---
name: mds-request-path
description: Ceph MDS请求处理路径专家。当用户询问MDS线程模型、客户端请求处理流程、消息分发、锁获取、MDCache请求生命周期、异步续跑模式时调用此技能。
---

# MDS 请求处理路径

## 〇、为什么需要理解 MDS 请求路径？

MDS 处理文件系统元数据操作（创建文件、目录遍历、权限检查等），其请求处理路径决定了元数据操作的延迟和吞吐：

```
没有请求路径理解的问题:
  客户端 open() 延迟高 → 不知道是锁竞争还是缓存未命中
  MDS CPU 100% → 不知道是哪些请求类型导致
  多 MDS 环境元数据不一致 → 不了解子树迁移和锁协议

掌握请求路径后:
  延迟高 → 定位: 是 acquire_locks 等待? 还是 RADOS 读写?
  CPU 高 → 分析: 请求类型分布, 是否需要增加 MDS
  不一致 → 理解: MDCache 协议、Migrator 迁移流程
```

---

## 一、线程分布图

### 1.1 MDS 线程模型全景

```
┌──────────────────────────────────────────────────────────────────┐
│                         MDS 进程                                 │
├──────────────────────────────────────────────────────────────────┤
│                                                                   │
│  ┌───────────────────────────────────────────────────┐           │
│  │ Messenger 网络线程 (AsyncMessenger Worker)         │           │
│  │   ├── epoll 事件循环检测到可读 fd                  │           │
│  │   ├── ProtocolV2 读取并解码消息                     │           │
│  │   └── ms_dispatch2() [获取 mds_lock 全局锁]         │           │
│  │       ├── handle_core_message() (mon map 等)       │           │
│  │       └── mds_rank->ms_dispatch()                   │           │
│  └───────────────────────────────────────────────────┘           │
│                          │ 同步处理                               │
│                          ▼                                        │
│  ┌───────────────────────────────────────────────────┐           │
│  │ MDSRank::handle_message() [消息路由]                │           │
│  │   └── 根据 message type 分发:                       │           │
│  │       ├── CEPH_MSG_CLIENT_REQUEST → server          │           │
│  │       ├── MSG_MDS_LOCK → locker                    │           │
│  │       ├── MDS_PORT_CACHE → mdcache                  │           │
│  │       └── MDS_PORT_MIGRATOR → mdcache->migrator    │           │
│  └───────────────────────────────────────────────────┘           │
│                                                                   │
│  ┌───────────────────────────────────────────────────┐           │
│  │ ProgressThread (唯一专用线程)                         │           │
│  │   └── _advance_queues()                              │           │
│  │       ├── 处理 finished_queue (回调完成)              │           │
│  │       └── 处理 waiting_for_nolaggy (延迟消息)         │           │
│  └───────────────────────────────────────────────────┘           │
│                                                                   │
│  ┌───────────────────────────────────────────────────┐           │
│  │ Finisher (通用回调线程)                               │           │
│  │   └── 处理 Context::complete() 回调                  │           │
│  │       (如 RADOS 操作完成通知)                         │           │
│  └───────────────────────────────────────────────────┘           │
│                                                                   │
│  ┌───────────────────────────────────────────────────┐           │
│  │ 其他线程                                            │           │
│  │   ├── beacon_timer       [MON 心跳]                 │           │
│  │   ├── MDLog 刷新线程     [日志定期刷新]              │           │
│  │   └── MDBalancer 线程    [负载均衡计算]              │           │
│  └───────────────────────────────────────────────────┘           │
└──────────────────────────────────────────────────────────────────┘
```

### 1.2 关键问题指南

**看到任何函数，要问：**

| 问题 | 如何回答 |
|------|---------|
| 这个函数在哪个线程执行？ | 若来自 `ms_dispatch2`，则是 Messenger Worker 线程（持有 `mds_lock`）; 若来自 `ProgressThread`，则是完成回调 |
| 它操作的数据被谁共享？ | 所有元数据操作共享 `mds_lock`，单线程串行化 |
| 是否需要锁？ | `mds_lock` 已在入口获取，内部操作无需额外大锁 |
| 它可能阻塞吗？ | 不应主动阻塞。等待锁时注册回调后返回 |
| 什么是 C_MDS_RetryRequest？ | 锁不可用时注册的续跑回调，锁释放后重新执行handler |

### 1.3 与 OSD 的关键区别

