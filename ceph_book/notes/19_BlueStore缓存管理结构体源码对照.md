# 2.3.2 BlueStore 缓存管理结构体源码对照

## 问题：书中介绍的缓存结构体在 Ceph 源码中是如何实现的？成员之间如何匹配？

### 解答

书中 2.3.2 小节介绍了 BlueStore 缓存管理的核心结构体，这些结构体在 Ceph 源码（`src/os/bluestore/BlueStore.h` 和 `BlueStore.cc`）中有明确的定义和实现。以下是详细的对照关系：

---

## 1. Cache 基类 → `BlueStore::CacheShard`

**源码位置**: `BlueStore.h:1470`

| 书中成员 (表 2-13) | 源码成员 | 说明 |
|-------------------|---------|------|
| `logger` | `PerfCounters *logger` | 用于缓存命中率相关的统计 |
| `lock` | `ceph::recursive_mutex lock` | 互斥锁，缓存相关的所有操作几乎都需要在 lock 的保护下进行 |
| `num_extents` | 移至 `BufferCacheShard` | 当前缓存的 extent 总数 |
| `num_blobs` | 移至 `BufferCacheShard` | 当前缓存的 blob 总数 |
| `last_trim_seq` | 改为 `age_bins` 机制 | 书中描述的是 mempool_seq 对比机制，源码中改用 `boost::circular_buffer<std::shared_ptr<int64_t>> age_bins` 实现缓存年龄分桶统计 |

**补充说明**:
- 源码中将 Cache 拆分为 `CacheShard`（通用基类）、`OnodeCacheShard`（Onode 缓存分片）和 `BufferCacheShard`（Buffer 缓存分片）三个层次
- `max` 和 `num` 原子变量用于控制缓存容量上限和当前使用量
- `_trim()` 方法通过比较当前使用量与 max 值来决定是否执行淘汰

---

## 2. OnodeSpace → `BlueStore::OnodeSpace`

**源码位置**: `BlueStore.h:1620`

| 书中成员 (表 2-14) | 源码成员 | 说明 |
|-------------------|---------|------|
| `cache` | `OnodeCacheShard *cache` | 表明自身归属于哪个 Cache 实例 |
| `onode_map` | `mempool::bluestore_cache_meta::unordered_map<ghobject_t,OnodeRef> onode_map` | 查找表，使用对象 ID 到 OnodeRef 的哈希映射 |

**补充说明**:
- `OnodeSpace` 作为中间结构，建立 Onode 和其归属的 Collection 之间的联系
- 每个 `Collection` 包含一个 `OnodeSpace onode_space` 成员（`BlueStore.h:1669`）
- 提供 `add_onode()`, `lookup()`, `rename()`, `clear()` 等方法管理 Onode 生命周期

---

## 3. BufferSpace → `BlueStore::BufferSpace`

**源码位置**: `BlueStore.h:419`

| 书中成员 (表 2-15) | 源码成员 | 说明 |
|-------------------|---------|------|
| `cache` | 通过方法参数传入 `BufferCacheShard* cache` | 表明自身归属于哪个 Cache 实例 |
| `buffer_map` | `buffer_map_t buffer_map` | 查找表，使用 boost::intrusive::set 以 Buffer::offset 为键组织 |
| `writing` | 移至 Buffer 的 `txc` 字段 | 包含脏数据的缓存队列，源码中通过 Buffer 的 `TransContext* txc` 关联事务上下文来管理 |

**补充说明**:
- `BufferSpace` 内嵌于 `Onode` 结构体中（`Onode` 包含 `BufferSpace buffer_space`）
- `buffer_map` 使用侵入式容器 `boost::intrusive::set`，避免额外的内存分配
- 提供 `discard()`, `read()`, `write()`, `finish_write()` 等核心接口（与书中表 2-17 一致）

---

## 4. Buffer → `BlueStore::Buffer`

**源码位置**: `BlueStore.h:312`

