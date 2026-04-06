---
name: wsl-qemu-ceph-deploy
description: 基于WSL+QEMU虚拟机部署简易Ceph学习环境专家。当用户询问如何在WSL中用QEMU搭建Ceph实验环境、Ceph最小化部署、vstart开发集群、cephadm单节点部署、Ceph数据流验证时调用此技能。
---

# 基于 WSL + QEMU 部署简易 Ceph 学习环境

## 〇、为什么需要本地学习环境？

Ceph 是一个分布式系统，生产环境通常需要多台物理机。但在本地学习时：

```
生产环境:
  3台物理机 × (MON + MGR + OSD) = 至少 3 台机器
  成本高、部署复杂、不适合学习

本地学习的需求:
  - 低成本 (一台笔记本即可)
  - 可重现 (随时销毁重建)
  - 可调试 (能进入任何组件内部)
  - 隔离 (不影响宿主机)
```

**本方案**: 在 WSL2 中运行 QEMU 虚拟机，模拟多节点 Ceph 集群。

---

## 一、整体架构

```
┌─────────────────────────────────────────────────────┐
│                    Windows 宿主机                      │
│  ┌───────────────────────────────────────────────┐   │
│  │              WSL2 (Ubuntu)                      │   │
│  │                                               │   │
│  │  ┌──────────┐  ┌──────────┐  ┌──────────┐    │   │
│  │  │ QEMU VM1 │  │ QEMU VM2 │  │ QEMU VM3 │    │   │
│  │  │ (node1)  │  │ (node2)  │  │ (node3)  │    │   │
│  │  │          │  │          │  │          │    │   │
│  │  │ MON+MGR  │  │  OSD.0   │  │  OSD.1   │    │   │
│  │  │ OSD.2    │  │          │  │          │    │   │
│  │  └──────────┘  └──────────┘  └──────────┘    │   │
│  │       │              │             │          │   │
│  │       └──────────────┴─────────────┘          │   │
│  │              虚拟网络 (tap/bridge)               │   │
│  │                                               │   │
│  │  ┌───────────────────────────────────────┐    │   │
│  │  │  管理工具: cephadm / vstart.sh         │    │   │
│  │  │  客户端: ceph CLI / rados / rbd        │    │   │
│  │  └───────────────────────────────────────┘    │   │
│  └───────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────┘
```

---

## 二、环境准备

### 2.1 WSL2 安装

```bash
# Windows PowerShell (管理员)
wsl --install -d Ubuntu-22.04

# 确认 WSL2 内核
wsl --status
# 默认版本: 2

# 更新 WSL 内核
wsl --update
```

### 2.2 WSL2 中安装 QEMU 和依赖

```bash
# 进入 WSL2
wsl -d Ubuntu-22.04

# 安装 QEMU 和工具链
sudo apt update && sudo apt install -y \
    qemu-system-x86 \
    qemu-utils \
    cloud-image-utils \
    libvirt-daemon-system \
    bridge-utils \
    net-tools \
    openssh-server \
    genisoimage \
    curl \
    wget

# 验证 QEMU
qemu-system-x86_64 --version
```

### 2.3 网络配置 (用户态网络)

WSL2 中创建虚拟网络较复杂，推荐使用 **用户态网络 (user networking)** 简化配置：

```bash
# 创建网桥 (可选，如果需要 VM 间通信)
sudo ip link add name br0 type bridge
sudo ip addr add 192.168.100.1/24 dev br0
sudo ip link set br0 up

# 启用 IP 转发
sudo sysctl -w net.ipv4.ip_forward=1
```

---

## 三、方案选择

根据学习深度和资源，有三种方案：

| 方案 | 适用场景 | 复杂度 | 资源需求 |
|------|---------|--------|---------|
| **A. vstart.sh** | 快速验证代码逻辑 | 最低 | 4GB RAM, 10GB 磁盘 |
| **B. cephadm 单节点** | 学习运维部署 | 中等 | 8GB RAM, 20GB 磁盘 |
| **C. QEMU 多节点** | 模拟真实集群 | 最高 | 16GB RAM, 40GB 磁盘 |

### 推荐学习路线

```
阶段 1: vstart.sh (1-2 天)
  └─ 目标: 理解 Ceph 基本数据流
       - 对象 → PG → OSD 映射
       - 读写 I/O 路径
       - 基本 CLI 操作

阶段 2: cephadm 单节点 (2-3 天)
  └─ 目标: 理解部署和配置
       - cephadm 工作原理
       - ceph.conf 配置项
       - 守护进程管理
       - 故障排查

阶段 3: QEMU 多节点 (3-5 天)
  └─ 目标: 理解分布式特性
       - 多节点通信
       - 故障模拟 (kill OSD/MON)
       - 数据恢复和重平衡
       - CRUSH 规则调优
```

---

## 四、三种方案对比 (从内核存储栈视角)

从你熟悉的 Linux 内核存储栈视角来理解，Ceph 本质上是一个**用户态的分布式存储栈**：

```
内核存储栈（你熟悉的）:
  VFS → 文件系统(ext4) → 块层(bio) → 驱动(nvme) → 硬件(SSD)

Ceph 存储栈（用户态分布式）:
  客户端(VFS/librados) → 网络(TCP/RDMA) → OSD(BlueStore) → 裸盘
```

内核存储栈的所有层都在**同一个内核地址空间**；Ceph 把这些层拆到了**多个主机、多个进程**中，通过网络把它们粘合起来。

### 4.1 角度一：主机结点个数及结点间关系

| 维度 | 方案A: vstart.sh | 方案B: cephadm 单节点 | 方案C: QEMU 多节点 |
|------|-----------------|---------------------|-------------------|
| **主机结点** | 1 个（WSL 本身） | 1 个（WSL 本身） | 3 个（3 个 QEMU VM） |
| **网络** | loopback (127.0.0.1) | 容器网络 (10.89.0.x) | 虚拟网络 (192.168.100.0/24) |
| **故障域** | 无（单机） | 无（单机） | 有（可模拟节点宕机、网络分区） |
| **类比内核** | 像在一个内核模块里跑所有层 | 像用 namespace 隔离的各层 | 像多台机器通过 SAN/NAS 互联 |

