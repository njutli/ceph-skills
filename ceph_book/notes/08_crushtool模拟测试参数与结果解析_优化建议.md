# Note 08 优化建议

## 1. 参数拼写修正
- **原文错误**：原文使用了 `--show/utilization` 和 `--showMappings`。
- **改正**：根据源码 `src/tools/crushtool.cc`，正确的参数应为：
  - `--show_utilization` (使用下划线，而非斜杠)
  - `--show_mappings` (使用下划线和小写，而非驼峰)
- **影响**：如果用户直接复制原文命令执行，会报错 `unrecognized option`。
