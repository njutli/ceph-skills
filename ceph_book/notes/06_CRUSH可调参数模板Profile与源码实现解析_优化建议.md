# Note 06 优化建议

## 1. 关键参数差异：`jewel` vs `optimal`
- **原文错误**：原文提到"生产环境中建议使用 `optimal`（等同于最新版 Jewel 配置）"。
- **改正**：`optimal` 并不完全等同于 `jewel`。在源码 `src/crush/CrushWrapper.h` 中，`set_tunables_optimal()` 调用了 `set_tunables_jewel()`，但额外设置了 **`crush->straw_calc_version = 1`**。建议明确区分 `jewel` 和 `optimal` 的这一差别，`optimal` 启用了 Straw2 算法的改进版权重计算逻辑。
