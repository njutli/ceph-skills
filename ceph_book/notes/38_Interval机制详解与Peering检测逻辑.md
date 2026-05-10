# 38. Interval 机制详解与 Peering 检测逻辑

本节深入探讨 Ceph 中 `Interval` 的本质、生成、存储、检测与释放机制，重点解析其在 Peering 数据恢复过程中的核心作用。

## 1. Interval 的本质与实体

### 1.1 定义
**Interval（区间）** 是指 OSDMap 的一个连续 Epoch 间隔。在此期间，PG 的 `Acting Set`、`Up Set`、副本数等属性保持恒定。
它是 Ceph 用来切分 PG 历史的“时间切片”。

### 1.2 源码实体
在源码中，Interval 的实体是结构体 `pg_interval_t`（定义于 `src/osd/PGPastIntervals.h`）：
```cpp
struct pg_interval_t {
  epoch_t first;             // 起始 Epoch
  epoch_t last;              // 结束 Epoch
  std::vector<int> acting;   // 该期间的 Acting Set
  int acting_primary;        // 该期间的 Primary OSD
  std::vector<int> up;       // 该期间的 Up Set
  int up_primary;            // 该期间的 Up Primary OSD
  bool maybe_went_rw = false;// 核心标志：该期间 PG 是否可能处理过读写
};
```
所有的 `pg_interval_t` 被组织在 `PastIntervals` 类中。

---

## 2. Interval 的生成机制

### 2.1 生成时机
Interval 的生成发生在 **OSD 收到新的 OSDMap 时**。核心逻辑位于 `PG::handle_advance_map` 函数。

### 2.2 生成逻辑

每当 OSDMap 更新，PG 会对比新旧 Map 计算出的 `acting` 和 `up` 集合。
- **变化触发**：只要这两个集合中的任何一个发生变化，就意味着**旧的 Interval 结束，新的 Interval 开始**。
- **填充旧 Interval**：系统会将旧 Interval 的 `first`、`last`、`acting` 等信息填充完整，并计算 `maybe_went_rw`（判断该期间 PG 是否可能处于 Active 状态并接受了写入）。
- **开启新 Interval**：更新 `info.history.same_interval_since` 为当前 Epoch，并记录新的 Acting/Up Set。

**源码实现示例 (`PG::handle_advance_map`)**：
```cpp
// src/osd/PG.cc (简化版)
void PG::handle_advance_map(OSDMapRef newmap, OSDMapRef oldmap) {
    // 1. 计算当前的 acting/up
    vector<int> new_acting;
    int new_acting_primary;
    int new_up_primary;
    vector<int> new_up;
    OSDMap::raw_pg_to_acting_up(newmap, pg_id, new_acting, new_acting_primary, new_up, new_up_primary);

    // 2. 判断是否发生变化
    bool acting_changed = (new_acting != info.history.acting) || (new_acting_primary != info.history.acting_primary);
    bool up_changed = (new_up != info.history.up) || (new_up_primary != info.history.up_primary);

    // 3. 如果变化了，关闭旧 Interval，开启新 Interval
    if (acting_changed || up_changed) {
        // 填充旧 Interval 的结束时间
        pg_interval_t interval;
        interval.first = info.history.same_interval_since;
        interval.last = newmap->get_epoch() - 1;
        interval.acting = info.history.acting;
        interval.acting_primary = info.history.acting_primary;
        // ... 填充其他字段

        // 关键：计算 maybe_went_rw
        // 逻辑很复杂，主要判断在此期间 Primary 是否存活且 up_thru 是否更新
        interval.maybe_went_rw = ...; 

        // 将旧 Interval 加入历史列表
        info.history.intervals.push_back(interval);

        // 开启新 Interval
        info.history.same_interval_since = newmap->get_epoch();
        info.history.acting = new_acting;
        // ... 更新其他 current 状态
    }
}
```

### 2.3 离线 OSD 的“记忆补全”
如果一个 OSD 在宕机期间错过了多个 OSDMap 更新，它的本地 `past_intervals` 列表会停滞在掉线的那一刻。
当它重新上线时，OSD Monitor 会将缺失的 OSDMap 序列发给它。OSD 会在内存中**重新计算（推算）**这段历史，当场生成缺失的 Interval 记录并插入列表。
**结论**：OSD 不需要当时“在场”也能拥有 Interval 记录，它是通过查阅“官方档案”（OSDMap 历史）来补全记忆的。

---

## 3. Interval 的存储位置

Interval 不会单独保存，而是作为 `pg_info_t`（PG 的核心元数据）的一部分。
1. **内存中**：保存在 `PG` 对象的 `info.history.intervals` 成员变量中。
2. **磁盘上**：当 PG 状态发生变化（如 Peering 完成、Map 更新）时，`pg_info_t` 会被序列化并写入到后端的 **RocksDB** 中。这意味着即使 OSD 重启，`past_intervals` 也不会丢失。

---

## 4. Peering 过程中的检测逻辑

### 4.1 检测者
检测动作由 **新当选的 Primary** 负责执行。

### 4.2 核心动作：逆向自查与“顺藤摸瓜”
新 Primary **不会**去读取其他 OSD 的 `past_intervals` 列表。它采用的是“自查”策略：
1. **自查**：只读取**自己本地**的 `past_intervals` 列表。
2. **推断**：根据列表中的成员信息，推断“谁可能持有最新数据”。
3. **问询**：向推断出的 OSD 发送 `Query` 消息，索要 `Info`（版本号摘要），而非完整的 Interval 列表。

### 4.3 为什么必须“继续向前遍历”
检测函数 `PG::build_prior` 会对 `past_intervals` 进行**逆序遍历**（从最近向最远）。
**绝对不能**在找到一个 OSD 后就停止，必须**继续向前遍历**，直到遇到“安全线”（`last_epoch_started`）为止。

