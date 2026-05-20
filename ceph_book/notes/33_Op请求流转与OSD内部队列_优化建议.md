# Note 33 优化建议

## 1. `op_shardedwq` 的优先级队列实现
- **原文问题**：原文提到 `op_shardedwq` 内部有优先级队列（Recovery vs Client IO）。
- **优化建议**：在现代 Ceph 中，这通常通过 **`ClassHandler`** 和 **`OpQueue`** 的优先级参数实现。Recovery/Backfill 请求通常带有更高的优先级权重，确保集群在恢复数据时不会被客户端 IO 饿死。可以补充说明 `osd_recovery_op_priority` 等配置参数的作用。

## 2. `Session Sub-queue` 的保序机制
- **原文问题**：原文提到会话子队列保证同一客户端的 Op 顺序执行。
- **优化建议**：补充说明 Ceph 使用 `client_reqid_t`（包含 client ID 和 tid）来唯一标识请求。如果客户端重发请求（由于超时未收到 ACK），OSD 会通过 `reqid` 去重，确保幂等性，同时不会破坏原有的会话队列顺序。

## 3. `waiting_for_map` 的触发细节
- **原文问题**：原文提到 Op 携带的 Epoch 大于 OSD 本地 Epoch 时进入 `waiting_on_map`。
- **优化建议**：实际上，OSD 还会检查 Op 携带的 Epoch 是否**小于** OSD 的 `osdmap_epoch`。如果太小，OSD 会直接返回 `EAGAIN` 让客户端更新 Map，而不是无限期等待。只有当 Op 的 Epoch **略新于** OSD（例如 OSD 正在处理新 Map 的间隙）时，才会短暂进入 `waiting_on_map`。

## 4. `do_request` 与 `do_op` 的分离
- **原文问题**：流程图中区分了 `do_request`（条件检查）和 `do_op`（执行）。
- **优化建议**：明确 `do_request` 主要负责 PG 状态检查、权限验证、快照上下文解析等前置逻辑。只有当所有检查通过，Op 才会被封装进 `OpContext` 并交给 `PGBackend` 执行实际的 `do_op`（数据读写）。这种分离使得前置检查失败时可以快速返回错误，避免不必要的资源消耗。
