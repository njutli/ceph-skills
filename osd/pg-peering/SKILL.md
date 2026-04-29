---
name: osd-pg-peering
description: Ceph OSD Placement Group Peering状态机专家。当用户询问PG状态转换、Peering流程(GetInfo/GetLog/GetMissing)、权威OSD选择、PGLog合并、恢复激活流程时调用此技能。
---

# OSD Placement Group Peering 状态机

## 〇、为什么需要 Peering？

为什么需要 Peering 流程？在分布式存储中，数据有多个副本分布在不同的 OSD 上：

```
没有 Peering 的问题:
  osd.0 (primary) 宕机后恢复
  osd.1 在 osd.0 宕机期间接收了新的写入
  → osd.0 的数据是旧的，osd.1 的数据是新的
  → 哪个是权威数据源？
  → 如何安全地同步数据而不丢失写入？
```

Peering 的核心目标是：**让 PG 的所有副本就"当前权威数据"达成一致，并安全地恢复不一致的副本**。

```
Peering 解决的问题:
  1. 确定权威数据源 (哪个 OSD 有最新的数据)
  2. 构建权威日志 (合并所有副本的日志)
  3. 发现缺失对象 (哪些对象在某些副本上缺失)
  4. 安全激活 (确保数据一致后才接受客户端 I/O)
  5. 恢复/Backfill (同步落后副本)
```

---

## 一、完整状态机

### 1.1 状态层次结构 (src/osd/PeeringState.h:554-587)

```
PeeringMachine (root)
├── Initial
├── Crashed
├── Reset
├── Started
│   ├── Start
│   │   └── (→ Primary 或 Stray)
│   ├── Primary
│   │   ├── WaitActingChange
│   │   ├── Peering
│   │   │   ├── GetInfo          (初始子状态)
│   │   │   ├── GetLog
│   │   │   ├── GetMissing
│   │   │   ├── WaitUpThru
│   │   │   ├── Incomplete
│   │   │   └── Down
│   │   └── Active
│   │       ├── Activating       (初始子状态)
│   │       ├── Clean
│   │       ├── Recovered
│   │       ├── Backfilling
│   │       ├── WaitRemoteBackfillReserved
│   │       ├── WaitLocalBackfillReserved
│   │       ├── NotBackfilling
│   │       ├── NotRecovering
│   │       ├── Recovering
│   │       ├── WaitRemoteRecoveryReserved
│   │       └── WaitLocalRecoveryReserved
│   ├── ReplicaActive
│   │   ├── RepNotRecovering     (初始子状态)
│   │   ├── RepRecovering
│   │   ├── RepWaitBackfillReserved
│   │   └── RepWaitRecoveryReserved
│   ├── Stray
│   └── ToDelete
│       ├── WaitDeleteReserved   (初始子状态)
│       └── Deleting
└── Crashed
```

### 1.2 状态转换表

| 当前状态 | 触发事件 | 下一状态 | 说明 |
|---------|---------|---------|------|
| Initial | Initialize | Reset | OSD 启动 |
| Reset | AdvMap | Started | 收到新 OSDMap |
| Started | MakePrimary | Primary | 成为主 OSD |
| Started | MakeStray | Stray | 成为从 OSD |
| Primary | 进入 | Peering/GetInfo | 初始子状态 |
| GetInfo | GotInfo | GetLog | 收集完所有 peer info |
| GetLog | NeedActingChange | WaitActingChange | 需要变更 acting set |
| GetLog | IsIncomplete | Incomplete | 找不到权威数据 |
| GetLog | 成功 | GetMissing | 日志合并完成 |
| GetMissing | NeedUpThru | WaitUpThru | 需要更新 up_thru |
| GetMissing | 成功 | Active/Activating | 激活 |
| WaitUpThru | ActMap | Active/Activating | up_thru 满足条件 |
| Activating | AllReplicasRecovered | Recovered | 恢复完成 |
| Activating | DoRecovery | WaitLocalRecoveryReserved | 需要恢复 |
| Activating | RequestBackfill | WaitLocalBackfillReserved | 需要 backfill |
| WaitLocalRecoveryReserved | LocalRecoveryReserved | WaitRemoteRecoveryReserved | 本地保留完成 |
| WaitRemoteRecoveryReserved | AllRemotesReserved | Recovering | 远端保留完成 |
| Recovering | AllReplicasRecovered | Recovered | 所有副本恢复完成 |
| Recovered | GoClean | Clean | 标记为 clean |
| WaitLocalBackfillReserved | LocalBackfillReserved | WaitRemoteBackfillReserved | 本地 backfill 保留 |
| WaitRemoteBackfillReserved | AllBackfillsReserved | Backfilling | 开始 backfill |
| Backfilling | Backfilled | Clean | backfill 完成 |
| ReplicaActive | RequestRecoveryPrio | RepWaitRecoveryReserved | 副本需要恢复 |
| RepRecovering | RecoveryDone | RepNotRecovering | 副本恢复完成 |

