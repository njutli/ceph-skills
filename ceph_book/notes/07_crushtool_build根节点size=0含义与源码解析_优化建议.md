# Note 07 优化建议

## 1. 适用场景补充
- **原文问题**：原文解释了 `size=0` 的含义和源码实现，但未提及实际使用场景。
- **优化建议**：补充说明 `crushtool --build` 通常用于**测试环境快速生成 CRUSH Map** 或 **离线模拟**。在生产环境中，CRUSH Map 通常通过 `ceph osd crush` 命令手动构建或由 Orchestrator (如 Rook/Cephadm) 自动生成，较少直接使用 `--build` 参数。

## 2. 命名规则细节
- **原文问题**：原文提到 `size > 0` 时生成 `root0`, `root1`。
- **优化建议**：可以补充说明，如果 `size > 0`，`crushtool` 会创建**多个** Root Bucket（例如 `root0`, `root1`），这在多集群联邦或数据隔离场景中有用，但单集群通常只需要一个 Root。`size=0` 确保了只生成一个收敛所有节点的单一 Root。

## 3. 源码位置更新
- **原文问题**：行号可能随版本变化。
- **优化建议**：建议注明源码版本（如 Ceph Reef/Squid），因为行号在不同版本中可能偏移。核心逻辑在 `src/tools/crushtool.cc` 的 `build_crush_map` 函数中。
