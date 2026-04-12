---
name: mds-cache-objects
description: CephFS MDS 核心缓存对象（CInode/CDentry/CDir）。当用户询问 MDS 内存数据结构、目录缓存、分布式元数据管理、子树分区时调用此技能。
---

# MDS 缓存核心数据结构

## 1. 问题起源

CephFS 需要在多个 MDS 之间分布式管理文件系统元数据。元数据包括：
- 文件/目录的属性（size, mode, uid, gid, time 等）
- 目录结构（哪些文件在哪个目录下）
- 硬链接关系

核心挑战：
1. **内存管理**：如何高效缓存海量元数据？
2. **一致性**：如何保证多 MDS 之间的元数据一致性？
3. **负载均衡**：如何将元数据分布到多个 MDS？
4. **持久化**：如何将元数据写入 RADOS？

## 2. 三大核心对象

```
                    CephFS 元数据缓存层次结构

     ┌─────────────────────────────────────────────────────┐
     │                     MDCache                         │
     │  ┌─────────────────────────────────────────────┐   │
     │  │               CInode (inode)                 │   │
     │  │  - 文件/目录的属性 (size, mode, uid...)      │   │
     │  │  - 指向父 CDentry (parent)                   │   │
     │  │  - 指向子 CDir 列表 (dirfrags)               │   │
     │  │  - Capability (客户端缓存权限)               │   │
     │  └─────────────────────────────────────────────┘   │
     │                       │                            │
     │                       │ parent_dn                  │
     │                       ▼                            │
     │  ┌─────────────────────────────────────────────┐   │
     │  │             CDentry (目录项)                 │   │
     │  │  - name: 文件名                              │   │
     │  │  - linkage: 指向 CInode 或 remote inode     │   │
     │  │  - 所在目录: CDir                            │   │
     │  └─────────────────────────────────────────────┘   │
     │                       │                            │
     │                       │ dir                        │
     │                       ▼                            │
     │  ┌─────────────────────────────────────────────┐   │
     │  │              CDir (目录分片)                 │   │
     │  │  - frag_t: 分片标识                          │   │
     │  │  - items: map<dentry_key, CDentry*>         │   │
     │  │  - 子树权限: dir_auth                        │   │
     │  │  - 属于: CInode (目录的 inode)              │   │
     │  └─────────────────────────────────────────────┘   │
     └─────────────────────────────────────────────────────┘
```

## 3. MDSCacheObject 基类

所有 MDS 缓存对象的基类，定义在 `src/mds/MDSCacheObject.h`。

### 3.1 核心功能

```c++
class MDSCacheObject {
  // 状态位
  unsigned state;
  
  static const int STATE_AUTH      = (1<<30);  // 是权威副本
  static const int STATE_DIRTY     = (1<<29);  // 有未持久化的修改
  static const int STATE_NOTIFYREF = (1<<28);  // 引用归零时通知
  static const int STATE_REJOINING = (1<<27);  // 副本尚未与主副本同步
  
  // 引用计数
  int ref;
  mempool::mds_co::flat_map<int,int> ref_map;  // 引用来源追踪
  
  // auth pin（防止在操作期间被导出）
  int auth_pins;
  
  // 副本映射（哪些 MDS 拥有副本）
  replica_map_type replica_map;  // mds_rank_t -> nonce
  unsigned replica_nonce;         // 本副本的 nonce
};
```

### 3.2 关键方法

| 方法 | 说明 |
|------|------|
| `get(by)` / `put(by)` | 引用计数管理，支持来源追踪 |
| `auth_pin(who)` / `auth_unpin(who)` | 锁定对象，防止导出 |
| `is_auth()` | 是否权威副本 |
| `is_dirty()` | 是否有未持久化修改 |
| `is_replicated()` | 是否有副本在其他 MDS |
| `add_waiter(mask, ctx)` | 添加等待者 |
| `take_waiting(mask, ls)` | 获取并清除等待者 |

## 4. CInode - Inode 缓存对象

定义在 `src/mds/CInode.h`，表示文件或目录的元数据。

### 4.1 继承关系

```c++
class CInode : public MDSCacheObject, 
               public InodeStoreBase, 
               public Counter<CInode> {
  // 继承自 InodeStoreBase
  inode_const_ptr    inode;       // inode 属性
  xattr_map_const_ptr xattrs;      // 扩展属性
  old_inode_map_const_ptr old_inodes;  // 快照历史版本
  
  // 继承自 MDSCacheObject
  // state, ref, auth_pins, replica_map...
};
```