---

## 二、触发 Peering 的事件

### 2.1 OSD 启动

```cpp
// src/osd/PG.cc:1075-1141
PG::read_state()
  → init_from_disk_state()
  → handle_initialize()
    → Initial 状态
```

OSD 启动时从磁盘读取 PG 状态，调用 `handle_initialize()` 进入 Initial 状态。

### 2.2 OSDMap 变更

```cpp
// src/osd/PeeringState.cc:572-596
PeeringState::advance_map()
```

当 OSD 收到新的 OSDMap 时触发。关键判断条件：

```cpp
// src/osd/PeeringState.cc:665-695
bool should_restart_peering() {
  // 1. acting/up/primary 发生变化
  // 2. 本地 OSD 从 down → up
}
```

### 2.3 OSD 恢复

```cpp
// src/osd/PeeringState.cc:202-229
PeeringState::check_recovery_sources()
```

之前宕机的 OSD 重新上线时，验证计划恢复的对端是否仍可用。

### 2.4 CRUSH 映射变更

```cpp
// src/osd/PeeringState.cc:698-881
PeeringState::start_peering_interval()
```

CRUSH 映射变化时：
- 清除所有 active/peered/down 状态
- 清除 primary 状态
- 调用 `on_new_interval()`

---

## 三、Peering 核心阶段

### 3.1 Phase 1: GetInfo

**目的**: 从所有对端收集 `pg_info_t`。

```
Primary 发送 pg_query_t::INFO 查询
  ↓
所有 peer 回复 MOSDPGNotify2 (含 pg_info_t)
  ↓
Primary 存储到 peer_info 映射表
  ↓
所有 info 收集完成 → GotInfo 事件 → GetLog
```

**关键代码**: `proc_replica_notify()` (PeeringState.cc:451-495)

### 3.2 Phase 2: GetLog

**目的**: 确定权威日志并构建主日志。

```
1. choose_acting() (PeeringState.cc:2521-2733)
   ├─ find_best_info() → 确定 auth_log_shard (权威日志分片)
   ├─ 计算期望的 acting set
   ├─ 确定 backfill targets
   └─ 确定 async recovery targets

2. 向 auth_log_shard 请求 LOG/FULLLOG

3. 收到日志后:
   ├─ proc_master_log() 或 proc_replica_log()
   ├─ merge_log() 合并日志
   ├─ rewind_divergent_log() 回滚分歧日志
   └─ 更新 peer_missing

4. 结果:
   ├─ acting set 需要变更 → WaitActingChange
   ├─ 找不到合适数据 → Incomplete
   └─ 成功 → GetMissing
```

### 3.3 Phase 3: GetMissing

**目的**: 发现所有 acting_recovery_backfill 分片上的缺失对象。

```
1. 向 peer 请求 FULLLOG 获取 pg_missing_t
2. build_might_have_unfound() 构建可能有 unfound 对象的 OSD 集合
3. discover_all_missing() 查询所有候选 peer
4. 结果:
   ├─ need_up_thru 设置 → WaitUpThru
   └─ 否则 → 激活
```

### 3.4 Phase 4: WaitUpThru (可选)

**目的**: 等待 OSD 的 `up_thru` epoch 在 OSDMap 中更新。

```
Primary 的 up_thru 必须 >= same_interval_since 才能激活
如果不满足，等待新 OSDMap 反映更新的 up_thru
```

### 3.5 Phase 5: 激活

