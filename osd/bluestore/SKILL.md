---
name: bluestore-storage-engine
description: Ceph BlueStore原生存储引擎专家。当用户询问BlueStore磁盘布局、Onode结构、Blob/Extent映射、写入路径、WAL/DB角色、Allocator类型、压缩机制时调用此技能。
---

# BlueStore 存储引擎

## 〇、为什么需要 BlueStore？

为什么需要 BlueStore？在 BlueStore 之前，Ceph 使用 FileStore（基于 XFS/Btrfs）：

```
FileStore 的双重写入问题:
  写入数据 → 文件系统日志 (journal) → 文件系统数据区
  写入放大: 2x (先写 journal，再写实际位置)
  元数据开销: 文件系统自身的 inode、目录树、块位图
  性能瓶颈: 文件系统层引入的额外 I/O 和内存开销
```

BlueStore 的核心创新是：**绕过文件系统，直接在裸块设备上管理数据**。

```
BlueStore 方案:
  数据 → 直接写入裸块设备 (一次写入)
  元数据 → RocksDB KV 存储 (由 BlueFS 管理)
  写入放大: ~1x (无双重写入)
  性能提升: 2-3x (相比 FileStore)
```

BlueStore 于 2016 年 (Jewel) 引入，2017 年 (Luminous) 成为默认存储引擎。

---

## 一、磁盘布局

### 1.1 物理布局

```
裸块设备
  │
  ├── Superblock (多处存储)
  │     bluestore_bdev_label_t: osd_uuid, size, btime, description, meta
  │
  ├── BlueFS 区域 (RocksDB 的极简文件系统)
  │     ├── WAL 设备 (快速，可选，通常 NVMe)
  │     ├── DB 设备 (RocksDB SST 文件、WAL、manifest)
  │     └── 慢速设备 (回退，通常 HDD)
  │
  └── 数据区域 (BlueStore 分配器管理的原始块范围)
        ├── FreelistManager 追踪已分配/空闲范围
        └── Allocator 按需分配新范围
```

### 1.2 KV 存储前缀 (src/os/bluestore/BlueStore.cc:132-143)

| 前缀 | 键 | 值 | 说明 |
|------|-----|-----|------|
| `S` (SUPER) | 键名 | 元数据 | 全局元数据: nid_max, blobid_max, freelist_type, ondisk_format |
| `T` (STAT) | 计数器 | int64 数组 | 统计计数器 |
| `C` (COLL) | 集合名 | bluestore_cnode_t | 集合 (对应 PG) |
| `O` (OBJ) | 对象键 | bluestore_onode_t | Onode 元数据 |
| `M` (OMAP) | u64(head) + key | value | OMAP 数据 |
| `P` (PGMETA_OMAP) | 同上 | value | 元数据集合的 OMAP |
| `m` (PERPOOL_OMAP) | s64(pool) + ... | value | 每池 OMAP |
| `p` (PERPG_OMAP) | u64(pool) + ... | value | 每 PG OMAP |
| `L` (DEFERRED) | ID | deferred_transaction | WAL: 延迟事务 |
| `B` (ALLOC) | u64 offset | u64 length | 空闲列表 |
| `b` (ALLOC_BITMAP) | 位图 | - | 位图式空闲列表 |
| `X` (SHARED_BLOB) | u64 SB ID | shared_blob | 共享 Blob |

---

## 二、BlueStore vs FileStore

| 方面 | FileStore | BlueStore |
|------|-----------|-----------|
| **存储层** | POSIX 文件系统 (XFS/Btrfs) | 裸块设备 (直接 I/O) |
| **元数据** | 文件系统 inode + xattr | RocksDB KV 存储 |
| **数据放置** | 文件系统管理块 | BlueStore 直接分配 |
| **双重写入** | 是: journal → 文件系统 | 否: 数据直接写入块设备 |
| **WAL** | 文件系统日志 | `PREFIX_DEFERRED` ("L") + BlueFS WAL |
| **开销** | 文件系统元数据、日志、碎片 | 最小: 仅 RocksDB 元数据 + BlueStore 头 |
| **校验和** | 依赖文件系统 | 每 Blob 可配置 (CRC32C, xxhash 等) |
| **压缩** | 文件系统级 | 每 Blob，可配置算法和模式 |

---

## 三、关键数据结构