**详细案例**：
假设 OSD 1 醒来成为 Primary，它的历史记录如下：
- **Interval C (90-100)**: 成员 `[OSD 1, OSD 2]`。OSD 1 在场。
- **Interval B (80-89)**: 成员 `[OSD 3, OSD 4]`。**OSD 1 此时宕机了！**
- **Interval A (70-79)**: 成员 `[OSD 1, OSD 2]`。
- **安全线 (`last_epoch_started`)**: 75。

**遍历过程**：
1. **检查 Interval C**：发现 OSD 2。将 **OSD 2** 加入名单。*（虽然 OSD 1 在场，但 OSD 2 可能有 OSD 1 短暂掉线期间的数据，或日志丢失，必须问）*
2. **检查 Interval B**：发现 OSD 3, 4。将 **OSD 3, 4** 加入名单。*（这是关键！OSD 1 宕机期间，数据可能只写到了 3 和 4。如果停止遍历，这些数据将永久丢失！）*
3. **检查 Interval A**：发现 OSD 2（已在名单）。
4. **检查更早**：遇到 Epoch < 75，触发 `break`，停止遍历。

最终名单：`{OSD 2, OSD 3, OSD 4}`。OSD 1 向它们同时发起查询，找回所有潜在数据。

### 4.4 为什么“在场”也要问（防御性设计）

即使 Primary 在某个 Interval 全程“在场”，它也会将该 Interval 的其他成员加入名单。这是为了应对：
- **脑裂（Split-Brain）**：网络分区导致多个 Primary 同时写入，日志分叉。
- **日志丢失**：Primary 掉电导致 Journal 未刷盘，数据比副本旧。
- **短暂盲区**：Primary 在 Interval 中间短暂断连，错过了部分写入。

### 4.5 源码实现：检测逻辑与 Down 状态的关系

检测的核心逻辑位于 `PG::build_prior` 函数中。这正是回答“什么样的 Interval 能跳过”以及“为什么会导致 Down 状态”的关键。

**源码实现示例 (`PG::build_prior`)**：
```cpp
// src/osd/PG.cc
void PG::build_prior(PriorSet *prior) {
    // 1. 逆序遍历 past_intervals (从最近的开始)
    for (auto i = info.history.intervals.rbegin(); i != info.history.intervals.rend(); ++i) {
        const pg_interval_t &interval = *i;

        // 【检测条件 A】：是否发生在 last_epoch_started 之前？
        // 如果是，说明是“旧账”，已经处理过了，直接停止遍历（因为更早的肯定也处理过了）
        // **可跳过**
        if (interval.last < info.last_epoch_started) {
            break; 
        }

        // 【检测条件 B】：maybe_went_rw 是否为 false？
        // 如果是 false，说明这段时间 PG 是“死”的，不可能有数据写入，直接跳过
        // **可跳过**
        if (!interval.maybe_went_rw) {
            continue;
        }

        // 4. 如果通过了检测，说明这个 Interval "有问题" (可能写过数据)
        // 将该 Interval 期间的 Acting Set 中的 OSD 加入探测列表 (probing)
        for (auto osd : interval.acting) {
            if (osd != whoami) { // 排除自己
                prior->probing.insert(osd);
            }
        }
    }
}
```

**与 Down 状态的直接关联**：
在表 4-15 中，**Down** 状态的定义是：“Peering 过程中，PG 检测到某个**不能被跳过**的 Interval 中……当前剩余在线的 OSD 不足以完成数据修复”。

结合源码，我们可以得出：
1. **什么样的 Interval 能跳过？**
   - 满足**条件 A**（发生在 `last_epoch_started` 之前，旧账已结）。
   - 满足**条件 B**（`maybe_went_rw` 为 false，期间 PG 未 Active，无数据写入）。
2. **什么样的 Interval 不能跳过？**
   - 同时**不满足**上述两个条件的 Interval。这意味着它既包含潜在的写入数据，又尚未被确认同步。
3. **为什么会变成 Down 状态？**
   - 当 Primary 遇到一个**不能跳过**的 Interval 时，它必须找到该 Interval 的 Acting Set 成员来恢复数据。
   - 如果这些成员全部宕机或离线（即 `prior->probing` 列表为空，或者纠删码场景下存活节点数不足 k 值），Primary 就无法完成数据修复。
   - 此时，Peering 流程无法继续，PG 只能将自己设置为 **Down** 状态，停止服务，直到那些离线的 OSD 重新上线。

---

## 5. Interval 的释放时机

`past_intervals` 不能无限增长。释放发生在 **Peering 成功完成** 之后。
1. **更新安全线**：在 Peering 最后阶段（`PG::activate`），新 Primary 会通知所有副本更新 `last_epoch_started` 为当前 Epoch。
2. **清理**：调用 `clear_past_intervals`，移除所有 `last < last_epoch_started` 的 Interval。
这些 Interval 的数据一致性已经得到保证，属于“已结旧账”，可以安全释放。

---

## 6. 总结：基于 Peering 视角的 Interval 生命周期

**Interval 是 Ceph 为了在 Peering 过程中高效恢复数据一致性而设计的“历史切片”。每当 OSDMap 变化导致 PG 的 Acting/Up Set 改变时，系统会生成一个新的 Interval 并记录其成员及是否可能处理过读写（`maybe_went_rw`）。当新 Primary 上任启动 Peering 时，它会通过逆向遍历本地的 Interval 历史列表（`past_intervals`），利用 `last_epoch_started` 作为安全线，筛选出所有“可能持有新数据”的历史成员构建 PriorSet，进而向它们拉取日志进行同步。一旦 Peering 成功完成并更新了安全线，这些 Interval 就完成了使命，随后被清理释放。**
