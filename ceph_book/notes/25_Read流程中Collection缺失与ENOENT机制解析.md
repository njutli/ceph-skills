# Read 流程中 Collection 缺失与 -ENOENT 机制解析

## 1. Collection 存在性检查的意义
在图 2-11 的 `read` 流程中，如果无法找到对应的 `collection`，BlueStore 会直接返回 `-ENOENT`。这基于以下逻辑：
- `collection` 是 PG 在底层存储（数据面）的物理实体（目录及 KV 索引）。
- 如果找不到 `collection`，等同于该 PG 在本 OSD 上不存在或未就绪，自然无法定位到具体的 Object。

## 2. Collection 的创建时机与 PG 状态
**前提误区**：集群就绪后，所有可能 PG 的 Collection 都会提前创建好。
**实际情况**：Collection 是**按需/动态创建**的，紧随 PG 状态流转：
1. **OSDMap 更新触发**：OSD 收到最新的 OSDMap，发现自身被分配了新区间的 PG，才会开始创建 Collection。
2. **状态机流转**：PG 经历 `creating`（新建） → `peering`（同步元数据/日志） → `activating`（等待底层就绪） → **`active`**（对外服务）。
   - **关键约束**：只有在底层成功创建了 Collection 并加载数据后，PG 才会进入最终的 `active` 状态。
   - 因此，**只要 PG 是 active 的，其 Collection 必然已就绪。**

## 3. 为什么正常 Read 操作会撞上 `-ENOENT`？
既然 PG 激活后 Collection 必在，那么客户端下发请求时为何还会遇到 Collection 缺失？这通常是**客户端与 OSD 状态不同步**导致的瞬态现象：

| 场景 | 现象 | 结果 |
| :--- | :--- | :--- |
| **客户端 OSDMap 过期** | 客户端根据旧版 Map，将请求发给了错误/不负责的 OSD，该 OSD 上尚未分配或正在处理该 PG，Collection 不存在。 | OSD 返回 `-ENOENT`，客户端捕获后触发 **OSDMap 更新**，随后重试正确 OSD。 |
| **PG 正在迁移/分裂** | 目标 OSD 正在接收数据并初始化 Collection，此时请求提前到达。 | 拦截失败，待 Collection 就绪后恢复正常。 |
| **Map 传播延迟** | Monitor 更新了 Map，但 OSD 侧处理尚未完成，处于短暂的不一致间隙。 | 瞬间报错自愈。 |

## 总结
返回 `-ENOENT` 是一个 **快速失败（Fail-Fast）** 的安全校验：防止 OSD 处理不属于自己或尚未就绪的 IO 请求，同时作为隐式信令，促使客户端修正路由信息。