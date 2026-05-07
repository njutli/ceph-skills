# 2.5 BlueStore 架构与组件关系解析

## 问题：OSD, PG, BlueStore, RocksDB, BlueFS 各组件在代码层面的关系是什么？

### 解答

从代码实现角度看，**OSD 进程负责所有“逻辑控制”和“分布式协调”，而 BlueStore 纯粹作为底层的“存储引擎”提供物理数据管理能力**。两者通过 `ObjectStore` 抽象接口严格解耦。RocksDB 和 BlueFS 则是 BlueStore 内部用于高效管理元数据的组件。

---

## 1. 职责边界（源码层级）

| 组件 | 源码路径 | 核心职责 | 关键数据结构 |
|:---|:---|:---|:---|
| **OSD** | `src/osd/` | 网络交互、PG 状态机（Peering/Active）、版本控制、主从同步、Recovery/Scrub、事务构建 | `PrimaryLogPG`, `ObjectContext`, `pg_info_t`, `pg_log` |
| **BlueStore** | `src/os/bluestore/` | 实现 `ObjectStore` 接口、内存缓存、磁盘空间分配、元数据持久化（RocksDB）、数据落盘 | `Collection`, `Onode`, `Blob`, `Allocator`, `BufferCacheShard` |
| **RocksDB** | 第三方库 | 存储所有元数据（对象属性、位置映射、PG 日志等），提供 KV 查询 | `DB`, `WriteBatch` |
| **BlueFS** | `src/os/bluestore/BlueFS.*` | RocksDB 专用的极简用户态文件系统，替代 XFS/ext4，减少 IO 开销 | `bluefs_fnode_t`, `bluefs_transaction_t` |

---

## 2. 组件层级架构图

```text
[ OSD 进程 ] (负责 PG 逻辑、网络、分布式一致性)
      │
      ▼
[ ObjectStore 接口 ] (BlueStore 实现此接口)
      │
      ├── 处理用户数据 (Object Payload) ──────────────────────┐
      │      │                                               │
      │      ▼                                               │
      │ [ Allocator ] (分配磁盘空间)                          │
      │      │                                               │
      │      ▼                                               │
      │ [ Block Device ] (裸磁盘，直接写数据 Blob)            │
      │                                                      │
      │                                                      │
      └── 处理元数据 (Onode, PG Log, Omap) ──────────────────┘
             │
             ▼
       [ RocksDB ] (KV 数据库，存储所有元数据)
             │
             ▼
       [ BlueFS ] (RocksDB 专用的极简用户态文件系统)
             │
             ▼
       [ Block Device ] (通常是同一块盘，或者是独立的高速 SSD)
```

---

## 3. 数据结构映射关系

OSD 层管理的是**逻辑状态**，BlueStore 层管理的是**物理布局**。它们通过以下结构一一对应：

| OSD 层（逻辑） | BlueStore 层（物理） | 交互方式 |
|:---|:---|:---|
| `PrimaryLogPG` | `BlueStore::Collection` | 1:1 映射。PG 是控制面实体，Collection 是数据面实体。 |
| `ObjectContext` | `BlueStore::Onode` | OSD 管版本/快照/锁；BlueStore 管磁盘映射 (ExtentMap) 与缓存。 |
| `ObjectStore::Transaction` | `BlueStore::TransContext` | OSD 构建操作列表（写数据/改属性/删对象），BlueStore 解析并执行。 |
| `pg_log` / `pg_info_t` | `omap` (RocksDB) | PG 日志和状态以 Key-Value 形式存在 BlueStore 的 Meta Object 中。 |

---

## 4. RocksDB 与 BlueFS 的定位

*   **为什么需要 RocksDB？**
    *   对象的位置映射（ExtentMap）、属性、PG 日志等元数据需要频繁读写和查询。使用 KV 数据库可以极大提高查找效率。
    *   BlueStore 将**数据与元数据分离**：数据直接写裸盘（Blob），元数据写 RocksDB。

*   **为什么需要 BlueFS？**
    *   RocksDB 原本运行在通用文件系统（如 XFS）上，但通用文件系统太重，消耗大量 CPU 和 IO。
    *   BlueFS 是 Ceph 为 RocksDB 定制的“极简文件系统”，只保留 RocksDB 需要的最小功能集（创建文件、读写、同步），去掉了所有不必要的开销。

---

## 5. 一次写操作的代码流转

1.  **OSD 接收与路由**：`OSD::ms_fast_dispatch` → `PG::queue_op` → `PrimaryLogPG::do_request`
2.  **构建事务**：OSD 在 `execute_ctx` 中将数据写入、属性更新、omap 修改等操作封装进 `ObjectStore::Transaction t`。
3.  **提交存储**：调用 `store->queue_transactions(osr, t, on_applied, on_commit)`。**此时控制权正式交给 BlueStore**。
4.  **BlueStore 执行**：
    *   解析 Transaction，定位到对应的 `Collection` 和 `Onode`。
    *   数据写入 `BufferSpace`（缓存），调用 `Allocator` 分配磁盘块，生成 `Blob` 并记录 Extent 映射。
    *   将元数据（Onode 信息、omap）写入 RocksDB，数据异步刷盘。
    *   刷盘完成后，通过回调 `on_applied` / `on_commit` 通知 OSD。
5.  **OSD 后续**：收到回调后，主 OSD 继续处理副本同步或向客户端返回 ACK。

---

## 6. 形象比喻

如果把 Ceph OSD 比作一家**图书馆**：

1.  **OSD (馆长)**：负责管理借阅规则、目录索引、处理读者请求。
2.  **Object (书的内容)**：直接存放在书架上（**裸磁盘**），为了拿取快，不包装在复杂的盒子里。
3.  **RocksDB (图书检索系统/电脑)**：记录每本书在哪个书架、第几层（元数据）。馆长查电脑就知道去哪拿书。
4.  **BlueFS (检索系统的专用操作系统)**：为了让检索电脑运行得更快，专门给它装了一个精简版系统，去掉了所有花哨的功能，只保证开机快、查得快。

**总结**：`BlueStore = 裸盘数据管理 (Allocator) + 元数据管理 (RocksDB on BlueFS)`。
