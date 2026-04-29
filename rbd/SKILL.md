---
name: rbd
description: RBD分布式块存储专家。当用户询问RBD架构、librbd客户端、ImageCtx、snapshot快照、exclusive lock排他锁、journal日志、object map对象映射、COW克隆、分层(layering)、条带化(striping)、迁移(migration)时调用此技能。
---

# RBD (RADOS Block Device) 分布式块存储

## 〇、为什么需要 RBD？

Ceph 的底层是 RADOS 对象存储，但云平台（OpenStack、Kubernetes）需要的是**块设备**。RBD 在 RADOS 之上提供虚拟块设备接口，解决以下问题：

```
问题: RADOS 存的是对象 (按 object name 读写)，上层需要块设备 (按 offset+length 读写)

RBD 的解决:
  用户视角: 一个虚拟磁盘 /dev/rbd0，支持随机读写
  │
  ▼
  librbd 将块 I/O 转换为 RADOS 对象操作:
  1. 按 striping 参数将 offset+length 映射到 object
  2. 通过 librados 向 OSD 发送对象读写请求
  3. 处理 snapshot/clone/journal/exclusive lock 等特性
```

RBD 广泛应用于：
- **OpenStack Cinder/Glance**: 虚拟机启动盘 + 镜像存储
- **Kubernetes CSI**: 通过 ceph-csi 为 Pod 提供持久卷 (PV)
- **iSCSI 网关**: 通过 tcmu-runner 导出为 iSCSI target

## 一、核心数据结构

### 1.1 ImageCtx (src/librbd/ImageCtx.h:80-270)

每个打开的 RBD 映像对应一个 `ImageCtx` 实例：

```cpp
struct ImageCtx {
    CephContext *cct;
    PerfCounters *perfcounter;              // 性能计数器

    // 映像基本属性
    IoCtx data_ctx;                         // 数据池 RADOS IOContext
    IoCtx md_ctx;                           // 元数据池 (默认 rbd pool)
    std::string header_oid;                 // "rbd_header.<image_id>" 对象名
    struct rbd_obj_header_ondisk header;     // 映像头: size, order, features 等

    // 快照
    std::vector<librados::snap_t> snaps;
    std::map<librados::snap_t, SnapInfo> snap_info;
    uint64_t snap_id;  // CEPH_NOSNAP = 当前映像, 其他值 = 某快照

    // 特性 (features)
    uint64_t features;         // 映像启用的特性位图
    uint64_t flags;            // 映像状态标志

    // 核心功能组件
    ExclusiveLock<ImageCtx> *exclusive_lock;  // 排他锁 (防多客户端并发写)
    Journal<ImageCtx> *journal;               // 日志 (崩溃一致性)
    ObjectMap<ImageCtx> *object_map;          // 对象存在位图 (加速 resize/discard)
    Operations<ImageCtx> *operations;         // 快照/克隆/迁移操作

    // I/O 辅助
    Readahead readahead;                      // 预读控制
    asio::ContextWQ *op_work_queue;           // 异步操作工作队列
    uint64_t readahead_max_bytes;
    uint64_t readahead_disable_after_bytes;

    // 高级特性
    MigrationInfo migration_info;             // 在线迁移上下文
    cls::rbd::GroupSpec group_spec;           // group 快照组成员
};
```

### 1.2 映像头对象 (cls/rbd/cls_rbd_types.h)

RBD 元数据存储在 RADOS 对象 `rbd_header.<image_id>` 中：

