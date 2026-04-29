---
name: osd-io-path
description: Ceph OSD I/O路径专家。当用户询问OSD线程模型、写入请求追踪、消息分发路径、ShardedOpWQ调度、BlueStore写入链路、从网络到磁盘的完整I/O流程时调用此技能。
---

# OSD I/O 路径：从网络到磁盘

## 〇、为什么需要理解 OSD I/O 路径？

OSD 是 Ceph 数据平面最核心的组件。理解请求从网络到达磁盘的完整路径，是排查性能问题和理解 OSD 架构的基础：

```
没有 I/O 路径理解的问题:
  客户端写入延迟高 → 不知道瓶颈在哪
  OSD CPU 占用高 → 不知道是调度问题还是磁盘问题
  写入卡住 → 不知道请求挂在哪个环节

掌握 I/O 路径后:
  延迟高 → 可以精确定位: 是 ShardedOpWQ 队列积压? 还是 BlueStore kv_sync 瓶颈?
  CPU 高 → 可以判断: 是 PG 操作太多? 还是无谓的上下文切换?
  卡住 → 可以追踪: 请求在哪个状态机环节等待?
```

---

## 一、线程分布图

### 1.1 OSD 线程模型全景

```
┌─────────────────────────────────────────────────────────────────────┐
│                          OSD 进程                                    │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  ┌──────────────────────────────────────────────┐                   │
│  │ Messenger 网络线程 (多个)                      │                   │
│  │   ├── ms_fast_dispatch()  [快速路径，无锁]     │                   │
│  │   │    └── Peering 消息 → enqueue_peering_evt │                   │
│  │   │    └── OSD 客户端 Op → OpRequest → enqueue_op                 │
│  │   ├── ms_dispatch()       [慢速路径，加 osd_lock]                 │
│  │   └── 心跳线程                                │                   │
│  └──────────────────────────────────────────────┘                   │
│                         │ 入队                                       │
│                         ▼                                            │
│  ┌──────────────────────────────────────────────┐                   │
│  │ ShardedOpWQ (统一操作队列)                      │                   │
│  │   └── ShardedThreadPool osd_op_tp              │                   │
│  │       ├── Worker 线程 0 (shard 0)               │                   │
│  │       ├── Worker 线程 1 (shard 1)               │                   │
│  │       ├── ... (按 PG 分片，每 shard 独立)        │                   │
│  │       └── Worker 线程 N (shard N)               │                   │
│  │                                                 │                   │
│  │   调度器: mClockScheduler 或 WeightedPriorityQueue               │
│  │   ├── 客户端 Op (PGOpItem)          [client 类]                   │
│  │   ├── Peering 事件 (PGPeeringItem)   [immediate 类]              │
│  │   ├── Scrub 操作 (PGScrub)           [background_best_effort 类] │
│  │   ├── Snap Trim (PGSnapTrim)         [background_best_effort 类] │
│  │   └── Recovery (PGRecovery)           [优先级决定类]               │
│  └──────────────────────────────────────────────┘                   │
│                         │ 取出执行                                   │
│                         ▼                                            │
│  ┌──────────────────────────────────────────────┐                   │
│  │ PG 处理 (在 Worker 线程上)                      │                   │
│  │   ├── PrimaryLogPG::do_op()        [客户端写操作]                 │
│  │   ├── PrimaryLogPG::do_request()   [其他请求]                     │
│  │   ├── PG::do_request()             [Recovery/Scrub]              │
│  │   └── OSD::dequeue_peering_evt()   [Peering 事件]               │
│  └──────────────────────────────────────────────┘                   │
│                         │ 写事务                                     │
│                         ▼                                            │
│  ┌──────────────────────────────────────────────┐                   │
│  │ BlueStore 线程                                  │                   │
│  │   ├── kv_sync_thread     [RocksDB 同步写入]                      │
│  │   ├── kv_finalize_thread  [事务完成回调]                          │
│  │   ├── finisher           [通用回调线程]                           │
│  │   └── AIO 完成线程        [磁盘 IO 完成通知]                     │
│  └──────────────────────────────────────────────┘                   │
│                                                                      │
│  ┌──────────────────────────────────────────────┐                   │
│  │ 其他线程                                        │                   │
│  │   ├── tick_timer         [定期 tick: 心跳/scrub 调度]            │
│  │   ├── beacon_timer       [MON 心跳]                              │
│  │   └── scrub_finalizer    [scrub 完成]                             │
│  └──────────────────────────────────────────────┘                   │
└─────────────────────────────────────────────────────────────────────┘
```

