---
name: ceph-overview
description: Ceph分布式存储系统整体概述。当用户询问Ceph架构、组件关系、数据流、设计哲学、MON/OSD/MDS/MGR/RGW职责、CRUSH算法原理时调用此技能。
---

# Ceph 分布式存储系统概述

## 1. 问题起源

在 Ceph 出现之前，企业级存储主要依赖两种方案：

**传统 SAN/NAS：**
- 专有硬件，昂贵且封闭
- 单控制器架构，存在 SPOF（单点故障）
- 扩展需要购买更大控制器（scale-up），而非添加节点（scale-out）

**早期分布式文件系统（如 GFS、Lustre）：**
- 依赖中心化的元数据服务器，成为性能瓶颈
- 元数据服务器故障 = 整个文件系统不可用
- 数据分布需要手动配置，无法自动均衡

## 2. 为什么需要 Ceph

Ceph 解决的核心问题是：**在 commodity hardware 上构建无单点、自管理、无限扩展的统一存储平台**。

| 问题 | 传统方案 | Ceph 的解决 |
|------|---------|------------|
| 单点故障 | 主从架构 | 所有组件多活，MON 使用 Paxos 共识 |
| 数据分布 | 中心元数据表 | CRUSH 算法，客户端自行计算 |
| 扩展性 | Scale-up | Scale-out，添加节点自动重平衡 |
| 统一存储 | 多套系统 | RADOS 之上提供 Block/File/Object 三种接口 |
| 自管理 | 人工运维 | 自动恢复、自动重平衡、自动选举 |

Ceph 由 Sage Weil 在 2003-2006 年博士期间设计，2006 年发表论文，同年代码开源。2014 年 Red Hat 收购 Inktank（Ceph 商业化公司），2015 年 Red Hat 收购 Ceph 商标。

## 3. 核心设计

Ceph 的设计哲学是**去中心化 + 智能客户端**：

```
                    Ceph 整体架构

    ┌─────────────────────────────────────────────────────┐
    │                   应用层接口                          │
    │  ┌──────────┐  ┌──────────┐  ┌──────────────────┐   │
    │  │  RBD     │  │ CephFS   │  │  RGW (S3/Swift)  │   │
    │  │ (块存储)  │  │ (文件存储) │  │  (对象存储)       │   │
    │  └────┬─────┘  └────┬─────┘  └────────┬─────────┘   │
    │       │             │                 │              │
    │  ┌────┴─────────────┴─────────────────┴──────────┐   │
    │  │              librados (客户端库)                │   │
    │  │  ┌─────────────┐  ┌──────────┐  ┌──────────┐  │   │
    │  │  │  Objecter   │  │MonClient │  │Messenger │  │   │
    │  │  └─────────────┘  └──────────┘  └──────────┘  │   │
    │  └────────────────────────────────────────────────┘   │
    └───────────────────────────┬───────────────────────────┘
                                │
    ┌───────────────────────────┴───────────────────────────┐
    │                    RADOS 集群                          │
    │  ┌─────────┐  ┌─────────┐  ┌─────────┐  ┌─────────┐  │
    │  │  MON    │  │  MON    │  │  MON    │  │  MGR    │  │
    │  │(Paxos)  │  │(Paxos)  │  │(Paxos)  │  │(Manager)│  │
    │  └─────────┘  └─────────┘  └─────────┘  └─────────┘  │
    │  ┌─────────┐  ┌─────────┐  ┌─────────┐  ┌─────────┐  │
    │  │  OSD    │  │  OSD    │  │  OSD    │  │  MDS    │  │
    │  │(存储)    │  │(存储)    │  │(存储)    │  │(元数据)  │  │
    │  └─────────┘  └─────────┘  └─────────┘  └─────────┘  │
    └───────────────────────────────────────────────────────┘
```

关键设计决策：
- **CRUSH 算法**：客户端自行计算数据位置，无需中心查找表
- **智能客户端**：客户端直接与 OSD 通信，不经过代理
- **Paxos 共识**：MON 集群强一致，保证集群元数据的可靠性
- **Capability 系统**：MDS 将权限委托给客户端，实现分布式缓存一致性

## 4. 五大守护进程

### MON (Monitor)
- 维护集群主数据（monmap、osdmap、mdsmap、mgrmap、pgmap）
- 使用 Paxos 实现多节点共识
- 通常部署 3 或 5 个节点（容忍 1 或 2 个故障）
- 代码位置：`src/mon/`

