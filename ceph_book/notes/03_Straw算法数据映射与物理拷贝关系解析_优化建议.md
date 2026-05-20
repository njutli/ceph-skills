# Note 03 优化建议

## 1. 核心概念修正：确定性 vs 随机性
- **原文问题**：原文提到 "straw 算法会随机地将...数据重新映射"。
- **优化建议**：CRUSH 映射是**确定性**的（Deterministic），而非随机。对于相同的输入（Object Name, CRUSH Map, Weights），输出永远一致。所谓的"随机"是指哈希函数带来的**统计均匀分布**特性。建议修改为 "CRUSH 算法通过确定性哈希计算，将数据均匀分布..."。
- **源码依据**：`src/crush/mapper.c` 中的 `crush_do_rule` 函数。输入相同的 `x` (PG ID) 和规则，输出相同的 OSD 列表。

## 2. 算法版本更新：Straw vs Straw2
- **原文问题**：标题和内容仅提及 "Straw"。
- **优化建议**：Ceph 目前默认使用 **Straw2** 算法。Straw2 相比 Straw 修复了权重分布不均的问题，并且在 OSD 增删时能更精确地控制数据迁移量。建议在笔记中补充 Straw2 的优势。
- **源码依据**：`src/crush/crush.h` 中定义了 `CRUSH_RULE_CHOOSELEAF_STRAW2`。

## 3. 术语精确化：Backfill vs Rebalance
- **原文问题**：原文将 Backfill 和 Rebalance 混用。
- **优化建议**：
  - **Backfill**：通常指为新 OSD 填充副本以满足 Pool 的 `size` 要求（例如从 2 副本变 3 副本，或新 OSD 加入后承担部分 Primary/Replica 角色）。
  - **Rebalance**：指在现有 OSD 之间移动数据以达到负载均衡（例如新 OSD 加入后，旧 OSD 释放部分数据给新 OSD）。
  - 建议在笔记中区分这两个概念，或者统称为 "数据重分布 (Data Redistribution)"。

## 4. 映射变化的触发机制
- **原文问题**：原文未提及映射变化是如何触发数据移动的。
- **优化建议**：补充说明：当 CRUSH 计算出的新 Acting Set 与当前 Acting Set 不一致时，OSDMap 的 Epoch 会增加。OSD 监控到 OSDMap 变化后，会启动 **Peering** 过程，随后根据差异开始 **Backfill/Recovery**。
- **源码依据**：`src/osd/OSD.cc` 中处理 `OSDMap` 更新的逻辑。

## 5. 总结优化
- 建议将"映射 = 决定谁该搬"改为"映射 = 计算数据的新归属（确定性计算）"。
- 建议将"拷贝 = 执行搬运动作"改为"数据移动 = 根据映射差异执行 Backfill/Rebalance"。
