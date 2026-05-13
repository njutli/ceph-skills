# RGW 索引对象与应用对象解析

## 1. 疑问与确认
**疑问**: 书中"7.2.2 存储桶"一节提到了"索引对象"和"应用对象"。我的理解是：这两个"对象"本质上都是 **RADOS 对象**。
> *   **索引对象**: 用于记录存储桶下的"用户自定义对象"列表（用于 List Bucket 等操作）。
> *   **应用对象**: 用于存储实际的"用户自定义对象"数据。
> *   **结论**: 这样理解对不对？

**确认**: **完全正确**。这正是 RGW 为了实现存储桶目录功能与实际数据解耦而设计的架构。

---

## 2. 深度解析
在底层存储实现中，逻辑上的"存储桶 (Bucket)"实际上被拆分成了两类不同的 RADOS 对象，分别存放在不同的存储池（Pool）中：

### A. 应用对象 (Data Object / Application Object)
*   **作用**: 承载用户上传的实际文件内容（Payload）。
*   **映射**: 用户的一个逻辑对象（比如 `video.mp4`），根据其大小，在底层会被切分成 1 个或多个 Shadow RADOS 对象（如笔记 58 所述）。
*   **存放位置**: `data_pool`（由 Placement Policy 指定）。

### B. 索引对象 (Index Object)
*   **作用**: 维护 Bucket 的命名空间和对象列表。它充当了 **目录项 (Directory Entry)** 的角色。
*   **机制**:
    *   RGW 使用 RADOS 的 **omap (Object Map)** 功能来存储键值对。
    *   **Key**: 对象名称 (例如 `video.mp4`)。
    *   **Value**: 指向具体 RADOS 数据对象的元数据引用（包括对象名、ETag、大小、权限等）。
*   **存放位置**: `index_pool`。
*   **分片机制 (Sharding)**: 为了支持海量对象，一个存储桶可以配置多个索引对象（如 `num_shards`），通过 Hash 算法将不同对象的路由到不同的索引对象上，从而并行化 `ListBucket` 操作，避免单对象写入瓶颈。

### C. 对比总结
| 维度 | 索引对象 (Index Object) | 应用对象 (App Object) |
| :--- | :--- | :--- |
| **类比** | 目录项 (Directory Entry) | 文件数据块 (File Data Block) |
| **核心内容** | 元数据映射 (Name -> ID) | 实际用户数据 (User Data) |
| **所在 Pool** | `index_pool` | `data_pool` |
| **底层类型** | RADOS 对象 | RADOS 对象 |
