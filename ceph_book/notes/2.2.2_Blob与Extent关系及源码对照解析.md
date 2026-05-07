# Q&A: BlueStore 表 2-7 (Blob) 与表 2-8 (Extent) 的关系及代码对应

## 问题
在阅读 2.2.2 节时，看到表 2-7 (Blob) 和表 2-8 (Extent)。
**疑问**：
1. 这两张表描述的结构体是什么关系？
2. 表中的成员如何对应到 Ceph 源码中的定义？特别是之前提到的 `PExtentVector` 究竟属于哪个表？

## 解答
这两张表描述了 BlueStore 中 **“逻辑地址”** 与 **“物理存储”** 的解耦与嵌套关系。

### 1. 核心关系：引用与展开
* **表 2-8 (Extent / 逻辑段)**：代表对象内部的一段**逻辑数据**。它不直接记录硬盘位置，而是通过成员 `blob` 指向一个物理容器。
* **表 2-7 (Blob / 物理块)**：这就是表 2-8 中 `blob` 成员**指向/展开**的结构。它负责描述数据在磁盘上的物理属性和实际位置。
* **一句话总结**：**表 2-7 是表 2-8 成员 `blob` 的详细实现**。多个逻辑 Extent 可以共享同一个物理 Blob（例如快照、克隆或数据共享场景）。

### 2. 源码与表格成员的精确对应

| 书中表格 | 成员名称 | Ceph 源码定义路径 | 对应代码实现与类型 |
| :--- | :--- | :--- | :--- |
| **表 2-8**<br>(Extent) | **`blob`** | `src/os/bluestore/BlueStore.h`<br>`struct Extent` | `BlobRef blob` (智能指针)。<br>该成员**引用**表 2-7 的结构体，是连接逻辑与物理的桥梁。 |
| ↳ **表 2-7**<br>(Blob) | **`extents`** | `src/os/bluestore/bluestore_types.h`<br>`struct bluestore_blob_t` | `PExtentVector extents`。<br>这才是真正的**物理位置数组**，属于表 2-7，而非表 2-8。 |
| ↳ 数组元素 | `物理段` | `src/os/bluestore/bluestore_types.h` | `bluestore_pextent_t`。<br>包含 `offset` (物理扇区) 和 `length`。 |

### 3. 关键代码实现解析
* **`PExtentVector` 的定义** (`bluestore_types.h`):
  ```cpp
  typedef mempool::bluestore_cache_other::vector<bluestore_pextent_t> PExtentVector;
  ```
  它是一个经过内存池优化的动态数组，专门用于存放物理段信息。
* **`Extent` 结构体** (`BlueStore.h`):
  ```cpp
  struct Extent {
    uint32_t logical_offset = 0; // 表 2-8: 逻辑起始地址
    uint32_t length = 0;         // 表 2-8: 逻辑长度
    BlobRef blob;                // 表 2-8: 指向表 2-7 的指针
    uint32_t blob_offset = 0;    // 表 2-8: Blob 内部偏移
  };
  ```

### 4. 💡 设计哲学：为什么拆分？
这种设计实现了 **逻辑与物理的彻底解耦** 和 **数据共享（Share-on-Disk）**：
* **寻址路径**：`对象逻辑地址` -> (查 Extent 表) -> `blob 指针` -> (查 Blob 表) -> `PExtentVector` -> `磁盘扇区`。
* **零拷贝共享**：在写时复制（COW）或快照场景中，上层只需创建新的 Extent（表 2-8），让其 `blob` 指针指向已有的 Blob（表 2-7）。无需复制底层物理数据，极大地节省了空间和写入延迟。
