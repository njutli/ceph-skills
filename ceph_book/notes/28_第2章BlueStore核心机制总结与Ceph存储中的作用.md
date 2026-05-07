# 第2章 BlueStore 核心机制总结：从用户操作看 Ceph 存储的完整运转

## 一、 BlueStore 在 Ceph 中的定位

FileStore 依赖本地文件系统（XFS/Ext4），写数据必须先写 Journal 再写数据文件——**双重写惩罚**严重拖垮性能。BlueStore 的核心设计哲学是绕过 VFS，直接管理裸块设备，用 kvDB（RocksDB）+ BlueFS 管理元数据，用 Allocator 直接分配物理空间，用 Cache 加速热数据访问。它在 Ceph 存储体系中是**承上启下的基石**：向上承接 PG/Collection/Object 的语义映射，向下直接操作裸盘 I/O。

---

## 二、 场景一：集群初始化与创建存储池

### 用户操作
```bash
ceph osd pool create mypool 128
ceph osd pool set mypool size 3
```

### BlueStore 内部发生了什么

1. **Monitor 分配 Pool ID**（如 pool_id=1），确定 PG 数量（128个），每个 PG 分配 PGID（如 `1.0`, `1.1`, ..., `1.127`）。
2. **CRUSH 计算 PG → OSD 映射**：每个 PGID 经 CRUSH 算法选出 3 个 OSD（如 PG `1.0` → OSD.0, OSD.3, OSD.7）。
3. **OSD 收到 OSDMap 更新**后，发现自己被分配了新的 PG。以 OSD.0 为例：
   - 调用 `BlueStore::mkfs`（如果首次部署已完成）。
   - 调用 `BlueStore::mount`：打开裸盘设备 → 加载 BlueFS → 挂载 RocksDB → 重建 Allocator 位图 → 重建内存 Cache。
4. **PG 进入 Creating 状态**，BlueStore 调用 `_create_collection`：
   - 在内存中创建 `Collection` 结构体（包含 `OpSequencer`、`onode_space`、`BufferCacheShard` 等）。
   - 在 kvDB 中写入 Collection 记录（Key: `collection_1.0`，Value: `bluestore_cnode_t`，包含 bits 等元信息）。此写入经由 **RocksDB 的 WAL** 持久化，RocksDB 调用 BlueFS 写入 `.log` 文件时又会追加一条 **BlueFS 日志**事务。
5. **PG Peering 完成后进入 Active**。此时如果客户端对 `1.0` 上的对象发 read，但 Collection 还不存在，BlueStore 直接返回 `-ENOENT`——这是一个快速失败的安全校验。

### 关键结构关系
```
Pool (pool_id=1, pg_num=128)
  └→ PG (pgid=1.0)  ──CRUSH──→  OSD.0, OSD.3, OSD.7
       └→ Collection (cnode.bits, onode_space, cache)   ← BlueStore 内存实体
            └→ kvDB Key: "collection_1.0" → Value: cnode  ← 持久化
```

---

## 三、 场景二：客户端首次写入新对象

### 用户操作
```bash
# 客户端通过 RBD 将一个虚拟块设备映射为 /dev/rbd0
rbd map mypool/myimage
# 向块设备写入一段数据
dd if=/dev/zero of=/dev/rbd0 bs=4K count=1 seek=0
```

RBD 将此写请求转化为对 RADOS 对象的写操作。例如对象名为 `rbd_data.1024`，写入 offset=0, length=4096（4KB，恰好 MAS 对齐）。

### BlueStore 写入流程（新写，4KB MAS 对齐，最简单的场景）

**第1步：PG 接收请求**
- `PrimaryLogPG` 收到写请求，通过 `queue_transactions` 提交至 BlueStore。
- 找到 PG 对应的 Collection，获取 `OpSequencer` 保序。

**第2步：查找/创建 Onode**
- 在 Collection 的 `onode_space` 中查找 `rbd_data.1024` 对应的 Onode。
- **这是新对象，Onode 不存在**，BlueStore 创建一个新的 Onode：
  - 分配内存 `Onode` 对象，初始化 `oid`、空的 `ExtentMap`、空的 `attrs`。
  - Onode 此时仅驻留内存，尚未持久化。