| 维度 | OSD | MDS |
|------|-----|-----|
| **并发模型** | ShardedOpWQ 多线程，按 PG 分片 | 全局 mds_lock，单线程串行 |
| **工作队列** | 有 OpWQ/PeeringWQ 等 | 无独立 WorkQueue，消息直接处理 |
| **锁粒度** | PG 级别锁 | 全局 mds_lock + 细粒度分布锁(Locker) |
| **异步续跑** | PG 内状态机 | C_MDS_RetryRequest 回调续跑 |
| **证据机制** | 无 | Capability 系统委托客户端权限 |

---

## 二、请求追踪图

### 2.1 客户端请求完整路径

```
客户端 --[CEPH_MSG_CLIENT_REQUEST]--> Messenger Worker 线程
                                            │
                                            ▼
                              MDSDaemon::ms_dispatch2()      [MDSDaemon.cc:1044]
                                            │
                              ┌──────────────┴──────────────┐
                              │ mds_lock.lock()               │
                              │  (全局公平互斥锁)               │
                              │ handle_core_message()?         │
                              │  → mon map, mds map 等核心消息 │
                              │ 否 → mds_rank->ms_dispatch()   │
                              └──────────────┬──────────────┘
                                            │
                                            ▼
                              MDSRankDispatcher::ms_dispatch()  [MDSRank.cc:1033]
                                            │
                                            ▼
                              MDSRank::_dispatch(m, true)      [MDSRank.cc:1054]
                                            │
                              ┌──────────────┴──────────────┐
                              │ 检查:                         │
                              │  - quiesce_dispatch()? 暂停?  │
                              │  - is_stale_message()? 过期?   │
                              │  - is_valid_message()? 有效?   │
                              │  - beacon.is_laggy()? 拥堵?   │
                              │    是 → waiting_for_nolaggy    │
                              │    否 → handle_message(m)     │
                              └──────────────┬──────────────┘
                                            │
                                            ▼
                              MDSRank::handle_message()       [MDSRank.cc:1227]
                                            │
                  ┌─────────────────────────┼─────────────────────────┐
                  │                         │                         │
                  ▼                         ▼                         ▼
     CEPH_MSG_CLIENT_REQUEST    MSG_MDS_LOCK                MDS_PORT_CACHE
     CEPH_MSG_CLIENT_SESSION   MSG_MDS_INODEFILECAPS         (resolve, rejoin,
     CEPH_MSG_CLIENT_CAPS      CEPH_MSG_CLIENT_CAPRELEASE     discover...)
     CEPH_MSG_CLIENT_LEASE     CEPH_MSG_CLIENT_CAPS
                  │                         │                         │
                  ▼                         ▼                         ▼
         Server::dispatch()        Locker::dispatch()       MDCache::dispatch()
          [Server.cc:317]          [Locker.cc:91]           [MDCache.cc:8451]
```

### 2.2 客户端元数据操作路径

```
Server::dispatch(m)                            [Server.cc:317]
  │
  ▼ 判断消息类型
  │
  ├── CEPH_MSG_CLIENT_SESSION → handle_client_session()
  ├── CEPH_MSG_CLIENT_RECONNECT → handle_client_reconnect()
  ├── CEPH_MSG_CLIENT_REQUEST → handle_client_request()
  │
  ▼
Server::handle_client_request(req)              [Server.cc:2603]
  │
  ├── 验证 session 状态
  ├── 检查请求去重 (replay/retry)
  │
  ▼
  mdr = mdcache->request_start(req)             [MDCache.cc:9921]
  │  → 创建 MDRequestRef
  │  → 注册到 active_requests
  │
  ▼
  Server::dispatch_client_request(mdr)          [Server.cc:2752]
  │
  ┌── switch (req->get_op()) ──────────────────────────────┐
  │                                                          │
  │  CEPH_MDS_OP_LOOKUP   → handle_client_getattr(mdr, true) │
  │  CEPH_MDS_OP_GETATTR  → handle_client_getattr(mdr, false)│
  │  CEPH_MDS_OP_SETATTR  → handle_client_setattr(mdr)      │
  │  CEPH_MDS_OP_READDIR  → handle_client_readdir(mdr)      │
  │  CEPH_MDS_OP_OPEN     → handle_client_open(mdr)          │
  │  CEPH_MDS_OP_CREATE   → handle_client_openc(mdr)        │
  │  CEPH_MDS_OP_UNLINK   → handle_client_unlink(mdr)       │
  │  CEPH_MDS_OP_RENAME   → handle_client_rename(mdr)       │
  │  CEPH_MDS_OP_MKDIR    → handle_client_mkdir(mdr)        │
  │  ...                                                     │
  └──────────────────────────────────────────────────────────┘
```

