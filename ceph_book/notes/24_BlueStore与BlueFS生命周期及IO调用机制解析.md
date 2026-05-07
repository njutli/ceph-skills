# BlueStore 与 BlueFS 生命周期及 IO 调用机制解析

## 1. BlueStore 与 BlueFS 生命周期关联

在源码实现中，BlueFS 的生命周期是完全**嵌入并受控于** BlueStore 的生命周期的。BlueFS 就像 BlueStore 的一个底层驱动程序。

| BlueStore 生命周期函数 | BlueFS 的调用逻辑 | `bluefs=false` 时的行为 |
| :--- | :--- | :--- |
| `BlueStore::mkfs` | 检测到启用 BlueFS -> 分配裸设备 -> 调用 **`BlueFS::mkfs`** 初始化超级块和裸盘结构。 | **不调用** BlueFS mkfs。仅格式化 BlueStore 自身的主数据结构（Superblock 等），RocksDB 将使用默认 POSIX 接口。 |
| `BlueStore::mount` | 初始化 `BlueFS` 对象 -> 调用 **`BlueFS::mount`** (重放日志, 恢复文件状态)。 | **不调用** BlueFS mount。RocksDB 直接挂载本地已存在的文件系统目录 (如 XFS/ext4)。 |

## 2. BlueFS Read/Write 流程示例

当 BlueFS 接管 RocksDB 的本地文件系统时，它通过用户态直接操作裸盘，绕过内核 VFS。

### 场景：RocksDB Flush 生成 SST (Write)

当 RocksDB 将内存数据落盘时，调用链路如下：

1.  **Trigger**: RocksDB MemTable 写满，通过 `BlueRocksEnv` 调用 `NewWritableFile("000042.sst")` 创建文件。
2.  **BlueFS::write() 处理**:
    *   **分配空间**: 调用 Allocator 从裸设备分配连续的物理空间。
    *   **写入**: 直接将数据拷贝并构造 IO 请求，通过 `libaio`/`io_uring` 下发**物理扇区写入**。
    *   **更新**: 更新内存中 `bluefs_fnode_t` 记录物理块映射。
    *   **记录日志**: 将分配和写入动作编码日志条目 (`op_bl`)，写入 BlueFS 专用 Log 文件。

### 场景：Compaction 读取 SST (Read)

RocksDB 后台整理数据时，需要读取旧文件：

1.  **Trigger**: `NewRandomAccessFile("000038.sst")` -> `Read(offset, len)`。
2.  **BlueFS::read() 处理**:
    *   **映射**: 在 `file_map` 中找到对应的 `fnode` 和物理块映射表。
    *   **定位**: 计算物理磁盘地址。
    *   **读取**: 直接下发命令从**物理磁盘**读取数据，返回给 RocksDB (无 Page Cache)。