**第3步：分配物理空间（COW 策略——但这里是新写，没有旧数据）**
- 因为写入的 4KB 恰好 MAS 对齐，**新写**走简单的 COW 路径（无需补齐读、无需 WAL）：
  1. `Allocator` 从空闲列表中分配一块 4KB 的物理空间（如物理偏移 LBA=0x10000）。
  2. 创建新的 `Blob`：
     - `bluestore_blob_t` 的 `extents` 字段记录 `pextent{offset=0x10000, length=4096}`。
     - 如配置了校验（默认 CRC32），计算 4KB 数据的校验和，存入 `csum_data`。
  3. 创建新的 `Extent`：
     - `logical_offset=0`, `length=4096`, `blob` 指向上面的 Blob, `blob_offset=0`。
  4. 将 Extent 插入 Onode 的 `ExtentMap`。

**第4步：Cache 更新**
- `BufferSpace::write()`：创建新 Buffer，状态 `STATE_WRITING`，保存写入的 4KB 脏数据。
- `BufferSpace::discard()`：因为是新写，无重叠旧缓存需要丢弃。

**第5步：TransContext 三阶段执行**

> **重要概念辨析：BlueStore 体系中共有三层"日志"**
>
> | | **RocksDB 的 WAL** | **BlueStore 的 WAL 写**（2.6.4 节） | **BlueFS 的日志** |
> |---|---|---|---|
> | **是什么** | RocksDB 自身的预写日志，保证 kvDB 事务的 ACID | 仅针对非 MAS 对齐覆盖写的保护日志条目 | BlueFS 自身元数据的唯一持久化形式 |
> | **存哪里** | BlueFS 管理的 `.log` 文件（优先存 WAL 设备） | **存入 kvDB 本身**，作为一条特殊的 KV 记录 | BlueFS 自身的 log 文件（优先存 WAL 设备） |
> | **何时产生** | 每次 kvDB 提交事务时都会写 | 仅当覆盖写且必须走 RMW 路径时才有 | 每次通过 BlueFS 创建/删除/修改文件时产生 |
> | **保护什么** | 保护 kvDB 自身崩溃可恢复 | 保护覆盖写的原数据不被写坏 | BlueFS 的目录和文件元数据（dir_map / file_map） |
> | **清理方式** | 内容被 flush 到 SSTable 后截断 | 覆盖写完成后从 kvDB 删除该 KV 条目 | 日志压缩（Compaction）：生成全量快照替代历史日志 |
>
> 三者的依赖关系：**BlueStore WAL 写**的内容最终存入 kvDB → 写入 kvDB 需要经过 **RocksDB 的 WAL** 保证持久性 → RocksDB 的 WAL 文件由 **BlueFS** 管理，BlueFS 自身的元数据变更又追加到自己的日志中。
>
> 新写场景**有 RocksDB 的 WAL**（保证元数据原子提交）和 **BlueFS 的日志**（记录 .log/.sst 文件的分配变更），**没有 BlueStore 的 WAL 写**（因为没有覆盖旧数据）。

- **阶段1：等待用户数据 I/O 完成**
  - 同步线程将 4KB 数据直接写入裸盘偏移 0x10000（绕过 OS 页缓存）。
  - 此时掉电安全：若下电，kvDB 中 Onode 尚未更新，新写入的数据无指针引用，等价于从未写入。
  - 用户数据 I/O 完成后，回调重新将 TransContext 加入同步线程继续处理。
- **阶段2：同步元数据至 kvDB**
  - 将以下元数据通过单个事务原子写入 RocksDB：
    - Onode（含 ExtentMap 分片信息）→ KV Key: `O_1_rbd_data.1024`，Value: 序列化后的 `bluestore_onode_t`。
    - Allocator 的 FreelistManager 更新（标记物理偏移 0x10000~0x10FFF 已分配）。
  - RocksDB 内部保证 ACID：先将事务写入 **RocksDB 的 WAL**（`.log` 文件，由 BlueFS 管理，优先存储在 WAL 设备上），再写 MemTable。**RocksDB 的 WAL** 确保即使掉电，下次 mount 时 RocksDB 可以重放恢复未完成的 SSTable 写入。
  - 写入 **RocksDB 的 WAL** 的过程中，RocksDB 调用 BlueFS 的写接口将 `.log` 文件内容写入裸盘。BlueFS 追加一条 **BlueFS 日志**事务到自身的 log 文件，记录 `.log` 文件的 extents 变化（即"RocksDB 的 .log 文件扩展到了哪些物理块"）。
  - **kvDB 事务提交成功 → 触发 `on_commit` 回调**，通知上层 PG："元数据已经安全落盘，数据不会丢了"。
  - **注意**：此时用户数据已经在阶段1写到了裸盘，元数据在阶段2已原子提交到 kvDB（经由 **RocksDB 的 WAL** + **BlueFS 日志** 持久化）。两者都安全了，客户端可以安全地收到写完成应答。