### 4.2 核心成员

```c++
class CInode {
  // 基本属性（只读，通过 project_inode 修改）
  inode_const_ptr inode;  // 包含：
  //   inodeno_t ino;       // inode 号
  //   uint64_t size;       // 文件大小
  //   uint32_t mode;       // 文件类型和权限
  //   uint32_t uid, gid;   // 所有者
  //   utime_t atime, mtime, ctime;
  //   ...
  
  // 父目录项
  CDentry* parent;
  
  // 目录分片（仅目录 inode 有）
  mempool::mds_co::map<frag_t, CDir*> dirfrags;
  fragtree_t dirfragtree;  // 分片树结构
  
  // Capability（客户端缓存权限）
  mempool::mds_co::map<client_t, Capability> client_caps;
  
  // 锁（分布式锁管理）
  // filelock - 文件内容锁
  // nestlock - 嵌套统计锁
  // dnlock - 目录项锁（在 parent CDentry 中）
  
  // 快照支持
  SnapRealm* snaprealm;
  old_inode_map_const_ptr old_inodes;  // 快照历史
  
  // 状态位
  static const int STATE_EXPORTING    = (1<<0);   // 正在导出
  static const int STATE_FREEZING     = (1<<2);   // 正在冻结
  static const int STATE_FROZEN       = (1<<3);   // 已冻结
  static const int STATE_AMBIGUOUSAUTH = (1<<4);  // 权限模糊
  static const int STATE_DIRTYPARENT  = (1<<9);   // 父目录 dirty
  static const int STATE_DIRTYRSTAT   = (1<<10);  // 递归统计 dirty
};
```

### 4.3 投影机制（Projection）

CInode 使用投影机制支持并发修改：

```c++
// 发起修改：创建投影
projected_inode project_inode(const MutationRef& mut, 
                               bool xattr = false, 
                               bool snap = false);

// 提交修改后：弹出并应用
void pop_and_dirty_projected_inode(LogSegmentRef const& ls, 
                                   const MutationRef& mut);

// 获取当前投影或原始值
const inode_const_ptr& get_projected_inode() const;
```

投影机制确保：
1. 正在进行的修改不会被其他操作看到
2. 多个并发修改可以排队
3. 日志提交后原子性应用到主副本

### 4.4 关键方法

```c++
// 路径操作
void make_path_string(std::string& s, bool projected=false);
CDir* get_dirfrag(frag_t fg);         // 获取目录分片
CInode* get_parent_inode();            // 获取父目录 inode

// 持久化
void store(MDSContext *fin);           // 写入 RADOS
void flush(MDSContext *fin);           // 强制刷盘
void fetch(MDSContext *fin);           // 从 RADOS 加载

// 快照
snapid_t get_oldest_snap();
void pre_cow_old_inode();              // Copy-On-Write

// Capability
Capability* get_client_cap(client_t c);

// 子树判断
bool is_subtree_root() const;          // 是否是子树根
```

## 5. CDentry - 目录项缓存对象

定义在 `src/mds/CDentry.h`，表示目录中的一个条目（文件名 → inode 的映射）。

### 5.1 核心成员

```c++
class CDentry : public MDSCacheObject, 
                public LRUObject, 
                public Counter<CDentry> {
  
  // 名称
  mempool::mds_co::string name;        // 文件名
  __u32 hash;                          // 文件名哈希值
  
  // 快照范围
  snapid_t first, last;               // 有效快照范围
  
  // 所属目录
  CDir* dir;                           // 所在的 CDir
  
  // 链接类型
  struct linkage_t {
    CInode* inode;                     // primary: 指向本地 CInode
    inodeno_t remote_ino;              // remote: 指向远程 inode
    unsigned char remote_d_type;       // remote: 文件类型
    CInode* referent_inode;            // referent remote: 引用的实际 inode
    inodeno_t referent_ino;            
    
    bool is_primary() const { return remote_ino == 0 && inode != nullptr; }
    bool is_remote() const { return remote_ino > 0 && referent_inode == nullptr; }
    bool is_null() const { return remote_ino == 0 && inode == nullptr; }
    bool is_referent_remote() const { return remote_ino > 0 && referent_ino != 0; }
  };
  linkage_t linkage;                   // 当前的链接状态
  list<linkage_t> projected;           // 投影中的链接状态
  
  // 版本
  version_t version;
  version_t projected_version;
  
  // 锁
  SimpleLock lock;                     // DN 锁
  
  // 客户端租约
  ClientLeaseMap client_leases;        // 客户端 -> 租约
};
```

