---
name: mon-paxos-consensus
description: Ceph MON集群Paxos共识专家。当用户询问Paxos状态机、提案-接受-提交流程、领导者选举、版本存储、Paxos服务框架、lease机制时调用此技能。
---

# MON 集群与 Paxos 共识

## 〇、为什么需要 Paxos？

为什么需要 Paxos 共识算法？Ceph MON 集群维护着集群的"真相"（monmap、osdmap、mdsmap 等），这些数据必须：

```
没有共识的问题:
  MON A 认为 osd.0 是 up
  MON B 认为 osd.0 是 down
  → 客户端收到不一致的 OSDMap
  → 数据写入错误的 OSD 或无法访问
```

Paxos 保证：**只要多数派 MON 存活，集群就能达成一致并继续服务**。

```
Ceph 的 Paxos 方案:
  - 单 Paxos 实例，所有服务共享
  - 每个服务有独立的 key 前缀 (osdmap:, mdsmap:, monmap:)
  - Lease 机制允许 Peon 提供读服务（无需经过 Leader）
  - 提案批处理减少共识开销
```

---

## 一、Paxos 状态机

### 1.1 状态定义 (src/mon/Paxos.h:207-237)

| 状态 | 名称 | 说明 |
|------|------|------|
| `STATE_RECOVERING` | recovering | 恢复阶段（Phase 1 - Prepare/Promise） |
| `STATE_ACTIVE` | active | 空闲；Peon 有有效 lease，Leader 可接受写入 |
| `STATE_UPDATING` | updating | Leader 正在提议新值（Phase 2 - Accept） |
| `STATE_UPDATING_PREVIOUS` | updating-previous | 提议之前未提交的值 |
| `STATE_WRITING` | writing | 正在异步写入磁盘 |
| `STATE_WRITING_PREVIOUS` | writing-previous | 正在异步写入之前的值 |
| `STATE_REFRESH` | refresh | 提交后刷新服务状态 |
| `STATE_SHUTDOWN` | shutdown | 关闭中 |

### 1.2 状态转换

**Leader 路径（赢得选举后）：**
```
leader_init()
  ├─ quorum == 1 → STATE_ACTIVE
  └─ quorum > 1  → STATE_RECOVERING → collect()
```

**恢复阶段（Phase 1）：**
```
STATE_RECOVERING
  ├─ Leader: collect() → 发送 OP_COLLECT 给所有 Peon
  ├─ Peon:   handle_collect() → 接受 PN, 回复 OP_LAST
  └─ Leader: handle_last()
       ├─ 所有 quorum 回复 → STATE_ACTIVE (extend_lease)
       └─ 发现未提交值 → STATE_UPDATING_PREVIOUS → begin()
```

**提议阶段（Phase 2）：**
```
STATE_ACTIVE
  ├─ trigger_propose() → propose_pending()
  └─ begin(value) → STATE_UPDATING
       ├─ Leader 写磁盘，发送 OP_BEGIN 给 Peon
       ├─ Peon: handle_begin() → STATE_UPDATING, 回复 OP_ACCEPT
       └─ Leader: handle_accept()
            └─ 所有 quorum 接受 → commit_start() → STATE_WRITING
                 └─ 异步提交完成 → commit_finish() → STATE_REFRESH
                      └─ do_refresh() → extend_lease() → STATE_ACTIVE
```

**Peon 路径：**
```
peon_init() → STATE_RECOVERING
  └─ handle_collect() → 回复 OP_LAST
  └─ handle_begin() → STATE_UPDATING → 回复 OP_ACCEPT
  └─ handle_commit() → store_state() → do_refresh()
  └─ handle_lease() → STATE_ACTIVE
```

---

## 二、版本存储

### 2.1 KV 存储布局 (src/mon/Paxos.h:38-108)

```
paxos:
  first_committed -> 1        (最老保留版本)
  last_committed  -> 4        (最新提交版本)
  last_pn         -> 42       (最后生成的提案号)
  accepted_pn     -> 42       (最后接受的提案号)
  1               -> value_1  (编码的 MonitorDBStore::Transaction)
  2               -> value_2
  3               -> value_3
  4               -> value_4
```

### 2.2 关键变量 (src/mon/Paxos.h:330-390)

```cpp
// Paxos 类成员
version_t first_committed;        // 最低可用版本
version_t last_committed;         // 最高提交版本
version_t last_pn;                // 最后生成的提案号 (持久化)
version_t accepted_pn;            // 最后接受的提案号 (持久化)
map<int, version_t> peer_first_committed;  // 每个 quorum 成员的最低版本
map<int, version_t> peer_last_committed;   // 每个 quorum 成员的最高版本
```

### 2.3 版本号编码

每个版本的值是一个 `ceph::buffer::list`，包含编码的 `MonitorDBStore::Transaction`。提交时：
1. 将 bufferlist 写入 `paxos:<version>`
2. 解码 bufferlist，原子应用其中的操作