- **阶段3：无 BlueStore WAL 写需要执行**
  - 本次是纯新写 + MAS 对齐，写到的全是新分配的物理空间，不涉及对旧数据的原地覆盖写，因此**不存在"写坏旧数据"的风险**，不需要 **BlueStore 的 WAL 写** 保护。
  - 没有旧空间需要释放（新写的旧空间=0），所以阶段2完成后此事务组即告结束。

**第6步：Cache 状态转换**
- `BufferSpace::finish_write()`：
  - 状态 `STATE_WRITING` → `STATE_CLEAN`（如 Buffer 未设置 `FLAG_NOCACHE`）。
  - Buffer 加入 Cache 的 warm_in 队列（2Q 入口）。

### 最终磁盘状态
```
裸盘布局:
┌──────────────────────┐
│ ... BlueStore Super   │  ← Slow 设备
│ LBA 0x10000: [4KB 用户数据] │  ← Slow 设备, 用户数据区
│ ...                   │
├──────────────────────┤
│ ... BlueFS Log        │  ← WAL 设备（BlueFS 日志，记录所有文件/目录变更）
│ ... RocksDB .log     │  ← WAL 设备（RocksDB 的 WAL 文件，由 BlueFS 管理）
│ ... RocksDB .sst      │  ← DB 设备（RocksDB 的 SSTable 文件，由 BlueFS 管理）
└──────────────────────┘

kvDB 中的 Key-Value:
  "O_1_rbd_data.1024" → bluestore_onode_t {
      size: 4096,
      attrs: {},
      extent_map_shards: [
        shard_info { offset: 0, bytes: XXX }
      ]
  }
  extent_map shard → [
      Extent { logical_offset: 0, length: 4096, blob_offset: 0,
               blob: bluestore_blob_t {
                   extents: [pextent{offset: 0x10000, length: 4096}],
                   flags: FLAG_CSUM | FLAG_MUTABLE,
                   csum_type: CRC32,
                   csum_data: [4 bytes],
               }
      }
  ]
  FreelistManager → 标记 0x10000~0x10FFF 已占用
```

---

## 四、 场景三：追加写（Append）—— 对象增长

### 用户操作
```bash
# 继续向同一对象追加写入 12KB 数据（offset=4096, length=12288）
dd if=/dev/zero of=/dev/rbd0 bs=12K count=1 seek=1
```

写入区间：[4096, 16384)，对象已有 [0, 4096)。

### 地址切分
```
已有数据: [0, 4096)   → 1 个 Extent/Blob 占据 LBA 0x10000
追加区间: [4096, 16384)
  头非对齐: 无（4096 是 MAS 对齐的）
  中MAS对齐: [4096, 8192)  → 4096B  → 新写，COW
              [8192, 12288) → 4096B  → 新写，COW
              [12288, 16384)→ 4096B  → 新写，COW
  尾非对齐: 无
```

因为这3段都是 MAS 对齐的新写（已有区间 [0, 4096) 之外无需覆盖），所以处理方式与场景二完全类似：