### 2.3 路径遍历与锁获取（以 open 为例）

```
Server::handle_client_open(mdr)                [Server.cc:4585]
  │
  ├── 1. 路径遍历: rdlock_path_pin_ref(mdr, ...) [Server.cc:3836]
  │     │
  │     ├── mdcache->path_traverse(mdr, ...)
  │     │     │
  │     │     ├── 从根 CInode 开始沿路径查找 CDentry/CInode
  │     │     ├── 缓存命中 → 直接返回
  │     │     ├── 缓存未命中 → 向权威 MDS 发送 discover 消息
  │     │     └── 本地不是权威 → forward_request() 转发
  │     │
  │     └── 对路径上每个 CDentry/CInode 获取读锁 (rdlock)
  │
  ├── 2. 锁获取: Locker::acquire_locks(mdr, lov)  [Locker.cc:243]
  │     │
  │     ├── 遍历 LockOpVec (lov)
  │     ├── 对每个锁:
  │     │     ├── rdlock_start(lock, mdr)
  │     │     ├── wrlock_start(op, mdr)
  │     │     └── xlock_start(lock, mdr)
  │     │
  │     ├── 如果锁不可立即获取:
  │     │     └── lock->add_waiter(WAIT_XX, new C_MDS_RetryRequest(mdcache, mdr))
  │     │         → 返回 false，请求挂起
  │     │         → 锁可用时回调 C_MDS_RetryRequest
  │     │         → MDCache::dispatch_request(mdr) 重新执行
  │     │
  │     └── 所有锁获取成功 → 返回 true
  │
  ├── 3. 执行操作
  │     ├── 查找或创建目标 CInode
  │     ├── 更新 inode 属性
  │     ├── 授予 Capability
  │     └── 记录 MDLog 条目
  │
  ├── 4. 回复客户端
  │     └── respond_to_request(mdr, 0)  [Server.cc]
  │
  └── 5. 释放锁
        └── Locker::drop_locks(mut)        [Locker.cc:876]
              ├── 释放 mdr->locks 中所有锁
              └── Locker::eval() 评估锁状态转变
                    → 可能触发新的 Cap 授权/撤销
```

### 2.4 异步续跑模式（锁等待）

```
Client Request ───> Server::handle_client_open()
                       │
                       ├── path_traverse() 成功
                       ├── acquire_locks()
                       │     ├── lock_A: rdlock 成功 ✓
                       │     ├── lock_B: wrlock 成功 ✓
                       │     └── lock_C: xlock 被其他客户端持有 ✗
                       │           └── lock_C->add_waiter(C_MDS_RetryRequest)
                       │                │
                       └── 返回 false，请求挂起
                            │
                   (Messenger Worker 线程释放 mds_lock，处理下一个消息)
                            │
                   ... 其他请求处理 ...
                            │
                   锁持有者释放 lock_C: Locker::eval()
                       │
                       ├── 锁状态转变
                       ├── 取出等待者: C_MDS_RetryRequest
                       └── 放入 finished_queue
                            │
                   ProgressThread 或 _advance_queues() 处理
                       │
                       ▼
                   MDCache::dispatch_request(mdr)  [MDCache.cc:10096]
                       │
                       ▼
                   Server::dispatch_client_request(mdr)  → 重新执行
                       │
                       ├── path_traverse() 跳过 (缓存已命中)
                       ├── acquire_locks()
                       │     ├── lock_A: rdlock 成功 ✓
                       │     ├── lock_B: wrlock 成功 ✓
                       │     └── lock_C: xlock 成功 ✓ (已释放)
                       │
                       ├── 执行操作
                       ├── 回复客户端
                       └── drop_locks()
```

### 2.5 MDCache 子系统间消息路由

