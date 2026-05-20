# Note 47 优化建议

## 1. `Backfill` 的并发控制机制
- **原文问题**：原文提到 Backfill 按对象逐个进行。
- **优化建议**：补充说明 Ceph 使用 **`BackfillReservation`** 机制来控制并发。Primary 会向所有 ActingBackfill 成员发送 reservation 消息，协商每个 OSD 可以同时处理多少个对象的 Backfill。这防止了 Backfill 占用过多 IO 带宽而饿死客户端请求。

## 2. 空事务（No-Op）的日志记录
- **原文问题**：原文提到下发空事务记录日志。
- **优化建议**：明确指出这个"空事务"实际上是一条 **`pg_log_entry_t`**，标记为 `MODIFY` 或 `WRITE` 操作。它确保即使 OSD 没有立即写入数据，它的 `pg_log` 也会与 Primary 保持一致。当该 OSD 稍后开始 Backfill 此对象时，它会发现自己的 Log 已经是最新版本，从而安全地跳过旧数据拷贝，直接标记为"已同步"。

## 3. `waiting_for_backfill` 队列
- **原文问题**：原文未提及写请求被阻塞时的具体队列。
- **优化建议**：当写请求因 Backfill 冲突被阻塞时，它会被放入 PG 内部的 **`waiting_for_backfill`** 队列。一旦 Backfill 线程完成该对象的处理并释放锁，会触发回调将该队列中的请求重新推入 `op_shardedwq` 继续执行。

## 4. EC 场景下的 Backfill 阻塞
- **优化建议**：在纠删码（EC）场景下，Backfill 的阻塞逻辑更加复杂。由于 EC 写入需要读取其他分片进行编码，Backfill 和客户端写请求可能会竞争相同的条带资源。Ceph 通过 **`ECBackend` 的排队机制** 确保编码计算不会因并发操作而出错。
