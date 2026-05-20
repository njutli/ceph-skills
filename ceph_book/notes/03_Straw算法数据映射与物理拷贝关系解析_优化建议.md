# Note 03 优化建议

## 1. 核心概念修正：确定性 vs 随机性
- **原文错误**：原文提到 "straw 算法会随机地将...数据重新映射"。
- **改正**：CRUSH 映射是**确定性**的（Deterministic），而非随机。对于相同的输入（Object Name, CRUSH Map, Weights），输出永远一致。所谓的"随机"是指哈希函数带来的**统计均匀分布**特性。建议修改为 "CRUSH 算法通过确定性哈希计算，将数据均匀分布..."。
- **源码依据**：`src/crush/mapper.c` 中的 `crush_do_rule` 函数。输入相同的 `x` (PG ID) 和规则，输出相同的 OSD 列表。
