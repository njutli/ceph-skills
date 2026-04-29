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
| [WSL+QEMU 部署指南](deploy/wsl-qemu/SKILL.md) | 三种部署方案、12天学习路线、数据流验证、故障模拟 | 1395 |

### 📐 核心算法

| Skill | 内容 | 行数 |
|-------|------|------|
| [CRUSH 算法](crush/crush-algorithm/SKILL.md) | 数据分布算法、5种bucket类型、Straw2数学原理、规则系统 | 364 |

### 🔒 MON 集群

| Skill | 内容 | 行数 |
|-------|------|------|
| [Paxos 共识](mon/mon-paxos/SKILL.md) | Paxos状态机、3阶段共识、领导者选举、服务框架、lease机制 | 454 |

### 💾 OSD 存储

| Skill | 内容 | 行数 |
|-------|------|------|
| [OSD I/O 路径](osd/osd-io-path/SKILL.md) | 线程分布、写入请求追踪、ShardedOpWQ调度、BlueStore写入链路 | 409 |
| [PG Peering 状态机](osd/pg-peering/SKILL.md) | 完整状态机、GetInfo/GetLog/GetMissing、权威OSD选择、恢复流程 | 460 |
| [BlueStore 引擎](osd/bluestore/SKILL.md) | 磁盘布局、Onode/Blob映射、写入路径、Allocator、压缩 | 457 |
| [Scrubbing 机制](osd/scrub/SKILL.md) | 浅层/深层Scrub、状态机、副本哈希比较、调度策略 | 335 |

### 🌐 通信层

| Skill | 内容 | 行数 |
|-------|------|------|
| [AsyncMessenger](messenger/async-messenger/SKILL.md) | 连接状态机、ProtocolV2协议、epoll事件循环、RDMA/DPDK | 518 |

### 💻 客户端

| Skill | 内容 | 行数 |
|-------|------|------|
| [Objecter](client/objecter/SKILL.md) | 对象→OSD映射、op提交、ObjectCacher写回、Striper条带化 | 539 |

### 📀 RBD 块存储

| Skill | 内容 | 行数 |
|-------|------|------|
| [RBD 块存储](rbd/SKILL.md) | ImageCtx、快照/克隆/COW分层、ExclusiveLock排他锁、Journal日志、ObjectMap、迁移 | 352 |

### ☁️ RGW 对象存储

| Skill | 内容 | 行数 |
|-------|------|------|
| [RGW 对象存储网关](rgw/SKILL.md) | S3/Swift API、SAL抽象层、Bucket Index分片、Multisite多站点、Lifecycle、IAM策略 | 422 |

### 📂 CephFS 文件系统

| Skill | 内容 | 行数 |
|-------|------|------|
| [MDS 请求路径](cephfs/mds-request-path/SKILL.md) | 线程分布、客户端请求追踪、异步续跑模式、子系统消息路由 | 471 |
| [MDS Capability 系统](cephfs/mds-capability/SKILL.md) | Cap位定义、授权/撤销/释放、分布式锁、内核cap管理 | 412 |
| [MDS 缓存核心数据结构](cephfs/mds-cache-objects/SKILL.md) | CInode/CDentry/CDir关系、硬链接、目录分片、子树分区 | 552 |
| [内核 CephFS 挂载](cephfs/kernel-client-mount/SKILL.md) | 挂载9阶段流程、MON连接、MDS会话建立、选项解析 | 442 |

## 推荐学习路线

路线将 ceph-skills 的 15 篇专题与《Ceph设计原理与实现》穿插结合，按三阶段推进，共约 6 周。

---

### 📅 第一阶段：建立全局认知（第 1-2 周，5 篇 Skill）

**目标**：知道 Ceph 有哪些组件、如何通信、数据怎么分布，能动手跑一个集群。

| 天数 | Skill | 核心收获 | 配合书章 |
|------|-------|---------|---------|
| 第 1 天 | [Ceph 整体概述](overview/SKILL.md) | 五大守护进程的职责、三条数据链路（块/文件/对象）、CRUSH 计算寻址 vs 中心元数据 | 前言 + 目录 |
| 第 2-4 天 | [WSL+QEMU 部署指南](deploy/wsl-qemu/SKILL.md) | 搭建最小 Ceph 集群，`rados put` 验证数据落盘，`vstart.sh` 了解开发环境 | — |
| 第 5-7 天 | [CRUSH 算法](crush/crush-algorithm/SKILL.md) | Straw2 伪代码→数学原理→rule 语法，理解"客户端自行计算数据位置" | **第 1 章 §1.1-1.2** |
| 第 8-10 天 | —（书） | 继续阅读第 1 章 §1.3 调制 CRUSH（编辑 map、定制规则、数据重平衡），这是全书最重要的实战章节 | **第 1 章 §1.3-1.4** |
| 第 11-14 天 | [Objecter](client/objecter/SKILL.md) | `pg_t` 计算 → `_calc_target` → Op 提交流程，理解客户端如何把对象 I/O 发给正确的 OSD | — |

**里程碑**：在部署的集群上用 `ceph osd map` 验证一个对象到底落到哪三个 OSD 上，结果和 CRUSH 手工计算一致。

---

### 📅 第二阶段：攻坚核心路径（第 3-4 周，6 篇 Skill）

**目标**：追踪一条写入请求从客户端到落盘的完整调用链，理解集群元数据的一致性保证。

