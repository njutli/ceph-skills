# Note 10 优化建议

## 1. 术语修正：Extent vs Block
- **原文问题**：原文使用了 "Block" 来描述存储单元。
- **优化建议**：BlueStore 底层使用的是 **Extent（不定长连续物理区域）** 而非固定大小的 "Block"。在部分覆盖写时，BlueStore 会进行 **Extent Split（分裂）**：将原 Extent 拆分为"新写入的小 Extent"和"保留的旧 Extent"。建议使用 "Extent" 替代 "Block" 以更贴合 BlueStore 源码（`src/os/bluestore/BlueStore.cc` 中的 `extent_map`）。

## 2. COW 机制的精确描述
- **原文问题**：原文将 BlueStore 的行为直接称为 COW。
- **优化建议**：BlueStore 对**元数据**（Onode, Extent Map）严格使用 COW（通过 RocksDB WAL/Manifest）。对**数据**，如果是 4K 对齐的覆盖写，BlueStore 支持 **Direct Overwrite**（原地覆盖），并不总是 COW。只有**非对齐写**或**部分覆盖写**时，才会表现出类似 COW 的行为（写新位置 + 更新映射）。建议补充这一细节，避免读者误以为 BlueStore 所有写入都是 COW。

## 3. 碎片回收机制
- **原文问题**：原文提到后台 Compaction 整理数据。
- **优化建议**：BlueStore 的碎片回收主要依赖 **Garbage Collection (GC)** 线程（`BlueStore::GarbageCollector`），它会扫描包含大量"空洞"或无效数据的 Extents，将其有效数据迁移并合并，释放旧空间。RocksDB 的 Compaction 主要负责元数据整理。建议区分数据 GC 和元数据 Compaction。

## 4. 性能权衡总结
- **优化建议**：可以补充说明，这种设计使得 BlueStore 的写入放大（Write Amplification）极低，因为大部分写入都是追加写（Append-only），充分利用了 SSD 的顺序写性能。