1. **Allocator 分配** 3 块 4KB 物理空间（如 LBA 0x11000, 0x12000, 0x13000）。
2. **创建 3 个新 Blob + 3 个新 Extent**，分别映射逻辑偏移 4096/8192/12288 到物理偏移 0x11000/0x12000/0x13000。
3. **但可选优化**：如果这 3 段数据与小 Blob 阈值（`min_alloc_size`）匹配，BlueStore 可能将它们合并到同一个 Blob（跨多个 pextent），减少元数据量。假设这里合并为一个 Blob：
   ```
   Blob {
     extents: [
       pextent{offset: 0x11000, length: 4096},
       pextent{offset: 0x12000, length: 4096},
       pextent{offset: 0x13000, length: 4096},
     ],
     flags: FLAG_CSUM,
     csum_data: [3 × 4 bytes CRC32]
   }
   Extent { logical_offset: 4096,  length: 12288, blob_offset: 0 }
   ```
   或分 3 个 Extent 各指向同一 Blob（带不同 blob_offset）。

4. **Onode 的 ExtentMap 更新**：追加 1~3 个新 Extent 条目，Onode.size 更新为 16384。
5. **Cache 更新**：
   - `discard()` 丢弃 [4096, 16384) 范围的旧缓存（本场景无旧缓存）。
   - `write()` 创建新的 `STATE_WRITING` Buffer。
6. **提交 kvDB**：Onode（含新的 ExtentMap 分片）+ FreelistManager 一起原子写入。写入过程经由 **RocksDB 的 WAL**（保证 ACID）和 **BlueFS 日志**（保证 .log/.sst 文件的元数据可恢复）持久化。

### 磁盘上 Onode 的 ExtentMap 变化
```
追加前:
  Extent { logical_offset: 0,     length: 4096,  blob_offset: 0 } → Blob_A @ LBA 0x10000

追加后:
  Extent { logical_offset: 0,     length: 4096,  blob_offset: 0  } → Blob_A @ LBA 0x10000
  Extent { logical_offset: 4096,  length: 12288, blob_offset: 0  } → Blob_B @ LBA 0x11000~0x13FFF
```

**关键点**：追加写总是新数据写入新空间，走 COW 路径，效率最高。这也是 Ceph RBD 默认以 4MB 条带化写入的原因——每次写入新的对象，不存在覆盖写问题。

---

## 五、 场景四：覆盖写（Overwrite）—— 最复杂的场景

### 用户操作
```bash
# 修改对象 rbd_data.1024 中间的数据：从 offset 5000 写入 13KB
dd if=/dev/zero of=/dev/rbd0 bs=13K count=1 seek=5000
# 实际发送: write(object="rbd_data.1024", offset=5000, length=13312)
```

当前对象 ExtentMap 状态（来自场景三）：
```
Extent { logical_offset: 0,     length: 4096,  blob_offset: 0 } → Blob_A
Extent { logical_offset: 4096,  length: 12288, blob_offset: 0 } → Blob_B
对象大小: 16384 bytes
```

### 地址切分（MAS = 4KB）

```
写入区间: [5000, 18312)

MAS边界: 0     4096   8192   12288  16384   ...
               |      |      |      |
  头非对齐: [5000, 8192)     → 3192B  ← 落入 Blob_A 尾 + Blob_B 头
  中MAS对齐: [8192, 12288)   → 4096B  ← 覆盖 Blob_B 中段
  中MAS对齐: [12288, 16384)  → 4096B  ← 覆盖 Blob_B 末段
  尾非对齐: [16384, 18312)   → 1928B  ← 超出原对象尾部，新写
```

### 逐段处理

#### 头非对齐 [5000, 8192) — 需要 RMW + WAL

**问题**：这段数据跨两个 Blob：
- [5000, 4096+4096=8192) 中 [5000, 8192) 这 3192B 不够一个 4KB 块。

**三分支判断**：
1. **能否直接 COW？** → 来看 Blob_A/B 是否被压缩过。假设这里未压缩，且 `FLAG_MUTABLE` 已设置。
2. **能否复用 unused 块？** → MAS = 块大小 = 4KB，Blob 内无 unused 块。不行。
3. **WAL 写**：必须走这条路径。