---

## 三、领导者选举

### 3.1 选举策略 (src/mon/ElectionLogic.h:194-199)

| 策略 | 说明 |
|------|------|
| CLASSIC | 最低 rank 获胜 |
| DISALLOW | 某些 rank 不允许成为 leader |
| CONNECTIVITY | 优先选择连通性好的 monitor |

### 3.2 选举流程

```
Phase 1 - Propose:
  Monitor 调用 logic.start() → call_election()
    ├─ 如果我们应该提议: propose_to_peers() → 发送 OP_PROPOSE
    └─ 如果我们应该推迟: _defer_to(who) → 发送 OP_ACK 给更高 rank

Phase 2 - Acknowledge:
  Peon 推迟后发送 OP_ACK 给提议者
  提议者追踪 acks: logic.acked_me

Phase 3 - Victory:
  当提议者获得多数派 ack → 获胜
  message_victory() → 计算 quorum 特性交集
  发送 OP_VICTORY 给所有 quorum 成员
  调用 mon->win_election() → Monitor 进入 LEADER 状态
```

### 3.3 连通性追踪

Monitor 之间通过 `MMonPing` 消息互相探测，`ConnectionTracker` 评分对端连通性，用于 CONNECTIVITY 策略。

---

## 四、Paxos 服务框架

### 4.1 服务层次 (src/mon/Monitor.h:677)

```
Monitor
  └── 单个 Paxos 实例 (paxos)
       └── paxos_service[] 数组:
            ├── [PAXOS_MONMAP]    MonmapMonitor
            ├── [PAXOS_OSDMAP]    OSDMonitor
            ├── [PAXOS_MDSMAP]    MDSMonitor
            ├── [PAXOS_AUTH]      AuthMonitor
            ├── [PAXOS_LOG]       LogMonitor
            ├── [PAXOS_MGR]       MgrMonitor
            ├── [PAXOS_MGRSTAT]   MgrStatMonitor
            ├── [PAXOS_HEALTH]    HealthMonitor
            ├── [PAXOS_CONFIG]    ConfigMonitor
            └── [PAXOS_KV]        KVMonitor
```

### 4.2 必须实现的虚方法 (src/mon/PaxosService.h:288-368)

| 方法 | 用途 | 调用时机 |
|------|------|---------|
| `create_initial()` | 创建初始状态 | 新集群首次提议 |
| `update_from_paxos(bool*)` | 从 KV 解码状态 | 每次 Paxos 提交后 |
| `create_pending()` | 创建新的 pending 状态 | 选举后，在 leader 上 |
| `encode_pending(TransactionRef)` | 编码 pending 变更到事务 | 提议时 |
| `preprocess_query(op)` | 处理只读查询 | 每次 dispatch |
| `prepare_update(op)` | 应用写入到 pending 状态 | 在 leader 上，写操作 |
| `should_propose(delay)` | 决定何时批量提议 | 每次 prepare_update 后 |
| `on_active()` | Paxos 活跃后的钩子 | 每次选举/提交后 |

### 4.3 消息路由 (src/mon/PaxosService.cc:47-151)

```
PaxosService::dispatch(op)
    │
    +-- is_readable(version) → 不可读则 wait_for_readable()
    +-- preprocess_query(op) → 只读操作直接处理
    +-- 如果不是 leader → forward_request_leader(op)
    +-- is_writeable() → 不可写则 wait_for_writeable()
    +-- prepare_update(op) → 修改 pending 状态
    +-- should_propose(delay) → 设置提议定时器（批处理）
    │
    ▼ (定时器触发)
    propose_pending()
```

---

## 五、完整的提议-接受-提交流程

### 5.1 消息类型 (src/mon/MMonPaxos.h:31-37)

| 操作码 | 常量 | 方向 | 用途 |
|--------|------|------|------|
| 1 | `OP_COLLECT` | Leader → Peon | Phase 1: Prepare |
| 2 | `OP_LAST` | Peon → Leader | Phase 1: Promise |
| 3 | `OP_BEGIN` | Leader → Peon | Phase 2: Accept |
| 4 | `OP_ACCEPT` | Peon → Leader | Phase 2: Accepted |
| 5 | `OP_COMMIT` | Leader → Peon | 通知提交 |
| 6 | `OP_LEASE` | Leader → Peon | 延长读 lease |
| 7 | `OP_LEASE_ACK` | Peon → Leader | 确认 lease |

### 5.2 Phase 1: 恢复/Prepare