```
MDCache::dispatch(m)                            [MDCache.cc:8451]
  │
  ├── MSG_MDS_RESOLVE          → handle_resolve()
  ├── MSG_MDS_RESOLVEACK       → handle_resolve_ack()
  ├── MSG_MDS_CACHEREJOIN      → handle_cache_rejoin()
  ├── MSG_MDS_DISCOVER         → handle_discover()
  ├── MSG_MDS_DISCOVERREPLY    → handle_discover_reply()
  ├── MSG_MDS_DIRUPDATE        → handle_dir_update()
  ├── MSG_MDS_CACHEEXPIRE      → handle_cache_expire()
  ├── MSG_MDS_DENTRYLINK       → handle_dentry_link()
  ├── MSG_MDS_DENTRYUNLINK     → handle_dentry_unlink()
  ├── MSG_MDS_FRAGMENTNOTIFY   → handle_fragment_notify()
  ├── MSG_MDS_FINDINO          → handle_find_ino()
  ├── MSG_MDS_OPENINO          → handle_open_ino()
  └── MSG_MDS_SNAPUPDATE       → handle_snap_update()

Migrator::dispatch(m)                          [MDCache.cc] (migrator 子模块)
  │
  ├── MSG_MDS_EXPORTDISCONNECT → handle_export_disconnect()
  ├── MSG_MDS_EXPORTDIR       → handle_export_dir()
  ├── ... (子树导出相关)
  └── MSG_MDS_IMPORTDIR       → handle_import_dir()

Locker::dispatch(m)                             [Locker.cc:91]
  │
  ├── MSG_MDS_LOCK             → handle_lock()          (MDS 间锁消息)
  ├── MSG_MDS_INODEFILECAPS    → handle_inode_file_caps()
  ├── CEPH_MSG_CLIENT_CAPS    → handle_client_caps()   (客户端 cap 消息)
  ├── CEPH_MSG_CLIENT_CAPRELEASE → handle_client_cap_release()
  └── CEPH_MSG_CLIENT_LEASE    → handle_client_lease()
```

---

## 三、对象关系图

### 3.1 MDSRank 核心子系统关系

```
MDSRank (src/mds/MDSRank.h)
  ├── Server *server             客户端请求处理
  │     └── handle_client_open(), handle_client_unlink(), ...
  ├── MDCache *mdcache           分布式元数据缓存
  │     ├── map<vinodeno_t, CInode*> inodes
  │     ├── map<dentry_key_t, CDentry*> dentries
  │     ├── map<dirfrag_t, CDir*> dirs
  │     └── Migrator *migrator   子树迁移
  ├── Locker *locker             分布式锁管理
  │     └── 锁状态评估、Cap 授权/撤销
  ├── MDLog *mdlog               元数据日志
  │     └── 日志段管理、提交回调
  ├── MDBalancer *balancer       负载均衡
  │     └── 子树负载统计、迁移决策
  └── ScrubStack *scrubstack     元数据校验
        └── CInode/CDir 一致性检查
```

### 3.2 MDRequest 请求生命周期

```
客户端请求到达
  │
  ▼
MDCache::request_start(req)            [MDCache.cc:9921]
  │  → 创建 MDRequestRef (mdr)
  │  → 注册到 active_requests map
  │
  ▼
Server::dispatch_client_request(mdr)
  │  → 根据 op 类型调用 handle_client_xxx()
  │
  ├── 成功完成:
  │     respond_to_request(mdr, rc)
  │     Locker::drop_locks(mdr)
  │     MDCache::request_cleanup(mdr)
  │     MDCache::request_finish(mdr)     [MDCache.cc:10018]
  │       → 从 active_requests 移除
  │
  ├── 需要等待锁:
  │     C_MDS_RetryRequest → finished_queue
  │     → mdcache->dispatch_request(mdr) 重试
  │
  ├── 需要转发到权威 MDS:
  │     MDCache::request_forward(mdr, who, port) [MDCache.cc:10071]
  │       → 发送 MSG_MDS_PEER_REQUEST
  │
  └── 需要从 RADOS 读取:
        → 提交异步读请求，注册 Context 回调
        → 回调中继续处理 (续跑模式)
```

### 3.3 MDRequest 数据结构

```
MDRequestImpl (MDCache.h:76 定义)
  ├── client_request / peer_request / internal_op  (请求来源)
  ├── LockOps locks                (已获取的锁列表)
  ├── vector<CInode*> in            (路径上的 CInode 数组)
  ├── vector<CDentry*> dn           (路径上的 CDentry 数组)
  ├── CInode *target_inode          (目标 CInode)
  ├── int op_type                   (操作类型)
  └── Session *session              (客户端会话)
```

---

## 四、核心代码位置

