# Note 50 优化建议

## 1. Peering 状态的细分
- **原文问题**：原文将 Peering 描述为 5 个阶段。
- **优化建议**：在现代 Ceph 中，Peering 过程被细分为更多的内部状态，如 `GetInfo`、`GetLog`、`GetMissing`、`WaitBackfillReserved`、`StartRecovery` 等。这些状态在 `PG::RecoveryState` 中通过状态机管理。建议补充说明这些细分状态，以便读者在查看 `ceph pg dump` 或日志时能更精准地定位问题。

## 2. `WaitBackfillReserved` 阶段
- **原文问题**：原文未提及 Backfill 资源预留。
- **优化建议**：在 `Activate` 之前，现代 Ceph 通常会插入一个 **`WaitBackfillReserved`** 阶段。Primary 会向所有 ActingBackfill 成员发送 Reservation 请求，确保它们有足够的 IO 带宽和 CPU 资源来执行 Recovery/Backfill。如果某个 OSD 拒绝（例如负载过高），Peering 会暂停，防止 Recovery 拖垮集群性能。

## 3. `ReplicaActive` 与 `Active` 的区别
- **原文问题**：原文提到通知 Peer 激活变为 `ReplicaActive`。
- **优化建议**：明确区分 Primary 的 `Active` 状态和 Replica 的 `ReplicaActive` 状态。Primary 进入 `Active` 后可以立即处理客户端 IO，而 Replica 进入 `ReplicaActive` 后主要负责接收 Recovery 数据和处理副本同步。只有当所有副本都达到 `ReplicaActive` 且数据完全同步（`clean`）时，PG 才会显示为 `active+clean`。

## 4. 源码位置
- **优化建议**：Peering 状态机的核心逻辑位于 `src/osd/PG.cc` 和 `src/osd/PeeringState.cc`。`build_prior`、`proc_replica_log`、`calc_missing` 等函数分别对应 GetInfo、GetLog、GetMissing 阶段的具体实现。