### 1.2 关键问题指南

**看到任何函数，要问：**

| 问题 | 如何回答 |
|------|---------|
| 这个函数在哪个线程执行？ | 查调用链。若来自 `_process()`，则是 ShardedOpWQ Worker 线程 |
| 它操作的数据被谁共享？ | PG 对象被多个 Op 共享，但同一 shard 内串行化 |
| 是否需要锁？ | ShardedOpWQ 每个 shard 串行处理同一 PG；跨 PG 无锁 |
| 它可能阻塞吗？ | Worker 线程不应阻塞。BlueStore 写入是异步的 |
| 回调在哪个线程执行？ | `Finisher` 线程或 `kv_finalize_thread` |

### 1.3 ShardedOpWQ 调度器类型 (src/osd/scheduler/)

| 调度器 | 说明 |
|--------|------|
| `mClockScheduler` | 基于 dmClock 的 QoS 调度，支持客户端优先级和预留带宽 |
| `ClassicWeightedPriorityQueue` | 传统加权优先级队列，按优先级和年龄调度 |

调度类：
| 类 | 用途 |
|-----|------|
| `scheduler_class::client` | 客户端 OSD 操作 |
| `scheduler_class::immediate` | Peering、PG 创建等高优先级 |
| `scheduler_class::background_best_effort` | Scrub、Snap Trim 等后台任务 |

---

## 二、请求追踪图

### 2.1 写入请求完整路径

