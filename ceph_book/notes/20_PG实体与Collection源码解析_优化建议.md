# Note 20 优化建议

## 1. `Collection` 与 `PG` 的对应关系
- **原文问题**：原文提到"每个 PG 对应一个 `Collection` 实例"。
- **优化建议**：明确说明 `Collection` 是 BlueStore（存储引擎）对 PG 的抽象。在 OSD 层面，PG 对应的是 `PrimaryLogPG` 对象。`PrimaryLogPG` 通过 `coll_t`（Collection ID）与 BlueStore 的 `Collection` 交互。这种分层设计使得 OSD 逻辑与底层存储引擎（BlueStore/KVStore）解耦。

## 2. `OpSequencer` 的作用
- **原文问题**：原文提到了 `OpSequencerRef osr`，但未详细解释。
- **优化建议**：补充说明 `OpSequencer` 的核心作用：保证同一个 PG 内的所有 IO 请求**严格有序执行**。Ceph 使用 `OpSequencer` 将并发请求串行化，避免并发写导致的数据不一致，同时允许不同 PG 的请求并行处理，最大化并发度。

## 3. `pgmeta_oid` 的 Key 结构
- **原文问题**：原文提到 PG 元数据写入 `pgmeta_oid` 的 omap。
- **优化建议**：补充说明 `pgmeta_oid` 内部使用的 Key 前缀。例如，`pg_info` 通常存储在特定的 Key 下，而 `pg_log` 条目则按 epoch 或 sequence 组织。这有助于理解 PG 恢复时如何从 RocksDB 中读取历史日志。

## 4. `SharedBlobSet` 的说明
- **原文问题**：原文提到 `SharedBlobSet` 用于 EC 场景。
- **优化建议**：`SharedBlobSet` 主要用于管理跨对象的共享 Blob（如 Clone/Snapshot），不仅限于 EC。在 EC 场景下，它可能用于管理 Stripe 之间的共享元数据，但其核心设计初衷是支持 COW/Clone 的零拷贝共享。
