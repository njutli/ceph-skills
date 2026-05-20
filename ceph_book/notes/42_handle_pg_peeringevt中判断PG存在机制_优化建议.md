# Note 42 优化建议

## 1. `pg_map` 的锁机制
- **原文问题**：原文提到 `pg_map` 查找，但未提及锁。
- **优化建议**：补充说明 `pg_map` 的查找通常需要获取 **`pg_map_lock`** 或类似的读写锁。为了避免锁竞争，现代 Ceph 使用了分片化的 PG 查找机制（如 `ShardedOpWQ` 对应的 PG 分片），确保不同 PG 的事件处理可以并行进行而不互相阻塞。

## 2. PG 不存在时的处理策略
- **原文问题**：原文提到"直接返回或触发创建流程"。
- **优化建议**：明确区分两种情况：
  - 如果是 **Peering 事件**（如 OSDMap 更新），且 PG 不存在，OSD 通常会调用 `get_or_create_pg(pgid)` 触发 PG 实例化，进入 `Creating` 状态。
  - 如果是 **普通 IO 或心跳消息**，且 PG 不存在，OSD 会直接丢弃消息或返回 `ENOSYS`/`ENOENT`，绝不会触发创建，防止恶意或错误请求导致资源耗尽。

## 3. `handle_pg_peeringevt` 的具体调用场景
- **原文问题**：原文未详细说明该函数的调用场景。
- **优化建议**：`handle_pg_peeringevt` 通常在 OSD 收到 `MOSDPGPeering` 消息或内部定时器触发时被调用。它负责将 Peering 相关的事件（如 `InfoRec`、`LogRec`、`Query`）分发给对应的 PG 实例状态机进行处理。

## 4. 源码位置
- **优化建议**：核心查找逻辑位于 `src/osd/OSD.cc` 的 `OSD::get_pg` 或 `OSD::_lookup_lock_pg` 函数。PG 的创建逻辑则位于 `OSD::handle_pg_create` 或 `OSD::maybe_create_pgs`。
