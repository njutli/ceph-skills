---
name: osd-scrub-mechanism
description: Ceph OSD Scrubbing机制专家。当用户询问Scrub原理、浅层/深层Scrub区别、Scrub状态机、副本哈希比较、Scrub调度、数据一致性校验时调用此技能。
---

# OSD Scrubbing 机制

## 〇、为什么需要 Scrubbing？

为什么需要 Scrubbing？在大规模存储集群中，数据损坏是不可避免的：

```
没有 Scrubbing 的问题:
  磁盘位翻转 (Bit Rot): 宇宙射线、磁盘老化导致静默数据损坏
  软件 Bug: 写入错误、恢复不完整导致副本不一致
  硬件故障: 控制器缓存损坏、传输错误
  
  没有 Scrubbing = 损坏的数据永久存在，直到用户读取时才发现
```

Scrubbing 的核心目标是：**主动检测并修复数据损坏，在用户发现之前解决问题**。

```
Scrubbing 方案:
  定期扫描 PG 的所有副本
  计算并比较每个对象的 CRC32C 校验和
  发现不一致时自动修复 (从权威副本复制)
  记录并报告所有检测到的错误
```

---

## 一、浅层 Scrub vs 深层 Scrub

### 1.1 对比

| 方面 | 浅层 Scrub | 深层 Scrub |
|------|-----------|-----------|
| **检查内容** | 对象元数据 (存在性、大小、属性、digest) | 元数据 + 读取所有数据字节重新计算 CRC32C |
| **I/O 开销** | 低 (仅读取元数据) | 高 (读取所有对象数据) |
| **频率** | 高 (默认 ~24h) | 低 (默认 ~1 周) |
| **检测能力** | 缺失对象、大小不匹配、属性损坏 | 位翻转、实际数据损坏、EC 分片不一致 |
| **PG 状态标志** | `PG_STATE_SCRUBBING` | `PG_STATE_SCRUBBING \| PG_STATE_DEEP_SCRUB` |

### 1.2 枚举定义 (src/osd/osd_types.h:2261-2262)

```cpp
enum class scrub_level_t : bool { shallow = false, deep = true };
enum class scrub_type_t : bool { not_repair = false, do_repair = true };
```

---

## 二、Scrub 状态机

### 2.1 顶层状态 (src/osd/scrubber/scrub_machine.h:267-288)

```
NotActive (静止状态)
  │
  ├── PrimaryActive (我们是主 PG)
  │     ├── PrimaryIdle (准备好接收新 scrub 请求)
  │     │      └─ [StartScrub 事件] →
  │     │
  │     └── Session (处理单个 scrub 会话)
  │           ├── ReservingReplicas (从副本 OSD 获取资源)
  │           │      └─ [RemotesReserved 事件] →
  │           │
  │           └── ActiveScrubbing (子状态机，见 2.2)
  │
  └── ReplicaActive (我们是副本 PG)
        ├── ReplicaIdle (准备好接收主的 scrub 请求)
        │      └─ [StartReplica 事件] →
        │
        └── ReplicaActiveOp (处理单个 chunk map 请求)
              ├── ReplicaWaitUpdates (等待 active_pushes 清除)
              └── ReplicaBuildingMap (构建 scrub map chunk)
```

### 2.2 ActiveScrubbing 子状态机 (仅主)

```
ActiveScrubbing
  │
  ├── PendingTimer (chunk 之间休眠，或重新排队)
  │      └─ [InternalSchedScrub 事件] →
  │
  ├── NewChunk (选择 chunk，验证可用性)
  │      ├─ [ChunkIsBusy 事件] → RangeBlocked
  │      │                           └─ [Unblocked 事件] →
  │      └─ [SelectedChunkFree 事件] →
  │
  ├── WaitPushes (等待 active_pushes 清除 — 恢复进行中)
  │      └─ [ActivePushesUpd 事件，当 pushes==0] →
  │
  ├── WaitLastUpdate (等待 last_update_applied — EC RMW 读取往返)
  │      └─ [InternalAllUpdates 事件] →
  │
  ├── BuildMap (为此 chunk 构建本地 scrub map)
  │      ├─ [IntBmPreempted 事件] → Drain ReplMaps
  │      │                                └─ [GotReplicas 事件] →
  │      └─ [IntLocalMapDone 事件] →
  │
  ├── WaitReplicas (等待所有副本 map 到达)
  │      └─ [GotReplicas 事件，所有 map 可用] →
  │           maps_compare_n_cleanup() →
  │
  └── WaitDigestUpdate (等待 digest 修复事务完成)
           └─ [DigestUpdate 事件，全部完成] →
                如果 m_end.is_max() → [ScrubFinished 事件] → PrimaryIdle
                否则 → [NextChunk 事件] → PendingTimer
```

### 2.3 主流程关键转换

