# Note 52 优化建议

## 1. `SubOp` 消息的现代演进
- **原文错误**：原文暗示 Recovery 和 Backfill 都通过 `SubOp` 消息进行。
- **改正**：在现代 Ceph 版本中，Recovery 和 Backfill 的消息传递机制已重构。Backfill 通常使用专门的 **`MOSDPGPush`** 和 **`MOSDPGPull`** 消息，而不是通用的 `SubOp`。`SubOp` 更多用于多副本池的实时同步（Replication）。建议区分 **Replication（实时同步）** 和 **Recovery/Backfill（后台修复）** 的消息差异。