### 5.2 三种链接类型

```
CDentry 链接类型：

1. Primary Link（主链接）
   ┌────────┐    linkage.inode    ┌────────┐
   │CDentry │ ─────────────────► │CInode  │
   │ (dir)  │                     │ (本地) │
   └────────┘                     └────────┘
   - 最常见：普通文件/目录
   - CInode 在本 MDS

2. Remote Link（远程链接）
   ┌────────┐    remote_ino      ┌────────┐
   │CDentry │ ────────────────► │ RADOS  │
   │ (dir)  │                   │ (硬链接)│
   └────────┘                   └────────┘
   - 硬链接：同一 inode 在多个目录中出现
   - inode 在其他 MDS 上或已从缓存逐出

3. Referent Remote（引用远程链接）
   ┌────────┐    referent_ino   ┌────────┐
   │CDentry │ ────────────────► │CInode  │
   │ (dir)  │   remote_ino       │ (缓存) │
   └────────┘                   └────────┘
   - 远程 inode 已被加载到缓存
   - referent_inode 指向缓存的 CInode
```

### 5.3 Projection 机制

CDentry 同样支持投影：

```c++
linkage_t* get_projected_linkage();
linkage_t* get_linkage();

// 推入投影
push_projected_linkage(CInode* inode);
push_projected_linkage(inodeno_t ino, char d_type);

// 弹出投影
linkage_t* pop_projected_linkage();
```

### 5.4 关键方法

```c++
// 路径
void make_path_string(std::string& s, bool projected=false);
void make_path(filepath& fp, bool projected=false);

// 远程链接管理
void link_remote(linkage_t* dnl, CInode* remote_in, CInode* ref_in=nullptr);
void unlink_remote(linkage_t* dnl);

// 客户端租约
ClientLease* get_client_lease(client_t c);
ClientLease* add_client_lease(Session* session);
```

## 6. CDir - 目录分片缓存对象

定义在 `src/mds/CDir.h`，表示目录的一个分片。

### 6.1 为什么分片？

```
目录分片的原因：

1. 大目录问题
   - 一个目录可能有数百万文件
   - 单个 MDS 无法有效缓存

2. 解决方案：目录分片（Fragmentation）
   ┌─────────────────────────────────────┐
   │          /data 目录                  │
   │  100 万个文件                         │
   │                                     │
   │  分片 0 (frag_t=0x0):  a-g         │
   │  分片 1 (frag_t=0x1):  h-n         │
   │  分片 2 (frag_t=0x2):  o-t         │
   │  分片 3 (frag_t=0x3):  u-z         │
   └─────────────────────────────────────┘
   
3. 子树分区（Subtree Partitioning）
   - 不同分片可以分配给不同 MDS
   - 实现大目录的负载均衡
```

### 6.2 核心成员

```c++
class CDir : public MDSCacheObject, 
             public Counter<CDir> {
  
  // 所属 inode
  CInode* inode;                       // 这个目录的 inode
  
  // 分片标识
  frag_t frag;                         // 分片号
  
  // 目录内容
  dentry_key_map items;                // map<dentry_key_t, CDentry*>
  
  // 统计信息
  fnode_const_ptr fnode;               // 包含：
  //   frag_info_t fragstat;           // 本分片统计
  //   nest_info_t rstat;              // 递归统计（包括子目录）
  
  // 投影的 fnode
  list<fnode_ptr> projected_fnode;
  
  // 子树权限
  mds_authority_t dir_auth;            // (auth, rep)
  // CDIR_AUTH_DEFAULT = (-1, -1) 表示继承父目录
  // CDIR_AUTH_UNKNOWN = (-2, -2) 表示权限模糊
  
  // 状态
  static const unsigned STATE_COMPLETE  = (1<<0);   // 内容完整
  static const unsigned STATE_EXPORTING = (1<<10);  // 正在导出
  static const unsigned STATE_IMPORTING = (1<<11);  // 正在导入
  static const unsigned STATE_FROZENTREE = (1<<1);  // 子树冻结
  static const unsigned STATE_STICKY     = (1<<13); // 固定不导出
  
  // 引用计数相关
  int num_head_items;                   // head dentry 数量
  int num_head_null;                     // null dentry 数量
  int num_snap_items;                    // 快照 dentry 数量
  int num_dirty;                         // dirty dentry 数量
};
```