#### 方案A (vstart.sh) — 1 个主机，进程间本地通信

```
WSL (唯一主机)
  ├── ceph-mon.a   (127.0.0.1:6789)
  ├── ceph-mgr.x   (127.0.0.1:6800)
  ├── ceph-osd.0   (127.0.0.1:6801)
  ├── ceph-osd.1   (127.0.0.1:6802)
  └── ceph-osd.2   (127.0.0.1:6803)

数据流:
  客户端 → MON (127.0.0.1:6789) 获取 OSDMap
  客户端 → OSD.0 (127.0.0.1:6801) 写入数据
  OSD.0 → OSD.1 (127.0.0.1:6802) 复制数据 (localhost TCP)
  OSD.0 → OSD.2 (127.0.0.1:6803) 复制数据 (localhost TCP)
```

所有进程跑在同一个 WSL 实例中，通过 **loopback TCP** 通信。没有真正的网络延迟，也没有网络故障。类比内核：就像把 VFS、ext4、块层、驱动层拆成 5 个用户态进程，通过 `127.0.0.1` 互相发 bio。

#### 方案B (cephadm 单节点) — 1 个主机，容器隔离

```
WSL (唯一主机)
  ├── container: mon.node1    (容器网络 10.89.0.x)
  ├── container: mgr.node1    (容器网络 10.89.0.x)
  ├── container: osd.0        (容器网络 10.89.0.x)
  ├── container: osd.1        (容器网络 10.89.0.x)
  └── container: osd.2        (容器网络 10.89.0.x)

数据流:
  客户端 → MON (容器IP) 获取 OSDMap
  客户端 → OSD.0 (容器IP) 写入数据
  OSD.0 → OSD.1 (容器IP) 复制数据 (容器间 TCP)
  OSD.0 → OSD.2 (容器IP) 复制数据 (容器间 TCP)
```

每个守护进程在独立的 **Podman 容器** 中，有独立的 network namespace。容器间通过虚拟网桥通信。类比内核：就像用 cgroup + namespace 把 VFS、ext4、块层隔离到不同的容器里，虽然物理上在同一台机器，但网络栈是隔离的。

#### 方案C (QEMU 多节点) — 3 个主机，真实网络

```
node1 (VM1: 192.168.100.10)     node2 (VM2: 192.168.100.11)     node3 (VM3: 192.168.100.12)
  ├── ceph-mon.a                  ├── ceph-mon.b                  ├── ceph-mon.c
  ├── ceph-mgr.x                  ├── ceph-mgr.y                  │
  └── ceph-osd.2                  ├── ceph-osd.0                  └── ceph-osd.1

数据流:
  客户端 → MON (192.168.100.10:6789) 获取 OSDMap
  客户端 → OSD.0 (192.168.100.11) 写入数据
  OSD.0 → OSD.1 (192.168.100.12:6803) 复制数据 (跨 VM TCP)
  OSD.0 → OSD.2 (192.168.100.10:6802) 复制数据 (跨 VM TCP)
```

3 个独立 VM，各有完整的内核、网络栈。数据通过 **虚拟网卡 → WSL 网桥 → 目标 VM** 传输。类比内核：就像 3 台物理机通过以太网互联，每台运行自己的存储栈，通过 RDMA/TCP 同步数据。

### 4.2 角度二：单结点内部进程及关系

从内核存储栈视角，Ceph 的守护进程可以这样类比：

| Ceph 进程 | 内核存储栈类比 | 作用 |
|-----------|--------------|------|
| **ceph-mon** | 超级块 + mount 信息 + 设备拓扑 | 维护集群"元数据真相"（哪些 OSD 在线、数据在哪） |
| **ceph-mgr** | sysfs + debugfs + 性能计数器 | 集群管理、监控、Dashboard |
| **ceph-osd** | 块设备驱动 + 文件系统 + I/O 调度器 | 实际存储数据、处理复制、scrub |
| **ceph-mds** | 内核 dentry/inode 缓存 | CephFS 元数据服务（仅文件存储需要） |

#### 方案A (vstart.sh) — 6 个进程

```
WSL 进程列表:
  ceph-mon    × 1   (mon.a)        — 集群元数据
  ceph-mgr    × 1   (mgr.x)        — 集群管理
  ceph-osd    × 3   (osd.0/1/2)    — 数据存储

进程关系:
  ceph-mon ←→ ceph-mgr    : mgr 向 mon 注册，报告状态
  ceph-mon ←→ ceph-osd    : mon 下发 OSDMap，osd 上报心跳
  ceph-osd ←→ ceph-osd    : 数据复制、心跳、peering
  ceph-mgr ←→ ceph-osd    : 收集性能指标

类比内核:
  mon  ≈ 维护全局 mount table + 设备拓扑的管理进程
  mgr  ≈ 收集 /sys/block/* 统计信息的监控进程
  osd  ≈ 每块盘一个 nvme 驱动进程，互相之间通过网络同步数据
```

#### 方案B (cephadm 单节点) — 6 个容器进程

```
WSL 容器列表 (每个容器 1 个主进程):
  podman run ceph-mon    × 1   — 容器内运行 ceph-mon
  podman run ceph-mgr    × 1   — 容器内运行 ceph-mgr
  podman run ceph-osd    × 3   — 容器内运行 ceph-osd

进程关系: 与方案A相同，但多了容器边界:
  - 每个进程有独立的 /etc/ceph/ceph.conf
  - 每个进程有独立的 keyring (认证密钥)
  - 数据目录通过 volume mount 映射到宿主机
  - 进程间通过容器网络通信 (有 NAT)

类比内核:
  与方案A相同，但每个"驱动进程"跑在独立的 mount namespace 中
  类似于用 chroot 隔离的多个驱动实例
```

#### 方案C (QEMU 多节点) — 每 VM 2-3 个进程

