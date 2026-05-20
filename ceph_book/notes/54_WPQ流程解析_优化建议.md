# Note 54 优化建议

## 1. WPQ 的现代演进
- **原文问题**：原文将 WPQ 描述为"新版"调度策略。
- **优化建议**：在现代 Ceph 版本中，**dmClock** 已经逐渐取代了传统的 Prio 和 WPQ 调度器，成为默认的 QoS 机制。dmClock 提供了更精确的分布式速率控制和预留/限制保障。建议注明"WPQ 对应中期版本的调度实现，现代 Ceph 推荐使用 dmClock"。

## 2. `rand() % total_prio` 的公平性
- **原文问题**：原文提到通过随机数选择队列。
- **优化建议**：补充说明 WPQ 的随机选择在大样本下趋近于权重比例，但在短时间内可能存在波动。为了平滑这种波动，WPQ 通常会结合 **Virtual Time（虚拟时间）** 或 ** deficit counting（赤字计数）** 机制，确保长期来看各队列获得的资源严格符合权重设定。

## 3. Cost Check 的阈值计算
- **原文问题**：原文公式 `rand(0~99) <= 100 - (req_size * 0.9)`。
- **优化建议**：指出这个公式中的系数（如 0.9）和 `max_cost` 是可调参数（如 `osd_op_cost_threshold`）。管理员可以根据硬件性能（如 SSD 的大 IO 吞吐能力）调整这些参数，以平衡小 IO 延迟和大 IO 吞吐。

## 4. 源码位置
- **优化建议**：WPQ 的调度逻辑位于 `src/osd/OpQueue.h` 和 `src/osd/OpQueue.cc` 的 `WeightedPriorityQueue` 类中。`choose_queue` 和 `choose_item` 函数分别实现了队列选择和 Cost 过滤。