```
客户端 --[CEPH_MSG_OSD_OP]--> Messenger 网络线程
                                    │
                                    ▼
                        OSD::ms_fast_dispatch()           [OSD.cc:7663]
                                    │
                        ┌───────────┴───────────┐
                        │ Peering 消息?           │
                        │  是 → enqueue_peering_evt()  │
                        │  否 ↓                   │
                        └─────────────────────────┘
                                    │
                                    ▼
                        OpRequestRef op = create_request()   [OSD.cc:7712]
                                    │
                                    ▼
                        OSD::enqueue_op(spg, op, epoch)      [OSD.cc:9877]
                                    │
                                    ▼
                   ┌── ShardedOpWQ.queue(PGOpItem) ─────────┐
                   │   (按 PG hash 分配到 shard)              │
                   └────────────────────────────────────────┘
                                    │
                          Worker 线程取出执行
                                    │
                                    ▼
                        ShardedOpWQ::_process()              [OSD.cc:11071]
                                    │
                                    ▼
                        PGOpItem::run(osd, sdata, pg, handle) [OpSchedulerItem.cc:23]
                                    │
                                    ▼
                        OSD::dequeue_op(pg, op, handle)      [OSD.cc:9935]
                                    │
                                    ▼
                        PG::do_request(op, handle)           [PG.cc]
                                    │
                                    ▼
                        PrimaryLogPG::do_request()             [PrimaryLogPG.cc:1836]
                                    │
                                    ▼
                        PrimaryLogPG::do_op()                 [PrimaryLogPG.cc:2002]
                                    │
                     ┌──────────────┴──────────────┐
                     │ 验证请求:                    │
                     │  - PG 是否活跃?              │
                     │  - 是否 backoff?             │
                     │  - 操作是否允许?              │
                     │  - 对象是否存在/可创建?       │
                     └──────────────┬──────────────┘
                                    │
                                    ▼
                        PrimaryLogPG::execute_ctx()           [PrimaryLogPG.cc:4219]
                                    │
                        ┌───────────┴───────────┐
                        │ 准备写事务:             │
                        │  1. 构建 PGTransaction  │
                        │  2. 分配新版本号         │
                        │  3. 设置 on_commit 回调  │
                        │  4. 设置 on_applied 回调 │
                        └───────────┬───────────┘
                                    │
                                    ▼
                        PrimaryLogPG::issue_repop()           [PrimaryLogPG.cc:11528]
                                    │
                        ┌───────────┴───────────┐
                        │ 1. 创建 RepGather      │
                        │ 2. on_all_commit 回调   │
                        │ 3. 向副本发送 MOSDOp     │
                        └───────────┬───────────┘
                                    │
                                    ▼
                        pgbackend->submit_transaction()       
                                    │
                    ┌───────────────┴───────────────┐
                    │                               │
          ReplicatedBackend                   ECBackend
          (ReplicatedBackend.cc:580)          (ECBackend.cc:951)
                    │                               │
                    ▼                               ▼
          1. 生成 ObjectStore::Transaction     1. 计算编码分片
          2. 发送 MOSDRepOp 给副本              2. 写入本地分片
          3. 写入本地 BlueStore                  3. 读写 RMW 流程
                    │
                    ▼
          BlueStore::queue_transactions()     [BlueStore.cc:15747]
                    │
          ┌────────┴────────┐
          │ TransContext 状态机│
          │ PREPARE → AIO_WAIT │
          │ → IO_DONE           │
          │ → KV_QUEUED         │
          │ → KV_SUBMITTED      │
          │ → KV_DONE           │
          │ → FINISHING → DONE  │
          └────────┬────────┘
                   │
         ┌─────────┴─────────┐
         │ kv_sync_thread     │
         │ RocksDB 写入       │
         │ (提交事务)          │
         └─────────┬─────────┘
                   │
         ┌─────────┴─────────┐
         │ kv_finalize_thread │
         │ 回调完成           │
         └─────────┬─────────┘
                   │
                   ▼
          C_OSD_RepopCommit::complete()
                   │
                   ▼
          PrimaryLogPG::repop_all_commit()
                   │
                   ▼
          回复客户端 ──[CEPH_MSG_OSD_OPREPLY]──> 客户端
```

### 2.2 读取请求路径

```
客户端 --[CEPH_MSG_OSD_OP (read)]--> ms_fast_dispatch()
    → enqueue_op() → ShardedOpWQ → PrimaryLogPG::do_op()
    → execute_ctx() (读操作)
    → pgbackend->objects_read_sync() 或 objects_read_async()
    → BlueStore 读取
    → 回复客户端
```

读取与写入的关键区别：
- 读取不需要副本同步，Primary 直接从本地 BlueStore 读取
- 读取不需要 `RepGather` 和 `issue_repop`
- 所有操作在同一个 Worker 线程上发起，BlueStore AIO 完成后再回调

### 2.3 Peering 事件路径

```
MOSDPGNotify / MOSDPGLog
    → ms_fast_dispatch()
    → enqueue_peering_evt()               [OSD.cc:9919]
    → op_shardedwq.queue(PGPeeringItem)  [优先级: osd_peering_op_priority]
    → ShardedOpWQ::_process()
    → PGPeeringItem::run()
    → OSD::dequeue_peering_evt()
    → PG::do_request()
    → PeeringState::handle_event()
```

Peering 事件使用 `immediate` 调度类，优先级高于客户端操作。

---

## 三、对象关系图

### 3.1 OSD 核心对象关系