### OSD (Object Storage Daemon)
- 存储数据对象到本地磁盘（BlueStore）
- 处理复制、恢复、scrubbing
- 每个 OSD 对应一块物理磁盘
- 代码位置：`src/osd/`

### MDS (Metadata Server)
- 仅 CephFS 需要，管理文件元数据
- 分布式缓存 + 动态子树分区
- Capability 委托机制
- 代码位置：`src/mds/`

### MGR (Manager)
- 监控、仪表板、插件平台
- Python 模块生态
- 代码位置：`src/mgr/`

### RGW (Rados Gateway)
- S3/Swift 兼容的对象存储网关
- REST API 接口
- 代码位置：`src/rgw/`

## 5. 核心概念：PG 与 Pool

在 Ceph 庞大的集群中，数据并不是直接散落在 OSD 上的。**PG (Placement Group)** 和 **Pool** 是连接“逻辑数据视图”与“物理存储设备”的中间层。

### 5.1 什么是 PG？
**PG (Placement Group，归置组)** 是 Ceph 数据分布和管理的核心逻辑单元。

在物理上，**PG 并不对应任何实际的目录或文件**，它纯粹是一个通过 Hash 计算得出的**虚拟 ID**（如 `1.5a3`）。Ceph 的数据流向并不是“对象 → OSD"，而是：
**对象 (Object) → 归置组 (PG) → OSD 组**

你可以把 **PG** 想象成一个**“逻辑集装箱”**：
*   **对数据而言**：PG 是一组对象的集合。成千上万个对象被哈希到同一个 PG ID 中。
*   **对集群而言**：PG 是管理粒度的基本单位。Ceph 的所有核心运维动作（副本复制、故障恢复、数据校验 Scrub、状态协商 Peering）都是**以 PG 为单位**进行的，而不是以单个文件/对象为单位。

### 5.2 为什么需要引入 PG？（解决元数据爆炸）
如果没有 PG，Ceph 需要维护 `10 亿个对象 ↔ OSD` 的精确映射：
*   **灾难场景**：100 个 OSD 的集群，每个 OSD 可能要记录上千万个对象的位置和健康状态。扩容或单盘故障时，集群需要重新计算每个对象的去向，元数据同步和状态交换会让整个集群陷入瘫痪。

引入 PG 后，逻辑分层解耦：
*   **元数据骤降**：假设集群规划 1024 个 PG，平均每个 OSD 负责约 10 个 PG。OSD 只需维护十几个 PG 的状态，而不是上千万个对象的状态。
*   **极速恢复**：OSD 故障时，只需要对比少量 PG 的版本号（`pg_log`）就能完成数据重建，效率提升几个数量级。

### 5.3 PG 与 Pool 的关系
**Pool (存储池)** 是 PG 的**逻辑容器与规则制定者**。

*   **包含关系**：一个 Pool 由一组连续的 PG 组成。例如 Pool `data` 配置 `pg_num=1024`，它就拥有编号为 `0~1023` 的所有 PG（对应 PG ID 为 `1.0` 到 `1.3ff` 等）。
*   **规则继承**：Pool 定义了其下属所有 PG 的物理行为策略：
    *   `size`: 存几份副本。
    *   `crush_rule`: 数据往哪里落。
    *   `pg_num`: 这个 Pool 切分成多少个逻辑块。
*   **隔离性**：不同 Pool 的 PG 互不干扰，可以实现“数据隔离”或“性能隔离”。



### 5.4 核心映射路径总结
```
客户端写入 object "photo.jpg"
   │
   ▼
1. 映射到 Pool:     pool_id = 1 (data)
   │
   ▼
2. 映射到 PG:       pg_id = hash(object_name) % pg_num = 352
   │                (得到 PG ID: 1.160 (即 Pool1 的第 352 号 PG))
   │
   ▼
3. 映射到 OSD:      CRUSH(pg_id=352, osdmap) → [osd.1(主), osd.3(备), osd.5(备)]
   │
   ▼
最终落盘:           数据被发送到 osd.1，并由 osd.1 负责分发给 osd.3 和 osd.5
```

## 6. 三大接口写入数据流