### 3.1 bluestore_pextent_t -- 物理范围 (src/os/bluestore/bluestore_types.h:98-112)

```cpp
struct bluestore_pextent_t {
    uint64_t offset = 0;   // 块设备上的物理偏移
    uint32_t length = 0;   // 长度 (字节)
};
```

简单的 `(offset, length)` 对，映射到裸块设备上的连续区域。`INVALID_OFFSET` 标记未分配区域 (稀疏 Blob 中的空洞)。

### 3.2 bluestore_blob_t -- Blob (src/os/bluestore/bluestore_types.h:491-1085)

Blob 是**磁盘上的一块数据**——存储的基本单位：

```cpp
struct bluestore_blob_t {
    PExtentVector extents;          // 设备上的原始数据位置
    uint32_t logical_length = 0;    // 原始 (未压缩) 长度
    uint32_t compressed_length = 0; // 压缩后长度
    uint32_t flags = 0;             // FLAG_COMPRESSED, FLAG_CSUM, FLAG_HAS_UNUSED, FLAG_SHARED
    unused_t unused = 0;            // 从未写入的部分位图 (稀疏追踪)
    uint8_t csum_type;              // CSUM_NONE, CSUM_CRC32C, CSUM_XXHASH32, ...
    uint8_t csum_chunk_order;       // 校验和块大小 = 1 << order
    buffer::ptr csum_data;          // 校验和值向量
};
```

**标志位**:
- `FLAG_COMPRESSED` (2): Blob 数据已压缩
- `FLAG_CSUM` (4): Blob 有校验和
- `FLAG_HAS_UNUSED` (8): Blob 有 "unused" 位图 (稀疏追踪)
- `FLAG_SHARED` (16): Blob 在对象间共享 (通过 SharedBlob)

### 3.3 bluestore_onode_t -- 持久化 Onode (src/os/bluestore/bluestore_types.h:1122-1295)

```cpp
struct bluestore_onode_t {
    uint64_t nid = 0;                    // 数字 ID (本地唯一)
    uint64_t size = 0;                   // 对象逻辑大小
    map<string, buffer::ptr> attrs;      // 扩展属性
    vector<shard_info> extent_map_shards; // 大对象的分片边界
    uint32_t expected_object_size = 0;   // 来自 alloc_hint
    uint32_t expected_write_size = 0;
    uint32_t alloc_hint_flags = 0;
    uint32_t segment_size = 0;           // 分片段大小 (v3 编码)
    uint8_t flags = 0;                   // FLAG_OMAP, FLAG_PGMETA_OMAP, ...
    map<uint32_t, uint64_t> zone_offset_refs; // Clone/zone 引用
};
```

### 3.4 BlueStore::Onode -- 内存 Onode (src/os/bluestore/BlueStore.h:1331-1467)

```cpp
struct Onode {
    Collection *c;
    ghobject_t oid;
    string key;                  // KV 存储键
    bluestore_onode_t onode;     // 持久化元数据
    bool exists;
    bool cached;
    ExtentMap extent_map;        // 逻辑偏移 → Blob 映射
    BufferSpace bc;              // 此 Onode 的缓冲区缓存
    atomic<int> flushing_count;  // 待刷新计数
};
```

### 3.5 BlueStore::Extent -- 逻辑范围 (src/os/bluestore/BlueStore.h:832-903)

```cpp
struct Extent {
    uint32_t logical_offset = 0;  // 对象内的逻辑偏移
    uint32_t blob_offset = 0;     // Blob 内的偏移
    uint32_t length = 0;          // 范围长度
    BlobRef blob;                 // 包含数据的 Blob
};
```

### 3.6 BlueStore::Blob (src/os/bluestore/BlueStore.h:650-826)

```cpp
struct Blob {
    int16_t id = -1;               // 跨范围 Blob 的 ID (>= 0)
    SharedBlobRef shared_blob;     // 共享状态 (用于共享 Blob)
    bluestore_blob_t blob;         // 解码的 Blob 元数据
    bluestore_blob_use_tracker_t used_in_blob;  // 引用追踪
};
```

### 3.7 BlueStore::SharedBlob (src/os/bluestore/BlueStore.h:546-598)

共享 Blob 允许多个对象引用相同的物理数据 (用于克隆)：

```cpp
struct SharedBlob {
    atomic_int nref = {0};
    bool loaded = false;
    CollectionRef collection;
    union {
        uint64_t sbid_unloaded;
        bluestore_shared_blob_t *persistent;
    };
};
```