```cpp
struct rbd_obj_header_ondisk {
    uint64_t size;               // 映像容量 (字节)
    uint64_t obj_version;        // 对象内部版本
    uint64_t order;              // 对象大小 = 1 << order (默认 22 → 4MB)
    uint64_t features;           // 特性位图 (见下方)
    uint64_t flags;              // 标志
    uint64_t snap_seq;           // 最近快照序号
    uint64_t parent_pool_id;     // 克隆父映像所在 pool
    string parent_image_id;      // 克隆父映像 ID
    uint64_t parent_snap_id;     // 克隆父映像快照 ID
    uint64_t stripe_unit;        // 条带单元 (默认 4MB)
    uint64_t stripe_count;       // 条带数
    uint64_t data_pool_id;       // 数据存储所在的 pool (erasure coded pool)
};

// 特性位图 (rbd_features.h)
#define RBD_FEATURE_LAYERING           (1ULL<<0)   // 分层 / COW 克隆
#define RBD_FEATURE_STRIPINGV2         (1ULL<<1)   // 条带化 v2
#define RBD_FEATURE_EXCLUSIVE_LOCK     (1ULL<<2)   // 排他锁
#define RBD_FEATURE_OBJECT_MAP         (1ULL<<3)   // 对象存在映射
#define RBD_FEATURE_FAST_DIFF          (1ULL<<4)   // 快速差异比较
#define RBD_FEATURE_DEEP_FLATTEN       (1ULL<<5)   // 深度展平
#define RBD_FEATURE_JOURNALING         (1ULL<<6)   // 日志
#define RBD_FEATURE_DATA_POOL          (1ULL<<7)   // 数据分层 (EC pool)
#define RBD_FEATURE_OPERATIONS         (1ULL<<8)   // 快照/克隆操作
#define RBD_FEATURE_MIGRATING          (1ULL<<9)   // 在线迁移
#define RBD_FEATURE_GROUPS             (1ULL<<10)  // 一致性组快照
```

## 二、核心流程

### 2.1 条带化：块偏移 → RADOS 对象映射

```
用户写入: offset=5MB, length=1MB
映像 order=22 (object_size=4MB), stripe_unit=4MB, stripe_count=3

               stripe 0                stripe 1
       ┌──────┬──────┬──────┬──────┬──────┬──────┐
       │ obj0 │ obj1 │ obj2 │ obj3 │ obj4 │ obj5 │ ...
       └──────┴──────┴──────┴──────┴──────┴──────┘
          4MB    4MB    4MB     4MB    4MB    4MB

对象名规则: rbd_data.<image_id>.<object_no>
   - object_no 基于 stripe 单元分配
   - offset 落在哪个 object_no = (offset / stripe_unit) * stripe_count
                                + (offset % stripe_unit) / single_object_portion

本例: stripe_unit=4MB, stripe_count=3, object_size=4MB
  object_no = (5MB / 4MB) * 3 + (1MB / (4MB/3)) = 1 * 3 + 0 = 3
  写入对象: rbd_data.<id>.00000000000003, 内部偏移 1MB
```

### 2.2 排他锁 (Exclusive Lock)

防止多个客户端同时写入同一映像：

```
客户端 A 尝试写
  │
  ▼
ExclusiveLock::request_lock()          // src/librbd/ExclusiveLock.cc
  │
  ├─ 1. 检查当前锁持有者 (从锁对象读取)
  │     lock_object = "rbd_lock.<image_id>"
  │
  ├─ 2. 无人持有 → 通过 RADOS compare-and-write 尝试获取
  │     写入: {locker=client_A, cookie=...}
  │
  ├─ 3. 获取成功 → exclusive_locked = true, 开始 I/O
  │     定期向 OSD 发送心跳续租 (默认 60s lease)
  │
  └─ 4. 获取失败 → 等待通知, 阻塞 I/O
  │
  ▼
Watch/Notify 机制:
  - 客户端 watch 锁对象, 当锁释放时收到 notify
  - 竞争获取锁 (分布式自旋锁)
```

代码位置: `src/librbd/ExclusiveLock.cc:request_lock()`

### 2.3 快照 (Snapshot)

```
创建快照: rbd snap create pool/image@snap1

过程:
1. 获取 ExclusiveLock
2. 暂停所有 I/O
3. 递增 snap_seq, 记录当前映像状态到 snap_info
4. 后续写入 → COW: 先拷贝旧数据到快照对象, 再覆盖写入
   - 对象 "rbd_data.<id>.<no>" 被 HEAD 修改
   - 原始数据以 "rbd_data.<id>.<no>.<snap_id>" 名保留
5. 恢复 I/O

快照删除: rbd snap rm pool/image@snap1
  - 遍历所有快照保护的对象, 删除不再引用的快照数据
  - 更新 snap_info
```

