# Note 24 优化建议

## 1. `bluefs=false` 的默认行为
- **原文问题**：原文提到 `bluefs=false` 时 RocksDB 使用默认 POSIX 接口。
- **优化建议**：明确指出在现代 Ceph 版本中，**BlueFS 是默认且唯一推荐**的 RocksDB 后端。`bluefs=false` 主要用于调试或极特殊的遗留环境。生产环境中几乎总是 `bluefs=true`。

## 2. IO 路径的零拷贝优化
- **原文问题**：原文提到 BlueFS 直接下发物理磁盘读取。
- **优化建议**：补充说明 BlueFS 支持 **`splice`** 或 **`io_uring`** 等高级 IO 机制，可以在某些场景下实现用户态到内核态的零拷贝或批量提交，进一步降低 RocksDB 刷盘时的 CPU 开销。

## 3. 多设备支持 (WAL/DB 分离)
- **原文问题**：原文未提及 BlueFS 的多设备管理。
- **优化建议**：BlueFS 支持将元数据日志（WAL）和数据文件（DB/SST）分配到不同的物理设备（如快盘和慢盘）。这通过 `BlueFS::add_block_device` 和文件分配策略实现，是优化 RocksDB 性能的重要手段。

## 4. 源码位置补充
- **优化建议**：BlueFS 的挂载和初始化逻辑位于 `src/os/bluestore/BlueFS.cc` 的 `mount()` 和 `mkfs()` 函数。RocksDB 与 BlueFS 的胶水代码位于 `src/rocksdb/env/BlueRocksEnv.cc`。