### 3.8 BlueStore::ExtentMap (src/os/bluestore/BlueStore.h:933-1221)

```cpp
struct ExtentMap {
    Onode *onode;
    extent_map_t extent_map;       // Extent → Blob 映射 (按 logical_offset 排序)
    blob_map_t spanning_blob_map;  // 跨分片的 Blob
    vector<Shard> shards;          // 分片元数据
    buffer::list inline_bl;        // 缓存的编码映射 (小/未分片对象)
    uint32_t needs_reshard_begin/end;
};
```

---

## 四、Onode 结构：逻辑对象到物理范围的映射链

```
Object (ghobject_t)
  │
  └─→ Onode (内存缓存，OnodeSpace 中按 ghobject_t 键索引)
        │
        └─→ bluestore_onode_t (持久化在 RocksDB PREFIX_OBJ 下)
        │     包含: nid, size, attrs, extent_map_shards, flags
        │
        └─→ ExtentMap (内存，从 onode 值延迟加载)
              │
              └─→ vector<Shard> (大对象按逻辑偏移范围分片)
              │
              └─→ extent_map_t (boost::intrusive::set<Extent>，按 logical_offset 排序)
                    │
                    └─→ Extent { logical_offset, blob_offset, length, BlobRef }
                          │
                          └─→ Blob { bluestore_blob_t, SharedBlobRef }
                                │
                                └─→ PExtentVector { bluestore_pextent_t { offset, length } }
                                      │
                                      └─→ 裸块设备偏移/长度
```

**查找示例**: 读取逻辑偏移 `0x1000` 处的对象：
1. 在 `OnodeSpace` 中查找 Onode (或从 RocksDB `PREFIX_OBJ` 加载)
2. `ExtentMap::seek_lextent(0x1000)` 找到覆盖该偏移的 Extent
3. Extent 指向一个 Blob
4. Blob 的 `bluestore_blob_t` 有 `extents[]` — 磁盘上的物理 pextent
5. `blob_offset` + Blob 的范围布局 = 物理磁盘偏移
6. 直接从 `BlockDevice` 读取该偏移

---

## 五、写入路径

### 5.1 入口: `_do_write()` (src/os/bluestore/BlueStore.cc:17612)

```
_do_write(txc, c, o, offset, length, bl, fadvise_flags)
  │
  ├─ _choose_write_options()     // 确定压缩、Blob 大小、校验和类型
  ├─ extent_map.fault_range()    // 从 DB 加载相关范围分片
  ├─ _do_write_data()            // 构建 WriteContext 与 Blob 分配
  │     ├─ _do_write_small()     // 小写入: 尝试重用现有 Blob 空间
  │     └─ _do_write_big()       // 大写入: 分配新 Blob
  │
  ├─ _do_alloc_write()           // 分配磁盘空间，写入 bdev
  │     ├─ [压缩如果启用]         // 压缩数据，检查压缩率
  │     ├─ alloc->allocate()     // 从分配器获取物理范围
  │     ├─ bdev->aio_write()     // 向块设备提交异步 I/O
  │     └─ [计算校验和]           // 为新 Blob 初始化校验和
  │
  ├─ _wctx_finish()              // 在旧范围打孔，更新引用计数
  ├─ gc.estimate()               // 评估压缩 Blob 的垃圾回收收益
  ├─ _do_gc()                    // 如果有益则执行垃圾回收
  ├─ extent_map.compress_extent_map()  // 合并相邻范围
  └─ extent_map.dirty_range()    // 标记范围分片脏以更新 DB
```

### 5.2 事务状态机: `_txc_state_proc()` (src/os/bluestore/BlueStore.cc:14401)

```
STATE_PREPARE
  │ (提交待处理的 AIO)
  ▼
STATE_AIO_WAIT
  │ (等待 AIO 完成)
  ▼
STATE_IO_DONE
  │ (应用 KV 事务，同步或排队到 kv_sync_thread)
  ▼
STATE_KV_QUEUED → STATE_KV_SUBMITTED
  │ (KV 事务提交到 RocksDB)
  ▼
STATE_KV_DONE
  │ (如果有延迟写入，排队)
  ▼
STATE_DEFERRED_QUEUED → STATE_DEFERRED_CLEANUP
  │
  ▼
STATE_FINISHING
  │ (_txc_finish: 释放引用，通知完成)
  ▼
STATE_DONE
```

