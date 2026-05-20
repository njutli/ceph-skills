# Note 10 优化建议

## 1. COW 机制的精确描述
- **原文错误**：原文将 BlueStore 的行为直接称为 COW（写时复制）。
- **改正**：BlueStore 对**元数据**（Onode, Extent Map）严格使用 COW（通过 RocksDB WAL/Manifest）。对**数据**，如果是 4K 对齐的覆盖写，BlueStore 支持 **Direct Overwrite**（原地覆盖），并不总是 COW。只有**非对齐写**或**部分覆盖写**时，才会表现出类似 COW 的行为（写新位置 + 更新映射）。建议补充这一细节，避免读者误以为 BlueStore 所有写入都是 COW。
