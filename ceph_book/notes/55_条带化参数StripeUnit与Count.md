# 55_RBD条带化：Stripe Unit 与 Count 解析

## 核心区别 (Table 6-2)

在 Table 6-2 `rbd_header` 元数据定义中，这两个参数定义了 RBD image 在底层 RADOS 对象分布的逻辑：

### 1. Stripe Unit (`stripe_unit`)
*   **定义**：条带单元大小。
*   **作用**：决定了数据“切块”的大小。即：每次向单个 RADOS 对象连续写入的数据量。
*   **效果**：当对 RBD 的写入达到 `stripe_unit` 大小时，数据会被路由到下一个对象。

### 2. Stripe Count (`stripe_count`)
*   **定义**：条带数量。
*   **作用**：决定了“分发给几个对象”。即：一个完整的数据条带（Stripe）由多少个对象组成。
*   **效果**：当数据依次写满 `stripe_count` 个对象后，下一个数据块会回到第一个对象（Object_0）。

## 疑问解答：回到 Object_0 会破坏原有数据吗？

**绝对不会。**

用户常误以为“回到 Object_0"意味着覆盖开头的数据。实际上，这里涉及两个维度的寻址：
1.  **对象维度（水平轮询）**：按照 `Object_0 -> Object_1 -> ... -> Object_n` 的顺序循环选择对象 ID。
2.  **对象内部（垂直追加）**：每个对象内部的**写入偏移量 (Offset)** 是递增的，永远不会重复覆盖。

### 示例模型
假设：`stripe_unit` = 4MB，`stripe_count` = 2。

*   **第 1 轮 (Offset 0 - 8MB)**
    *   Block 1 (0~4MB) -> 写入 **Object 0**，Offset: 0。
    *   Block 2 (4~8MB) -> 写入 **Object 1**，Offset: 0。

*   **第 2 轮 (Offset 8MB - 16MB)** —— *此时轮询回到 Object 0*
    *   Block 3 (8~12MB) -> 写入 **Object 0**，但 **Offset 变为 4MB**。*(追加)*
    *   Block 4 (12~16MB) -> 写入 **Object 1**，但 **Offset 变为 4MB**。*(追加)*

**结果**：Object 0 内部会包含 Block 1 和 Block 3 的数据，物理存储上是连续增长的，互不干扰。