### 5.3 KV 同步线程: `_kv_sync_thread()` (src/os/bluestore/BlueStore.cc:15057)

专用的 KV 同步线程：
1. 批量处理多个 `TransContext` KV 事务
2. 调用 `bdev->flush()` 确保数据 AIO 已写入磁盘
3. 向 RocksDB 提交单个 `db->submit_transaction_sync(synct)`
4. 这是**持久性屏障** — 返回后，数据在磁盘上且元数据已提交

---

## 六、WAL 和 DB 的角色

### RocksDB (DB)

- **存储**: Onodes (`PREFIX_OBJ`)、Collections (`PREFIX_COLL`)、SharedBlobs (`PREFIX_SHARED_BLOB`)、OMAP 数据、超级元数据 (`PREFIX_SUPER`)、统计 (`PREFIX_STAT`)、空闲列表 (`PREFIX_ALLOC`)
- **角色**: 所有元数据的真相来源。每次写入都会更新 onodes 和范围映射。
- **同步点**: `db->submit_transaction_sync()` 是持久性屏障。

### WAL (延迟写入) — `PREFIX_DEFERRED` ("L")

- **存储**: `bluestore_deferred_transaction_t` 记录，包含待处理的写入
- **角色**: 当写入无法直接放置时 (例如会导致读-修改-写的小随机写入)，BlueStore 延迟处理。延迟写入在发出前先记录到 RocksDB 的 `PREFIX_DEFERRED` 下。
- **延迟写入** 由 `_deferred_submit_unlock()` 批量提交。
- **重放**: 启动时，`_deferred_replay()` (line 15636) 重放 RocksDB 中任何待处理的延迟事务。

### BlueFS

BlueFS 是支持 RocksDB 的**极简文件系统**。它管理：
- **WAL 设备**: RocksDB WAL 的快速设备 (可选，通常 NVMe)
- **DB 设备**: RocksDB SST 文件的设备 (通常 SSD)
- **慢速设备**: 回退设备 (通常 HDD)

BlueFS 仅提供足够的文件系统语义 (目录、文件、范围) 供 RocksDB 运行，无需完整 POSIX 文件系统的开销。

---

## 七、分配器类型

### 7.1 StupidAllocator

- **数据结构**: `std::vector<interval_set_t> free` — 按大小分组的空闲区间 (2 的幂分桶)
- **策略**: 从满足请求的最小桶中首次适配
- **优点**: 简单，内存开销低
- **缺点**: 可能随时间碎片化，O(log n) 分配
- **适用**: 小型 OSD，测试

### 7.2 BitmapAllocator

- **数据结构**: 两级位图层次结构 (`AllocatorLevel01Loose` + `AllocatorLevel02`)
- **策略**: 空闲/已分配块的位级追踪，分层汇总用于快速搜索
- **优点**: 快速分配/释放，适合小 `min_alloc_size` 的 SSD
- **缺点**: 大设备内存使用较高
- **适用**: 生产环境 NVMe/SSD OSD (现代 Ceph 默认)

### 7.3 HybridAllocator

- **组成**: 主分配器 (AvlAllocator 或 Btree2Allocator) + 辅助 BitmapAllocator
- **策略**: 主分配器处理大多数分配；无法满足时溢出到位图分配器
- **变体**:
  - `HybridAvlAllocator`: AVL 树 + 位图
  - `HybridBtree2Allocator`: B-tree v2 + 位图
- **适用**: 内存受限的大型 HDD OSD

---

## 八、压缩支持

### 配置 (每集合)

```cpp
optional<int> compression_algorithm;     // snappy, zlib, zstd, lz4
optional<int> compression_mode;          // none, passive, aggressive, force
optional<double> compression_req_ratio;  // 所需压缩率阈值
optional<int64_t> comp_min_blob_size;
optional<int64_t> comp_max_blob_size;
```

### 压缩流程 (`_do_alloc_write`, BlueStore.cc:17083)

1. 如果启用压缩且 Blob 足够大，尝试压缩
2. 检查压缩后大小 <= `original_size * compression_required_ratio`
3. 如果压缩成功且满足比率阈值：
   - 在 Blob 上设置 `FLAG_COMPRESSED`
   - 存储 `logical_length` (原始) 和 `compressed_length` (压缩后)
   - 前置 `bluestore_compression_header_t` (类型、长度、compressor_message)
   - 填充到 `min_alloc_size`