### 6.3 子树分区

```
子树分区（Subtree Partitioning）：

           根目录 (/) [MDS.0 权威]
              │
     ┌────────┼────────┐
     │        │        │
   home     var      tmp [MDS.1 权威]
     │                 │
   alice             project1
     │                 │
  [MDS.2 权威]       src [MDS.3 权威]

定义：
- 子树根：CDir.dir_auth != CDIR_AUTH_DEFAULT
- 子树包含该目录及其所有子目录，直到下一个子树根
- 权威 MDS：dir_auth.first
- 副本 MDS：dir_auth.second (如果有)

示例：
- /home/alice 是子树根，MDS.2 权威
  - /home/alice 及其子目录都在 MDS.2 上
  - /home 在 MDS.0 上
- /tmp/project1/src 是子树根，MDS.3 权威
```

### 6.4 关键方法

```c++
// 查找
CDentry* lookup(std::string_view name, snapid_t snap=CEPH_NOSNAP);
CDentry* lookup_exact_snap(std::string_view name, snapid_t last);

// 添加 dentry
CDentry* add_primary_dentry(std::string_view name, CInode* in, ...);
CDentry* add_remote_dentry(std::string_view name, CInode* ref_in, ...);
CDentry* add_null_dentry(std::string_view name, ...);

// 移除 dentry
void remove_dentry(CDentry* dn);
void unlink_inode(CDentry* dn, bool adjust_lru=true);

// 分片操作
void split(int bits, vector<CDir*>* subs, ...);  // 分裂
void merge(const vector<CDir*>& subs, ...);       // 合并
bool should_split() const;                        // 是否应分裂
bool should_merge() const;                        // 是否应合并

// 子树操作
bool is_subtree_root() const { return dir_auth != CDIR_AUTH_DEFAULT; }
bool contains(CDir* x);                           // 是否包含 x

// 导入/导出
void encode_export(buffer::list& bl);
void decode_import(buffer::list::const_iterator& blp, ...);
```

## 7. 对象关系图

```
              对象关系和引用

     ┌──────────────────────────────────────────────────┐
     │                    MDCache                         │
     │                                                    │
     │   inode_map: map<vinodeno_t, CInode*>             │
     │   dentry_map: map<dentry_key_t, CDentry*>        │
     │   dir_map: map<dirfrag_t, CDir*>                  │
     │                                                    │
     └──────────────────────────────────────────────────┘
                          │
                          │ 管理
                          ▼
     ┌──────────────────────────────────────────────────┐
     │                    CInode                         │
     │                                                    │
     │   ino() ─────────────────────────────────► inode_t│
     │   parent ──────────────────────────────► CDentry* │
     │   dirfrags: map<frag_t, CDir*>                   │
     │   client_caps: map<client_t, Capability>         │
     │   filelock, nestlock, ...                        │
     │                                                    │
     └──────────────────────────────────────────────────┘
              │                          │
              │ parent                    │ get_dirfrag(frag)
              ▼                          │
     ┌───────────────────┐               │
     │     CDentry       │               │
     │                   │               │
     │   name            │               │
     │   hash            │               │
     │   dir ────────────┼───────────────┘
     │   linkage.inode ──┼──────► CInode (remote 或 primary)
     │   lock            │
     └───────────────────┘
              │
              │ items
              ▼
     ┌───────────────────┐
     │      CDir         │
     │                   │
     │   inode ───────────┼──────► CInode (目录本身)
     │   frag            │
     │   items: map<dentry_key, CDentry*>│
     │   fnode          │
     │   dir_auth       │
     └───────────────────┘

典型的路径查找流程：
1. 从根 CInode (ino=1) 开始
2. 获取其 CDir (frag=0)
3. 在 CDir.items 中查找 name="home" 的 CDentry
4. 获取 CDentry.linkage.inode (CInode)
5. 重复 2-4 直到目标
```

## 8. 状态机

### 8.1 权威状态迁移