| 书中成员 (表 2-16) | 源码成员 | 说明 |
|-------------------|---------|------|
| `space` | `BufferSpace *space` | 表明自身归属于哪个 BufferSpace 实例 |
| `state` | `uint16_t state` | 枚举值：`STATE_EMPTY`, `STATE_CLEAN`, `STATE_WRITING`（与书中一致） |
| `cache_private` | `uint16_t cache_private` | 当 Buffer 存在于 Cache 中时，用于在 Cache 的不同队列之间进行调整 |
| `flags` | `uint32_t flags` | 目前仅定义 `FLAG_NOCACHE = 1`（与书中一致） |
| `offset` | `uint32_t offset` | 数据在 extent 中的逻辑起始地址 |
| `length` | `uint32_t length` | 数据长度 |
| `data` | `ceph::buffer::list data` | 数据本身，使用 Ceph 的 bufferlist 结构 |
| `seq` | `TransContext* txc` | 书中描述为写操作序列号，源码中改为直接关联事务上下文 |
| `lru_item` | `boost::intrusive::list_member_hook<> lru_item` | 用于将 Buffer 插入 Cache 的数据缓存队列 |
| `state_item` | `boost::intrusive::set_member_hook<> set_item` | 用于将 Buffer 插入 BufferSpace 的 buffer_map |

**补充说明**:
- 源码中增加了 `cache_age_bin` 成员用于缓存年龄分桶统计
- Buffer 状态转换与书中图 2-6 完全一致：`STATE_WRITING` → `STATE_CLEAN` → `STATE_EMPTY`

---

## 5. TwoQCache → `TwoQBufferCacheShard`

**源码位置**: `BlueStore.cc:1339`

| 书中成员 (表 2-18) | 源码成员 | 说明 |
|-------------------|---------|------|
| `onode_lru` | 由 `LruOnodeCacheShard` 实现 | 全局 Onode LRU 队列，源码中 Onode 和 Buffer 缓存分片独立实现 |
| `warm_in` | `list_t warm_in` | 对应 A1in 队列，新进入的温数据 |
| `buffer.hot` | `list_t hot` | 对应 Am 队列，热数据 |
| `warm_out` | `list_t warm_out` | 对应 A1out 队列，被淘汰的空页面影子队列 |
| `buffer_bytes` | `uint64_t buffer_bytes` | 全局缓存容量使用统计 |
| `buffer_list_bytes` | `uint64_t list_bytes[BUFFER_TYPE_MAX]` | 每种队列容量使用统计，用于应用淘汰策略 |

**补充说明**:
- 源码中定义了 `BUFFER_NEW`, `BUFFER_WARM_IN`, `BUFFER_WARM_OUT`, `BUFFER_HOT` 四种状态
- `cache_private` 字段用于标识 Buffer 当前位于哪个队列
- Onode 缓存使用 `LruOnodeCacheShard`（`BlueStore.cc:1089`），仅实现 LRU 策略，与书中描述"针对 Onode 采用 LRU；针对 Buffer 才是真正的 2Q"一致
- Buffer 缓存可选择 `LruBufferCacheShard` 或 `TwoQBufferCacheShard`，通过工厂方法 `BufferCacheShard::create()` 创建

---

## 6. 缓存分片机制 (Sharding) 与 数据/元数据隔离

### 6.1 全局缓存池与局部指针
书中提到“BlueStore 会实例化多个缓存”以减少锁碰撞，源码中通过**分片列表（Pool）**与**局部指针（Pointer）**实现：

*   **全局缓存池**：`BlueStore` 类中维护了一个 `BufferCacheShard*` 的 vector 列表（`BlueStore.h:2374`）。这是整个实例管理的所有缓存分片的“仓库”。
    ```cpp
    mempool::bluestore_cache_buffer::vector<BufferCacheShard*> buffer_cache_shards;
    ```
*   **局部指针**：每个 `Collection`（对应一个 PG）持有一个指向该列表中某个特定分片的指针（`BlueStore.h:1658`）。
    ```cpp
    BufferCacheShard *cache;       ///< our cache shard
    ```
*   **绑定逻辑**：当 Collection 创建时，根据其 ID (`coll_t`) 的哈希值计算索引，从全局池中分配一个分片：
    ```cpp
    // BlueStore.cc
    buffer_cache_shards[cid.hash_to_shard(buffer_cache_shards.size())]
    ```

### 6.2 数据缓存 vs 元数据缓存
源码明确区分了**数据**（用户 Payload）和**元数据**（Extent Map 等）的缓存路径：

