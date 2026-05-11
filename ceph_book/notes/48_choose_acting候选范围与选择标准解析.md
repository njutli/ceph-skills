# 48. choose_acting候选范围与选择标准解析

## Q: choose_acting是从哪些OSD里选出Acting Set的？选择标准是什么？
## Q: 为什么“如果当前want Acting列表大小等于存储池副本数，则终止”？难道会大于吗？

### 1. 候选范围 (Where to choose from?)
`choose_acting` 并非从集群所有 OSD 中筛选，而是从特定的集合中筛选，范围是 **Prior Set**（或 Up Set 与存活的旧 Acting Set 的并集）。
筛选前提：
*   **存活（Up）**：节点网络可达。
*   **有日志（Has Info）**：节点必须携带该 PG 的有效 InfoRec（历史日志）。空节点不能作为权威候选者。

### 2. 选择标准 (Criteria)
*   **权威日志优先 (Authoritative Log)**：核心标准是找出拥有**最完整、版本最新日志**的 OSD (`auth_log_shard`)。若 `Up Primary` 日志不是最新的，它会退位。
*   **满足最小恢复条件**：
    *   **副本池**：存活 OSD 数不低于 `min_size`。
    *   **纠删码池 (EC)**：存活 OSD 数不少于 `k` 值。否则进入 `Incomplete`。
*   **Primary 确定**：优先选新 Up Primary，若其不可恢复，则选权威日志所在的 OSD 作为临时 Primary。

### 3. 列表大小控制 (Why terminate at size?)
书中提到“如果当前 want Acting 列表大小等于存储池副本数，则终止”，是为了**从过多的候选者中截断**。
*   **现象**：为什么“Pool Size=2，但 OSD 1, 2, 3 都有数据”？
    *   通常是因为**旧节点“苏醒”**或**切换延迟**。
    *   *场景*：PG 原本在 OSD [1, 2]。OSD 2 故障，OSD 3 顶替（Backfill）。
    *   当 OSD 3 正在同步或刚同步完，而 OSD 2 此时恢复上线。此时 OSD 1、2、3 都认为自己持有该 PG 数据。
    *   对于 `choose_acting` 而言，它面对的是一个“超员”的候选池 {1, 2, 3}。
*   **动作**：算法按优先级顺序将 OSD 加入 `want_acting`。一旦发现列表长度达到 `size`（如选入 1 和 2），便**立即停止**。
*   **目的**：防止列表冗余。多余的 OSD（如 3）不会被加入，后续会被视为 Stray 节点处理。
