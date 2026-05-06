# Q&A: `extent map` 代码定义与表 2-10 的对应关系

## 问题
在 2.2.2 节中，表 2-10 定义了 `extent map` 的磁盘数据结构。
**疑问**：
1. `extent map` 在 Ceph 源码中的具体定义在哪里？
2. 代码中的成员与书中表 2-10 是如何对应的？

## 解答

### 1. 代码定义位置
`extent map` 的核心定义位于 `src/os/bluestore/BlueStore.h` 中，它被定义为 `BlueStore` 类内部的一个结构体 `ExtentMap`。

### 2. 表 2-10 与源码成员的精确对应

| 书中表 2-10 成员 | 源码成员 (BlueStore.h) | 对应代码行号 | 说明 |
| :--- | :--- | :--- | :--- |
| **`extent_map`** | `extent_map_t extent_map` | Line 935 | 逻辑段集合。类型为 `boost::intrusive::set<Extent>`，存储了对象内所有的逻辑 Extent。 |
| **`spanning_blob_map`** | `blob_map_t spanning_blob_map` | Line 936 | 跨分片 Blob 集合。类型为 `std::map<int, BlobRef>`。 |
| **`shards`** | `vector<Shard> shards` | Line 945 | 分片数组。每个分片是一个 `Shard` 结构体。 |
| ↳ `offset` | `shard_info->offset` | Line 939 (指向 Line 1129) | 分片的逻辑起始地址。存储在 `bluestore_onode_t::shard_info` 中。 |
| ↳ `shard_info` | `bluestore_onode_t::shard_info` | Line 1128 (`bluestore_types.h`) | 分片的元数据信息，包含 `offset` (起始地址) 和 `bytes` (编码长度)。 |
| ↳ `loaded` | `bool loaded` | Line 941 | 标记该分片是否已从 KVDB 加载到内存。 |
| ↳ `dirty` | `bool dirty` | Line 942 | 标记该分片是否被修改，需要重新编码并写回 KVDB。 |

### 3. 代码结构解析 (`BlueStore.h` Line 933-945)
```cpp
/// a sharded extent map, mapping offsets to lextents to blobs
struct ExtentMap {
  Onode *onode;
  extent_map_t extent_map;        ///< map of Extents to Blobs (表 2-10: extent_map)
  blob_map_t spanning_blob_map;   ///< blobs that span shards (表 2-10: spanning_blob_map)

  struct Shard {
    bluestore_onode_t::shard_info *shard_info = nullptr; // (表 2-10: shard_info)
    unsigned extents = 0;  
    bool loaded = false;   ///< true if shard is loaded (表 2-10: loaded)
    bool dirty = false;    ///< true if shard is dirty (表 2-10: dirty)
  };

  mempool::bluestore_cache_meta::vector<Shard> shards;    ///< shards (表 2-10: shards)
  
  // ...
};
```

### 4. 补充说明：`shard_info` 的定义 (`bluestore_types.h` Line 1128)
书中提到的 `shard_info`（表 2-9）在代码中定义如下：
```cpp
struct shard_info {
  uint32_t offset = 0;  ///< logical offset for start of shard (表 2-9: offset)
  uint32_t bytes = 0;   ///< encoded bytes (表 2-9: bytes)
  // ...
};
```
这个结构体在 `bluestore_onode_t` (onode 磁盘结构) 中以 vector 形式持久化存储 (`extent_map_shards`)，而在内存中的 `ExtentMap` 结构里，它被 `Shard` 结构体引用，并附带了 `loaded` 和 `dirty` 等运行时状态。