由于 Ceph 的三种接口（块/对象/文件）客户端实现不同，写入路径也有区别。
下文中 `【客户端】` 表示运行在用户空间（或内核空间）的组件，`【服务端】` 表示运行在存储服务器上的 Ceph 守护进程。

### 6.1 块存储 (RBD)

Ceph 提供两种块设备访问方式：**内核模块** (生成 `/dev/rbdX`) 和 **用户态库** (常用于 QEMU)。

**(1) 内核态路径 (/dev/rbdX)**
内核模块 `rbd.ko` 直接实现协议栈，不使用 `librbd`。

```
【客户端】(运行在 Linux 内核中)
  │
  ├─ 用户应用 (对 /dev/rbdX 执行 I/O)
  │
  ▼
  ├─ VFS → Generic Block Layer (提交 bio)
  │
  ▼
  ├─ RBD Driver (drivers/block/rbd.c)
  │    ├─ rbd_queue_workfn: 接收 block I/O
  │    └─ 将 bio 转换为 ceph osd_request
  │
  ▼
  └─ OSD Client (net/ceph/osd_client.c) ← 内核版 Objecter/CRUSH
       │
       ▼
──────────────────────────────────────────────────────────────
【服务端】
     OSD.X (Primary OSD)
       └─ BlueStore (src/os/bluestore/BlueStore.cc)
```

**(2) 用户态路径 (librbd, 常用于 QEMU/KVM)**
此路径不生成块设备文件，而是由 QEMU 进程直接读写镜像。

```
【客户端】(运行在 QEMU 进程中)
  │
  ├─ Guest VM (访问虚拟磁盘 vda)
  │
  ▼
  ├─ QEMU Block Driver (drive=rbd:id=myimage...)
  │    (构造 librados rbd image name)
  │
  ▼
  ├─ librbd (src/librbd/ImageCtx)
  │
  ▼
  └─ librados / Objecter (src/osdc/Objecter.cc)
       │
       ▼
──────────────────────────────────────────────────────────────
【服务端】
     OSD.X (Primary OSD)
       └─ BlueStore
```

### 6.2 对象存储 (RGW)

```
【客户端】(运行在外部 HTTP 客户端)
  │
  ├─ S3/Swift Client (curl / boto / aws-cli)
  │    (发送 HTTP PUT request)
  │
  │  网络发送
  ▼
──────────────────────────────────────────────────────────────
【服务端】(RGW 网关进程 + OSD 集群)
  │
  ▼
  ├─ RGW Gateway (src/rgw)
  │    (解析 HTTP 请求，鉴权，将 Bucket+Key 映射为 RADOS 对象名)
  │
  ▼
  └─ librados (集成在 radosgw 进程内部)
       (作为客户端代理组件，计算 CRUSH，联系 OSD)
       │
       │  网络发送 (内部集群通信)
       ▼
     OSD.X (Primary OSD)
       (src/osd/OSD.cc)
       │
       ▼
     BlueStore
       └─ 数据写入磁盘后回复 RGW
```

### 6.3 文件存储 (CephFS - 内核态)

**注意**：内核态 CephFS (`mount -t ceph`) **不经过用户态 librados**，因此不调用 `Objecter`。它使用内核内置的等效组件。

```
【客户端】(Linux 内核空间 + 用户空间)
  │
  ├─ 用户应用 (write syscall)
  │
  ▼
  ├─ VFS 层 (fs/ceph/file.c)
  │
  ├─ 元数据操作 (lookup, getattr 等)
  │    └─ MDS Client (fs/ceph/mds_client.c)
  │       (发送请求给 MDS 守护进程)
  │
  ▼
  └─ 数据读写操作 (read/write file data)
       └─ OSD Client (net/ceph/osd_client.c)  ← 内核版的 Objecter
          │
          ├─ 内核 CRUSH (net/ceph/crush.c): 计算对象所在的 OSD
          │
          │  网络发送
          ▼
──────────────────────────────────────────────────────────────
【服务端】(运行在存储集群节点上)
       │
       ▼
     OSD.0 (Primary OSD)
       (src/osd/OSD.cc)
       │
       ▼
     BlueStore
       └─ 写入磁盘，分发到副本，回复内核客户端
```

### 关键总结

