# Note 16 优化建议

## 1. 分片机制的背景
- **原文问题**：原文直接讨论分片问题，未解释为什么需要分片。
- **优化建议**：补充说明 **Extent Map Sharding** 的背景：对于极大对象（如 RBD 镜像），Extent Map 可能包含数百万条目。为了节省内存和加速查找，BlueStore 将 Map 按逻辑地址范围切分为多个 Shard（分片），每个 Shard 独立存储在 RocksDB 中。只有访问对应逻辑范围时，才加载对应的 Shard。

## 2. `spanning_blob` 的存储位置
- **原文问题**：原文提到 `spanning_blob_map`，但未说明其存储方式。
- **优化建议**：明确指出 `spanning_blob_map` 中的 Blob 信息通常存储在 **第一个引用它的 Shard** 中，或者作为一个独立的元数据条目存在。其他 Shard 在解码时，会通过 `consume_spanning_blob` 机制从共享区域加载这些跨分片 Blob 的引用。

## 3. 性能影响
- **原文问题**：原文未提及性能影响。
- **优化建议**：补充说明 `spanning_blob` 会增加元数据管理的复杂度。在 GC（垃圾回收）或 Compaction 时，处理跨分片 Blob 需要格外小心，以避免破坏其他 Shard 的引用。过多的 `spanning_blob` 可能会导致内存碎片和加载延迟。

## 4. 源码位置
- **优化建议**：核心逻辑位于 `src/os/bluestore/BlueStore.h` 的 `struct ExtentMap` 和 `struct ShardInfo` 中。`decode_spanning_blobs` 函数负责处理跨分片 Blob 的反序列化。
