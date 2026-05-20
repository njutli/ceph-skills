# Note 09 优化建议

## 1. 概念区分：`OSDMap` 权重 vs `CrushMap` 权重
- **原文错误**：原文提到"Monitor 会更新全局的 CRUSH Map"。
- **改正**：准确来说，`ceph osd reweight` 更新的是 `OSDMap` 中的 **`new_weight` 数组**（运行时权重），而不是 `CrushMap` 的拓扑结构或静态权重。CRUSH 算法在计算时，会读取 `OSDMap` 中的 `reweight` 值与 `CrushMap` 中的静态权重相乘，得到最终的有效权重。
- **源码依据**：`src/mon/OSDMonitor.cc` 中 `pending_inc.new_weight[id] = ww;`。