### 2.4 克隆 (Clone) —— 基于 COW 的分层

```
rbd clone pool/parent@snap1 pool/child

原理:
1. 子映像的 header 设置 parent_pool_id, parent_image_id, parent_snap_id
2. 子映像读取时:
   - 先查自身是否有该对象 (rbd_data.<child_id>.<no>)
   - 没有 → 去父映像查找 (rbd_data.<parent_id>.<no>.<snap_id>)
3. 子映像写入时:
   - COW: 先将父映像数据拷贝到子映像, 再写入 (仅首次)

展平 (flatten): rbd flatten pool/child
  - 将所有仍未拷贝的父映像数据逐对象拷贝到子映像
  - 完成后清除 parent 引用, 子映像完全独立
```

### 2.5 Journal 日志

为支持 RBD Mirror（跨集群异步复制）和崩溃一致性：

```
启用 journaling 后:
  写请求 → Journal::append_op_event(OpEvent)
          → 写入 journal_object (RADOS 对象, 环形缓冲)
          → commit 到 journal
          → 执行真实 I/O 到数据对象
          → 记录 journal 位置到 header

崩溃恢复:
  重新打开映像 → 读取 journal event → 从上次 commit 位置重放未完成的操作
```

代码位置: `src/librbd/Journal.cc:append_io_event()`

### 2.6 Object Map

用位图标记每个对象是否存在（加速 resize、discard、export 等操作）：

```
ObjectMap 结构:
  rbd_object_map.<image_id> 对象存储位图
  bit 0 = 对象 0 是否存在
  bit 1 = 对象 1 是否存在
  ...

用途:
  - resize (扩大): 标记新对象为不存在 (稀疏分配)
  - resize (缩小): 标记超出范围的对象为不存在
  - discard: 标记被丢弃区域的对象为不存在
  - export: 只导出存在的对象 (节省带宽)
```

代码位置: `src/librbd/ObjectMap.cc:set_object_map_state()`

### 2.7 迁移 (Migration)

在线将 RBD 映像从一个 pool 迁移到另一个 pool：

```
rbd migration prepare pool/src@snap1 pool/dst
rbd migration execute pool/dst

过程:
1. prepare: 为目标映像设置 source 信息 (pool, image, snap)
2. execute:
   - 创建目标映像, 设置 parent = src@snap (利用 clone 机制)
   - 后台逐对象从源拷贝数据到目标
   - 拷贝完成后清除 parent 引用
3. commit: 目标完全独立, 可删除源
```

代码位置: `src/librbd/migration/`

## 三、关键设计特性

| 特性 | 说明 | 启用方式 |
|------|------|---------|
| **Layering** | COW 克隆, 基于快照创建即时子映像 | `rbd feature enable` |
| **Exclusive Lock** | 排他锁防止并发写, RBD Mirror 的前提 | 自动用于 journal/write |
| **Journaling** | 崩溃一致性 + RBD Mirror 源 | `rbd feature enable journaling` |
| **Object Map** | 加速 resize/discard/export | `rbd feature enable object-map` |
| **Fast Diff** | 高效计算两个快照间的差异块 | 需要 object-map |
| **Deep Flatten** | 递归展平多层 clone 链 | `rbd flatten --no-progress` |
| **Mirroring** | 跨集群异步复制 (基于 journal) | `rbd mirror image enable` |
| **Migration** | 在线跨 pool 迁移 | `rbd migration` |
| **Encryption** | LUKS 加密格式 | `rbd encryption format` |
| **Group Snapshots** | 多映像一致性快照 | `rbd group snap create` |
| **Sparseness** | 稀疏分配, 未写入的对象不占空间 | 默认 (依赖 object-map) |
| **Trash** | 延迟删除, 误删可恢复 | `rbd trash` |

## 四、关键代码位置