```
OSD (src/osd/OSD.h)
  ├── OSDService (全局共享服务)
  │     ├── ObjectStore *store      (BlueStore 实例)
  │     ├── Messenger *cluster_messenger  (集群内部网络)
  │     ├── Messenger *client_messenger   (客户端网络)
  │     ├── OSDMapRef osdmap       (当前 OSDMap)
  │     └── ShardedOpWQ op_shardedwq  (操作调度队列)
  │
  ├── PG (src/osd/PG.h)
  │     ├── PeeringState recovery_state  (状态机)
  │     ├── PGPool &pool           (引用)
  │     ├── pg_info_t &info        (引用)
  │     ├── OSDService *osd         (回到全局服务)
  │     ├── ObjectStore::CollectionHandle ch  (存储引擎句柄)
  │     ├── PGBackend *pgbackend   (后端引擎)
  │     │     └── ReplicatedBackend 或 ECBackend
  │     └── SnapshotContext *snapcontext  (快照上下文)
  │
  └── PrimaryLogPG (src/osd/PrimaryLogPG.h)
        └── 继承 PG
              ├── OpContext (每次操作的上下文)
              │     ├── ObjectContextRef obc  (对象上下文)
              │     ├── PGTransactionUPtr op_t  (事务)
              │     └── RepGather *repop  (复制收集器)
              └── ObjectContext (src/osd/ObjectContext.h)
                    ├── object_rwlock    (读写锁)
                    └── erosion_manifest (EC 清单)
```

### 3.2 BlueStore 写入事务对象关系

```
TransContext (BlueStore.h:2443)
  ├── CollectionRef ch          (所属集合)
  ├── State state              (事务状态机)
  │     STATE_PREPARE → STATE_AIO_WAIT → ... → STATE_DONE
  ├── vector<WriteContext>#ifdef write_conflict  (冲突写入)
  ├── IntervalSet<uint64_t> allocated  (已分配空间)
  └── Context *on_commit       (完成回调 = C_OSD_RepopCommit)

BlueStore::Onode (BlueStore.h:1331)
  ├── Collection *c            (所属集合)
  ├── ghobject_t oid            (对象键)
  ├── ExtentMap extent_map      (逻辑偏移 → Blob 映射)
  └── BufferSpace bc            (缓冲区缓存)

BlueStore::Blob (BlueStore.h:650)
  ├── bluestore_blob_t blob     (持久化 Blob 元数据)
  ├── SharedBlobRef shared_blob (共享 Blob 引用)
  └── PExtentVector extents    (物理范围列表)
```

---

## 四、核心代码位置

| 功能 | 文件 | 行号 |
|------|------|------|
| **消息分发入口** | | |
| ms_fast_dispatch | src/osd/OSD.cc | 7663 |
| ms_dispatch (慢路径) | src/osd/OSD.cc | 7550 |
| _dispatch (慢路径分发) | src/osd/OSD.cc | 7822 |
| **ShardedOpWQ** | | |
| ShardedOpWQ 定义 | src/osd/OSD.h | 1728-1815 |
| ShardedOpWQ::_process | src/osd/OSD.cc | 11071 |
| enqueue_op | src/osd/OSD.cc | 9877 |
| enqueue_peering_evt | src/osd/OSD.cc | 9919 |
| dequeue_op | src/osd/OSD.cc | 9935 |
| **调度器** | | |
| OpSchedulerItem 类型 | src/osd/scheduler/OpSchedulerItem.h | 218-560 |
| OpSchedulerItem::run | src/osd/scheduler/OpSchedulerItem.cc | 23-230 |
| mClockScheduler | src/osd/scheduler/mClockScheduler.h | - |
| ClassicWeightedPriorityQueue | src/osd/scheduler/ClassicWeightedPriorityQueue.h | - |
| **PG 处理** | | |
| PrimaryLogPG::do_op | src/osd/PrimaryLogPG.cc | 2002 |
| PrimaryLogPG::do_request | src/osd/PrimaryLogPG.cc | 1836 |
| PrimaryLogPG::execute_ctx | src/osd/PrimaryLogPG.cc | 4219 |
| PrimaryLogPG::issue_repop | src/osd/PrimaryLogPG.cc | 11528 |
| **后端** | | |
| ReplicatedBackend::submit_transaction | src/osd/ReplicatedBackend.cc | 580 |
| ECBackend::submit_transaction | src/osd/ECBackend.cc | 951 |
| **BlueStore** | | |
| queue_transactions | src/os/bluestore/BlueStore.cc | 15747 |
| _kv_sync_thread | src/os/bluestore/BlueStore.cc | 15057 |
| _kv_finalize_thread | src/os/bluestore/BlueStore.cc | 15331 |
| _txc_state_proc | src/os/bluestore/BlueStore.cc | 14401 |
| TransContext 状态定义 | src/os/bluestore/BlueStore.h | 2443-2455 |

