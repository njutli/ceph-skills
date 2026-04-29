# Ceph Skills

基于 Ceph 主线源码分析总结的结构化知识集合。每个 SKILL.md 文件深入讲解一个 Ceph 子系统或机制，包含核心设计、数据结构、完整代码流程、关键代码位置（精确到行号）以及常见问题。

> 本项目的组织方式参考了 [linux-kernel-skills](https://github.com/linux-kernel-skills) 的模式，以源码为基础，从问题起源到代码实现逐层剖析。

## 快速导航

### 🏛️ 架构概览

| Skill | 内容 | 行数 |
|-------|------|------|
| [Ceph 整体概述](overview/SKILL.md) | 架构、五大守护进程、数据流、协议版本、10年演进 | 339 |

### 🚀 入门与部署

| Skill | 内容 | 行数 |
|-------|------|------|
| [WSL+QEMU 部署指南](deploy/wsl-qemu/SKILL.md) | 三种部署方案、12天学习路线、数据流验证、故障模拟 | 1394 |

### 📐 核心算法

| Skill | 内容 | 行数 |
|-------|------|------|
| [CRUSH 算法](crush/crush-algorithm/SKILL.md) | 数据分布算法、5种bucket类型、Straw2数学原理、规则系统 | 359 |

### 🔒 MON 集群

| Skill | 内容 | 行数 |
|-------|------|------|
| [Paxos 共识](mon/mon-paxos/SKILL.md) | Paxos状态机、3阶段共识、领导者选举、服务框架、lease机制 | 447 |

### 💾 OSD 存储

| Skill | 内容 | 行数 |
|-------|------|------|
| [OSD I/O 路径](osd/osd-io-path/SKILL.md) | 线程分布、写入请求追踪、ShardedOpWQ调度、BlueStore写入链路 | 395 |
| [PG Peering 状态机](osd/pg-peering/SKILL.md) | 完整状态机、GetInfo/GetLog/GetMissing、权威OSD选择、恢复流程 | 448 |
| [BlueStore 引擎](osd/bluestore/SKILL.md) | 磁盘布局、Onode/Blob映射、写入路径、Allocator、压缩 | 445 |
| [Scrubbing 机制](osd/scrub/SKILL.md) | 浅层/深层Scrub、状态机、副本哈希比较、调度策略 | 329 |

### 🌐 通信层

| Skill | 内容 | 行数 |
|-------|------|------|
| [AsyncMessenger](messenger/async-messenger/SKILL.md) | 连接状态机、ProtocolV2协议、epoll事件循环、RDMA/DPDK | 511 |

### 💻 客户端

| Skill | 内容 | 行数 |
|-------|------|------|
| [Objecter](client/objecter/SKILL.md) | 对象→OSD映射、op提交、ObjectCacher写回、Striper条带化 | 527 |

### 📀 RBD 块存储

| Skill | 内容 | 行数 |
|-------|------|------|
| [RBD 块存储](rbd/SKILL.md) | ImageCtx、快照/克隆/COW分层、ExclusiveLock排他锁、Journal日志、ObjectMap、迁移 | — |

### ☁️ RGW 对象存储

| Skill | 内容 | 行数 |
|-------|------|------|
| [RGW 对象存储网关](rgw/SKILL.md) | S3/Swift API、SAL抽象层、Bucket Index分片、Multisite多站点、Lifecycle、IAM策略 | — |

### 📂 CephFS 文件系统

| Skill | 内容 | 行数 |
|-------|------|------|
| [MDS 请求路径](cephfs/mds-request-path/SKILL.md) | 线程分布、客户端请求追踪、异步续跑模式、子系统消息路由 | 458 |
| [MDS Capability 系统](cephfs/mds-capability/SKILL.md) | Cap位定义、授权/撤销/释放、分布式锁、内核cap管理 | 400 |
| [MDS 缓存核心数据结构](cephfs/mds-cache-objects/SKILL.md) | CInode/CDentry/CDir关系、硬链接、目录分片、子树分区 | 544 |
| [内核 CephFS 挂载](cephfs/kernel-client-mount/SKILL.md) | 挂载9阶段流程、MON连接、MDS会话建立、选项解析 | 435 |

## 推荐学习路线

```
入门 (1-2周)
  │
  ├─ 1. Ceph 整体概述          → 理解架构和组件关系
  ├─ 2. WSL+QEMU 部署指南       → 动手搭建实验环境
  ├─ 3. CRUSH 算法             → 理解数据分布原理
  └─ 4. Objecter               → 理解客户端 I/O 路径
       │
       ▼
进阶 (2-4周)
  │
  ├─ 5. AsyncMessenger         → 理解通信层
  ├─ 6. OSD I/O 路径            → 理解 OSD 线程模型和写入链路
  ├─ 7. Paxos 共识             → 理解 MON 集群共识
  ├─ 8. PG Peering 状态机       → 理解数据一致性协商
  ├─ 9. RBD 块存储             → 理解 RADOS 之上如何构建块设备
  └─ 10. RGW 对象存储网关      → 理解 S3/Swift API 的 RADOS 映射
       │
       ▼
深入 (4-6周)
  │
  ├─ 11. BlueStore 引擎         → 理解底层存储
  ├─ 12. MDS 请求路径           → 理解 MDS 线程模型和异步续跑
  ├─ 13. MDS Capability 系统    → 理解分布式缓存一致性
  ├─ 14. MDS 缓存数据结构       → 理解 CInode/CDentry/CDir 关系
  └─ 15. Scrubbing 机制         → 理解数据完整性保障
```

## 每个 SKILL.md 的结构

每个技能文件遵循统一的组织方式：

```
〇、为什么需要这个机制？     — 问题起源，传统方案的不足
一、核心数据结构             — 关键结构体定义（带源码行号）
二、核心流程                 — 完整的调用链和状态转换
三、关键设计特性             — 设计哲学和权衡
四、关键代码位置             — 精确到文件:行号
五、常见问题与陷阱           — 实战中遇到的问题和解答
六、参考文献                 — 官方文档和论文
```

## 源码参考

本项目的分析基于以下代码库：

- **Ceph**: `https://github.com/ceph/ceph` (主线代码位于 `linux/ceph`)
- **Linux Kernel CephFS**: `https://github.com/torvalds/linux` (fs/ceph/, net/ceph/)

## 项目结构

```
ceph-skills/
├── overview/                   # 架构概览
│   └── SKILL.md                # Ceph 整体架构概述
├── deploy/                     # 部署指南
│   └── wsl-qemu/               # WSL+QEMU 学习环境
├── crush/                      # CRUSH 数据分布
│   └── crush-algorithm/        # CRUSH 算法详解
├── mon/                        # Monitor 集群
│   └── mon-paxos/              # Paxos 共识机制
├── osd/                        # OSD 存储
│   ├── osd-io-path/            # OSD I/O 路径与线程模型
│   ├── pg-peering/             # PG 状态机与 Peering
│   ├── bluestore/              # BlueStore 存储引擎
│   └── scrub/                  # OSD Scrubbing 机制
├── messenger/                  # 通信层
│   └── async-messenger/        # AsyncMessenger 通信层
├── client/                     # 客户端
│   └── objecter/               # Objecter OSD 客户端
├── rbd/                        # RBD 块存储
│   └── SKILL.md                # RBD 架构与实现
├── rgw/                        # RGW 对象存储
│   └── SKILL.md                # RGW 网关架构与实现
├── cephfs/                     # CephFS 文件系统
│   ├── mds-request-path/       # MDS 请求处理路径
│   ├── mds-capability/         # MDS Capability 系统
│   ├── mds-cache-objects/      # MDS 缓存核心数据结构
│   └── kernel-client-mount/    # 内核 CephFS 挂载流程
└── ceph_book/                  # 学习参考书
    ├── Ceph设计原理与实现_MinerU.md
    └── images/                 # 书中插图
```

## 贡献

欢迎提交 Issue 或 Pull Request 来补充和修正内容。每个 SKILL.md 中的代码行号基于特定版本的 Ceph 源码，如果源码更新导致行号变化，也欢迎提交修正。

## License

本项目内容采用 [CC BY-SA 4.0](https://creativecommons.org/licenses/by-sa/4.0/) 许可。
