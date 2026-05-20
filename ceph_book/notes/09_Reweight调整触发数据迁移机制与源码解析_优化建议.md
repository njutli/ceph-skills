# Note 09 优化建议

## 1. 概念区分：`OSDMap` 权重 vs `CrushMap` 权重
- **原文问题**：原文提到"Monitor 会更新全局的 CRUSH Map"。
- **优化建议**：准确来说，`ceph osd reweight` 更新的是 `OSDMap` 中的 **`new_weight` 数组**（运行时权重），而不是 `CrushMap` 的拓扑结构或静态权重。CRUSH 算法在计算时，会读取 `OSDMap` 中的 `reweight` 值与 `CrushMap` 中的静态权重相乘，得到最终的有效权重。
- **源码依据**：`src/mon/OSDMonitor.cc` 中 `pending_inc.new_weight[id] = ww;`。

## 2. 移除 `UpmapItem` 引用
- **原文问题**：原文在"状态检测"部分提到了 `UpmapItem`。
- **优化建议**：`UpmapItem` 是用于 `ceph osd pg-upmap` 命令的显式 PG 覆盖映射机制，与 `reweight` 的自动 CRUSH 重计算机制无关。建议删除 `UpmapItem` 的提及，避免混淆。

## 3. 迁移触发流程精确化
- **原文问题**：原文描述触发过程较为笼统。
- **优化建议**：补充说明：OSD 在 `activate_map()` 函数中加载新 OSDMap 后，会调用 `calc_acting()` 重新计算 `acting` 和 `up` 集合。如果 `new_up` 与 `old_up` 不一致，PG 状态机会进入 `Backfilling` 状态，从而触发数据迁移。

## 4. 术语统一
- **原文问题**：原文混用了 `Backfill` 和 `Recovery`。
- **优化建议**：在 `reweight` 场景下，数据迁移通常称为 **Backfill**（因为数据是完整的，只是需要复制到新位置）。`Recovery` 通常指 OSD 宕机恢复后的数据修复。建议统一使用 `Backfill` 或 `数据重分布`。