---

## 五、常见问题与陷阱

### Q1: 为什么 OSD 使用 ShardedOpWQ 而不是普通线程池？

**A**: ShardedOpWQ 按 PG 分片（shard），同一个 PG 的操作在同一 shard 上串行执行，避免了 PG 锁争用。不同 PG 的操作可以并行处理。mClock 调度器进一步提供了 QoS 保证，防止单个客户端饿死其他客户端。

### Q2: 写入操作的 on_commit 回调在哪个线程执行？

**A**: BlueStore 的 `on_commit` 回调在 `kv_finalize_thread` 中执行。当 `kv_sync_thread` 将事务写入 RocksDB 并完成后，将事务移到 `kv_committing_to_finalize` 队列，`kv_finalize_thread` 取出后调用 `_txc_state_proc()` 进入 `STATE_FINISHING`，最终调用 `on_commit->complete(0)`。

### Q3: ms_fast_dispatch 和 ms_dispatch 的区别？

**A**: `ms_fast_dispatch` 不加 OSD 全局锁，直接入队到 ShardedOpWQ，延迟极低。`ms_dispatch` 需要获取 `osd_lock`，用于处理需要全局状态一致的消息（如 OSDMap 更新）。Peering 消息和客户端 I/O 走快速路径，管理消息走慢速路径。

### Q4: 一个写入请求经过多少次线程切换？

**A**:
1. **Messenger 线程**: 接收消息，入队 ShardedOpWQ (~0次上下文切换)
2. **ShardedOpWQ Worker 线程**: 取出处理，调用 do_op → execute_ctx → issue_repop
3. **同一线程**: 提交 BlueStore AIO 写入（异步提交，不等待）
4. **AIO 完成线程**: 磁盘写入完成，触发 IO_DONE
5. **kv_sync_thread**: 同步写入 RocksDB
6. **kv_finalize_thread**: 调用 on_commit 回调
7. **Messenger 线程**: 发送回复给客户端

实际上只有 3-4 次实质性的线程切换（Worker → AIO完成 → kv_sync → kv_finalize）。

### Q5: Recovery 操作如何与客户端 I/O 竞争调度？

**A**: `PGRecovery` 的调度类由优先级决定：高优先级恢复（如 `must_repair`）比客户端 I/O 优先级更高，低优先级恢复（如 `periodic_regular`）使用 `background_best_effort` 类。mClock 调度器通过预留带宽和权重配置确保恢复不会完全饿死客户端 I/O，同时客户端 I/O 也不会完全阻塞恢复。

### 相关技能

- [BlueStore 引擎](../bluestore/SKILL.md) — I/O 路径最终写入 BlueStore
- [Objecter](../../client/objecter/SKILL.md) — 客户端发起的 op 通过 Objecter 到达 OSD
- [PG Peering](../pg-peering/SKILL.md) — Peering 恢复流程与 I/O 路径的并发调度

### 参考文献

1. **OSD 开发者文档**: https://docs.ceph.com/en/latest/dev/osd_overview/
2. **mClock 调度器论文**: "mClock: Handling Throughput Variability for Hypervisor IO Scheduling" (OSDI 2010)
3. **dmClock 实现**: https://github.com/ceph/dmclock
4. **源码**: `src/osd/OSD.cc`, `src/osd/PG.cc`, `src/osd/OpScheduler.cc"
