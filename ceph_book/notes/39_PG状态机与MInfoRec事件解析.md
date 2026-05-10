# 39. PG 状态机与 MInfoRec 事件解析

本节澄清 PG 状态机的作用范围以及 Peering 过程中关键事件 `MInfoRec` 的触发机制。

## 1. 状态机是针对“单个 PG 实例”而言的

图 4-9 所示的 PG 状态机，是**针对每个 OSD 上的单个 PG 实例（PG Instance）**的，而不是针对整个逻辑 PG 的全局状态。

**源码证据：**
在 Ceph 源码中，`PG` 类（定义于 `src/osd/PG.h`）直接继承了 `boost::statechart::state_machine`：

```cpp
class PG : public boost::statechart::state_machine< PG, Reset >, ... {
    // ...
};
```

这意味着，OSD 进程中每加载一个 PG 副本，就会在内存中实例化一个**独立的 `PG` 对象**，每个对象都拥有自己独立的状态机。它们各自根据自己收到的消息（Event）进行状态流转。

## 2. MInfoRec 事件的含义

`MInfoRec` 代表 **Message Info Received**。

*   **场景**：在 Peering 过程中，**Primary PG 实例** 会向其他 OSD（Peers）发送 `Query` 消息索要 Info。
*   **触发**：当 **Primary PG 实例** 收到 Peer 回复的 `MOSDPGInfo` 消息时，就会触发 `MInfoRec` 事件。
*   **作用**：该事件驱动 Primary 的状态机从“等待 Info"的状态向前推进（例如进入 `GetLog` 或 `GetMissing` 状态），以便开始构建 PriorSet 和同步日志。

**总结：**
虽然 Replica（从副本）也有自己的状态机，但 `MInfoRec` 这个特定事件主要发生在 **Primary** 侧。Replica 侧的状态机通常是由收到 Primary 发来的写请求、Log 同步请求（`MOSDPGLog`）等消息来驱动的。