```
node1 (VM1):
  ceph-mon.a    × 1   — MON (集群元数据)
  ceph-mgr.x    × 1   — MGR (集群管理)
  ceph-osd.2    × 1   — OSD (存储数据)

node2 (VM2):
  ceph-mon.b    × 1   — MON (集群元数据)
  ceph-mgr.y    × 1   — MGR (standby)
  ceph-osd.0    × 1   — OSD (存储数据)

node3 (VM3):
  ceph-mon.c    × 1   — MON (集群元数据)
  ceph-osd.1    × 1   — OSD (存储数据)

进程关系:
  跨 VM:
    mon.a ←→ mon.b ←→ mon.c   : Paxos 共识 (选举 leader，同步 monmap/osdmap)
    osd.0 ←→ osd.1 ←→ osd.2   : 数据复制、心跳、peering
    mon   ←→ osd              : 下发 OSDMap，上报心跳

  VM 内:
    mon ←→ mgr   : mgr 向本地 mon 注册
    mon ←→ osd   : 本地通信 (同 VM 内 TCP)

类比内核:
  3 台机器，每台有自己的存储驱动 (osd)
  3 台机器各有一个拓扑管理器 (mon)，通过 Paxos 达成一致
  数据写入时: 客户端 → 主 osd → 通过网络复制到另 2 台的 osd
  这类似于 3 台服务器通过 SAN 做同步复制，但 Ceph 是在用户态用 TCP 做的
```

### 4.3 核心差异总结

| 维度 | 方案A | 方案B | 方案C |
|------|-------|-------|-------|
| **进程隔离** | 无（同一用户空间） | 容器隔离（namespace） | VM 隔离（完整内核） |
| **网络** | loopback | 容器网桥 | 虚拟网卡 |
| **MON 共识** | 1 个 MON（无共识） | 1 个 MON（无共识） | 3 个 MON（Paxos 共识） |
| **OSD 复制** | 本地 TCP 复制 | 容器 TCP 复制 | 跨 VM TCP 复制 |
| **可模拟故障** | 只能 kill 进程 | 只能 stop 容器 | 可关 VM、断网、kill 进程 |

---

## 五、方案 A: vstart.sh (最快上手)

vstart.sh 是 Ceph 源码树中的开发集群脚本，最适合快速理解数据流。

### 5.1 编译 Ceph

```bash
cd /home/i_ingfeng/ceph

# 安装编译依赖
sudo apt install -y \
    git cmake gcc g++ make \
    libboost-all-dev libfmt-dev libaio-dev \
    libssl-dev libcurl4-openssl-dev \
    libedit-dev libsnappy-dev liblz4-dev \
    libblkid-dev libudev-dev \
    python3-dev python3-pip python3-venv \
    libgoogle-perftools-dev \
    libibverbs-dev librdmacm-dev \
    ninja-build

# 创建构建目录
mkdir build && cd build

# 配置 (包含 MDS 以支持 CephFS)
cmake .. \
    -DCMAKE_BUILD_TYPE=RelWithDebInfo \
    -DWITH_RADOSGW=OFF \
    -DWITH_MGR=ON \
    -DWITH_MDS=ON \
    -DWITH_KVS=OFF \
    -DWITH_SPDK=OFF \
    -DWITH_JAEGER=OFF \
    -DWITH_SYSTEMD=OFF \
    -G Ninja

# 编译 (使用 -j 并行)
ninja -j$(nproc)
```

### 5.2 启动开发集群

```bash
cd /home/i_ingfeng/ceph/build

# 启动最小集群 (1 MON, 1 MGR, 3 OSD, 1 MDS)
../src/vstart.sh -n -d \
    -m 127.0.0.1 \
    --mds \
    --without-dashboard \
    --bluestore

# 参数说明:
#   -n        : 不重新编译
#   -d        : 使用 debug 模式
#   -m        : MON 地址
#   --mds     : 启动 MDS 守护进程 (支持 CephFS)
#   --bluestore : 使用 BlueStore (默认)
```

### 5.3 验证集群

```bash
# 设置环境变量
export PATH=$PWD/bin:$PATH
export CEPH_CONF=$PWD/ceph.conf

# 查看集群状态
ceph -s

# 预期输出:
#   cluster:
#     id:     xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx
#     health: HEALTH_OK
#   services:
#     mon: 1 daemons, quorum a
#     mgr: x(active)
#     mds: cephfs:1 {0=a=up:active}
#     osd: 3 osds: 3 up, 3 in
#   data:
#     pools:   1 pools, 1 pgs
#     objects: 0 objects, 0 B
#     usage:   3 OSDs used, X GiB used
#     pg_state:  1 active+clean
```

### 5.4 基本数据流验证

```bash
# 1. 创建测试 pool
ceph osd pool create testpool 8 8

# 2. 写入对象
echo "hello ceph" | rados -p testpool put myobject -

# 3. 查看对象位置
ceph osd map testpool myobject
# 输出: osdmap eXX pool 'testpool' (X) object 'myobject' -> pg X.XXXXXXXX -> up [0,1,2] acting [0,1,2]

# 4. 读取对象
rados -p testpool get myobject -
# 输出: hello ceph

# 5. 删除对象
rados -p testpool rm myobject
```

### 5.5 观察数据流

```bash
# 实时查看集群事件
ceph -w

# 查看 PG 状态
ceph pg dump | head -20

# 查看 OSD 树
ceph osd tree

# 查看 OSD 详细信息
ceph osd dump

# 停止一个 OSD 观察恢复
../src/stop-osd.sh 1
ceph -w
# 观察 PG 状态变化: active+clean → active+degraded → active+clean (恢复后)
../src/run-osd.sh 1
```

### 5.6 CephFS 客户端流程 (方案A)

**为什么需要 MDS？** Ceph 提供三种存储接口，MDS 仅用于 CephFS（文件存储）：

```
Ceph 三种接口:
  ├── RBD (块存储)  → 不需要 MDS (直接对象读写)
  ├── RGW (对象存储) → 不需要 MDS (REST API → 对象)
  └── CephFS (文件存储) → 需要 MDS (管理目录树/inode)
```