**WAL 写流程**：
```
1. 补齐读：从磁盘 LBA 0x10000 读取 Blob_A 的 [4096, 8192) 区域（整块）。
2. 合并：将新写入的 3192B 合并到读出的块中 [5000-4096, 8192-4096) 这段位置。
3. 填充：块头部 [0, 5000-4096=904) 不属于本次写入，保留旧数据（或用 0 填充）。
   最终得到一个完整的 4KB 块，包含了旧数据+新数据的合并结果。
4. 生成 BlueStore WAL 条目：将合并后的 4KB 块数据编码为一条 KV 记录，加入 TransContext。
   这条 KV 记录将在阶段 2 随其他元数据一起原子写入 kvDB。
5. 此 WAL 条目的作用：如果阶段 3 的原地覆盖写因掉电而中断，下次 mount 时
   BlueStore 可以从 kvDB 中读出这条 WAL 记录，重放覆盖写操作，保证数据完整性。
6. 完整持久化路径：BlueStore WAL 条目 → 写入 RocksDB（经由 RocksDB 的 WAL 持久化）
   → RocksDB 调用 BlueFS 写 .log 文件 → BlueFS 追加 BlueFS 日志事务到自身 log。
   三层日志依次保证：BlueFS 日志保证 BlueFS 元数据可恢复，
   RocksDB 的 WAL 保证 kvDB 事务可重放，BlueStore WAL 条目保证覆盖写可重放。
```

#### 中MAS对齐 [8192, 12288) 和 [12288, 16384) — COW

这两段是对齐的覆盖写，直接走 COW：
1. 从 Allocator 分配 2 块新 4KB 空间（如 LBA 0x14000, 0x15000）。
2. 创建新 Blob_C 和 Blob_D（或合并为一个大 Blob，含 2 个 pextent）。
3. 旧 Blob_B 中 [8192, 16384) 对应的物理空间标记为待释放。
4. 新 Extent 指向新 Blob。

#### 尾非对齐 [16384, 18312) — 新写（超过原对象尾部）

这段超出了原对象大小，是纯粹的**新写**：
1. 从 Allocator 分配 1 块 4KB 空间（如 LBA 0x16000）。
2. 不足 4KB 的部分（16384~18312 = 1928B）用 0 填充至 4KB。
3. 创建新 Blob_E，新 Extent 的 `blob_offset = 0`。

### TransContext 三阶段执行

**阶段1：等待 COW I/O 完成**
- 中对齐部分（新 Blob_C、Blob_D）和尾部新写（Blob_E）的数据同步写入裸盘。
- 掉电安全：若此时掉电，kvDB 中 Onode 的 ExtentMap 还是指向旧 Blob，新 Blob 的空间未在 Allocator 中标记已分配，无影响。

**阶段2：同步元数据至 kvDB**
- 将所有元数据变更和 WAL 条目通过单个事务原子写入 RocksDB：
  - **Onode 更新**：大小变为 18312，ExtentMap 更新（旧 Extent 被替换为新 Extent，Blob_B 中间的部分逻辑区间被 COW 分割）。
  - **FreelistManager 更新**：新分配空间标记已占用；旧空间（[8192, 16384) 对应的 Blob_B 物理块，以及 Blob_A 中 [4096, 8192) 的物理块——等 WAL 写完后才真正释放）标记为"待释放"。
  - **BlueStore WAL 条目**：包含头非对齐部分合并后的 4KB 数据，作为 kvDB 的一条特殊 KV 记录存入。
- RocksDB 先将整个事务写入 **RocksDB 的 WAL**（`.log` 文件，由 BlueFS 管理，优先存储在 WAL 设备上），保证 ACID，然后提交到 MemTable。在写入 **RocksDB 的 WAL** 的过程中，BlueFS 会追加 **BlueFS 日志**事务，记录 `.log` 文件和后续可能产生的 `.sst` 文件的物理块分配变更。
- **kvDB 事务提交成功 → 触发 `on_commit` 回调**。
- **注意**：此时 COW 部分的数据（阶段 1）已经安全写在裸盘新地址，元数据（阶段 2）也已通过 **RocksDB 的 WAL** + **BlueFS 日志** 原子提交到 kvDB。而头非对齐部分的原地覆盖写尚未执行，但其安全由 kvDB 中的 **BlueStore WAL 条目** 保证——即使掉电，也能通过 WAL 重放恢复。

**阶段3：执行 BlueStore WAL 覆盖写**
- 阶段 2 完成，**BlueStore WAL 条目**已安全存在于 kvDB 中。现在可以安全地执行原地覆盖写：
  - 将 WAL 记录的 4KB 合并块写回裸盘 LBA 0x10000 区域的 [4096, 8192) 位置。
