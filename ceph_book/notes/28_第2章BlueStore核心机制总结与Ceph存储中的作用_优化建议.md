# Note 28 优化建议

## 1. `io_uring` 支持
- **原文问题**：原文主要基于 `libaio` 描述 IO 路径。
- **优化建议**：现代 Ceph 已全面支持 **`io_uring`** 作为 BlueStore 的默认异步 IO 后端。相比 `libaio`，`io_uring` 提供了更低的系统调用开销和更好的批量提交/完成队列机制。建议在"数据写入裸盘"部分补充 `io_uring` 的提及。

## 2. `deferred_write` 机制
- **原文问题**：原文未详细提及 BlueStore 的 `deferred_write`（延迟写）机制。
- **优化建议**：BlueStore 为了减少频繁的小块 RMW 对 RocksDB 的压力，会将部分非对齐覆盖写放入 `deferred_txn`（延迟事务队列），在后台批量合并执行。这是 BlueStore 优化写入放大的重要手段，建议补充。

## 3. `min_alloc_size` 的影响
- **原文问题**：原文假设 MAS 为 4KB。
- **优化建议**：补充说明 `min_alloc_size`（默认 HDD 64KB, SSD 4KB）对 MAS 切分的决定性影响。在 HDD 场景下，64KB 的 MAS 会导致更多的非对齐写和 RMW，这也是 Ceph 在 HDD 上写入性能较差的原因之一。

## 4. 压缩策略的演进
- **原文问题**：原文以 Snappy 为例。
- **优化建议**：现代 Ceph 默认推荐 **ZSTD** 压缩算法，它在压缩比和解压速度上取得了更好的平衡。BlueStore 还支持按 Pool 配置不同的压缩模式（`none`, `passive`, `aggressive`, `force`）。
