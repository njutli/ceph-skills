# Note 15 优化建议

## 1. 补充 `SharedBlob` 机制
- **原文问题**：原文提到"多个逻辑 Extent 可以共享同一个物理 Blob"，但未解释如何管理共享引用。
- **优化建议**：补充说明 BlueStore 引入了 **`SharedBlob`** 结构（位于 RocksDB 的独立命名空间 `SHARED_BLOB_PREFIX`）。当多个 Onode（对象）的 Extent 指向同一个 Blob 时，`SharedBlob` 负责维护引用计数。只有当引用计数降为 0 时，物理空间才会被真正释放。这是实现克隆（Clone）和快照（Snapshot）零拷贝的核心。

## 2. `blob_offset` 的作用
- **原文问题**：原文在 `Extent` 结构体中提到了 `blob_offset`，但未详细解释。
- **优化建议**：明确 `blob_offset` 的含义：它表示该 Extent 引用的数据在 Blob 内部的起始偏移。因为一个 Blob 可能很大，而 Extent 只引用其中的一部分，所以需要 `blob_offset` 来精确定位。

## 3. 源码路径更新
- **原文问题**：源码路径准确，但可以补充命名空间。
- **优化建议**：指出 `bluestore_blob_t` 和 `PExtentVector` 定义在 `bluestore_types.h` 中，而内存中的 `Extent` 和 `BlobRef` 定义在 `BlueStore.h` 的 `BlueStore` 类内部。这反映了"磁盘持久化结构"与"内存运行时结构"的分离。