1. **RBD 和 RGW 都在用户态**：经过 `librados` → `Objecter` → `AsyncMessenger` → OSD。
2. **内核 CephFS 走内核态**：不经过 `librados`，而是直接调用内核的 `ceph_osd_client` → `ceph_msgr` → OSD。
3. **无论哪条路径，数据最终都通过相同的机制落盘到 BlueStore。**

## 7. 关键数据结构

### OSDMap 核心字段

```c++
/* src/osd/osd_types.h */
struct OSDMap {
    uuid_fsid;              // 集群唯一标识
    epoch_t epoch;          // 当前版本号（每次变更 +1）
    utime_t created;        // 创建时间
    int32_t pool_max;       // 最大 pool ID
    uint32_t flags;         // 集群标志
    
    // OSD 信息
    int32_t max_osd;        // 最大 OSD 数量
    vector<osd_info_t> osd_info;   // 每个 OSD 的信息
    vector<uint8_t> osd_state;      // OSD 状态 (up/down/in/out)
    
    // Pool 信息
    map<int64_t, pg_pool_t> pools;  // 所有 pool 配置
    
    // CRUSH
    CrushWrapper crush;     // CRUSH 映射
    
    // 计算 PG → OSD 映射
    int pg_to_up_acting_osds(pg_t pg, ...);
};
```

### CRUSH 规则

```
rule replicated_rule {
    id 0
    type replicated
    min_size 1
    max_size 10
    step take default
    step chooseleaf firstn 0 type host
    step emit
}
```

## 8. 协议版本

| 协议 | 版本 | 用途 |
|------|------|------|
| CEPH_OSDC_PROTOCOL | 24 | OSD 客户端/服务端 |
| CEPH_MDSC_PROTOCOL | 32 | MDS 客户端/服务端 |
| CEPH_MONC_PROTOCOL | 15 | Monitor 客户端/服务端 |
| CEPH_MON_PROTOCOL | 13 | MON 集群内部 |
| CEPH_OSD_PROTOCOL | 10 | OSD 集群内部 |
| CEPH_MDS_PROTOCOL | 37 | MDS 集群内部 |

## 9. 关键代码位置

```
src/
├── mon/              # Monitor 守护进程
│   ├── Monitor.cc    # 核心 Monitor 类
│   ├── Paxos.cc      # Paxos 共识实现
│   ├── Elector.cc    # 领导者选举
│   ├── OSDMonitor.cc # OSD 地图管理
│   └── MonClient.cc  # 客户端连接 Monitor
│
├── osd/              # OSD 守护进程
│   ├── OSD.cc        # 核心 OSD 类
│   ├── PG.cc         # Placement Group
│   ├── PeeringState.cc # PG 状态机
│   ├── PrimaryLogPG.cc # 主 PG 实现
│   ├── ReplicatedBackend.cc # 复制后端
│   └── ECBackend.cc  # 纠删码后端
│
├── mds/              # Metadata Server
│   ├── MDSRank.cc    # 活跃 MDS 核心
│   ├── MDCache.cc    # 分布式元数据缓存
│   ├── Locker.cc     # 分布式锁管理器
│   ├── Server.cc     # 客户端请求处理
│   └── MDBalancer.cc # 负载均衡
│
├── mgr/              # Manager
│   ├── Mgr.cc        # Manager 守护进程
│   └── PyModuleRegistry.cc # Python 模块管理
│
├── rgw/              # Rados Gateway
│   ├── rgw_main.cc   # RGW 入口
│   ├── rgw_rest_s3.cc # S3 协议
│   └── rgw_sal.h     # 存储抽象层
│
├── msg/              # 消息层
│   ├── async/AsyncMessenger.cc # 异步消息
│   ├── async/AsyncConnection.cc # 连接管理
│   └── async/ProtocolV2.cc # V2 协议
│
├── crush/            # CRUSH 算法
│   ├── crush.c       # 核心算法 (C 代码)
│   ├── mapper.c      # 映射函数
│   └── CrushWrapper.cc # C++ 封装
│
├── client/           # 用户态 CephFS 客户端
│   ├── Client.cc     # 核心客户端
│   └── Inode.cc      # Inode 缓存
│
├── osdc/             # OSD 客户端
│   ├── Objecter.cc   # OSD 操作发送
│   ├── ObjectCacher.cc # 对象缓存
│   └── Striper.cc    # 条带化
│
├── librados/         # RADOS API
│   └── RadosClient.cc
│
├── librbd/           # RBD API
│   └── ImageCtx.cc
│
└── common/           # 公共基础设施
    ├── buffer.cc     # 零拷贝缓冲区
    ├── WorkQueue.cc  # 线程池
    └── Finisher.cc   # 异步回调
```