```
权威迁移：

        [权威 MDS.A]                    [权威 MDS.B]
             │                               │
             │    EXPORT 导出请求            │
             │──────────────────────────────►│
             │                               │
             │    STATE_EXPORTING             │
             │    编码并传输数据              │
             │                               │
             │    IMPORT 导入完成            │
             │◄──────────────────────────────│
             │                               │
        [副本 MDS.A]                    [权威 MDS.B]

关键状态：
- STATE_EXPORTING: 正在导出
- STATE_IMPORTING: 正在导入
- STATE_AMBIGUOUSAUTH: 权限模糊（迁移过程中）
```

### 8.2 冻结状态

```
冻结操作：

正常状态 ─────────────► 冻结中 (STATE_FREEZING)
    │                        │
    │                        │ 等待所有 auth_pin 释放
    │                        │
    │                        ▼
    │                    已冻结 (STATE_FROZEN)
    │                        │
    │◄────────────────────────┘
    │      解冻
    ▼
正常状态

用途：
- 子树迁移
- MDS 故障恢复
- 目录分片/合并
```

## 9. 持久化

### 9.1 对象存储结构

```
RADOS 对象：

每个 CInode 存储：
- 对象名: <ino>.inode
- 内容: inode_t + xattrs + old_inodes (快照)

每个 CDir 存储：
- 对象名: <ino>.<frag>.dir
- 内容: fnode_t + 所有 dentry

写入流程：
1. 修改内存对象（CInode/CDir/CDentry）
2. 标记 STATE_DIRTY
3. 加入 LogSegment 的 dirty 列表
4. MDLog 写入操作日志
5. 异步 flush 到 RADOS
6. 清除 STATE_DIRTY
```

### 9.2 日志（Journal）

```c++
// src/mds/LogSegment.h
class LogSegment {
  list<CInode*> dirty_inodes;
  list<CDir*> dirty_dirs;
  list<CDentry*> dirty_dentries;
  // ...
};

// 所有修改先写日志
// 然后异步刷到 RADOS
```

## 10. 内存池（Memory Pool）

所有 MDS 缓存对象使用专用内存池：

```c++
// src/include/mempool.h
namespace mempool {
  namespace mds_co {  // MDS Cache Object pool
    // 分配器
    template<typename T>
    using pool_allocator = ...
    
    // 类型
    using string = std::basic_string<char, std::char_traits<char>, pool_allocator<char>>;
    template<typename K, typename V>
    using map = std::map<K, V, std::less<K>, pool_allocator<std::pair<const K, V>>>;
  }
}

// 使用
mempool::mds_co::string name;
mempool::mds_co::map<dentry_key_t, CDentry*> items;
```

好处：
1. 统一管理 MDS 内存使用
2. 方便统计和调试
3. 减少内存碎片

## 11. 关键代码位置

```
src/mds/
├── CInode.h         # CInode 定义
├── CInode.cc        # CInode 实现
├── CDentry.h        # CDentry 定义
├── CDentry.cc       # CDentry 实现
├── CDir.h           # CDir 定义
├── CDir.cc          # CDir 实现
├── MDSCacheObject.h # 基类
├── MDCache.h        # 缓存管理器
├── MDCache.cc       # 缓存管理实现
│
├── Locker.h         # 分布式锁管理
├── Locker.cc        # 锁状态机
├── Migrator.h       # 子树迁移
├── Migrator.cc      # 导入/导出实现
├── Server.h         # 客户端请求处理
├── Server.cc        # 元数据操作入口
│
├── Capability.h     # 客户端 capability
├── SimpleLock.h     # 简单锁
├── ScatterLock.h    # 散列锁
└── LogSegment.h     # 日志段
```

## 12. 相关组件依赖

```
依赖关系：

CInode/CDentry/CDir
       │
       ├── MDSCacheObject (基类: 引用计数、状态、等待)
       │
       ├── InodeStoreBase (持久化序列化)
       │
       ├── Locker (分布式锁)
       │      └── SimpleLock, ScatterLock
       │
       ├── MDCache (缓存管理)
       │      └── 对象查找: inode_map, dentry_map, dir_map
       │
       ├── Migrator (子树迁移)
       │      └── encode_export, decode_import
       │
       ├── Server (请求处理)
       │      └── handle_client_*
       │
       └── MDLog (日志)
              └── LogSegment (脏对象列表)
```

## 13. 参考资料

1. **Ceph 论文**: "Ceph: A Scalable, High-Performance Distributed File System" (OSDI 2006)
2. **源码注释**: `src/mds/CInode.h`, `src/mds/CDentry.h`, `src/mds/CDir.h`
3. **设计文档**: Ceph 源码 `doc/dev/` 目录