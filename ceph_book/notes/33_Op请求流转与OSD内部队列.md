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

### 3. Op 调度核心：为什么 op_shardedwq 需要分片？

在 Ceph 的源码架构中，`op_shardedwq` 的设计是为了在**高并发性能**与**操作保序性**之间找到完美的平衡点。

### 3.1. 是“一个”队列还是“多个”队列？

*   **对外宏观表现（一个接口）**：对于上层调用者（如网络线程或 Dispatcher 模块），`op_shardedwq` 看起来只是一个单一的对象。调用者只需调用类似 `enqueue_op()` 的方法，不需要关心底层细节。
*   **内部微观实现（多个分片队列）**：在其内部，维护了一个数组（Vector），包含 `num_shards`（通常由配置 `osd_op_num_shards` 控制，如 16 或 32）个独立的物理子队列（Shard List）。

### 3.2. 为什么要采用分片设计？

这主要为了解决分布式存储中典型的“并发 vs 一致性”矛盾：
*   **如果只有单队列**：为保证同一个 PG 的 Op 不乱序，全局必须共用一个队列。但这会导致数十个核心同时抢夺同一把锁，严重由于**锁竞争**降低吞吐量。
*   **如果每个 PG 一个队列**：Ceph 集群通常有成千上万个 PG（Placement Groups）。维护数万个队列及对应的线程资源调度开销巨大，难以管理。
*   **折中方案（Sharded）**：将几千个 PG 映射到几十个队列中。每个子队列拥有独立的锁，不同的 CPU 线程（Worker）可以并行拉取不同队列中的 Op 进行处理，大幅提升了 OSD 的整体并发性能。

### 3.3. Op 如何决定进入哪个子队列？

Op 的入队逻辑是基于 **PG ID** 的确定性路由（Deterministic Routing），这确保了**同一个 PG 的所有 Op 永远进入同一个子队列**。

*   **分配逻辑**：通过获取 PG ID 的哈希值并对分片数量取模。
    `shard_id = hash(pg_id) % num_shards;`
*   **保序效果**：无论客户端如何并发发送请求，只要属于 `PG 1.2` 的 Op，计算出的 `shard_id` 永远固定。该 Op 在子队列中严格遵循 **FIFO（先进先出）** 原则执行，从根本上防止了同一个对象被并发修改导致的乱序和数据不一致。

## 4. op_shardedwq 的内部三级队列结构

在确定了 Op 归属的 PG 分片队列（Shard List）后，`op_shardedwq` 内部还采用了**多级分层架构**。这种设计的目的是在保证**单个客户端操作严格顺序**（保序）的同时，兼顾**不同客户端及不同任务的并发执行效率**。

### 4.1. 层级拆解
从宏观到微观，Op 在 Shard 内部经历的队列过滤层级如下：

1.  **第 1 级：PG 分片队列 (Shard Queue)**
    *   **分流依据**：PG ID 的 Hash 值（如前所述）。
    *   **作用**：**大粒度分流**。解决全局锁竞争问题。同一个 PG 的所有 Op 必定落入同一个 Shard。

2.  **第 2 级：优先级队列 (Priority Queue)**
    *   **分流依据**：Op 的任务优先级（如：Recovery、Scrub、Client IO）。
    *   **作用**：**QoS 保障**。确保 Peering 等系统关键路径操作（Recovery IO）不会被普通的客户端数据读写阻塞。

3.  **第 3 级：会话子队列 (Session Sub-queue)**
    *   **分流依据**：客户端地址/身份（Client ID / Entity Name）。
    *   **作用**：**隔离与保序**。这是保证保序性的核心——同一个客户端发送到同一个 PG 的 Op，永远在一个 FIFO 队列里排队，绝不会发生“后来的 Op 先执行”导致的逻辑覆盖。

### 4.2. 结构示意图
```text
[OSD op_shardedwq]
   │
   └── [Shard N] (对应某个 PG Hash 分片)
        │
        └── (内部按优先级分层)
             ├── [Priority: High/Recovery]  (高优先级区)
             │    ├── [Session: Client A] -> Op1 -> Op2 
             │    └── [Session: Client B] -> Op4
             │
             └── [Priority: Low/Standard]   (普通优先级区)
                  ├── [Session: Client A] -> Op3
                  └── [Session: Client C] -> Op5
```

### 4.3. 综合场景举例
**前提**：客户端 **Client A** 和 **Client B** 同时对 **PG 1.2** 进行操作。
*   **Client A** 发送：`写 Op1`（高优先级/Recovery）、`写 Op2`（普通优先级）和 `写 Op3`（普通优先级）。
*   **Client B** 发送：`读 Op4`（高优先级）。

**流转过程**：
1.  **Level 1 (Shard)**：所有 Op 均进入处理 PG 1.2 的那个 Shard。
2.  **Level 2 (Priority)**：
    *   `Op1` 和 `Op4` 被分流至 **High Priority** 区。
    *   `Op2` 和 `Op3` 被分流至 **Low Priority** 区。
