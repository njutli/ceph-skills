# MemStore vs BlueStore 核心区别

## 1. 存储介质与持久性

| 维度 | MemStore | BlueStore |
|------|----------|-----------|
| **介质** | 纯 RAM | 裸盘（直接管理块设备） |
| **持久性** | 易失（重启丢失） | 持久化 |
| **崩溃恢复** | 无 | 支持（WAL + 校验和） |

## 2. 架构设计

**MemStore** (`src/os/memstore/MemStore.h`):
- 数据存储在 `std::map<coll_t, collection>` 内存结构中
- 对象数据直接保存在 `bufferlist` 内存缓冲区
- 无文件系统层，无块设备抽象
- 代码量小，实现简单

**BlueStore** (`src/os/bluestore/BlueStore.h`):
- **绕过文件系统**，直接管理裸块设备（`/dev/sdX`）
- 自研元数据管理（RocksDB 存储元数据 + 对象映射）
- 三层分离架构：
  - `block`: 主数据区（对象数据）
  - `block.db`: 元数据/小 IO 加速区（RocksDB）
  - `block.wal`: 预写日志（Write-Ahead Log）

## 3. 数据布局对比

```
MemStore 内存布局:
  ┌─────────────────────────────────┐
  │  std::map<coll, collection>     │
  │    └─ map<oid, Object>          │
  │         └─ bufferlist (数据)    │
  │         └─ map<xattr> (属性)    │
  └─────────────────────────────────┘
  全部在 RAM 中，无持久化结构


BlueStore 磁盘布局:
  ┌─────────────────────────────────────────┐
  │  block device (裸盘)                     │
  │  ├── BlueFS (自研微型文件系统)           │
  │  │   ├── RocksDB (元数据: 对象偏移映射)  │
  │  │   └── 小对象数据 (< 64KB)             │
  │  ├── 主数据区 (大对象直接写入)           │
  │  └── 空闲空间管理 (Bitmap/分配器)        │
  │                                          │
  │  block.db (可选, SSD):                   │
  │  └── RocksDB 元数据加速                  │
  │                                          │
  │  block.wal (可选, NVMe):                 │
  │  └── 预写日志 (崩溃恢复)                 │
  └─────────────────────────────────────────┘
```

## 4. 功能特性

| 特性 | MemStore | BlueStore |
|------|----------|-----------|
| **校验和** | ❌ | ✅ (CRC32C/xxhash) |
| **压缩** | ❌ | ✅ (LZ4/ZSTD/Snappy) |
| **EC 支持** | ❌ | ✅ (Erasure Coding) |
| **碎片整理** | ❌ | ✅ (在线 defragmentation) |
| **多租户 QoS** | ❌ | ✅ (mClock 调度器) |
| **Scrub** | 基础 | 深度校验 (checksum 验证) |
| **克隆/快照** | 基础 | ✅ (高效 copy-on-write) |

## 5. 性能特征

```
MemStore:
  - 读/写: ~纳秒级 (纯内存操作)
  - 无 IO 延迟，无磁盘瓶颈
  - 受限于系统内存大小

BlueStore:
  - 读/写: ~微秒-毫秒级 (取决于介质)
  - 小 IO 优化: 写入 RocksDB (内存缓冲 + 批量刷盘)
  - 大 IO 优化: 直接写入裸盘 (零拷贝)
  - 支持 io_uring/SPDK 异步 IO
```

## 6. 使用场景

| 场景 | 推荐 | 原因 |
|------|------|------|
| **vstart 开发调试** | MemStore | 快速启动，无权限问题，无需磁盘 |
| **单元测试** | MemStore | 可重现，无 IO 噪声 |
| **性能基准测试** | BlueStore | 真实反映生产性能 |
| **生产部署** | BlueStore | 唯一生产级存储引擎 (FileStore 已废弃) |
| **学习数据流** | 两者皆可 | MemStore 更简单，BlueStore 更真实 |

## 7. 源码关键文件

```
MemStore:
  src/os/memstore/MemStore.h       # 主类定义
  src/os/memstore/MemStore.cc      # 实现 (~2000 行)

BlueStore:
  src/os/bluestore/BlueStore.h     # 主类定义 (~4000 行)
  src/os/bluestore/BlueStore.cc    # 实现 (~10000+ 行)
  src/os/bluestore/BlueFS.cc       # 自研微型文件系统
  src/os/bluestore/Allocator.cc    # 空间分配器
  src/os/bluestore/Compression.cc  # 压缩引擎
```

## 总结

**MemStore** 是开发调试工具，**BlueStore** 是生产级存储引擎。
- 学习 Ceph 数据流/PG 机制 → 用 **MemStore**（快速、简单）
- 学习存储引擎/IO 路径/压缩/校验 → 用 **BlueStore**（真实）
- 生产环境 → **只能用 BlueStore**