- 覆盖写 I/O 完成 → 同步线程生成新的 kvDB 事务：删除 **BlueStore WAL 条目** + 将旧物理块加入 Allocator 释放。此事务同样经由 **RocksDB 的 WAL** + **BlueFS 日志** 持久化。
- 触发 `on_readable` 回调。
- **注意**：如果阶段 3 执行中途掉电，下次 mount 时 BlueStore 会读出 kvDB 中残留的 **BlueStore WAL 条目**，重新执行覆盖写（即 WAL 重放），保证数据最终完整。

### 覆盖写后的 ExtentMap
```
覆盖写前:
  Extent { logical_offset: 0,     length: 4096,  blob_offset: 0 } → Blob_A @ LBA 0x10000
  Extent { logical_offset: 4096,  length: 12288, blob_offset: 0 } → Blob_B @ LBA 0x11000~0x13FFF

覆盖写后:
  Extent { logical_offset: 0,     length: 5000,  blob_offset: 0    } → Blob_A' (截断后剩余部分)
  Extent { logical_offset: 5000,  length: 3192,  blob_offset: ???  } → 待WAL覆盖写回原位
  Extent { logical_offset: 8192,  length: 4096,  blob_offset: 0    } → Blob_C @ LBA 0x14000  (COW新写)
  Extent { logical_offset: 12288, length: 4096,  blob_offset: 0    } → Blob_D @ LBA 0x15000  (COW新写)
  Extent { logical_offset: 16384, length: 1928,  blob_offset: 0    } → Blob_E @ LBA 0x16000  (新写+0填充)

注: Blob_A 的 [4096, 8192) 部分被WAL写原地覆盖; Blob_B 的 [8192, 16384) 被COW替换;
    Blob_A 的 [0, 4096) 保持不变; 旧Blob_B的物理空间待释放。
```

### Cache 变化
```
1. BufferSpace::discard(5000, 13312):
   - 对象原有 [0, 16384) 对应的 Buffer 被裁剪，[5000, 16384) 重叠部分被丢弃
   - 保留 [0, 5000) 部分在 Cache 中（状态仍为 STATE_CLEAN）

2. BufferSpace::write(5000, 13312, data):
   - 创建新 Buffer，状态 STATE_WRITING，保存新写入的 13312 字节脏数据

3. finish_write() 后:
   - Buffer 状态转为 STATE_CLEAN，进入 Cache 的 warm_in 队列
```

---

## 六、 场景五：读取对象数据

### 用户操作
```bash
# 读取对象 rbd_data.1024 的 [0, 1000) 区域
rbd read mypool/myimage --offset 0 --length 1000
```

### BlueStore 读取流程

1. **PG 路由**：客户端根据 CRUSH 计算 PG → OSD → 发送读请求到目标 OSD。
2. **Collection 查找**：
   - BlueStore 在内存中查找 PG 对应的 Collection 指针。
   - 如果找不到 → 返回 `-ENOENT`（说明 PG 尚未在本 OSD 上 active）。
3. **Onode 查找**：
   - 在 Collection 的 `onode_space.onode_map` 中查找 `rbd_data.1024` 的 Onode。
   - 如果未命中（Cache miss）→ 从 kvDB 读取，反序列化为 Onode 对象，加入 onode_space。
4. **ExtentMap 加载**：
   - 如果 Onode 的 ExtentMap 尚未完全加载（分片按需加载机制），则根据 [0, 1000) 所在的分片范围，从 kvDB 读取对应 shard_info，解码出相关 Extent。
5. **定位数据**：
   - 逻辑偏移 [0, 1000) 命中 Extent `{logical_offset: 0, length: 4096, blob_offset: 0} → Blob_A`。
   - 计算 Blob_A 的物理偏移：0x10000 + (0 - 0) = 0x10000。
6. **BufferSpace 查询**：
   - 先查 Cache：BufferSpace::read(0, 1000)。
   - 命中 → 直接从 Buffer 返回数据，无需读盘。
   - 未命中 → 直接到裸盘 LBA 0x10000 读取。