MDS 在内核存储栈视角的类比：

```
内核文件系统 (ext4):
  VFS (open/read/write)
    → ext4_lookup()  (查找 dentry)
    → ext4_iget()    (加载 inode)
    → ext4_file_read() (读数据)

  所有元数据 (dentry/inode) 都在本地磁盘的 inode table 里
  所有缓存 (dcache/icache) 都在本地内核内存里

CephFS:
  VFS (open/read/write)
    → CephFS 客户端 (内核模块 fs/ceph/ 或 ceph-fuse)
    → 向 MDS 请求元数据 (lookup/getattr)
    → MDS 返回 inode/dentry 信息
    → 客户端缓存元数据，后续操作本地完成
    → 数据 I/O 直接发 OSD (不经过 MDS)

  元数据 (dentry/inode) 在 MDS 进程的内存里
  缓存分布在 MDS + 所有客户端
```

**MDS 的核心职责：**

| 职责 | 内核类比 | MDS 实现 |
|------|---------|---------|
| **目录树管理** | VFS dcache + 磁盘目录块 | 内存中的目录树，分片 (fragment) 存储 |
| **Inode 管理** | VFS icache + 磁盘 inode table | 内存中的 inode 对象，带版本号 |
| **分布式缓存一致性** | 本地缓存无需一致性 | **Capability 系统** (核心机制) |
| **动态子树分区** | 单文件系统单挂载点 | 目录子树可动态迁移到不同 MDS |
| **快照** | 快照子卷 | 快照元数据管理 |

**在方案A中启用 CephFS：**

```bash
# 1. 创建 CephFS 需要的 metadata pool 和 data pool
ceph osd pool create cephfs_metadata 8 8
ceph osd pool create cephfs_data 8 8

# 2. 创建 CephFS 文件系统 (自动关联 MDS)
ceph fs new myfs cephfs_metadata cephfs_data

# 3. 验证 MDS 状态
ceph fs status
# 预期: myfs - 1 clients, 1 MDS (a: active)

# 4. 挂载 CephFS (内核客户端)
sudo mkdir -p /mnt/cephfs
sudo mount -t ceph 127.0.0.1:6789:/ /mnt/cephfs \
    -o name=admin,secret=$(ceph auth get-key client.admin)

# 5. 验证挂载
df -h /mnt/cephfs
echo "hello cephfs" > /mnt/cephfs/test.txt
cat /mnt/cephfs/test.txt
```

**CephFS 数据流详解 (方案A)：**

```
mount -t ceph 127.0.0.1:6789:/ /mnt/cephfs

阶段 1: 挂载 (仅一次)
  客户端 → MON (127.0.0.1:6789)
    获取: monmap, osdmap, mdsmap
  客户端 → MDS (127.0.0.1:6804)
    建立会话，获取根 inode (CEPH_INO_ROOT = 1)
    MDS 授予 Capability (FILE_CACHE, FILE_RD)

阶段 2: 元数据操作 (经过 MDS)
  open("/mnt/cephfs/foo")
    → 客户端检查本地缓存是否有 foo 的 inode
    → 如果没有: 向 MDS 发送 lookup 请求 (127.0.0.1:6804)
    → MDS 返回 inode 信息 + caps
    → 客户端缓存 inode

  ls /mnt/cephfs/
    → 向 MDS 发送 readdir 请求
    → MDS 返回目录条目列表
    → 客户端缓存 dentry

阶段 3: 数据 I/O (不经过 MDS!)
  read("/mnt/cephfs/foo")
    → 客户端从 inode 获取文件 layout (条带化参数)
    → 计算文件偏移对应的 RADOS 对象
    → 通过 CRUSH 计算对象在哪些 OSD 上
    → 直接向 OSD 发送读请求 (127.0.0.1:6801/6802/6803)

  write("/mnt/cephfs/foo")
    → 计算对象位置
    → 直接向 OSD 发送写请求 (不经过 MDS!)
    → 如果写改变了文件大小: 向 MDS 发送 cap flush 更新 inode size

阶段 4: 缓存一致性 (Capability 系统)
  场景: 客户端 A 和 B 同时访问同一文件

  1. 客户端 A 打开文件 → MDS 授予 FILE_CACHE cap
  2. 客户端 A 本地缓存元数据
  3. 客户端 B 也要打开同一文件 → MDS 检测到冲突
  4. MDS 向客户端 A 发送 cap revoke
  5. 客户端 A 刷盘脏数据、失效缓存 → 回复 MDS
  6. MDS 将 cap 授予客户端 B

  这就像内核里的 inode 锁 + 页缓存回写，但跨网络、跨客户端
```

**停止集群：**

```bash
../src/stop.sh
```

---

## 六、方案 B: cephadm 单节点部署

### 6.1 安装 Ceph 包

```bash
# 添加 Ceph 仓库密钥和源
curl -fsSL https://download.ceph.com/keys/release.asc | sudo gpg --dearmor -o /etc/apt/keyrings/ceph-archive-keyring.gpg
echo "deb [signed-by=/etc/apt/keyrings/ceph-archive-keyring.gpg] https://download.ceph.com/debian-quincy/ $(lsb_release -sc) main" | sudo tee /etc/apt/sources.list.d/ceph.list

sudo apt update
sudo apt install -y cephadm
```

### 6.2 Bootstrap 单节点集群

```bash
# 获取 WSL2 IP
ip addr show eth0 | grep 'inet ' | awk '{print $2}' | cut -d/ -f1
# 假设输出: 172.24.80.10

# Bootstrap (单节点模式)
sudo cephadm bootstrap --mon-ip 172.24.80.10 --single-host-defaults

# --single-host-defaults 自动设置:
#   osd_crush_chooseleaf_type = 0  (允许同主机多 OSD)
#   osd_pool_default_size = 2      (默认 2 副本)
#   mgr_standby_modules = False    (禁用 standby 模块)
```

### 6.3 添加 OSD

