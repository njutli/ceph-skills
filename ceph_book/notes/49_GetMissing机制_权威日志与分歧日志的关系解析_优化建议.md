# Note 49 优化建议

## 1. `proc_replica_log` 的现代演进
- **原文问题**：原文提到 `PGLog::proc_replica_log` 处理分歧日志。
- **优化建议**：在现代 Ceph 版本中，日志处理逻辑已被重构到 `PeeringState` 和 `PG::RecoveryState` 中。`proc_replica_log` 的核心思想（比对版本、生成 Missing）依然适用，但具体的函数调用链可能已变化。建议注明"对应早期基于 PGLog 的处理逻辑"。

## 2. `is_divergent` 标记的精确含义
- **原文问题**：原文提到 `is_divergent` 标记分歧对象。
- **优化建议**：补充说明 `is_divergent` 不仅标记对象需要回滚，还决定了 Recovery 时的操作类型。如果 `is_divergent` 为 true，OSD 会先执行 `rollback`（将对象恢复到 `prior_version`），然后再执行 `fetch`（拉取权威版本）。这确保了即使在复杂的脑裂场景下，数据也能安全地回退到一致点。

## 3. 情形 2 的"直接删除"
- **原文问题**：原文提到情形 2 直接物理删除对象。
- **优化建议**：明确指出这种删除操作也会记录在 `pg_log` 中，作为一条 `DELETE` 或 `LOST_REVERT` 条目。这是为了确保即使删除操作中途掉电，重启后 OSD 也能知道该对象应该被删除，防止"幽灵对象"复活。

## 4. 脑裂（Split-Brain）的自动愈合
- **原文问题**：原文在情形 4 中提到了脑裂。
- **优化建议**：补充说明 Ceph 的脑裂愈合完全依赖于 **Interval 机制**。只有处于合法 Interval（即被 Monitor 正式承认的 Primary 任期）内的写入才会被纳入权威日志。任何在非法 Interval 内的写入（即使版本号更高）都会被标记为分歧并最终回滚。这保证了 Ceph 在极端网络分区下的数据强一致性。