4. 如果压缩失败或比率不满足：回退到原始存储

### 压缩头 (src/os/bluestore/bluestore_types.h:1344-1365)

```cpp
struct bluestore_compression_header_t {
    uint8_t type = Compressor::COMP_ALG_NONE;  // 算法标识符
    uint32_t length = 0;                        // 压缩数据长度
    optional<int32_t> compressor_message;       // 算法特定元数据
};
```

### 垃圾回收 (src/os/bluestore/BlueStore.h:1253-1326)

对于压缩的 Blob，部分覆盖写入会留下"垃圾"——已分配但不再引用的空间。`GarbageCollector` 评估是否值得读取压缩 Blob 中的剩余有效数据，解压缩，并以未压缩 (或更好压缩) 的方式重写以回收空间。

---

## 九、关键代码位置

| 功能 | 文件 | 行号 |
|------|------|------|
| BlueStore 类定义 | src/os/bluestore/BlueStore.h | 253-4415 |
| Onode (内存) | src/os/bluestore/BlueStore.h | 1331-1467 |
| Collection | src/os/bluestore/BlueStore.h | 1655-1732 |
| Blob (内存) | src/os/bluestore/BlueStore.h | 650-826 |
| Extent (lextent) | src/os/bluestore/BlueStore.h | 832-903 |
| ExtentMap | src/os/bluestore/BlueStore.h | 933-1221 |
| SharedBlob | src/os/bluestore/BlueStore.h | 546-598 |
| TransContext 状态机 | src/os/bluestore/BlueStore.h | 2443-2455 |
| KVSyncThread | src/os/bluestore/BlueStore.h | 2302-2309 |
| GarbageCollector | src/os/bluestore/BlueStore.h | 1253-1326 |
| bluestore_pextent_t | src/os/bluestore/bluestore_types.h | 98-112 |
| bluestore_blob_t | src/os/bluestore/bluestore_types.h | 491-1085 |
| bluestore_onode_t | src/os/bluestore/bluestore_types.h | 1122-1295 |
| bluestore_shared_blob_t | src/os/bluestore/bluestore_types.h | 1092-1117 |
| KV 前缀 | src/os/bluestore/BlueStore.cc | 132-143 |
| _do_write (写入入口) | src/os/bluestore/BlueStore.cc | 17612-17705 |
| _do_alloc_write (分配+磁盘写入) | src/os/bluestore/BlueStore.cc | 17051-17610 |
| _txc_state_proc (状态机) | src/os/bluestore/BlueStore.cc | 14401-14517 |
| _kv_sync_thread (KV 同步) | src/os/bluestore/BlueStore.cc | 15057-15400 |
| Allocator 基类 | src/os/bluestore/Allocator.h | 25-97 |
| StupidAllocator | src/os/bluestore/StupidAllocator.h | 17-67 |
| BitmapAllocator | src/os/bluestore/BitmapAllocator.h | 16-60 |
| HybridAllocator | src/os/bluestore/HybridAllocator.h | 13-150 |
| BlueFS (RocksDB 极简 FS) | src/os/bluestore/BlueFS.h | 1-1260 |

---

## 十、常见问题与陷阱

### Q1: BlueStore 的"延迟写入" (Deferred Write) 是什么？

**A**: 当写入无法直接放置到裸块设备上时 (例如小随机写入已分配的 Blob 中间，需要读-修改-写)，BlueStore 将写入记录到 RocksDB 的 `PREFIX_DEFERRED` 下，稍后批量提交。这避免了小块分配的开销，但增加了元数据写入量。

### Q2: SharedBlob 的作用是什么？

**A**: SharedBlob 允许多个 Onode 引用相同的物理 Blob 数据。主要用于克隆操作——克隆对象共享底层数据，仅在写入时复制 (COW)。SharedBlob 通过引用计数管理物理范围的生命周期。

### Q3: ExtentMap 分片的意义？

**A**: 对于大对象 (如 RBD 镜像)，ExtentMap 可能非常大。分片将范围映射拆分为多个 RocksDB 键，允许按需加载相关部分，而不是每次 I/O 都加载整个映射。分片边界由 `extent_map_shards` 向量定义。