```
1. Leader 调用 collect(oldpn)     [Paxos.cc:161-226]
   - 生成新提案号: last_pn = (last_pn/100 + 1) * 100 + rank
   - 发送 OP_COLLECT (pn, first_committed, last_committed)

2. Peon 收到 handle_collect(op)   [Paxos.cc:230-331]
   - 如果 pn > accepted_pn: 接受并持久化
   - 回复 OP_LAST (first_committed, last_committed, accepted_pn)
   - 如果 leader 落后，发送未提交的值

3. Leader 收到 handle_last(op)    [Paxos.cc:475-607]
   - 存储共享的值
   - 当 num_last == quorum.size():
     - 发现未提交值 → STATE_UPDATING_PREVIOUS → begin()
     - 否则 → STATE_ACTIVE → extend_lease()
```

### 5.3 Phase 2: Accept

```
4. Leader 调用 begin(value)       [Paxos.cc:620-716]
   - 存储值到 last_committed+1
   - 发送 OP_BEGIN 给所有 Peon

5. Peon 收到 handle_begin(op)     [Paxos.cc:719-776]
   - 验证 pn == accepted_pn
   - 存储值到 last_committed+1
   - 转到 STATE_UPDATING
   - 回复 OP_ACCEPT

6. Leader 收到 handle_accept(op)  [Paxos.cc:779-819]
   - 添加到 accepted 集合
   - 当 accepted == quorum → commit_start()
```

### 5.4 Phase 3: Commit

```
7. Leader 调用 commit_start()     [Paxos.cc:854-895]
   - 创建事务: 更新 last_committed, 解码并应用值的操作
   - 排队异步写入
   - 转到 STATE_WRITING

8. 异步回调 C_Committed::finish() [Paxos.cc:832-844]
   - commit_finish() [Paxos.cc:897-958]:
     - last_committed++
     - 发送 OP_COMMIT 给所有 Peon
     - 转到 STATE_REFRESH
     - do_refresh() 更新所有服务
     - extend_lease()
     - finish_round() → STATE_ACTIVE

9. Peon 收到 handle_commit(op)    [Paxos.cc:961-979]
   - store_state() 持久化值
   - do_refresh() 更新服务
```

### 5.5 Lease 机制

```
10. Leader 调用 extend_lease()    [Paxos.cc:981-1030]
    - lease_expire = now + mon_lease
    - 发送 OP_LEASE 给所有 Peon

11. Peon 收到 handle_lease(op)    [Paxos.cc:1109-1157]
    - 更新 lease_expire
    - 转到 STATE_ACTIVE
    - 回复 OP_LEASE_ACK

12. Leader 收到 handle_lease_ack  [Paxos.cc:1159-1184]
    - 所有 quorum 确认后取消超时
```

---

## 六、客户端请求路由

### 6.1 请求入口 (src/mon/Monitor.cc:4688-4936)

```
Monitor::dispatch_op(op)
    │
    ├── MSG_MON_COMMAND → handle_command()
    │   ├─ 解析 JSON 命令，查找 MonCommand
    │   ├─ 检查权限
    │   └─ 不是 leader → forward_request_leader()
    │
    ├── CEPH_MSG_MON_GET_OSDMAP, MSG_OSD_BOOT, etc.
    │   → paxos_service[PAXOS_OSDMAP]->dispatch(op)
    │
    ├── MSG_MDS_BEACON → paxos_service[PAXOS_MDSMAP]->dispatch(op)
    │
    ├── MSG_MON_PAXOS → paxos->dispatch(op)
    │
    └── MSG_MON_ELECTION → elector.dispatch(op)
```

### 6.2 非 Leader 请求转发 (src/mon/Monitor.cc:4224-4266)

```
Peon 收到写请求:
  1. PaxosService::dispatch() 检查 mon.is_leader()
  2. 不是 leader → mon.forward_request_leader(op)
  3. 将原始消息包装为 MForward（含 session、capabilities、features）
  4. 发送给当前 leader
  5. Leader 通过 handle_forward() 接收，重建原始消息并重新 dispatch
```

### 6.3 完整命令路径

```
ceph osd crush add ...
    │
    ▼
客户端发送 MSG_MON_COMMAND 到任意 Monitor
    │
    ▼
Monitor::dispatch_op() → handle_command()
    │
    ▼ (在 leader 上)
命令路由到 OSDMonitor (module name = "osd")
    │
    ▼
OSDMonitor::dispatch()
    │
    ├── is_readable() 检查
    ├── preprocess_query() → 处理读
    ├── prepare_update() → 修改 pending_inc
    ├── should_propose(delay) → 设置提议定时器
    │
    ▼ (定时器触发)
PaxosService::propose_pending()
    │
    ├── encode_pending() → 编码 pending_inc 到事务
    └── paxos.trigger_propose()
    │
    ▼
完整 Paxos 共识: COLLECT → LAST → BEGIN → ACCEPT → COMMIT
    │
    ▼
commit_finish() → do_refresh() → OSDMonitor::update_from_paxos()
    │
    └── 应用提交的 incremental 到 osdmap
    │
    ▼
C_Committed::finish() → _active() → on_active()
    │
    └── 唤醒等待者，回复客户端
```

