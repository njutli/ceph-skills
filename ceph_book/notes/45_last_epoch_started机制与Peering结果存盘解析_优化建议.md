# Note 45 优化建议

## 1. `Activate` 消息的现代演进
- **原文问题**：原文提到 Primary 发送 `Activate` 消息给 Replica。
- **优化建议**：在现代 Ceph 版本中，`Activate` 消息的部分职责已被整合到 `MOSDPGLog` 和 `MOSDPGInfo` 消息流中。Peering 过程更加流水线化，Primary 在收集到足够的 `Query` 回复后，直接推送 Log 和 Missing 列表，随后触发本地激活。建议注明"对应早期基于显式 Activate 消息的实现"。

## 2. `history.last_epoch_started` 的持久化时机
- **原文问题**：原文提到 Primary 收到 ACK 后更新 `history`。
- **优化建议**：补充说明 `history.last_epoch_started` 的更新是与 `pg_log` 的截断（Trim）操作绑定的。当所有副本确认激活后，Primary 会安全地丢弃旧版本的 Log 条目，并更新 `history` 以记录新的安全线。这确保了即使 Primary 崩溃，重启后也能知道哪些 Log 是安全的、哪些需要重新同步。

## 3. 缺失数据（Missing Data）的同步细节
- **原文问题**：原文提到 Missing Data 包含新增对象、覆盖旧版本和 PG_Log。
- **优化建议**：明确指出 Missing Data 的同步分为两个阶段：
  - **Log Sync**：首先同步 `pg_log_entry_t`，让 Replica 知道哪些对象需要更新。
  - **Data Recovery/Backfill**：然后根据 Log 中的 `ObjectModDesc`，拉取实际的对象数据或执行删除操作。这种分离设计使得 OSD 可以先快速对齐元数据，再在后台慢慢搬运大数据块。

## 4. 源码位置
- **优化建议**：`last_epoch_started` 的更新逻辑位于 `src/osd/PG.cc` 的 `PG::advance_map` 和 `PG::publish_stats_to_osd_map` 函数。`MInfoRec` 的处理位于 `PG::Recovering` 或 `PG::Active` 状态的事件处理函数中。