```bash
# 创建文件模拟磁盘 (每个 10GB)
for i in 0 1 2; do
    sudo truncate -s 10G /var/local/osd${i}.img
    sudo losetup /dev/loop${i} /var/local/osd${i}.img
done

# 使用 loop 设备创建 OSD
sudo ceph orch daemon add osd /dev/loop0:/dev/loop1:/dev/loop2

# 或使用目录模式 (更简单，但不推荐生产)
sudo mkdir -p /var/local/osd{0,1,2}
sudo ceph orch daemon add osd node1:/var/local/osd0
sudo ceph orch daemon add osd node1:/var/local/osd1
sudo ceph orch daemon add osd node1:/var/local/osd2
```

### 6.4 验证

```bash
# 使用 cephadm shell 进入 Ceph 环境
sudo cephadm shell

# 在 shell 内:
ceph -s
ceph osd tree
ceph osd df
```

### 6.5 基本操作

```bash
# 创建 pool
ceph osd pool create mypool 16 16

# 写入数据
rados -p mypool put testfile /etc/hostname

# 查看对象分布
ceph osd map mypool testfile

# 模拟 OSD 故障
ceph osd out 0
ceph osd down 0

# 观察数据迁移
ceph -w

# 恢复 OSD
ceph osd in 0
```

### 6.6 CephFS 客户端流程 (方案B)

**在方案B中启用 CephFS：**

```bash
sudo cephadm shell

# 1. 创建 CephFS 文件系统 (cephadm 会自动启动 MDS)
ceph fs volume create myfs

# 2. 验证 MDS 已启动
ceph fs status
# 预期: myfs - 0 clients, 1 MDS (node1: active)

ceph orch ls
# 预期: 看到 mds.myfs 服务

# 3. 获取 admin key
ceph auth get-key client.admin > /tmp/ceph.admin.key
```

**在 WSL 中挂载 CephFS：**

```bash
# 1. 确保 WSL 安装了 ceph-common
sudo apt install -y ceph-common

# 2. 复制 keyring 到 WSL
sudo cephadm shell -- ceph auth get-key client.admin | sudo tee /etc/ceph/ceph.client.admin.keyring

# 3. 获取 MON 地址 (容器 IP)
sudo cephadm shell -- ceph mon dump | grep mon

# 4. 挂载 CephFS
sudo mkdir -p /mnt/cephfs
sudo mount -t ceph <MON_CONTAINER_IP>:6789:/ /mnt/cephfs \
    -o name=admin,secret=$(cat /etc/ceph/ceph.client.admin.keyring)

# 5. 验证
echo "hello from cephadm" > /mnt/cephfs/test.txt
cat /mnt/cephfs/test.txt
```

**CephFS 数据流详解 (方案B)：**

```
                    方案B CephFS 数据流

WSL 宿主机 (客户端)
  │
  │ mount -t ceph <MON_IP>:6789:/ /mnt/cephfs
  │
  ├─→ MON 容器 (10.89.0.x:6789)
  │     获取 monmap, osdmap, mdsmap
  │
  ├─→ MDS 容器 (10.89.0.y:6800)
  │     建立会话，获取根 inode
  │     授予 Capability
  │
  └─→ OSD 容器 (10.89.0.z:6801/6802/6803)
        直接数据 I/O (不经过 MDS)

容器网络通信:
  客户端 → MON: 容器网桥 (br-podman) TCP
  客户端 → MDS: 容器网桥 (br-podman) TCP
  客户端 → OSD: 容器网桥 (br-podman) TCP
  OSD → OSD:   容器网桥 (br-podman) TCP (数据复制)

与方案A的差异:
  - 网络: 容器网桥 vs loopback
  - 隔离: 每个进程独立 namespace
  - 认证: 每个容器独立 keyring
  - 但数据流逻辑完全相同
```

**关键理解：MDS 只在元数据操作时参与**

```
元数据操作 (经过 MDS):
  open()    → MDS lookup
  getattr() → MDS getattr
  readdir() → MDS readdir
  rename()  → MDS rename
  unlink()  → MDS unlink
  mkdir()   → MDS mkdir

数据 I/O (不经过 MDS):
  read()    → 客户端 → OSD (直接)
  write()   → 客户端 → OSD (直接)
  mmap()    → 客户端 → OSD (直接)
```

---

## 七、方案 C: QEMU 多节点部署

### 7.1 准备云镜像

```bash
# 下载 Ubuntu Cloud Image
cd /home/i_ingfeng
mkdir -p ceph-cluster/images
cd ceph-cluster/images

wget https://cloud-images.ubuntu.com/jammy/current/jammy-server-cloudimg-amd64.img

# 为每个节点创建磁盘副本
for i in 1 2 3; do
    qemu-img create -f qcow2 -b jammy-server-cloudimg-amd64.img -F qcow2 node${i}.qcow2 20G
done
```

### 7.2 创建 cloud-init 配置

```bash
# 为每个节点创建 user-data
for i in 1 2 3; do
    cat > node${i}-user-data << EOF
#cloud-config
hostname: node${i}
fqdn: node${i}.ceph.local
users:
  - name: ceph
    sudo: ALL=(ALL) NOPASSWD:ALL
    shell: /bin/bash
    ssh_authorized_keys:
      - $(cat ~/.ssh/id_rsa.pub 2>/dev/null || echo "ssh-ed25519 YOUR_KEY_HERE")
ssh_pwauth: true
chpasswd:
  expire: false
  list: |
    ceph:ceph123
    root:ceph123
runcmd:
  - apt-get update
  - apt-get install -y openssh-server net-tools iproute2
  - systemctl enable ssh
  - systemctl start ssh
  - echo "192.168.100.10 node1" >> /etc/hosts
  - echo "192.168.100.11 node2" >> /etc/hosts
  - echo "192.168.100.12 node3" >> /etc/hosts
EOF
done

# 创建 meta-data
for i in 1 2 3; do
    cat > node${i}-meta-data << EOF
instance-id: node${i}
local-hostname: node${i}
EOF
done

# 生成 cloud-init ISO
for i in 1 2 3; do
    genisoimage -output node${i}-cidata.iso -volid cidata -joliet -rock node${i}-user-data node${i}-meta-data
done
```

