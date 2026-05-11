# Note 46：PG Temp机制与Backfill主节点保护解析

## 为什么需要 PG Temp？
在 Ceph 集群扩容或重平衡（Rebalance）时，CRUSH 规则可能会重新计算 PG 的分布，导致某个新的 OSD（例如 OSD B）成为 PG 的“第一顺位”（Up Set[0]），理应成为 Primary。
然而，通常这个新 OSD B 上并没有该 PG 的数据（或者是旧数据）。如果强制让 OSD B 立即作为 Primary，它因为没有数据而无法提供服务；如果让它先执行 Backfill（回填）拿到所有数据后再成为 Primary，那么在整个回填期间（可能持续数小时），该 PG 将处于 Peering 状态，无法响应客户端请求。

**PG Temp 的出现就是为了解决这个矛盾：在不中断服务的前提下，让旧的 OSD（已有完整数据）继续担任 Primary，直到新的 OSD 完成数据同步。**

## PG Temp 的机制
PG Temp 是 OSDMap 中的一种**临时覆盖规则**。
当 Monitor 计算 OSDMap 时，如果发现新的 Up Set[0] 需要 Backfill（且数据量较大），它会在 OSDMap 中为该 PG 设置一个 `pg_temp` 映射。
这个映射告诉集群：“虽然 CRUSH 算出来的 Primary 是 B，但在 Backfill 完成前，**请强制把 A（旧 Primary）当作临时 Primary**。”

## 详细案例：扩容场景下的 PG Temp

假设有一个 2 副本存储池，包含 **OSD A**（旧数据节点）和 **OSD B**（新加入节点）。

### 阶段 1：初始状态
*   **OSDMap**: `up=[A, B]`, 无 `pg_temp`。
*   **Primary**: OSD A。
*   **状态**: PG 状态为 `active+clean`，A 负责处理所有读写请求，B 负责同步。

### 阶段 2：CRUSH权重调整，B 成为第一顺位
*   **OSDMap**: `up=[B, A]`。按照 CRUSH 规则，**OSD B 应该是 Primary**。
*   **冲突**: 但是 OSD B 是新节点，上面是空的（或者数据不完整），它需要执行 **Backfill** 才能拿到数据。
*   **后果（如果不使用 PG Temp）**:
    *   OSD B 试图成为 Primary -> 发现自己没数据 -> 阻塞。
    *   OSD B 等待 A 把数据传过来 -> **耗时数小时**。
    *   在这数小时内，客户端无法读写。

### 阶段 3：引入 PG Temp（解决方案）
*   **Monitor 操作**: 监测到 B 需要 Backfill。
*   **设置 PG Temp**: Monitor 在 OSDMap 中写入 `pg_temp=[A, B]`。
*   **效果**:
    *   虽然规则上 B 是老大，但有了 `pg_temp`，**OSD A 被强制保留为 Primary**。
    *   集群状态迅速恢复为 `active+clean`（或带有 degraded）。
    *   **OSD A**：继续处理客户端请求，并作为 Backfill 的数据源发给 B。
    *   **OSD B**：作为 Replica，在后台默默接收数据。

### 阶段 4：Backfill 完成
*   **OSD B**: 数据同步完毕，拥有最新数据。
*   **Monitor 操作**: 移除该 PG 的 `pg_temp`。
*   **OSDMap**: `up=[B, A]`, 无 `pg_temp`。
*   **结果**: OSD B 正式接任 Primary，OSD A 变为 Replica。

## 总结
*   **PG Temp 的本质**：让有数据的 OSD 先干活（Serving IO），保证高可用性。
*   **副作用**：这会导致当前的数据分布不符合 CRUSH 的最佳负载均衡状态（称为 Remapping 状态），但这只是暂时的。
