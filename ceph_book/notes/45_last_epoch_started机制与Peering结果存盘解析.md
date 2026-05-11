# Note 45：last_epoch_started 机制与Peering结果存盘解析

## pg_info_t 中的两个 last_epoch_started
在 Ceph OSD 源码中，`pg_info_t` 包含两个关键的 `last_epoch_started` 属性：

1.  **`info.last_epoch_started`（Local Durability Watermark）**：
    *   **含义**：**本机 OSD** 已完成激活事务（Activate Transaction）落盘的最新 Epoch。
    *   **更新时机**：本机完成事务落盘后立即更新。

2.  **`info.history.last_epoch_started`（Authoritative Global Boundary）**：
    *   **含义**：**Primary 认证并下发的**、“所有 Acting Set 均已落盘”的历史安全线。
    *   **更新时机**：**不是**在 ACK 产生时立即更新，而是由 Primary 在收到所有确认并落盘自身 history 后，通过消息机制（如 `MInfoRec`）**主动下发**给 Replica 更新。

## “和Peering结果一并存盘”
这里的“Peering结果”指的是：Primary 在 Peering 阶段计算出的**权威 PG 状态（Authoritative PG State）**。
*   包括：权威 `pg_log`、对齐后的 `pg_info`（含 `info.last_epoch_started`）以及**计算好的完整历史状态**（含 `history.last_epoch_started`）。
*   这些状态组成一个事务包，**由 Primary 一次性下发**，Replica 负责整体落盘。这就是“一并存盘”的真正含义——历史不是后来追加的，而是激活包的一部分。

## 源码落盘时序与状态演进
我们将严格追踪**“谁在什么时候真正把这些值写入了磁盘”**，以及**缺失数据（Missing Data）**在此过程中的角色。

### 阶段 1：Epoch 100（初始正常）
*   **Acting Set**: [A (Primary), B (Replica)]
*   **Primary 落盘 Info**:
    *   A 决定激活 Epoch 100，将自身的 **info = 100** 写入事务。
*   **下发激活**:
    *   A 向 B 发送 **Activate 消息**（包含 `info=100` 以及已计算好的 `history=100`）。
    *   *注意：A 此时还没有更新本地的 history。*
*   **Replica 落盘 Info**:
    *   B 收到消息，落盘激活事务。此时只更新本地的 **info = 100**。
    *   *关键点*：B 此时的 **history 仍是旧值（例如 90）**。B 在 ACK 回复中告诉 A 自己已激活到 100，但 B 并不会自己单方面把 history 改成 100。
*   **Primary 落盘 History (权威确认)**:
    *   A 收到 B 的 ACK。触发 `AllReplicasActivated` 状态迁移，确认所有副本已落盘。
    *   **A 的动作**：A 将自身的 **history = 100** 写入磁盘。此时 100 成为权威全局状态。
*   **Replica 落盘 History**:
    *   A 发送 `MInfoRec`（包含 `history=100`）给 B。
    *   B 收到后，最终将自身的 **history 更新为 100** 并落盘。
*   **阶段结果**：A、B 最终均为 `info=100, history=100`。

### 阶段 2：Epoch 110（B 掉线，新 OSD C 加入）
*   **Acting Set**: [A (Primary), C (Replica)]
*   **下发激活**: A 向 C 发送 Activate（Epoch 110，包含 info/history 目标值）。
*   **Replica 落盘 Info**:
    *   C 落盘事务。C 的 **info 更新为 110**，但 **history 仍停留在 100**（等待 Primary 确认）。
*   **Primary 落盘 History**:
    *   A 收到 C 的 ACK。A 更新自身的 **history = 110** 并落盘。
*   **Replica 落盘 History**:
    *   A 通知 C。C 接收权威 Info，最终将 **history 落盘为 110**。
*   **阶段结果**：A、C 均为 `info=110, history=110`。B 停留在旧状态。

### 阶段 3：Epoch 120（A 掉线，C 成为 Primary，B 上线）
*   **Acting Set**: [C (Primary), B (Replica)]
    *   C 接替 A，B 带着 `Epoch 100` 的旧状态上线。
*   **Peering 与缺失数据补全**:
    *   B 汇报旧状态，C 发现 B 落后。
    *   **“缺失数据（Missing Data）”内容**：指 B 在掉线期间（Epoch 100 -> 120），未能接收到的所有**新增 Object 数据**、**被覆盖对象的旧版本**以及对应的 **PG_Log 变更记录**。
    *   C 通过 Recovery 机制将这些缺失数据同步给 B。
*   **Replica 落盘 Info**:
    *   C 下发 Activate（Epoch 120）。B 将收到的完整数据（含缺失数据修复）落盘，更新 **info = 120**。
*   **Primary 落盘 History**:
    *   C 收到 B 的 ACK，确认 B 已安全。C 更新自身的 **history = 120** 并落盘。
*   **Replica 落盘 History**:
    *   C 将权威 History 下发给 B。B 最终将 **history 落盘为 120**。
*   **阶段结果**：历史链条在 C 的带领下重新衔接，全局水位成功跃升至 120。

**总结源码中的严格顺序**：
1.  **Primary 决定值**（激活 Epoch = info/history 目标值）。
2.  **Replica 落盘 Info**（仅确认本地已就绪，**不修改** history）。
3.  **Primary 落盘 History**（收到确认后，确立权威历史）。
4.  **Replica 落盘 History**（接收并同步权威历史）。