### 7.3 启动 QEMU 虚拟机

```bash
# 创建启动脚本
cat > start-nodes.sh << 'SCRIPT'
#!/bin/bash

NODES=(
    "node1 192.168.100.10 2221 4096"
    "node2 192.168.100.11 2222 2048"
    "node3 192.168.100.12 2223 2048"
)

for entry in "${NODES[@]}"; do
    read -r name ip port mem <<< "$entry"

    qemu-system-x86_64 \
        -name ${name} \
        -m ${mem} \
        -smp 2 \
        -hda images/${name}.qcow2 \
        -cdrom images/${name}-cidata.iso \
        -netdev user,id=net0,hostfwd=tcp::${port}-:22 \
        -device virtio-net-pci,netdev=net0 \
        -enable-kvm \
        -cpu host \
        -daemonize \
        -pidfile ${name}.pid \
        -display none \
        -serial mon:stdio

    echo "Started ${name} (SSH port: ${port})"
done
SCRIPT

chmod +x start-nodes.sh
./start-nodes.sh
```

### 7.4 连接节点

```bash
# 等待 cloud-init 完成 (约 1-2 分钟)
sleep 120

# SSH 连接到各节点
ssh -p 2221 -o StrictHostKeyChecking=no ceph@localhost  # node1
ssh -p 2222 -o StrictHostKeyChecking=no ceph@localhost  # node2
ssh -p 2223 -o StrictHostKeyChecking=no ceph@localhost  # node3
```

### 7.5 在 QEMU 集群上部署 Ceph

```bash
# 在 node1 上安装 cephadm
ssh -p 2221 ceph@localhost
sudo apt update && sudo apt install -y cephadm

# Bootstrap MON
sudo cephadm bootstrap --mon-ip 192.168.100.10 --single-host-defaults

# 复制 SSH key 到其他节点
sudo ceph cephadm get-pub-key > ~/ceph.pub
ssh-copy-id -f -i ~/ceph.pub root@node2
ssh-copy-id -f -i ~/ceph.pub root@node3

# 添加主机到集群
sudo ceph orch host add node2 192.168.100.11
sudo ceph orch host add node3 192.168.100.12

# 部署 MON 到其他节点
sudo ceph orch apply mon "node1,node2,node3"

# 部署 MGR
sudo ceph orch apply mgr "node1,node2"

# 在每个节点上创建 OSD (使用目录模式模拟)
for node in node1 node2 node3; do
    ssh ${node} "sudo mkdir -p /var/local/osd0"
    sudo ceph orch daemon add osd ${node}:/var/local/osd0
done

# 验证
sudo ceph -s
sudo ceph orch ls
sudo ceph orch ps
```

### 7.6 CephFS 客户端流程 (方案C)

**在方案C中启用 CephFS：**

```bash
# SSH 到 node1
ssh -p 2221 ceph@localhost

# 1. 部署 MDS (在 node1 上)
sudo ceph orch apply mds myfs --placement="1 node1"

# 2. 创建 CephFS 需要的 pools
sudo ceph osd pool create cephfs_metadata 16 16
sudo ceph osd pool create cephfs_data 16 16

# 3. 创建 CephFS 文件系统
sudo ceph fs new myfs cephfs_metadata cephfs_data

# 4. 验证 MDS 状态
sudo ceph fs status
# 预期: myfs - 0 clients, 1 MDS (node1: active)

sudo ceph orch ls
# 预期: 看到 mds.myfs 服务
```

**从 WSL 挂载 CephFS (跨 VM 访问)：**

```bash
# 1. 确保 WSL 安装了 ceph-common
sudo apt install -y ceph-common

# 2. 从 node1 获取 admin key
ssh -p 2221 ceph@localhost "sudo ceph auth get-key client.admin" > /tmp/ceph.admin.key

# 3. 挂载 CephFS (通过 node1 的 MON)
sudo mkdir -p /mnt/cephfs
sudo mount -t ceph 192.168.100.10:6789:/ /mnt/cephfs \
    -o name=admin,secret=$(cat /tmp/ceph.admin.key)

# 4. 验证
echo "hello from QEMU cluster" > /mnt/cephfs/test.txt
cat /mnt/cephfs/test.txt
```

**CephFS 数据流详解 (方案C)：**

```
                    方案C CephFS 数据流

WSL 宿主机 (客户端: 172.24.80.10)
  │
  │ mount -t ceph 192.168.100.10:6789:/ /mnt/cephfs
  │
  ├─→ node1 VM (192.168.100.10)
  │     ├── ceph-mon.a   : 获取 monmap, osdmap, mdsmap
  │     ├── ceph-mgr.x   : 集群管理
  │     └── ceph-mds.a   : 元数据服务 (lookup, getattr, cap grant)
  │
  ├─→ node2 VM (192.168.100.11)
  │     ├── ceph-mon.b   : MON 备份 (Paxos 共识)
  │     ├── ceph-mgr.y   : MGR standby
  │     └── ceph-osd.0   : 数据 I/O (直接)
  │
  └─→ node3 VM (192.168.100.12)
        ├── ceph-mon.c   : MON 备份 (Paxos 共识)
        └── ceph-osd.1   : 数据 I/O (直接)

跨 VM 网络通信:
  客户端 → MON (node1): WSL → 虚拟网桥 → node1 virtio-net (TCP 6789)
  客户端 → MDS (node1): WSL → 虚拟网桥 → node1 virtio-net (TCP 6804)
  客户端 → OSD (node2): WSL → 虚拟网桥 → node2 virtio-net (TCP 6801)
  客户端 → OSD (node3): WSL → 虚拟网桥 → node3 virtio-net (TCP 6802)

  OSD 间复制:
    OSD.0 (node2) → OSD.1 (node3): node2 → 虚拟网桥 → node3 (TCP)
    OSD.0 (node2) → OSD.2 (node1): node2 → 虚拟网桥 → node1 (TCP)

  MON 间 Paxos:
    mon.a (node1) ↔ mon.b (node2) ↔ mon.c (node3): 跨 VM TCP

与方案A/B的差异:
  - 网络: 真实虚拟网络 vs loopback/容器网桥
  - 延迟: 有真实网络延迟 (ms 级)
  - 故障域: 可模拟 VM 宕机、网络分区
  - Paxos: 3 个 MON 真实共识 (方案A/B 只有 1 个 MON)
  - 数据复制: 跨 VM TCP 复制 (最接近生产环境)
```

