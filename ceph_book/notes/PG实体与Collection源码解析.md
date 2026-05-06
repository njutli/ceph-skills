# PG 的实体与 Collection 结构体解析

## 问题：PG 的实体是什么？Collection 怎么定义？PG 信息会落盘吗？

### 解答

在 Ceph 的两级映射架构中，PG（Placement Group）是一个核心概念。虽然它被称为“逻辑载体”，但在 Ceph 的源码实现中，PG 拥有实实在在的“实体”，横跨控制面、数据面和持久化层。

---

## 1. PG 的三重实体

PG 在 Ceph 系统中表现为三个层面的实体：

| 层面 | PG 的实体形态 | 核心职责 | 源码位置 |
|:---|:---|:---|:---|
| **控制面** (OSD 进程内存) | `PrimaryLogPG` 对象 | 状态机管理（Peering/Active）、数据恢复、主从同步、Scrub 校验。这是 PG 的“大脑”。 | `src/osd/PrimaryLogPG.h` |
| **数据面** (BlueStore 内存) | `Collection` 结构体 | 管理该 PG 下所有对象的缓存（Onode/Buffer）、事务顺序执行。这是 PG 的“躯干”。 | `src/os/bluestore/BlueStore.h` |
| **持久化** (磁盘) | 特殊 Meta Object (`pgmeta_oid`) | 存储 `pg_info_t` 和 `pg_log`，确保崩溃后能重建状态。这是 PG 的“记忆”。 | `src/osd/PG.cc` |

---

## 2. Collection 结构体定义

在 BlueStore 中，每个 PG 对应一个 `Collection` 实例。它是 PG 在存储引擎层面的内存管理结构。

**源码位置**：`src/os/bluestore/BlueStore.h:1655`

```cpp
struct Collection : public CollectionImpl {
  BlueStore *store;
  OpSequencerRef osr;                  // 操作序列化器，保证该 PG 内操作有序执行
  BufferCacheShard *cache;             // 数据缓存分片，管理该 PG 的用户数据缓存
  bluestore_cnode_t cnode;             // 落盘信息，记录 PG 的 bits（用于判断对象是否属于该 PG）
  ceph::shared_mutex lock;             // 集合锁，保护 Collection 内部状态
  bool exists;                         // 标记 Collection 是否存在

  SharedBlobSet shared_blob_set;       // 共享 Blob 集合，用于实现跨对象的数据共享（如 EC 场景）

  // 核心：管理该 PG 下所有 Onode 的缓存
  // 使用 OnodeSpace 避免全局锁竞争，实现按 PG 隔离的元数据缓存
  OnodeSpace onode_space;             

  // Pool 选项，如压缩策略、校验算法等，允许不同 Pool 的 PG 有不同的行为
  pool_opts_t pool_opts;
  std::optional<int> compression_algorithm;
  // ... 其他配置

  ContextQueue *commit_queue;          // 提交队列
  std::unique_ptr<Estimator> estimator;// 空间估算器

  // ... 方法
  OnodeRef get_onode(const ghobject_t& oid, bool create);
  void split_cache(Collection *dest);  // PG 分裂时拆分缓存
};
```

**关键成员说明**：
*   **`onode_space`**: 包含一个 `OnodeCacheShard *cache` 指针和一个 `onode_map`。它建立了对象 ID 到内存 `Onode` 对象的映射。
*   **`cnode`**: 对应 `bluestore_cnode_t`，包含 `bits` 成员，用于在 `contains()` 方法中通过掩码运算判断对象是否属于该 PG。

---

## 3. PG 信息落盘机制

PG 的元数据（状态、版本、日志等）必须持久化，否则 OSD 重启后无法恢复数据一致性。

**落盘形式**：
在 BlueStore 中，PG 的元数据被存储为该 Collection 内部的一个**特殊对象（Meta Object）**，其 ID 由 `pgid.make_pgmeta_oid()` 生成。

**源码流程**：
1.  **标记脏数据**：当 PG 状态发生变化（如版本更新、Peering 状态改变）时，`PeeringState` 会设置 `dirty_info = true`。
2.  **准备写入**：在事务提交前，调用 `PeeringState::write_if_dirty`，进而调用 `PG::prepare_write`。
3.  **序列化与写入**：
    *   `prepare_info_keymap`: 将 `pg_info_t`、`past_intervals` 等序列化为 key-value 对。
    *   `write_log_and_missing`: 将 PG Log 和 Missing 列表序列化。
    *   `t.omap_setkeys`: 通过 BlueStore 的 omap 接口，将这些元数据写入磁盘上的 `pgmeta_oid` 对象。

**源码位置**：`src/osd/PG.cc:908`
```cpp
void PG::prepare_write(
  pg_info_t &info,
  pg_info_t &last_written_info,
  PastIntervals &past_intervals,
  PGLog &pglog,
  bool dirty_info,
  bool dirty_big_info,
  bool need_write_epoch,
  ObjectStore::Transaction &t) 
{
  // ...
  if (dirty_big_info || dirty_info) {
    int ret = prepare_info_keymap(
      cct, &km, &key_to_remove,
      get_osdmap_epoch(), info, last_written_info, past_intervals,
      dirty_big_info, need_write_epoch, ...);
  }
  // 写入 Log 和 Missing 列表
  pglog.write_log_and_missing(t, &km, coll, pgmeta_oid, pool.info.require_rollback());
  // 将元数据 Key-Value 写入磁盘
  if (!km.empty())
    t.omap_setkeys(coll, pgmeta_oid, km);
}
```

---

## 4. 概念辨析：Object、OSD 与 PG

你的理解非常准确，这里做一个形象的总结：

*   **Object (货物)**：具体的数据实体，落盘为 Blob/Extent，内存中有 Onode/Buffer 缓存。
*   **OSD (仓库管理员)**：守护进程，负责网络交互、协议处理、调度 PG 工作。
*   **PG (货架)**：
    *   **逻辑容器**：将成千上万个 Object 打包，降低管理粒度。
    *   **工作单元**：迁移、恢复、Scrub 都是以 PG 为单位进行。
    *   **解耦层**：如果没有 PG，Object 直接映射到 OSD，扩缩容或故障时会导致元数据风暴。通过 PG 这一层，Ceph 实现了平滑的数据重平衡。