```
Activate 事件触发 → Active/Activating

activate() (PeeringState.cc:2910-3264):
  1. 更新 last_epoch_started 和 last_interval_started
  2. 设置 pg_committed_to = info.last_update
  3. 向所有副本发送 MOSDPGLog (含缺失集合)
  4. 设置 PG_STATE_ACTIVATING
  5. roll_forward() 所有日志条目
  6. pl->on_activate():
     - 重放日志 (roll_forward)
     - 释放阻塞的操作
     - 排队恢复 (如果需要)
```

---

## 四、权威 OSD 选择算法

### 4.1 find_best_info() (PeeringState.cc:1689-1782)

```
排序标准 (按优先级):
1. last_epoch_started >= max_last_epoch_started (不能太旧)
2. 不是 incomplete (last_backfill 必须是 max)
3. 复制池: 偏好更新的 last_update
   EC 池: 偏好更旧的 last_update
4. 如果平局: 偏好更长的 log_tail (更多历史)
5. 如果平局: 偏好没有缺失对象的 OSD
6. 如果平局: 偏好当前 primary (pg_whoami)
```

### 4.2 select_replicated_primary() (PeeringState.cc:1852-1915)

```
1. 从 up_primary (CRUSH 指定的 primary) 开始
2. 如果 up_primary 是 incomplete 或需要 backfill → 回退到 auth_log_shard
3. 如果 up_primary 缺失对象过多 (超过 osd_force_auth_primary_missing_objects) → 回退
```

---

## 五、关键数据结构

### 5.1 pg_shard_t (src/osd/osd_types.h:179-208)

```cpp
struct pg_shard_t {
    int32_t osd;         // OSD ID
    shard_id_t shard;    // 分片 ID (复制池为 NO_SHARD)
};
```

### 5.2 pg_info_t (src/osd/osd_types.h:3070-3143)

```cpp
struct pg_info_t {
    spg_t pgid;
    eversion_t last_update;       // 最后应用的版本
    eversion_t last_complete;     // PG 完全同步到的版本
    epoch_t last_epoch_started;   // 最后启动的 epoch
    epoch_t last_interval_started;
    version_t last_user_version;
    eversion_t log_tail;          // 最老的日志条目
    hobject_t last_backfill;      // backfill 位置
    interval_set<snapid_t> purged_snaps;
    pg_stat_t stats;
    pg_history_t history;         // epoch 追踪
};
```

### 5.3 pg_log_t (src/osd/osd_types.h:4679+)

```cpp
struct pg_log_t {
    eversion_t head;              // 最新条目
    eversion_t tail;              // 最老条目之前的版本
    eversion_t can_rollback_to;   // 可回滚边界
    list<pg_log_entry_t> log;     // 日志条目
    list<pg_log_dup_t> dups;      // 重复检测条目
};
```

### 5.4 pg_log_entry_t (src/osd/osd_types.h:4475-4623)

```cpp
struct pg_log_entry_t {
    enum { MODIFY, CLONE, DELETE, LOST_REVERT, LOST_DELETE, LOST_MARK, PROMOTE, CLEAN, ERROR };
    ObjectModDesc mod_desc;   // 回滚描述
    hobject_t soid;           // 对象 ID
    osd_reqid_t reqid;        // 请求 ID (重复检测)
    eversion_t version, prior_version, reverting_to;
    version_t user_version;
    int32_t op;               // 操作类型
    shard_id_set written_shards;  // EC: 写入的分片
};
```

---

## 六、PGLog 在 Peering 中的使用

### 6.1 日志合并

```cpp
// PeeringState.cc:3292-3301
merge_log()
  → 合并对端日志到主日志
  → 识别并回滚分歧条目
```

### 6.2 分歧日志回滚

```cpp
// PeeringState.cc:3303-3310
rewind_divergent_log()
  → 回滚主节点自身的分歧条目
  → 对每个分歧条目调用 rollback()
```

### 6.3 主日志处理

```cpp
// PeeringState.cc:3384-3543
proc_master_log()
  → 合并权威日志到本地日志
  → 处理 EC 部分写入
  → 更新 peer_missing
```

### 6.4 日志前滚

```cpp
// PeeringState.cc:3261-3263
pg_log.roll_forward(&info, rollbacker.get())
  → 激活后前滚所有条目使其可见
```

---

## 七、激活后的恢复流程

### 7.1 恢复状态链