7. **校验和验证**（如果 Blob 启用了 `FLAG_CSUM`）：
   - 从 Blob 的 `csum_data` 中取出对应块的校验和。
   - 对读回的数据重新计算 CRC32，与存储值比较。不匹配 → 数据损坏，向上报错。
8. **返回数据**：
   - 若跳过了读盘：直接返回 Cache 中的数据。
   - 若执行了读盘：`did_read()` 创建新 Buffer（STATE_CLEAN），加入 BufferSpace → 加入 Cache。

### 如果对象不存在
```
客户端发送 read → OSD 查找 Onode → kvDB 中不存在
→ BlueStore 返回 -ENOENT → PG 通知客户端 "对象不存在"
```

---

## 七、 场景六：压缩场景下的覆盖写

### 用户操作（假设已开启压缩）
```bash
ceph osd pool set mypool compression_algorithm snappy
ceph osd pool set mypool compression_mode aggressive
# 写入一段可压缩数据后，BlueStore 发现压缩比 > 阈值，对 Blob 执行压缩
# 然后客户端覆盖写该 Blob 的中间部分
```

### 压缩后的数据状态

假设 Blob_B 原有 12KB 数据，经 Snappy 压缩后变为 6KB：
```
Blob_B (compressed) {
  flags: FLAG_COMPRESSED,
  compressed_length_orig: 12288,     ← 原始长度
  compressed_length: 6144,            ← 压缩后长度
  extents: [pextent{offset: 0x20000, length: 6144}],  ← 物理空间仅6KB
  csum_data: [...]                    ← 校验和
}
```

### 覆盖写时发生了什么

当客户端写 [5000, 8192) 这段，正好跨过压缩后的 Blob：

1. **能否用 COW？** → **直接走 COW**。因为 Blob 已被压缩，若要走 RMW 必须先把整块数据读出来→解压→修改→重压，代价太大。
2. **COW 处理**：
   - 分配新的 4KB 物理空间。
   - 新写入的 3192B 数据（+ 0 填充至 4KB）直接写入新 Blob，**不压缩**。
   - 旧 Blob_B（压缩的）保持不变，但其引用计数减少。
   - ExtentMap 更新：插入新 Extent 指向新 Blob，旧 Extent 被替换或截断。
3. **读惩罚**：后续读 [0, 5000) 仍需从压缩 Blob_B 读取并解压，但这是 COW 的固有代价——读时需要从两个 Blob 拼凑完整数据。

这就是书中所说的"引入数据压缩的一个重大缺陷在于其对不完全覆盖写的负面影响"。

---

## 八、 场景七：客户端删除对象

### 用户操作
```bash
rados rm mypool rbd_data.1024
```

### BlueStore 删除流程

1. **查找 Onode**：从 Collection 的 `onode_space` 中定位 `rbd_data.1024` 的 Onode。
2. **遍历 ExtentMap**：
   - 对每个 Extent → 获取其 Blob → 遍历 Blob 的 `pextent` 列表。
   - 将所有物理空间加入 **待释放列表**：
     - Blob_A: LBA 0x10000, length=4096
     - Blob_B: LBA 0x20000, length=6144 (压缩后)
     - Blob_C: LBA 0x14000, length=4096
     - ...
3. **生成 TransContext**：
   - 删除 Onode 的 kvDB 条目（Key: `O_1_rbd_data.1024`）。
   - 删除 ExtentMap 的所有分片 kvDB 条目。
   - FreelistManager 标记上述物理空间为空闲。
4. **提交 kvDB**：原子生效，对象元数据彻底消失，物理空间回收至 Allocator。此事务同样经由 **RocksDB 的 WAL** + **BlueFS 日志** 持久化。
5. **Cache 清理**：
   - Onode 从 `onode_space` 中移除，从 LRU 队列淘汰。
   - BufferSpace 中所有 Buffer 被移除。

---

## 九、 串联总结：BlueStore 如何支撑 Ceph 存储