| 缓存类型 | 对应结构体 | 缓存内容 | 算法 | 内存占比 |
|:---|:---|:---|:---|:---|
| **数据缓存** | `BufferCacheShard` | `Buffer` (实际对象内容/字节流) | LRU 或 2Q | 默认 ~10% |
| **元数据缓存** | `OnodeCacheShard` | `Onode` (含 Extent Map 映射表) | LRU | 默认 ~90% |

*   **配置参数**：`bluestore_cache_meta_ratio`（默认 0.9），即 90% 的缓存空间用于元数据。这是因为元数据查找是 IO 路径的第一步，且 Extent Map 通常较小，放入内存收益极大。

---

## 结构体之间的关联关系

```
BlueStore (全局管理器)
  │
  ├── [全局缓存池] buffer_cache_shards (vector<BufferCacheShard*>)
  │      │
  │      ├── Shard 0 (TwoQBufferCacheShard) ───┐
  │      ├── Shard 1 (LruBufferCacheShard)     │  <-- 多个 Collection 可能共享同一个 Shard
  │      └── Shard N ...                       │
  │                                            │
  ├── Collection A (PG 1.0)                     │
  │      │                                      │
  │      ├── OnodeSpace onode_space ────────────┤
  │      │      │                               │
  │      │      └── onode_map (查找 Onode)      │
  │      │                                      │
  │      └── BufferCacheShard *cache ───────────┘ (指向 Shard 0)
  │             │
  │             └── 管理该 PG 下所有 Buffer 的淘汰与热度
  │
  └── Collection B (PG 1.1)
         │
         └── BufferCacheShard *cache ───────────┐ (可能指向 Shard 1)
                                                │
  ...                                           │
                                                │
Onode (对象元数据)                               │
  │                                             │
  ├── Extent Map (逻辑偏移 -> 物理块映射)        │ <== 缓存在 OnodeCacheShard 中
  │                                             │
  └── BufferSpace buffer_space                  │
         │                                      │
         └── buffer_map (boost::intrusive::set) │
                │                               │
                └── Buffer (实际数据内容) ──────┘ <== 缓指向的 BufferCacheShard 中
                       ├── state (EMPTY/CLEAN/WRITING)
                       ├── cache_private (NEW/WARM_IN/WARM_OUT/HOT)
                       └── lru_item (链接到 Cache 队列)
```

**关键关联点**:
1. **Collection → OnodeSpace**: 每个 Collection 通过 OnodeSpace 管理其下属的所有 Onode
2. **Collection → CacheShard**: 每个 Collection 关联一个 OnodeCacheShard 和一个 BufferCacheShard
3. **BlueStore → CacheShard Pool**: BlueStore 维护全局 Shard 列表，Collection 通过哈希映射绑定到具体 Shard
4. **Onode → BufferSpace**: 每个 Onode 内嵌一个 BufferSpace，管理该对象的所有缓存 Buffer
5. **BufferSpace → Buffer**: BufferSpace 通过 buffer_map 管理多个 Buffer
6. **Buffer → CacheShard**: Buffer 通过 lru_item 和 cache_private 链接到 CacheShard 的具体队列中

---

## 源码与书籍的差异总结

| 差异点 | 书籍描述 | 源码实现 |
|-------|---------|---------|
| Cache 结构 | 单一 Cache 类 | 拆分为 CacheShard/OnodeCacheShard/BufferCacheShard 三层 |
| trim 机制 | last_trim_seq 对比 mempool_seq | 使用 age_bins 分桶统计 + max 阈值控制 |
| Buffer 序列号 | seq 字段 | 改为 TransContext* txc 直接关联事务 |
| writing 队列 | BufferSpace 中的 writing_list | 通过 Buffer 的 txc 字段关联，不再单独维护队列 |
| 缓存分片 | 提及多实例减少锁碰撞 | 实际实现为：BlueStore 维护全局 Shard 列表，Collection 通过哈希映射绑定到具体 Shard |
| 缓存分类 | 笼统描述 | 明确区分：BufferCacheShard 存数据，OnodeCacheShard 存元数据（默认 9:1 比例） |