```
Activating
  ├─ 无需恢复 → Recovered → Clean
  ├─ 需要恢复 → WaitLocalRecoveryReserved
  │                → WaitRemoteRecoveryReserved
  │                    → Recovering
  │                        → AllReplicasRecovered → Recovered → Clean
  └─ 需要 Backfill → WaitLocalBackfillReserved
                         → WaitRemoteBackfillReserved
                             → Backfilling
                                 → Backfilled → Clean
```

### 7.2 副本路径

```
Stray → (ActMap 含正确角色) → ReplicaActive/RepNotRecovering
  ├─ RequestRecoveryPrio → RepWaitRecoveryReserved
  │                            → RemoteRecoveryReserved
  │                                → RepRecovering
  │                                    → RecoveryDone → RepNotRecovering
  └─ RequestBackfillPrio → RepWaitBackfillReserved
                               → RemoteBackfillReserved
                                   → RepRecovering
                                       → RecoveryDone → RepNotRecovering
```

---

## 八、关键代码位置

| 功能 | 文件 | 行号 |
|------|------|------|
| 状态机定义 | src/osd/PeeringState.h | 595-1402 |
| 状态层次注释 | src/osd/PeeringState.h | 554-587 |
| find_best_info() | src/osd/PeeringState.cc | 1689-1782 |
| choose_acting() | src/osd/PeeringState.cc | 2521-2733 |
| select_replicated_primary() | src/osd/PeeringState.cc | 1852-1915 |
| activate() | src/osd/PeeringState.cc | 2910-3264 |
| merge_log() | src/osd/PeeringState.cc | 3292-3301 |
| proc_master_log() | src/osd/PeeringState.cc | 3384-3543 |
| proc_replica_log() | src/osd/PeeringState.cc | 3545-3574 |
| rewind_divergent_log() | src/osd/PeeringState.cc | 3303-3310 |
| start_peering_interval() | src/osd/PeeringState.cc | 698-881 |
| should_restart_peering() | src/osd/PeeringState.cc | 665-695 |
| try_mark_clean() | src/osd/PeeringState.cc | 3675-3713 |
| PG::read_state() | src/osd/PG.cc | 1075-1141 |
| pg_info_t 定义 | src/osd/osd_types.h | 3070-3143 |
| pg_log_t 定义 | src/osd/osd_types.h | 4679+ |
| pg_log_entry_t 定义 | src/osd/osd_types.h | 4475-4623 |
| pg_shard_t 定义 | src/osd/osd_types.h | 179-208 |

---

## 九、常见问题与陷阱

### Q1: Peering 和 Recovery 的区别？

**A**:
- **Peering**: 协商阶段，确定权威数据源、构建主日志、发现缺失对象。此阶段**不接受客户端 I/O**。
- **Recovery**: 执行阶段，根据 Peering 的结果同步缺失对象。此阶段 PG 已进入 Active 状态，**可以接受客户端 I/O** (但可能有性能影响)。

### Q2: 什么情况下 PG 会进入 Incomplete 状态？

**A**: 当所有已知对端的 `pg_info_t` 都不满足以下条件时：
- `last_epoch_started >= max_last_epoch_started` (太旧)
- `last_backfill == max` (incomplete)
- `last_update >= min_last_update_acceptable`

这通常意味着数据丢失，需要人工干预。

### Q3: WaitUpThru 状态的作用？

**A**: 确保 Primary OSD 的 `up_thru` epoch 足够新。这是防止"脑裂"的安全机制——如果 primary 在宕机后恢复，必须等待 MON 确认它确实已经重新上线，才能激活 PG。

### 相关技能

- [OSD I/O 路径](../osd-io-path/SKILL.md) — Peering 后 PG 进入 Active，恢复操作与客户端 I/O 走同一调度器
- [Scrubbing](../scrub/SKILL.md) — Scrub 使用同一 PG 状态机，与 Peering 共享状态变量
- [Paxos 共识](../../mon/mon-paxos/SKILL.md) — OSDMap 变更通过 Paxos 推进，触发 Peering

### 参考文献

1. **PG 开发者文档**: https://docs.ceph.com/en/latest/dev/peering/
2. **源码**: `src/osd/PG.h`, `src/osd/PeeringState.h`, `src/osd/PeeringState.cc`
3. **Ceph 论文**: "Ceph: A Scalable, High-Performance Distributed File System" (OSDI 2006)"
