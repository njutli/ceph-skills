# 41. PG 创建机制与 Monitor 下发逻辑解析

本节解答阅读 4.3.2 小节"创建 PG"时产生的两个核心疑问，深入理解 Ceph 控制面与数据面的协作。

## 1. 疑问一：标题是“创建 PG"，为什么开头说收到“存储池创建命令”？

这是**因果关系**和**触发源**的问题。

*   **PG 是 Pool 的从属资源**：在 Ceph 中，用户不能直接“创建一个 PG"。用户只能创建“存储池（Pool）”或者调整存储池的 `pg_num`。
*   **触发链条**：
    1.  用户执行 `ceph osd pool create mypool 64`。
    2.  Monitor 收到这个命令，计算出这个 Pool 需要 64 个 PG。
    3.  Monitor 更新 OSDMap，在 Map 中增加了这 64 个 PG 的映射关系。
    4.  OSD 收到新的 OSDMap，发现“哎？多了几个 PG 需要我处理”。
    5.  **于是，“创建 PG"的动作被触发了。**

所以，虽然这节讲的是 PG 的创建过程，但**入口**必然是存储池的创建（或扩容）。

---

## 2. 疑问二：为什么“向每个 OSD 下发”？只给 Primary 下发不就够了吗？

文中提到“向池中每个 OSD 下发批量创建 PG 命令”，随后又说“仅需要向当前 Primary 所在的 OSD 下发 PG 创建请求”，这看似矛盾，实则描述了**不同粒度**的视角。

### 2.1 宏观视角（Pool 级别）：为什么“向每个 OSD 下发”？
当你创建一个存储池（Pool）时，这个 Pool 包含成百上千个 PG。这些 PG 是根据 CRUSH 算法分布在集群的所有 OSD 上的。
*   OSD 1 是 PG 1, 5, 9 的 Primary。
*   OSD 2 是 PG 2, 6, 10 的 Primary。
*   OSD 3 是 PG 3, 7, 11 的 Primary。

因为**每个 OSD 都会充当一部分 PG 的 Primary**，所以 Monitor 必须向**集群中的每个 OSD** 发送消息，告诉它们：“嘿，你有几个新 PG 需要创建。”
这就是书中说的“向池中每个 OSD 下发批量创建 PG 命令”。

### 2.2 微观视角（单个 PG 级别）：为什么“仅向 Primary 下发”？
虽然 Monitor 给每个 OSD 都发了消息，但**消息的内容是经过过滤的**。
*   Monitor 发给 OSD 1 的列表里，**只包含** OSD 1 是 Primary 的 PG（如 PG 1, 5, 9）。
*   Monitor **绝不会**在发给 OSD 1 的列表中包含 PG 2（因为 PG 2 的 Primary 是 OSD 2）。

对于**任意一个特定的 PG**（比如 PG 1），Monitor 确实**只通知了它的 Primary（OSD 1）**，而没有直接通知它的 Replica（OSD 2 和 OSD 3）。
这就是书中说的“仅需要向当前 Primary 所在的 OSD 下发 PG 创建请求”。

### 2.3 为什么不矛盾？
这两句话描述的是同一个过程的两个侧面：
*   **“向每个 OSD 下发”**：描述的是 Monitor 的**广播行为**（为了覆盖 Pool 里所有的 PG）。
*   **“仅向 Primary 下发”**：描述的是**单个 PG 的创建机制**（Monitor 不直接指挥 Replica，而是让 Primary 去指挥）。

### 2.4 为什么要这样设计？
这是为了**减轻 Monitor 的压力**并**利用 Primary 的协调能力**。
*   如果 Monitor 需要直接通知每个 PG 的所有 Replica，那么通信量将是巨大的（N 个 PG × 3 个副本）。
*   现在，Monitor 只需要通知 N 个 Primary。然后由 Primary 在 Peering 过程中去“摇人”（通知 Replica 创建 PG）。这样就把协调工作分散到了各个 OSD 上，提高了扩展性。

---

## 3. 总结

*   **Monitor** 是**发令枪**：决定“谁当老大”以及“开始干活”。它给每个 OSD 发了一份“任务清单”，但这份清单里**只列出了该 OSD 当老大的项目**。
*   **Primary** 是**领跑员**：收到指令后，自己先跑起来，并招呼后面的 Replica 一起跑（在 Peering 过程中自动创建 Replica）。

两者并不矛盾，而是上下游的协作关系。