## 10. MDS 核心数据结构

MDS 管理文件系统元数据的核心缓存对象：

| 对象 | 说明 | 定义位置 |
|------|------|----------|
| `MDSCacheObject` | 基类：引用计数、状态管理、等待队列 | `src/mds/MDSCacheObject.h` |
| `CInode` | Inode 缓存：文件/目录属性、快照、capability | `src/mds/CInode.h` |
| `CDentry` | 目录项：文件名到 inode 的映射，三种链接类型 | `src/mds/CDentry.h` |
| `CDir` | 目录分片：支持大目录负载均衡和子树分区 | `src/mds/CDir.h` |

详细说明请参考 [MDS 缓存核心数据结构](../cephfs/mds-cache-objects/SKILL.md)。

## 11. 10年演进 (2015-2025)

| 年份 | 版本 | 特性 | 解决的问题 |
|------|------|------|-----------|
| 2015 | Hammer | Firefly LTS 支持结束 | 企业级稳定性 |
| 2016 | Jewel | BlueStore 引入 | 替代 FileStore，减少 I/O 放大 |
| 2017 | Luminous | BlueStore 默认 | 性能提升 2-3x |
| 2018 | Mimic | RGW 多站点改进 | 跨地域复制 |
| 2019 | Nautilus | mgr 模块生态成熟 | 可观测性提升 |
| 2020 | Octopus | cephadm 部署工具 | 简化运维 |
| 2021 | Pacific | CRUSH straw2 优化 | 数据分布更均匀 |
| 2022 | Quincy | 内核 CephFS fscrypt | 文件加密 |
| 2023 | Reef | 内核 CephFS large folio | 页缓存性能 |
| 2024 | Squid | 内核 CephFS idmapped mounts | 容器化支持 |
| 2025 | TBD | Crimson OSD 成熟 | Seastar 高性能 OSD |

## 12. 与其他特性的关系

```
RADOS (底层对象存储)
  │
  ├── CRUSH: 数据分布算法，被所有客户端使用
  │
  ├── BlueStore: OSD 本地存储引擎
  │
  ├── PG (Placement Group): 数据复制和恢复的基本单位
  │
  └── OSDMap: 集群状态快照，客户端计算数据位置的基础
       │
       ├── MON: 维护和分发 OSDMap
       │
       └── Objecter: 客户端订阅 OSDMap 更新
            │
            ├── librados: 直接使用
            ├── libcephfs: 通过 ObjectCacher 间接使用
            └── librbd: 直接使用

CephFS 子系统关系：
  │
  └── MDS (元数据服务器)
       │
       ├── MDCache: 管理所有缓存对象 (CInode/CDentry/CDir)
       │    ├── inodes: map<vinodeno_t, CInode*>
       │    ├── dentries: map<dentry_key_t, CDentry*>
       │    └── dirs: map<dirfrag_t, CDir*>
       │
       ├── Locker: 分布式锁管理
       │    ├── SimpleLock: 简单锁 (DN 锁、file 锁)
       │    └── ScatterLock: 散列锁 (nest 锁、pg stats)
       │
       ├── Migrator: 子树迁移
       │    └── encode/decode CInode/CDir/CDentry
       │
       └── Server: 处理客户端请求
            └── 调用 MDCache 方法操作缓存对象
```

## 13. 参考文献与资源

### 官方文档
1. **Ceph 架构文档**: https://docs.ceph.com/en/latest/architecture/
2. **CRUSH 论文**: https://ceph.com/assets/pdfs/weil-crush-sc06.pdf

### 学术论文
3. **"Ceph: A Scalable, High-Performance Distributed File System"** - Weil et al., OSDI 2006
   - Ceph 原始设计论文，详细描述 CRUSH 算法和分布式元数据管理
4. **"The CRUSH Algorithm"** - Weil et al., SC 2006
   - CRUSH 算法的完整数学推导和性能分析

### 关键资源
5. **Ceph 开发者文档**: https://docs.ceph.com/en/latest/dev/
6. **Ceph 邮件列表**: ceph-devel@vger.kernel.org