```
1. PrimaryIdle 收到 StartScrub → ReservingReplicas
2. ReservingReplicas 收到所有副本的 ReplicaGrant → ActiveScrubbing
3. ActiveScrubbing 入口 → PendingTimer → NewChunk
4. NewChunk → WaitPushes → WaitLastUpdate → BuildMap
5. BuildMap → WaitReplicas (本地 map + 所有副本 map 到达后)
6. WaitReplicas → maps_compare_n_cleanup() → WaitDigestUpdate
7. WaitDigestUpdate → 如果还有 chunk: NextChunk → PendingTimer
                      否则: ScrubFinished → PrimaryIdle
```

---

## 三、副本间哈希比较

### 3.1 比较流程

```
Step 1: 每个 OSD 构建 ScrubMap
  Primary 和每个副本独立扫描分配范围内的本地对象
  每个 map 条目 (ScrubMap::object) 包含:
    - 对象属性 (object_info, snapset)
    - 对象大小
    - 数据 digest (CRC32C) — 深层 scrub 时计算，浅层时读取存储的 digest
    - OMAP digest (CRC32C)
    - 错误标志 (read_error, stat_error, ec_hash_mismatch, ...)

Step 2: Primary 向副本请求 maps
  发送 MOSDRepScrub 消息，指定 chunk 范围 [start, end)

Step 3: 副本构建并返回 maps
  每个副本为请求范围构建自己的 ScrubMap，通过 MOSDRepScrubMap 返回

Step 4: Primary 合并并比较 maps
  1. 将所有 maps 合并到 this_chunk->received_maps
  2. 构建"权威集合" — 所有 map 中提到的对象的并集
  3. 对每个对象，选择权威来源 (select_auth_object()):
     - 优先 Primary 的副本
     - 选择 object_info_t 版本最新的分片
     - 版本相同时，优先 digest 信息更多的
     - 拒绝有读错误、统计错误、EC 不匹配的分片
  4. 将权威对象与所有其他副本比较 (compare_obj_details()):
     - 大小检查: 存储的大小是否匹配？
     - Digest 检查 (深层 scrub): 计算的 CRC32C 是否匹配权威 digest？
     - OMAP digest 检查: OMAP CRC32C 是否匹配？
     - 属性检查: 对象属性是否匹配？
     - EC 特定: 对纠删码池，执行 EC CRC 编码/解码验证
```

### 3.2 EC 特定深层 Scrub

对纠删码池的深层 scrub：
1. 从所有可用分片收集 CRC32C digest
2. 填充较短的分片以匹配权威分片长度
3. 使用 EC 解码重建缺失的数据分片
4. 重新编码并验证校验分片匹配预期 CRC
5. 如果单个分片解码不正确，标记为损坏

---

## 四、Scrub 调度机制

### 4.1 三层架构

```
OSD 层 (OsdScrub)
  └─ 管理 OSD 级资源、环境条件、基于 tick 的调度器
       │
       ▼
ScrubQueue
  └─ 按 not_before 时间排序的 SchedEntry 优先级队列
       │
       ▼
PG 层 (ScrubJob)
  └─ 每个 PG 有一个 ScrubJob，含两个 SchedTarget (浅层和深层)
```

### 4.2 紧急度级别 (优先级从低到高)

| 紧急度 | 说明 |
|--------|------|
| `periodic_regular` | 标准调度的浅层/深层 scrub |
| `must_scrub` | 无效的 last-scrub 时间戳；高优先级浅层 scrub |
| `after_repair` | 恢复完成后触发；始终深层，不自动修复 |
| `repairing` | 之前的浅层 scrub 发现错误；带修复标志的深层 scrub |
| `operator_requested` | 管理员手动请求的 scrub |
| `must_repair` | 管理员请求的修复 (最高优先级) |

### 4.3 调度流程

```
1. PG 成为 PrimaryActive → schedule_scrub_with_osd() 注册 PG 到 OSD 队列

2. update_targets() 计算 not_before 时间:
   - last_scrub_stamp + osd_scrub_min_interval (浅层)
   - last_deep_scrub_stamp + osd_deep_scrub_interval (深层)
   - 随机化防止惊群效应

3. OSD tick 调用 OsdScrub::initiate_scrub():
   - 检查 OSD 级限制 (并发、CPU 负载、时间窗口、恢复)
   - 调用 pop_ready_entry() 找到最高优先级的合格目标
   - 调用 initiate_a_scrub() 锁定 PG 并调用 start_scrub_session()

4. start_scrub_session() 验证 PG 状态 (active+clean，非快照修剪)
```

### 4.4 环境限制 (src/osd/scrubber_common.h:95-110)

```cpp
struct OSDRestrictions {
    bool max_concurrency_reached;   // 此 OSD 上并发 scrub 过多
    bool random_backoff_active;     // 随机退避
    bool cpu_overloaded;            // 负载平均值超过 osd_scrub_load_threshold
    bool restricted_time;           // 不在允许的 scrub 时间/日期内
    bool recovery_in_progress;      // OSD 正在恢复且 osd_scrub_during_recovery 为 false
};
```

---

## 五、关键数据结构

### 5.1 ScrubMap (src/osd/osd_types.h:6581-6642)