```
src/librbd/
├── librbd.cc                    # 7818行, CLI 到 C API 转换
├── internal.cc                  # 内部实现 (I/O 调度)
├── ImageCtx.cc/h                # 映像上下文 (核心类)
├── ImageState.cc/h              # 映像打开/关闭状态机
├── ImageWatcher.cc/h            # 映像元数据变更 watch
├── Operations.cc/h              # 快照/克隆/展平操作
├── ExclusiveLock.cc/h           # 排他锁
├── ManagedLock.cc/h             # 通用锁管理
├── Journal.cc/h                 # 日志 (崩溃一致性/Mirror 回放)
├── ObjectMap.cc/h               # 对象存在位图
├── DeepCopyRequest.cc/h         # 深度拷贝 (clone flatten)
├── WatchNotifyTypes.cc/h        # Watch/Notify 消息类型
│
├── api/                         # 各特性的 API 层
│   ├── Migration.cc             # 在线迁移
│   ├── Mirror.cc                # RBD Mirror
│   ├── Group.cc                 # Group 快照
│   └── Trash.cc                 # 延迟删除
│
├── crypto/                      # 加密
│   └── luks/                    # LUKS 格式支持
│
├── mirror/                      # RBD Mirror 完整实现
│   ├── ImageReplayer.cc         # 日志回放
│   ├── InstanceReplayer.cc      # 实例级回放管理
│   └── PoolReplayer.cc          # Pool 级回放
│
├── migration/                   # 在线迁移
│   ├── NativeFormat.cc          # 原生格式
│   ├── RawFormat.cc             # 原始格式
│   └── QCOWFormat.cc            # QCOW2 格式
│
├── operation/                   # 异步操作
│   ├── ResizeRequest.cc         # 调整容量
│   ├── SnapshotCreateRequest.cc # 创建快照
│   └── FlattenRequest.cc        # 展平
│
└── journal/                     # 日志子系统
    ├── StandardPolicy.cc        # 标准日志策略
    └── TagData.cc               # 日志标签
```

```
src/cls/rbd/                     # RADOS 对象类 (服务端)
├── cls_rbd.cc                   # 服务端方法 (快照/header/group/trash)
├── cls_rbd_client.cc            # 客户端调用封装
└── cls_rbd_types.h              # 共享数据结构定义
```

## 五、常见问题与陷阱

1. **`rbd: feature required: exclusive-lock, object-map, fast-diff, deep-flatten, journaling`**
   - 客户端或内核不支持映像特性时出现。解决方案：`rbd feature disable <image> <feature>` 或用支持该特性的客户端版本。

2. **`rbd: image has watchers - meaning another client is already accessing it`**
   - 排他锁被其他客户端持有。检查 `rbd status pool/image`。如果是残留锁，可手动 blacklist 旧客户端后移除。

3. **克隆链过长导致性能下降**：子映像的每次首次读取需要沿 clone 链逐级查找。建议定期 `rbd flatten` 断链。

4. **kernel rbd 不支持 journaling**：内核 rbd 客户端 (`krbd`) 不支持 journal feature。如果需要在 krbd 上使用映像，需 disable journal。
   补救：`rbd feature disable pool/image journaling`

5. **object-map 与 sparse 读取**：启用 object-map 后能用 `OBJECT_EXISTS` 标志跳过不存在对象的读取，极大加速 export 和 resize，但引入额外写开销（每次 I/O 需更新位图）。

6. **RBD Mirror 与 exclusive lock 的强依赖**：journaling 必须与 exclusive lock 同时启用。如果锁被抢占，mirror 会暂停直到重新获取。

## 六、参考文献

1. **RBD 官方文档**: https://docs.ceph.com/en/latest/rbd/
2. **RBD Mirror 设计**: https://docs.ceph.com/en/latest/rbd/rbd-mirroring/
3. **librbd C API**: https://docs.ceph.com/en/latest/rbd/librbdpy/
4. **"RBD: RADOS Block Device"** - Ceph 官方架构文档

### 相关技能

- [Objecter](../client/objecter/SKILL.md) — RBD 通过 librados/Objecter 向 OSD 发送 I/O
- [CRUSH 算法](../crush/crush-algorithm/SKILL.md) — 对象到 OSD 的映射依赖 CRUSH