**模拟真实故障场景 (仅方案C能做到)：**

```bash
# 场景 1: MDS 节点宕机
# 在 WSL 中关闭 node1 VM
kill $(cat node1.pid)

# 观察:
# - MDS 变为 down
# - 已挂载的 CephFS 客户端: 元数据操作卡住 (lookup/open 等)
# - 数据 I/O (read/write) 仍然正常 (不经过 MDS)
# - MON quorum 仍然正常 (2/3)

# 恢复 node1
./start-nodes.sh  # 重新启动 node1

# 场景 2: 网络分区 (模拟)
# 在 node2 上阻断到 node3 的网络
ssh -p 2222 ceph@localhost
sudo iptables -A OUTPUT -d 192.168.100.12 -j DROP

# 观察:
# - OSD.0 (node2) 和 OSD.1 (node3) 之间的心跳断开
# - PG 状态变化: active+clean → active+degraded
# - MON quorum 仍然正常 (node1+node2 或 node1+node3)
# - 数据仍然可读写 (通过剩余的 OSD)

# 恢复网络
sudo iptables -D OUTPUT -d 192.168.100.12 -j DROP
```

---

## 八、学习路线详解

### 阶段 1: 基本数据流 (vstart.sh)

```
Day 1: 理解对象存储模型
  ├── 对象 (Object): 数据的基本单位
  │   ├── 对象名 (OID)
  │   ├── 数据 (bufferlist)
  │   └── 属性 (xattrs)
  │
  ├── Pool: 对象的逻辑分组
  │   ├── pg_num: PG 数量
  │   ├── size: 副本数
  │   └── min_size: 最小可写副本数
  │
  └── PG (Placement Group): 对象的集合
       ├── PG ID = hash(object_name) % pg_num
       └── PG → OSD 映射通过 CRUSH 算法

Day 2: 验证读写路径
  ├── 写入路径:
  │   1. 客户端计算对象对应的 PG
  │   2. 通过 CRUSH 找到 PG 的 acting OSD 集合
  │   3. 写入 primary OSD
  │   4. Primary 复制到 replica OSDs
  │   5. 所有 replica 确认后回复客户端
  │
  └── 读取路径:
      1. 客户端计算对象对应的 PG
      2. 通过 CRUSH 找到 OSD
      3. 从 OSD 读取数据

Day 3: 观察故障恢复
  ├── 停止一个 OSD
  │   ├── 观察 PG 状态: active+clean → active+degraded
  │   └── 验证数据仍然可读
  │
  ├── 恢复 OSD
  │   ├── 观察 PG 状态: degraded → recovering → active+clean
  │   └── 观察数据同步过程
  │
  └── 使用 ceph -w 观察事件流
```

### 阶段 2: 配置和原理 (cephadm 单节点)

```
Day 4-5: 理解 ceph.conf 配置
  ├── [global] 段:
  │   ├── fsid: 集群唯一标识
  │   ├── mon_host: MON 地址列表
  │   ├── public_network: 客户端网络
  │   └── cluster_network: 内部复制网络
  │
  ├── [mon] 段:
  │   ├── mon_allow_pool_delete: 允许删除 pool
  │   └── mon_osd_full_ratio: OSD 满水位
  │
  ├── [osd] 段:
  │   ├── osd_pool_default_size: 默认副本数
  │   ├── osd_pool_default_pg_num: 默认 PG 数
  │   ├── osd_crush_chooseleaf_type: CRUSH 故障域
  │   └── osd_max_scrubs: 最大并发 scrub
  │
  └── [client] 段:
      ├── rbd_cache: RBD 缓存
      └── rbd_cache_size: 缓存大小

Day 6-7: 理解守护进程
  ├── ceph-mon: 集群状态维护
  │   ├── monmap: MON 成员信息
  │   ├── osdmap: OSD 拓扑
  │   └── Paxos: 共识算法
  │
  ├── ceph-mgr: 集群管理
  │   ├── Dashboard: Web UI
  │   ├── Prometheus: 指标导出
  │   └── 插件系统
  │
  ├── ceph-osd: 数据存储
  │   ├── BlueStore: 存储引擎
  │   ├── Journal/WAL: 预写日志
  │   └── Scrub: 数据校验
  │
  └── ceph-mds: 元数据服务 (CephFS 需要)
      ├── 分布式缓存
      ├── Capability 系统
      └── 动态子树分区
```

### 阶段 3: 分布式特性 (QEMU 多节点)

```
Day 8-9: 故障模拟
  ├── MON 故障
  │   ├── 停止一个 MON
  │   ├── 观察 quorum 变化
  │   └─- 恢复 MON
  │
  ├── OSD 故障
  │   ├── 停止一个 OSD
  │   ├── 观察 PG 降级
  │   ├── 验证数据可读性
  │   └── 观察自动恢复
  │
  ├── MDS 故障
  │   ├── 停止 MDS
  │   ├── 观察元数据操作卡住
  │   ├── 验证数据 I/O 仍然正常
  │   └── 恢复 MDS
  │
  └── 网络分区模拟
      ├── 使用 iptables 阻断节点间通信
      ├── 观察 split-brain 处理
      └── 恢复网络

Day 10-11: CRUSH 调优
  ├── 查看当前 CRUSH map
  │   ├── ceph osd crush dump
  │   └── 理解 bucket 层次
  │
  ├── 修改故障域
  │   ├── 从 host 改为 rack
  │   └── 观察数据迁移
  │
  ├── 权重调整
  │   ├── ceph osd crush reweight
  │   └── 观察数据重新分布
  │
  └── 自定义规则
      ├── 创建 SSD-only pool
      └── 创建 SSD+HDD 分层 pool

Day 12: 性能测试
  ├── rados bench 测试
  │   ├── rados bench -p testpool 60 write
  │   ├── rados bench -p testpool 60 seq
  │   └── rados bench -p testpool 60 rand
  │
  ├── rbd 性能测试
  │   ├── rbd create testimg --size 1G
  │   ├── rbd map testimg
  │   └── fio 测试
  │
  ├── CephFS 性能测试
  │   ├── mount -t ceph ...
  │   ├── fio --directory=/mnt/cephfs ...
  │   └── mdtest (元数据性能)
  │
  └── 监控指标
      ├── ceph osd perf
      ├── ceph df detail
      └── iostat / sar
```

