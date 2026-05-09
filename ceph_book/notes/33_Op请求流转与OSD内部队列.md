# 33. Op请求流转与OSD内部队列机制

本节重点剖析客户端 Op 请求到达 OSD 后的内存流转路径，以及决定 Op 调度行为的 Epoch 对齐机制。

## 1. Op 在 OSD 内部的流转路径

Op 从网络层到达 OSD 开始，到最终进入工作线程执行，会经历多个内部队列的检查与暂存。这些队列确保了 Ceph 在复杂的分布式环境（如 Peering、OSD 启停、CRUSH 变更）下的状态一致性与数据可靠性。

流转顺序通常如下：

1. **waiting_for_pg**：
   * **触发条件**：OSD 收到 Op 时，目标 PG（Placement Group）的实例尚未在内存中创建。
   * **典型场景**：OSD 刚启动、PG 正在进行分裂（Split）或合并（Merge），或者 Op 请求早于 PG 初始化完成到达。
   * **处理**：将 Op 暂存，直到 PGService 初始化完成。

2. **waiting_for_active**：
   * **触发条件**：PG 已在内存中，但并未处于 `Active` 状态（如处于 `Peering`、`Backfilling` 或 `Degraded` 中）。
   * **作用**：只有 Active 状态的 PG 才能对外提供读写服务。此队列用于在 PG 状态机恢复正常前拦截 IO 请求。

3. **waiting_on_map**（时序对齐）：
   * **触发条件**：客户端发送 Op 时携带的 OSDMap Epoch（版本） **大于** OSD 本地持有的 Epoch。
   * **作用**：OSD 发现自己“视野落后”，无法确认自己是否还有权限处理该 Op（例如确认自己是否仍是该 PG 的 Primary）。此时将 Op 挂起，等待 OSDMonitor 推送最新的 OSDMap。

4. **op_shardedwq**（执行队列）：
   * **触发条件**：所有前置检查通过（PG 就绪、状态 Active、OSDMap Epoch 已对齐且确认自己是 Primary）。
   * **作用**：这是 Op 的终点。Worker 线程从此队列拉取 Op，构建 `OpContext`，并开始执行数据落盘与副本复制逻辑。

## 2. OSDMap Epoch 的对齐机制

### 2.1. Epoch 的定义
Epoch 代表 **OSDMap 的版本号**。它是一个单调递增的整数，集群的任何拓扑变更（OSD 状态改变、权重调整、CRUSH 规则更新）都会触发 OSDMap 版本更新（Epoch++）。

### 2.2. 为什么 Epoch 关系决定调度策略？
Ceph 采用了一种基于 Epoch 的**视图一致性（View Consistency）**模型。客户端和 OSD 必须基于同一版本的集群拓扑才能确保操作的正确性。

* **客户端视角的“真理”**：客户端在封装 Op 时，会记录当时的 OSDMap。例如，根据 `v100` 的 OSDMap，客户端计算出 `PG_x.0` 的 Primary 是 `OSD A`。
* **OSD 视角的“真理”**：OSD 通过 OSDMonitor 实时同步 Map。

#### 核心冲突场景
**场景**：OSD A 的本地 Map 版本停留在 `v98`，而客户端携带 `v100` 发来 Op，指定 OSD A 为 Primary。

1. **不确定性风险**：在 `v99` 版 Map 中，也许 `OSD A` 已经被标记为 Down，或者 `OSD C` 变成了新的 Primary。如果 OSD A 依据过时信息处理 Op，会导致“过期的 Primary”写入数据（即陈旧写入），严重时会引发脑裂。
2. **处理流程**：
   * OSD 识别到 `Client Epoch (100) > OSD Epoch (98)`。
   * Op 被放入 `waiting_on_map`。
   * OSD 请求 OSDMonitor 同步到 `v100`。
   * 同步完成后，OSD 重新评估：如果依据 `v100` 它依然是 Primary，则放行 Op 至 `op_shardedwq`；否则通知客户端出错。

### 3. 总结：队列的本质
* **waiting_for_pg / waiting_for_active**：负责 **PG 就绪态** 检查。
* **waiting_on_map**：负责 **集群拓扑一致性（Epoch 对齐）** 检查。
* **op_shardedwq**：负责 **并发执行**。

这些机制共同构成了 Ceph OSD 的“守门员系统”，确保只有满足所有分布式约束条件的 IO 请求才会被真正执行。