---

## 七、关键代码位置

| 功能 | 文件 | 行号 |
|------|------|------|
| Paxos 状态枚举 | src/mon/Paxos.h | 207-237 |
| Paxos 版本变量 | src/mon/Paxos.h | 330-390 |
| Paxos 存储布局文档 | src/mon/Paxos.h | 38-108 |
| collect() - Phase 1 开始 | src/mon/Paxos.cc | 161-226 |
| handle_collect() - Peon 承诺 | src/mon/Paxos.cc | 230-331 |
| handle_last() - Leader 处理承诺 | src/mon/Paxos.cc | 475-607 |
| begin() - Phase 2 开始 | src/mon/Paxos.cc | 620-716 |
| handle_begin() - Peon 接受值 | src/mon/Paxos.cc | 719-776 |
| handle_accept() - Leader 计数 | src/mon/Paxos.cc | 779-819 |
| commit_start() - 写磁盘 | src/mon/Paxos.cc | 854-895 |
| commit_finish() - 提交后 | src/mon/Paxos.cc | 897-958 |
| handle_commit() - Peon 提交 | src/mon/Paxos.cc | 961-979 |
| extend_lease() | src/mon/Paxos.cc | 981-1030 |
| handle_lease() - Peon 获取 lease | src/mon/Paxos.cc | 1109-1157 |
| propose_pending() | src/mon/Paxos.cc | 1537-1560 |
| get_new_proposal_number() | src/mon/Paxos.cc | 1271-1302 |
| leader_init() / peon_init() | src/mon/Paxos.cc | 1355-1398 |
| MMonPaxos 操作类型 | src/mon/MMonPaxos.h | 31-37 |
| Elector dispatch | src/mon/Elector.cc | 675-778 |
| handle_propose | src/mon/Elector.cc | 277-321 |
| handle_victory | src/mon/Elector.cc | 372-400 |
| Monitor::win_election | src/mon/Monitor.cc | 2326-2411 |
| Monitor::lose_election | src/mon/Monitor.cc | 2413-2439 |
| PaxosService::dispatch | src/mon/PaxosService.cc | 47-151 |
| PaxosService::propose_pending | src/mon/PaxosService.cc | 206-270 |
| PaxosService 虚方法 | src/mon/PaxosService.h | 288-368 |
| Monitor::dispatch_op | src/mon/Monitor.cc | 4688-4936 |
| Monitor::forward_request_leader | src/mon/Monitor.cc | 4224-4266 |

---

## 八、关键设计特性

### 8.1 单 Paxos 实例

所有服务共享一个 Paxos 实例。每个服务有独立的 key 前缀（如 `osdmap:`, `mdsmap:`, `monmap:`），版本独立追踪。

### 8.2 Lease 读优化

Peon 在 lease 有效期内可以直接服务读请求，无需经过 Leader。这是避免完整共识开销的关键优化。

### 8.3 提案批处理

`should_propose()` 实现阻尼 -- 快速连续的更新被批量合并，通过 `paxos_propose_interval` 和 `paxos_min_wait` 参数控制。

### 8.4 原子状态应用

通过 Paxos 提议的值是编码的 `MonitorDBStore::Transaction`。提交时，Paxos 原子写入编码事务并应用其操作，确保服务状态和 Paxos 版本始终一致。

---

## 九、常见问题与陷阱

### Q1: 提案号 (PN) 如何保证唯一性和单调性？

**A**: PN 公式为 `(last_pn/100 + 1) * 100 + rank`。每 100 为一轮，rank 保证同一轮内不同 monitor 的 PN 不同。这确保了 PN 的单调递增和全局唯一。

### Q2: 为什么需要 `first_committed` 和 `last_committed` 的范围追踪？

**A**: 不同 monitor 可能在不同时间提交，导致版本范围不一致。Paxos 在恢复阶段比较各成员的版本范围，确保 leader 拥有所有已提交的值，并将缺失的值同步给落后的成员。

### Q3: Lease 超时后会发生什么？

**A**: Peon 的 lease 超时后，状态从 ACTIVE 变为 RECOVERING，不再服务读请求。Leader 检测到 lease_ack 超时后会重新 extend_lease。如果 Leader 故障，剩余 monitor 会触发新的选举。

### 参考文献

1. **"Paxos Made Simple"** - Leslie Lamport (2001)
2. **Monitor 开发者文档**: https://docs.ceph.com/en/latest/dev/mon-bootstrap/
3. **源码**: `src/mon/Monitor.cc`, `src/mon/Paxos.cc`, `src/mon/Elector.cc`
4. **"Paxos Made Live"** - Chandra et al. (Google, 2007) — Paxos 工业实现的经验"
