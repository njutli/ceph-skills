# Q&A: BlueStore 元数据转 KV 对存储与索引机制

## 问题
在 2.2 节中提到 “所有元数据都设计成可以和用户数据分开存放，目前统一以键值对的形式存放在 kvDB 中”。
**疑问**：
1. 元数据是如何转换成键值对（Key-Value）的？
2. 转换成键值对存储后，读取数据时是如何索引的？请举个例子。

## 解答
这是 BlueStore 架构的核心设计，目的是将 **控制面 (Control Plane)** 与 **数据面 (Data Plane)** 彻底分离。
*   **数据面**：用户数据以裸数据的形式直接写入 Block Device，路径最短，性能最高。
*   **控制面**：所有描述数据位置、属性的信息全部存入 KVDB (RocksDB)。

### 1. 转换机制：元数据 -> KV 对
BlueStore 通过一种**紧凑的二进制编码（Binary Encoding）**将复杂的对象结构转换为 KV 对，而不是人类可读的纯字符串。
*   **Key (键)**：采用 **`前缀 + 二进制序列化`** 的结构。
    *   **前缀 (Prefix)**：源码（`src/os/bluestore/BlueStore.cc`）中定义了明确的前缀常量，如 `PREFIX_COLL` ("C") 用于标识 Collection，`PREFIX_OBJ` ("O") 用于标识对象。
    *   **编码**：前缀之后紧跟的是 Collection ID、Object ID 以及元数据类型的**二进制哈希值或序列化结构体**。这保证了极小的存储体积和极快的解析速度（源码避免使用字符串查找的开销）。
*   **Value (值)**：存储实际的物理信息或属性结构体（如序列化的 `bluestore_onode_t`）。

### 2. 索引与读取过程：查找 Onode 再解析分片
当系统需要读取一个对象时，它不直接读数据盘，而是先通过 KV 查找**Onode**。

**具体示例**：
假设客户端请求读取对象 `rbd_data.1024` 的内容。

1.  **构造 Key**：
    OSD 收到请求，生成对应的二进制 Key：`['O', Collection ID序列化, Object ID序列化]`（即 `PREFIX_OBJ` 组合）。

2.  **查询 KVDB (RocksDB) 获取 Onode**：
    向 RocksDB 发起查询。RocksDB 返回的 Value 不是直接的物理地址，而是该对象的**元数据核心 —— Onode**（序列化后的 `bluestore_onode_t` 结构）。

3.  **解析分片并定位物理地址**：
    对于大对象，其物理地址映射（Extent Map）可能被切分为多个 **Shard**（分片）。
    *   BlueStore 首先从 Onode 中读取 `extent_map_shards` 数组，确定目标偏移量位于哪个 Shard 中。
    *   根据 Shard 信息，在内存中（或缓存未命中时回查 KVDB）加载对应的 Shard 数据。
    *   从 Shard 中解析出最终的 **Extent** 列表，提取物理地址（如 `LBA=5000, Length=4MB`）。

4.  **直接物理读取**：
    BlueStore 拿到物理 LBA 后，**完全绕过文件系统逻辑**，直接向底层磁盘发起 I/O 读取指令。

5.  **返回数据**：
    将读到的数据返回给前端应用。

**优势**：
因为 RocksDB 是内存型数据库，Key 的索引都在内存中（LSM-Tree 结构），所以查找过程极快。而一旦找到位置，直接读硬盘，没有任何中间层开销。