```cpp
struct ScrubMap {
    map<hobject_t, object> objects;  // 对象 → ScrubMap::object
    eversion_t valid_through;        // 此 map 有效的 PG 版本
};

struct ScrubMap::object {
    map<string, buffer::ptr> attrs;  // 对象属性
    uint64_t size;                   // 对象大小
    uint32_t digest;                 // 数据 digest (CRC32C)
    uint32_t omap_digest;            // OMAP digest (CRC32C)
    uint32_t errors;                 // 错误标志
};
```

### 5.2 scrub_chunk_t (src/osd/scrubber/scrub_backend.h:273-301)

```cpp
struct scrub_chunk_t {
    map<pg_shard_t, ScrubMap> received_maps;  // 所有副本 map + 主自己的 map
    set<hobject_t> all_chunk_objects;         // 所有 maps 中对象的并集
    map<hobject_t, pg_shard_t> authoritative; // 损坏对象到其对等方的映射
    vector<inconsistent_obj_wrapper> m_inconsistent_objs;
    ScrubCounterSet m_error_counts;           // 浅层/深层错误计数器
    map<pg_shard_t, vector<uint32_t>> m_ec_digest_map;  // EC 特定 CRC 数据
};
```

### 5.3 ScrubJob (src/osd/scrubber/scrub_job.h:123-384)

```cpp
struct ScrubJob {
    SchedTarget shallow_target;  // 浅层 scrub 目标
    SchedTarget deep_target;     // 深层 scrub 目标
    bool blocked = false;        // 是否被阻塞
    bool registered = false;     // 是否已注册到 OSD 队列
};
```

---

## 六、关键代码位置

| 功能 | 文件 | 行号 |
|------|------|------|
| FSM 状态声明 | src/osd/scrubber/scrub_machine.h | 42-262 (事件), 267-288 (状态) |
| FSM 状态实现 | src/osd/scrubber/scrub_machine.cc | 129-1100 |
| scrub_compare_maps() | src/osd/scrubber/scrub_backend.cc | 253-275 |
| select_auth_object() | src/osd/scrubber/scrub_backend.cc | 443-552 |
| compare_obj_in_maps() | src/osd/scrubber/scrub_backend.cc | 963-1056 |
| match_in_shards() | src/osd/scrubber/scrub_backend.cc | 1226-1468 |
| build_scrub_map_chunk() | src/osd/scrubber/pg_scrubber.cc | 1443-1500 |
| initiate_scrub() | src/osd/scrubber/osd_scrub.cc | 98-149 |
| restrictions_on_scrubbing() | src/osd/scrubber/osd_scrub.cc | 186-217 |
| adjust_shallow_schedule() | src/osd/scrubber/scrub_job.cc | 114-152 |
| adjust_deep_schedule() | src/osd/scrubber/scrub_job.cc | 254-291 |
| ScrubMap 定义 | src/osd/osd_types.h | 6581-6642 |
| PG 状态标志 | src/osd/osd_types.h | 1038-1049 |
| scrub_level_t | src/osd/osd_types.h | 2261 |
| PgScrubber 类 | src/osd/scrubber/pg_scrubber.h | 254-1099 |
| ScrubBackend 类 | src/osd/scrubber/scrub_backend.h | 310-570 |
| ScrubJob 类 | src/osd/scrubber/scrub_job.h | 123-384 |
| ScrubQueue 类 | src/osd/scrubber/osd_scrub_sched.h | 154-265 |
| Store (持久错误存储) | src/osd/scrubber/ScrubStore.h | 47-178 |

---

## 七、常见问题与陷阱

### Q1: 浅层 scrub 如何检测数据损坏？

**A**: 浅层 scrub 不读取实际数据。它依赖之前存储的 digest (CRC32C)。如果对象在写入时计算并存储了 digest，浅层 scrub 会比较存储的 digest 是否匹配。如果 digest 从未存储过，浅层 scrub 无法检测数据损坏。深层 scrub 总是重新读取数据并计算 digest。

### Q2: Scrub 会影响性能吗？

**A**: 会，尤其是深层 scrub。调度系统通过以下机制减轻影响：
- 并发限制 (`osd_max_scrubs`)
- CPU 负载检查 (`osd_scrub_load_threshold`)
- 时间窗口限制 (可配置允许的 scrub 小时/日期)
- 随机退避 (防止所有 PG 同时 scrub)
- 恢复期间可禁用 scrub (`osd_scrub_during_recovery`)

### Q3: 自动修复是如何工作的？

**A**: 当 scrub 发现不一致时：
1. 选择权威来源 (最新版本、无错误的副本)
2. 如果设置了 `auto_repair` 标志，从权威副本读取数据
3. 将正确数据写入损坏的副本
4. 更新对象的 digest 和元数据
5. 记录修复操作到 OSD 日志

### 参考文献

1. **Scrubbing 开发者文档**: https://docs.ceph.com/en/latest/dev/osd_internals/scrub/
2. **源码**: `src/osd/scrub_machine.cc`, `src/osd/PG.cc` (scrub 相关部分)
3. **Ceph 运维文档 (Scrub)**: https://docs.ceph.com/en/latest/rados/configuration/osd-config-ref/#scrubbing"