| 天数 | Skill | 核心收获 | 配合书章 |
|------|-------|---------|---------|
| 第 15-16 天 | [AsyncMessenger](messenger/async-messenger/SKILL.md) | ProtocolV2 握手、epoll 事件循环、连接状态机，理解网络层如何复用连接、心跳保活 | — |
| 第 17-18 天 | [OSD I/O 路径](osd/osd-io-path/SKILL.md) | ShardedOpWQ 调度 → FastDispatch → ReplicatedBackend，追踪 op 在 OSD 内部穿越的每条线程 | **第 4 章 PG 读写** |
| 第 19-20 天 | [Paxos 共识](mon/mon-paxos/SKILL.md) | Paxos propose→accept→commit 三阶段、lease 机制、OSDMap epoch 如何单调推进 | — |
| 第 21-23 天 | [PG Peering 状态机](osd/pg-peering/SKILL.md) | GetInfo→GetLog→GetMissing→Active 全状态变迁，权威 OSD 如何选出，脑裂如何避免 | **第 4 章 状态迁移** |
| 第 24-25 天 | [RBD 块存储](rbd/SKILL.md) | ImageCtx → 条带化 → ExclusiveLock → 快照/克隆 COW 分层 → Journal | **第 6 章 RBD** |
| 第 26-28 天 | [RGW 对象存储网关](rgw/SKILL.md) | S3 请求 → SAL 抽象 → Bucket Index 分片 → 最终落到 RADOS 对象的完整映射 | **第 7 章 RGW** |

**里程碑**：能用 `rbd map` 挂载块设备后，追踪 `dd if=/dev/zero of=/dev/rbd0 bs=4k count=1` 的完整路径：客户端 Objecter 计算 OSD → AsyncMessenger 发送 → OSD op_shardedwq 调度 → BlueStore 写入。

---

### 📅 第三阶段：深入子系统（第 5-6 周，5 篇 Skill）

**目标**：理解底层存储引擎、分布式文件系统元数据管理、数据完整性保障。

| 天数 | Skill | 核心收获 | 配合书章 |
|------|-------|---------|---------|
| 第 29-31 天 | [BlueStore 引擎](osd/bluestore/SKILL.md) | Onode→Blob→Extent 三层映射、WAL/DB/Slow 三设备布局、BitmapAllocator 空间分配 | **第 2 章 §2.1-2.6** |
| 第 32-33 天 | [MDS 请求路径](cephfs/mds-request-path/SKILL.md) | MDS 单线程+异步续跑模式、MDRequest 生命周期、MDS 内部子系统消息路由 | **第 8 章 CephFS** |
| 第 34-35 天 | [MDS Capability 系统](cephfs/mds-capability/SKILL.md) | Cap 位定义（AsXx/BsXx/CxXx）、授权/撤销/释放流程、Loner 机制、内核 cap 同步 | **第 8 章** |
| 第 36-38 天 | [MDS 缓存核心数据结构](cephfs/mds-cache-objects/SKILL.md) | CInode/CDentry/CDir 三对象关联、硬链接表示、目录分片、子树分区 | **第 8 章** |
| 第 39-40 天 | [内核 CephFS 挂载](cephfs/kernel-client-mount/SKILL.md) | mount 9 阶段：MON 连接→认证→MDS 会话→cap→superblock→root dentry | — |
| 第 41-42 天 | [Scrubbing 机制](osd/scrub/SKILL.md) | shallow vs deep scrub、副本间哈希比较、调度策略、自动修复 | **第 4 章 数据修复** |

**里程碑**：能解释一次 4KB 随机写如何经过 BlueStore 三层映射落到 NVMe 的哪个 sector，以及 `mv /mnt/cephfs/a /mnt/cephfs/b` 在 MDS 内部触发的锁操作和缓存状态变迁。

---

### 📋 全书配套阅读索引

| 书章节 | 推荐阅读时机 | 对应 Skill |
|--------|-------------|-----------|
| 第 1 章 CRUSH | 第一阶段（必读） | crush/crush-algorithm |
| 第 2 章 §2.1-2.6 BlueStore | 第三阶段 | osd/bluestore |
| 第 2 章 §2.7 配置参数 | 用到时查阅 | — |
| 第 3 章 纠删码 | 学完 replicated pool 后按需阅读 | —（暂缺 EC Skill） |
| 第 4 章 PG 读写与状态迁移 | 第二阶段 | osd/osd-io-path + osd/pg-peering |
| 第 5 章 QoS | 需要性能隔离时再翻 | —（暂缺 QoS Skill） |
| 第 6 章 RBD | 第二阶段 | rbd/SKILL |
| 第 7 章 RGW | 第二阶段 | rgw/SKILL |
| 第 8 章 CephFS | 第三阶段 | cephfs/mds-* 系列 |
| 第 9 章 实战案例 | 学完所有模块后翻看 | — |

### 💡 使用建议

1. **顺序灵活**：第二阶段中 RBD 和 RGW 相互独立，可交换顺序；第三阶段中 BlueStore 和 MDS 系列也无先后依赖。
2. **动手优先**：每个 Skill 都标注了源码行号，<u>强烈建议在 IDE 中打开对应文件对照阅读</u>。理解分布式系统最好的方式就是"阅读源码，运行它，调试它"。
3. **书当字典**：不要试图从头到尾通读《Ceph设计原理与实现》。学到哪个子系统，翻到对应那十几页，把它当"设计意图说明书"用。
4. **版本注意**：《Ceph设计原理与实现》基于 Ceph Kraken (2017)，部分配置参数名和默认行为已变化。涉及具体配置时以 ceph-skills 中的源码行号和 Ceph 最新文档为准。

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

- **Ceph**: `https://github.com/ceph/ceph`
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
