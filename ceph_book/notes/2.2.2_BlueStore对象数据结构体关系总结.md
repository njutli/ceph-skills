# 总结：BlueStore 对象相关数据结构体及关系 (2.2.2 节)

在 2.2.2 小节中，Ceph BlueStore 为了高效管理对象数据，设计了一套**逻辑与物理分离**的数据结构体系。以下是这些核心结构体的总结及其相互关系：

### 1. 核心结构体概览

| 结构体名称 | 层级 | 作用简述 | 关键成员 |
| :--- | :--- | :--- | :--- |
| **`bluestore_pextent_t`** | 物理底层 | 描述数据在磁盘上的**物理位置**（扇区号+长度）。 | `offset`, `length` |
| **`bluestore_blob_t`** | 物理容器 | **物理数据块**。管理一段连续的物理空间及其属性（如压缩、校验）。 | `extents` (PExtentVector), `flags`, `csum_data` |
| **`Extent`** | 逻辑映射 | **逻辑段**。描述对象内部的一段数据（逻辑偏移）映射到哪个 Blob。 | `logical_offset`, `blob` (引用), `blob_offset` |
| **`ExtentMap`** | 逻辑集合 | **对象的映射表**。管理该对象所有的 Extent 以及分片（Shard）信息。 | `extent_map` (Extent 集合), `spanning_blob_map`, `shards` |
| **`Onode`** | 对象核心 | **对象元数据**。代表一个对象，包含 OID 映射、属性及 ExtentMap 的分片指针。 | `oid`, `attrs`, `extent_map_shards` (持久化分片信息) |

---

### 2. 结构体关系图解

它们之间的关系是一个从**“对象”**到**“物理磁盘”**的层层引用链条：

```text
[Onode] (对象元数据)
   │
   ├─ 包含 ──> [ExtentMap] (逻辑映射表)
   │              │
   │              ├─ 包含多个 ──> [Extent] (逻辑段)
   │              │                  │
   │              │                  └─ 引用 ──> [Blob] (物理容器)
   │              │                                 │
   │              │                                 ├─ 包含 ──> [PExtentVector] (物理段数组)
   │              │                                 │              │
   │              │                                 │              └─ 元素 ──> [pextent_t] (物理地址)
   │              │
   │              └─ 管理 ──> [Shard] (分片信息)
   │                             │
   │                             └─ 记录在 Onode 的 extent_map_shards 中
   │
   └─ 包含 ──> [attrs] (对象属性)
```

---

### 3. 详细关系解析

1.  **Onode 是入口**：
    每个对象在 BlueStore 中都有一个 `Onode`。它是 KVDB 中索引对象的 Key，里面存着对象的基本信息（如大小、创建时间）和指向 `ExtentMap` 的指针。

2.  **ExtentMap 负责“切分”与“索引”**：
    `Onode` 通过 `ExtentMap` 来管理对象的数据。为了节省内存，`ExtentMap` 被切分成多个 **Shard**（分片）。`Onode` 中只保留分片的元数据（`extent_map_shards`），按需加载具体的 Extent。

3.  **Extent 连接逻辑与物理**：
    `ExtentMap` 里装满了 `Extent`。每个 `Extent` 告诉系统：*“对象的第 X 字节到第 Y 字节的数据，存放在某个 Blob 中。”*
    *   **注意**：`Extent` 是逻辑的，它不直接知道磁盘扇区，它只知道 `Blob`。

4.  **Blob 是物理实体**：
    `Blob` 是实际占用磁盘空间的容器。它里面有一个 `PExtentVector`（物理段数组），记录了数据具体存在磁盘的哪些扇区。
    *   **共享机制**：多个 `Extent`（甚至不同对象的 `Extent`）可以指向同一个 `Blob`（例如快照、克隆场景），实现**零拷贝共享**。

5.  **PExtentVector 是最终落脚点**：
    这是最底层的结构，直接对应磁盘的 LBA（逻辑块地址）。

### 💡 总结
这套设计的核心在于 **解耦**：
*   **逻辑层（Extent）** 负责“用户看到的数据在哪里”。
*   **物理层（Blob）** 负责“数据实际存在磁盘的哪里”。
*   中间通过 **引用（Reference）** 连接，使得 Ceph 能够轻松实现**数据共享（快照）**、**压缩**、**校验**以及**高效的内存分片管理**。