3.  **Level 3 (Session)**：
    *   在 High 区：`Op1` 进入 Client A 的会话队列，`Op4` 进入 Client B 的队列。
    *   在 Low 区：`Op2` 和 `Op3` 进入 Client A 的会话队列。

**执行顺序**：
Worker 线程会优先拉取 High 区任务 -> 此时可能并行执行 A 的 `Op1` 和 B 的 `Op4`。
待 High 区清空，Worker 再拉取 Low 区任务 -> 严格按照顺序执行 `Op2` -> `Op3`（保证了 A 的顺序性）。

## 5. PG 内部队列与 op_shardedwq 的联动机制

在理解了 `op_shardedwq` 的内部结构后，我们需要理清它与书中**表 4-12** 提到的 PG 内部等待队列（如 `waiting_for_active`, `waiting_for_map` 等）之间的关系。

### 5.1. 核心区别：执行通道 vs 等待室

*   **`op_shardedwq` (主干道/执行通道)**：
    这是 OSD 的核心工作队列。所有的 Op 只有在这里，并且由 Worker 线程拉取执行 `do_request` 校验时，才能被真正处理。它是所有 IO 请求流转的枢纽。

*   **PG 内部等待队列 (等待室/停车场)**：
    这是 **PG 对象** 内部维护的多个 FIFO 暂存队列。当 Worker 线程从 `op_shardedwq` 取出 Op 处理时，如果发现当前 **PG 状态异常** 或 **对象数据不可用**，Op **不会被丢弃**，而是被**重新入队 (Requeue)** 到这些特定的等待队列中。这些队列里的 Op 处于 **“挂起 (Suspended)”** 状态。

### 5.2. 回流机制：Op 如何重新进入 op_shardedwq？
图 4-1 和 4-2 展示了 Op 如何被“阻塞”，但隐藏了“唤醒”的过程。Op **不会主动轮询** 检查条件是否满足，而是**被动等待“系统事件”将其唤醒**。

当阻塞条件解除时，OSD 的特定**状态机流转**或**回调函数**会主动去等待队列中捞取 Op，并将其重新推入 `op_shardedwq` (即“回流”)：

| 等待队列 (Staging Queues) | 阻塞条件 | **回流触发器（谁在什么时候唤醒？）** |
| :--- | :--- | :--- |
| **waiting_for_map** | Op 携带的 Epoch 比 OSD 新 | **OSD 收到新 Map**：OSD 调用 `consume_map` 处理新 Map 时，会主动检查此队列，将 Op 踢回 `op_shardedwq` 重试。 |
| **waiting_for_active** | PG 尚未进入 Active 状态 | **PG 状态机流转**：当 PG 完成 Peering 切换为 `Active` 状态时，`PG::on_activate` 被调用，它扫描此队列并将 Op 重新送入 `op_shardedwq`。 |
| **waiting_for_unreadable_object**| 对象处于丢失/修复状态 | **Recovery 完成**：当后台数据恢复流程 (Recovery/Backfill) 修复了该对象，PGBackend 触发回调唤醒 Op。 |
| **waiting_for_scrub** | 对象正在进行一致性检查 | **Scrub 完成回调**：一致性检查结束释放锁时，唤醒被搁置的 Op。 |

### 5.3. 完整的“生命周期闭环”
Op 的生命周期并非单向直线，而是一个“检查 -> 阻塞 (休眠) -> 唤醒 -> 重试”的循环：

```text
       [1. 网络接收]
           │
           ▼
      (op_shardedwq) ◄───────────────────────┐
           │                                 │
           │ [Worker 线程拉取]                 │ (2. 唤醒回流)
           │                                 │ 当外部系统事件触发
           ▼                                 │ (如新 Map 到达 / PG 变 Active)
     ┌──────────────┐                        │
     │  do_request  │ (图 4-1 流程)             │
     │  (条件检查)   │                        │
     └─────┬───────┘                        │
           │                                │
    ┌──────┴──────┐                         │
    │ 条件是否满足？│                         │
    └──────┬──────┘                         │
           │                                │
      (NO) ▼ (YES)                          │
   ┌──────────────┐   (1. 阻塞/挂起)    ┌──────────────┐
   │ PG 内部等待队列 │ ─────────────────► │    do_op     │ (图 4-2 流程)
   │ (休眠/排队)    │   (不消耗 CPU)       │    (执行)     │
   └──────────────┘                    └──────────────┘
```

**总结**：表 4-12 中的队列是条件不满足时的**停车场**，而 `op_shardedwq` 是条件满足后的**执行通道**。通过事件驱动的回调机制，OSD 实现了高效且无死循环的 Op 调度。

---

## 6. 总结：队列的本质
* **waiting_for_pg / waiting_for_active**：负责 **PG 就绪态** 检查。
* **waiting_on_map**：负责 **集群拓扑一致性（Epoch 对齐）** 检查。
* **op_shardedwq**：负责 **并发执行**。

这些机制共同构成了 Ceph OSD 的“守门员系统”，确保只有满足所有分布式约束条件的 IO 请求才会被真正执行。
