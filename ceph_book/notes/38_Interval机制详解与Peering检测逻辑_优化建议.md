# Note 38 优化建议

## 1. `maybe_went_rw` 的计算细节
- **原文问题**：原文提到 `maybe_went_rw` 判断该期间 PG 是否可能处理过读写。
- **优化建议**：补充说明 `maybe_went_rw` 的计算依赖于 `up_thru` 和 `last_epoch_started`。如果 Primary 在该 Interval 期间成功向 Monitor 报告了存活状态（`up_thru` 更新），则 `maybe_went_rw` 为 true。这是 Peering 时判断是否需要向该 Interval 成员查询数据的关键依据。

## 2. `build_prior` 与 `PriorSet`
- **原文问题**：原文提到了 `prior->probing` 列表。
- **优化建议**：明确 `PriorSet` 包含三个关键集合：`probing`（需要查询的 OSD）、`down`（已知宕机但可能持有数据的 OSD）、`ec_pp`（EC 场景下的伪主节点）。在副本池中，如果 `down` 集合中的 OSD 数量超过了允许的丢失限制（`min_size` 或 `allow_incomplete_clones`），PG 会进入 `down` 状态。

## 3. `last_epoch_started` 的更新时机
- **原文问题**：原文提到 Peering 成功后更新 `last_epoch_started`。
- **优化建议**：详细说明 `last_epoch_started` 是在 `PG::publish_stats_to_osd_map` 或 `PG::log_entered_state` 中更新的。它标志着"从这个 Epoch 开始，所有之前的数据一致性已经得到保证"。这个值会持久化到 `pg_info_t` 中，即使 OSD 重启也不会丢失。

## 4. EC 场景下的 Interval 检测
- **原文问题**：原文主要以副本池为例。
- **优化建议**：补充说明在纠删码（EC）场景下，Interval 检测更加复杂。Primary 需要确保收集到至少 K 个 Shard 的日志才能重建权威历史。如果存活的 Shard 不足 K 个，即使部分 OSD 在线，PG 也会进入 `down` 状态，因为数据无法恢复。
