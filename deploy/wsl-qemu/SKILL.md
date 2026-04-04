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

## 四、方案 A: vstart.sh (最快上手)

vstart.sh 是 Ceph 源码树中的开发集群脚本，最适合快速理解数据流。

### 4.1 编译 Ceph

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

# 配置 (最小化编译，仅编译必要组件)
cmake .. \
    -DCMAKE_BUILD_TYPE=RelWithDebInfo \
    -DWITH_RADOSGW=OFF \
    -DWITH_MGR=ON \
    -DWITH_MDS=OFF \
    -DWITH_KVS=OFF \
    -DWITH_SPDK=OFF \
    -DWITH_JAEGER=OFF \
    -DWITH_SYSTEMD=OFF \
    -G Ninja

# 编译 (使用 -j 并行)
ninja -j$(nproc)
```

### 4.2 启动开发集群

```bash
cd /home/i_ingfeng/ceph/build

# 启动最小集群 (1 MON, 1 MGR, 3 OSD)
../src/vstart.sh -n -d \
    -m 127.0.0.1 \
    --without-dashboard \
    --bluestore

# 参数说明:
#   -n        : 不重新编译
#   -d        : 使用 debug 模式
#   -m        : MON 地址
#   --bluestore : 使用 BlueStore (默认)
```

### 4.3 验证集群

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
#     osd: 3 osds: 3 up, 3 in
#   data:
#     pools:   1 pools, 1 pgs
#     objects: 0 objects, 0 B
#     usage:   3 OSDs used, X GiB used
#     pg_state:  1 active+clean
```

### 4.4 基本数据流验证

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

### 4.5 观察数据流

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

### 4.6 停止集群

```bash
../src/stop.sh
```

---

## 五、方案 B: cephadm 单节点部署

### 5.1 安装 Ceph 包

```bash
# 添加 Ceph 仓库密钥和源
curl -fsSL https://download.ceph.com/keys/release.asc | sudo gpg --dearmor -o /etc/apt/keyrings/ceph-archive-keyring.gpg
echo "deb [signed-by=/etc/apt/keyrings/ceph-archive-keyring.gpg] https://download.ceph.com/debian-quincy/ $(lsb_release -sc) main" | sudo tee /etc/apt/sources.list.d/ceph.list

sudo apt update
sudo apt install -y cephadm
```

### 5.2 Bootstrap 单节点集群

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

### 5.3 添加 OSD

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

### 5.4 验证

```bash
# 使用 cephadm shell 进入 Ceph 环境
sudo cephadm shell

# 在 shell 内:
ceph -s
ceph osd tree
ceph osd df
```

### 5.5 基本操作

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

---

## 六、方案 C: QEMU 多节点部署

### 6.1 准备云镜像

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

### 6.2 创建 cloud-init 配置

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

### 6.3 启动 QEMU 虚拟机

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

### 6.4 连接节点

```bash
# 等待 cloud-init 完成 (约 1-2 分钟)
sleep 120

# SSH 连接到各节点
ssh -p 2221 -o StrictHostKeyChecking=no ceph@localhost  # node1
ssh -p 2222 -o StrictHostKeyChecking=no ceph@localhost  # node2
ssh -p 2223 -o StrictHostKeyChecking=no ceph@localhost  # node3
```

### 6.5 在 QEMU 集群上部署 Ceph

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

---

## 七、学习路线详解

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
  └── 监控指标
      ├── ceph osd perf
      ├── ceph df detail
      └── iostat / sar
```

---

## 八、常用调试命令速查

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

## 九、关键代码位置

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

## 十、常见问题与陷阱

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