---

## 九、常用调试命令速查

### 集群状态

```bash
# 基本状态
ceph -s                    # 集群状态
ceph health detail         # 详细健康
ceph versions              # 各组件版本

# 监控状态
ceph mon dump              # MON 映射
ceph mon stat              # MON 状态
ceph quorum_status         # 法定人数状态

# OSD 状态
ceph osd status            # OSD 状态
ceph osd tree              # OSD 树
ceph osd df                # OSD 磁盘使用
ceph osd perf              # OSD 性能

# PG 状态
ceph pg stat               # PG 统计
ceph pg dump               # PG 详细信息
ceph pg dump_stuck         # 卡住的 PG

# MDS 状态
ceph fs status             # CephFS 状态
ceph fs dump               # CephFS 详细信息
ceph mds stat              # MDS 状态
```

### 数据操作

```bash
# Pool 操作
ceph osd pool ls           # 列出 pool
ceph osd pool create test 8 8  # 创建 pool
ceph osd pool rm test test --yes-i-really-really-mean-it  # 删除 pool

# 对象操作
rados -p testpool put obj file   # 上传对象
rados -p testpool get obj file   # 下载对象
rados -p testpool rm obj         # 删除对象
rados -p testpool ls             # 列出对象

# 对象位置
ceph osd map testpool obj    # 查看对象映射
```

### 故障排查

```bash
# 查看日志
cephadm logs --name mon.node1
cephadm logs --name osd.0

# 进入容器
cephadm enter --name mon.node1

# 查看守护进程
ceph orch ps
ceph orch ls

# Admin socket
ceph --admin-daemon /var/run/ceph/ceph-mon.node1.asok config show
```

---

## 十、关键代码位置

| 功能 | 文件 | 说明 |
|------|------|------|
| vstart.sh | src/vstart.sh | 开发集群启动脚本 |
| cephadm | src/cephadm/cephadm.py | 部署工具主文件 |
| cephadm 常量 | src/cephadm/cephadmlib/constants.py | 默认路径和常量 |
| cephadm 部署 | src/cephadm/cephadmlib/deploy.py | 部署类型定义 |
| 守护进程定义 | src/cephadm/cephadmlib/daemons/ceph.py | MON/MGR/OSD 定义 |
| 示例配置 | src/sample.ceph.conf | 示例 ceph.conf |
| 架构文档 | doc/architecture.rst | 架构概述 |
| 安装文档 | doc/cephadm/install.rst | cephadm 安装指南 |
| 故障排查 | doc/cephadm/troubleshooting.rst | 故障排查指南 |
| 配置参考 | doc/rados/configuration/ceph-conf.rst | 配置参考 |
| PG 故障排查 | doc/rados/troubleshooting/troubleshooting-pg.rst | PG 问题排查 |

---

## 十一、常见问题与陷阱

### Q1: WSL2 中 QEMU 性能很差怎么办？

**A**:
1. 确保启用了 KVM: `ls -la /dev/kvm`
2. WSL2 默认不支持 KVM，需要使用 WSLg 或 Windows 11 的嵌套虚拟化
3. 替代方案: 使用 Docker 容器代替 QEMU VM (`docker run` 多个 Ubuntu 容器)
4. 最简单方案: 直接用 vstart.sh，无需 VM

### Q2: 单节点集群 PG 一直是 active+remapped 怎么办？

**A**: 检查 `osd_crush_chooseleaf_type`:
```bash
# 单节点必须设置为 0
ceph config set global osd_crush_chooseleaf_type 0
```

### Q3: OSD 启动失败，报 "no space left on device"？

**A**: BlueStore 需要足够的磁盘空间。确保:
- 每个 OSD 至少 1GB 可用空间
- 文件系统支持扩展属性 (xattr)
- 使用 `df -h` 检查磁盘空间

### Q4: 如何快速重建实验环境？

**A**: 使用脚本自动化:
```bash
# 销毁
./stop-nodes.sh
rm -rf ceph-cluster/images/node*.qcow2

# 重建
./create-nodes.sh
./start-nodes.sh
sleep 120
# 重新部署 Ceph
```

### Q5: vstart.sh 和 cephadm 的区别？

**A**:
- **vstart.sh**: 开发工具，不依赖 systemd/container，直接启动进程。适合调试代码。
- **cephadm**: 生产部署工具，使用容器 (Podman/Docker) 管理守护进程。适合学习运维。

### Q6: CephFS 挂载后元数据操作卡住但数据 I/O 正常？

**A**: 这是 MDS 故障的典型症状。MDS 只负责元数据操作 (lookup/open/getattr/readdir)，数据 I/O (read/write) 直接走客户端 → OSD。检查 MDS 状态：
```bash
ceph mds stat
ceph fs status
```

### Q7: 为什么 CephFS 需要 MDS 而 RBD/RGW 不需要？

**A**:
- **RBD**: 块设备接口，只有读写操作，直接映射到对象 → 不需要元数据服务
- **RGW**: 对象存储接口，对象名就是 key，直接计算位置 → 不需要元数据服务
- **CephFS**: 文件系统接口，有目录树、inode、权限、硬链接等复杂元数据 → 需要 MDS 管理