| 功能 | 文件 | 行号 |
|------|------|------|
| **消息分发入口** | | |
| MDSDaemon::ms_dispatch2 | src/mds/MDSDaemon.cc | 1044 |
| MDSRankDispatcher::ms_dispatch | src/mds/MDSRank.cc | 1033 |
| MDSRank::_dispatch | src/mds/MDSRank.cc | 1054 |
| MDSRank::handle_message | src/mds/MDSRank.cc | 1227 |
| **ProgressThread** | | |
| ProgressThread::entry | src/mds/MDSRank.cc | 995 |
| _advance_queues | src/mds/MDSRank.cc | 1316 |
| **请求开始/结束** | | |
| MDCache::request_start | src/mds/MDCache.cc | 9921 |
| MDCache::request_finish | src/mds/MDCache.cc | 10018 |
| MDCache::request_forward | src/mds/MDCache.cc | 10071 |
| MDCache::dispatch_request | src/mds/MDCache.cc | 10096 |
| **Server 处理** | | |
| Server::dispatch | src/mds/Server.cc | 317 |
| Server::handle_client_request | src/mds/Server.cc | 2603 |
| Server::dispatch_client_request | src/mds/Server.cc | 2752 |
| Server::handle_client_open | src/mds/Server.cc | 4585 |
| Server::handle_client_getattr | src/mds/Server.cc | 4224 |
| Server::handle_client_unlink | src/mds/Server.cc | (search) |
| Server::respond_to_request | src/mds/Server.cc | (search) |
| **锁管理** | | |
| Locker::dispatch | src/mds/Locker.cc | 91 |
| Locker::acquire_locks | src/mds/Locker.cc | 243 |
| Locker::rdlock_start | src/mds/Locker.cc | 1789 |
| Locker::wrlock_start | src/mds/Locker.cc | 1979 |
| Locker::xlock_start | src/mds/Locker.cc | 2135 |
| Locker::drop_locks | src/mds/Locker.cc | 876 |
| **MDCache 消息** | | |
| MDCache::dispatch | src/mds/MDCache.cc | 8451 |
| MDCache::path_traverse | src/mds/MDCache.cc | (search) |
| **MDSRank 子系统** | | |
| MDSRank::handle_message 路由 | src/mds/MDSRank.cc | 1227-1308 |

---

## 五、常见问题与陷阱

### Q1: MDS 为什么使用全局锁而不是像 OSD 一样分片？

**A**: MDS 操作本质上需要目录路径遍历（从根到目标），路径上的每级都可能需要加锁。与 OSD 按 PG 独立处理不同，MDS 的元数据操作天然需要跨目录边界。全局 `mds_lock` 保证了一次只处理一个请求，避免了复杂的锁排序问题。多 MDS 场景下，每个 MDS 负责自己的子树，子树内串行处理。

### Q2: C_MDS_RetryRequest 的续跑模式会不会导致活锁？

**A**: 不会。每次续跑都会重新获取锁，如果锁一直不可用，请求会在队列中等待。续跑机制保证了：
1. 请求不会永远持有 `mds_lock` 等待锁
2. 释放 `mds_lock` 让其他请求有机会释放锁
3. 锁可用时立即续跑，延迟最低

### Q3: 多 MDS 环境下请求转发是如何工作的？

**A**: 当 MDS 发现请求的元数据不属于自己的子树时：
1. 通过 `MDCache::request_forward()` 发送 `MSG_MDS_PEER_REQUEST` 给权威 MDS
2. 权威 MDS 处理后直接回复客户端（不经过转发 MDS）
3. 转发 MDS 会等待客户端重试或超时

### Q4: MDS 到 RADOS 的异步操作如何续跑？

**A**: MDS 需要读写 RADOS 时（如加载 CInode/CDir、写 MDLog），提交异步操作并注册 `Context` 回调。回调在 `Finisher` 线程中执行，触发 `finished_queue`，由 `ProgressThread` 处理。这与锁续跑类似，但触发源不同：

- 锁续跑：由 Locker::eval() 触发
- RADOS 续跑：由 ObjectStore/Objecter 回调触发

### Q5: 为什么 MDS 不使用 Ceph 的 WorkQueue/ThreadPool？

**A**: MDS 元数据操作需要访问共享的 MDCache 状态（CInode/CDentry/CDir），这些状态之间有复杂的依赖关系（路径遍历、锁依赖、缓存一致性）。使用 WorkQueue 会在多个线程间产生大量锁争用，反而降低性能。全局 `mds_lock` + 异步续跑是更简单、更可预测的模型。