| 用户操作 |BlueStore 数据面动作 |BlueStore 元数据面动作 |kvDB 变化 |BlueStore WAL |RocksDB WAL |BlueFS 日志 |
|----------|---------------------|----------------------|----------|-------------|------------|-----------|
| 创建 Pool | — | 创建 Collection | 写入 Collection KV | 无 | ✅ 保证 Collection KV 事务 ACID | ✅ 记录 .log 文件 extents 变更 |
| 首次写入新对象 | 分配物理块 → 写裸盘 | 创建 Onode+Extent+Blob | 写入 Onode、ExtentMap 分片、FreelistManager | 无 | ✅ 保证上述 KV 事务 ACID | ✅ 记录 .log/.sst 文件 extents 变更 |
| 追加写 | 分配新空间 → 写裸盘 | 追加 Extent，可能合并 Blob | 更新 Onode（size↑）、新增 ExtentMap 分片、FreelistManager | 无 | ✅ | ✅ |
| 覆盖写（对齐） | COW 分配新块 → 写新位置 | 旧 Extent/Blob 标记释放，新建 Extent/Blob | 更新 Onode、旧块标记释放 | 无 | ✅ | ✅ |
| 覆盖写（非对齐） | 补齐读→合并→COW写新块 + WAL保护原地覆盖写 | WAL 条目写入kvDB，旧空间暂不释放 | 阶段2：写Onode+ExtentMap+FreelistManager+**BlueStore WAL条目**；阶段3完成后：删除WAL条目、释放旧空间 | ✅ 作为 KV 记录存入 kvDB，保护原地覆盖写 | ✅ 保证含 WAL 条目的事务 ACID | ✅ |
| 读取 | 直读裸盘/命中 Cache | 查 Onode→ExtentMap→Blob→物理地址 | 可能读 Onode/ExtentMap shard (Cache miss 时) | 无 | 可能读 SSTable | 可能读 |
| 删除 | 释放物理空间 | 删除 Onode+Extent+Blob | 删除 Onode KV、ExtentMap 分片 KV，FreelistManager 更新 | 无 | ✅ | ✅ |
| 压缩后覆盖写 | COW 到新块（不解压旧块） | ExtentMap 更新，旧 Blob ref↓ | 更新 Onode/ExtentMap | 无 | ✅ | ✅ |
| 掉电恢复 | — | mount 时检测 kvDB 中残留的 BlueStore WAL 条目 | 读取 WAL 条目，重放未决的覆盖写操作 | 重放未完成的 WAL 覆盖写 | 重放未完成的 kvDB 事务，恢复 ACID | 重放完整日志链或加载最新快照，恢复 dir_map/file_map |

> **关于三层"日志"的完整澄清**：
>
> 所有写操作（包括新写）都依赖 **RocksDB 的 WAL**（存储在 BlueFS 管理的 `.log` 文件中）来保证 kvDB 事务的 ACID。RocksDB 每次写 `.log` 文件时，BlueFS 会追加 **BlueFS 日志**事务来记录 `.log` 文件的物理块变更，保证 BlueFS 自身元数据可恢复。
>
> **BlueStore 的 WAL 写**（2.6.4 节所述）是 kvDB 中的一条特殊 KV 记录，仅在非 MAS 对齐覆盖写时才产生，用于保护原地覆盖写操作的崩溃安全性。
>
> **BlueFS 的日志**是 BlueFS 自身元数据的唯一持久化形式（日志结构文件系统的核心），每次 RocksDB 写 `.log` 或 `.sst` 文件都会触发 **BlueFS 日志**追加。BlueFS 定期执行日志压缩，将全量状态快照替代历史日志。
>
> 三者层次不同、目的不同、存储位置也不同，但存在逐级依赖：**BlueStore WAL 写** → 存入 kvDB → 写入需经过 **RocksDB 的 WAL** → 写 `.log` 文件需经过 **BlueFS 日志**。

**核心设计范式**：
- **所有元数据变更是原子的**（kvDB 的 ACID 保证），所以 COW 新写在阶段 2 之前掉电是安全的。
- **所有非对齐覆盖写必须先写 WAL 再写数据**，所以阶段 3 掉电可以通过 WAL 重放恢复。
- **物理空间分配/释放与元数据绑定**：Allocator 的位图在 kvDB 中持久化（FreelistManager），确保 mount 可重建。
- **Cache 两级加速**：Onode（LRU）加速元数据查找，Buffer（2Q）加速用户数据读取。两者共享全局缓存空间，默认 90% 给元数据。