# Note 06 优化建议

## 1. 关键参数差异：`jewel` vs `optimal`
- **原文问题**：原文提到"生产环境中建议使用 `optimal`（等同于最新版 Jewel 配置）"。
- **优化建议**：`optimal` 并不完全等同于 `jewel`。在源码 `src/crush/CrushWrapper.h` 中，`set_tunables_optimal()` 调用了 `set_tunables_jewel()`，但额外设置了 **`crush->straw_calc_version = 1`**。
- **影响**：`straw_calc_version = 1` 启用了 Straw2 算法的改进版权重计算逻辑，能更精确地处理非整数权重和极小权重 OSD。建议明确区分 `jewel` 和 `optimal` 的这一细微但重要的差别。

## 2. `allowed_bucket_algs` 的具体值
- **原文问题**：原文仅提到"允许使用的 Bucket 算法，如 straw, straw2"。
- **优化建议**：可以补充说明 `allowed_bucket_algs` 是一个位掩码（Bitmask）。在 `jewel` 中，它允许 `UNIFORM`, `LIST`, `STRAW`, `STRAW2`。而在现代 Ceph 中，`STRAW2` 是绝对主流，其他算法（如 `LIST`）由于数据迁移量过大已不再推荐。

## 3. `straw_calc_version` 的作用
- **原文问题**：原文未解释 `straw_calc_version` 的含义。
- **优化建议**：补充说明 `straw_calc_version` 控制的是 Straw/Straw2 算法内部计算"权重->概率"转换的数学公式版本。版本 1 修复了版本 0 中在某些权重分布下数据倾斜的问题。

## 4. 源码位置补充
- **优化建议**：可以补充 `src/crush/crush.h` 中的 `struct crush_map` 定义，这是所有 tunables 参数的底层存储结构。
