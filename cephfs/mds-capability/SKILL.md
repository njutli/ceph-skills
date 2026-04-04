---
name: mds-capability-system
description: Ceph MDS分布式Capability系统专家。当用户询问Capability位定义、cap授权/撤销/释放流程、分布式锁管理、MDS与客户端权限委托、cap状态机时调用此技能。
---

# MDS Capability 系统

## 〇、为什么需要 Capability 系统？

为什么需要 Capability？在分布式文件系统中，多个客户端同时访问同一文件，必须保证元数据一致性：

```
没有 Capability 的问题:
  客户端 A 缓存了 inode 属性 (uid, mode, size)
  客户端 B 修改了同一个 inode
  → 客户端 A 的缓存变为脏数据，但 A 不知道
  
传统解决方案:
  每次访问都向 MDS 查询 → 性能极差
  或者完全禁用客户端缓存 → 性能极差
```

Capability 系统的核心思想是：**MDS 将权限委托给客户端，客户端在权限范围内自由缓存，MDS 需要时收回权限**。

```
Capability 方案:
  MDS 授予客户端 FILE_CACHE cap → 客户端可以缓存文件数据
  MDS 授予客户端 FILE_WR cap   → 客户端可以本地缓冲写入
  当其他客户端需要访问时 → MDS 撤销冲突的 cap
  客户端收到 revoke → 刷盘并确认 → MDS 授权给其他客户端
```

---

## 一、Capability 位定义

### 1.1 基本权限原语 (src/include/ceph_fs.h:808-815)

```c
CEPH_CAP_PIN         = 1    // 仅 pin，无具体权限
CEPH_CAP_GSHARED     = 1    // 共享读取
CEPH_CAP_GEXCL       = 2    // 独占读写
CEPH_CAP_GCACHE      = 4    // (文件) 客户端可缓存读取
CEPH_CAP_GRD         = 8    // (文件) 客户端可读取
CEPH_CAP_GWR        = 16    // (文件) 客户端可写入
CEPH_CAP_GBUFFER    = 32    // (文件) 客户端可缓冲写入
CEPH_CAP_GWREXTEND  = 64    // (文件) 客户端可扩展 EOF
CEPH_CAP_GLAZYIO   = 128    // (文件) 客户端可延迟 I/O
```

### 1.2 锁域位移 (src/include/ceph_fs.h:821-824)

```c
CEPH_CAP_SAUTH      = 2    // AUTH 锁位移
CEPH_CAP_SLINK      = 4    // LINK 锁位移
CEPH_CAP_SXATTR     = 6    // XATTR 锁位移
CEPH_CAP_SFILE      = 8    // FILE 锁位移
```

### 1.3 四个锁域的 Capability

| 锁域 | SHARED | EXCL | 对应 inode 属性 |
|------|--------|------|----------------|
| **AUTH** | bits 2-3 | bits 2-3 | uid, gid, mode |
| **LINK** | bits 4-5 | bits 4-5 | nlink (硬链接数) |
| **XATTR** | bits 6-7 | bits 6-7 | 扩展属性 |
| **FILE** | bits 8-15 | bits 8-15 | size, mtime, layout |

### 1.4 FILE Capability 详解

```c
CEPH_CAP_FILE_SHARED   = (1 << 8)   // 共享读取
CEPH_CAP_FILE_EXCL     = (1 << 8)   // 独占读写
CEPH_CAP_FILE_CACHE    = (1 << 10)  // 缓存读取 (page cache)
CEPH_CAP_FILE_RD       = (1 << 11)  // 读取权限
CEPH_CAP_FILE_WR       = (1 << 12)  // 写入权限
CEPH_CAP_FILE_BUFFER   = (1 << 13)  // 缓冲写入 (异步写)
CEPH_CAP_FILE_WREXTEND = (1 << 14)  // 扩展文件大小
CEPH_CAP_FILE_LAZYIO   = (1 << 15)  // 延迟 I/O (不立即同步)
```

### 1.5 组合掩码 (src/include/ceph_fs.h:865-883)

```c
CEPH_CAP_ANY_SHARED  = AUTH_SHARED | LINK_SHARED | XATTR_SHARED | FILE_SHARED
CEPH_CAP_ANY_RD      = ANY_SHARED | FILE_RD | FILE_CACHE
CEPH_CAP_ANY_EXCL    = AUTH_EXCL | LINK_EXCL | XATTR_EXCL | FILE_EXCL
CEPH_CAP_ANY_FILE_WR = FILE_WR | FILE_BUFFER | FILE_EXCL
CEPH_CAP_ANY_WR      = ANY_EXCL | ANY_FILE_WR
CEPH_CAP_ANY         = ANY_RD | ANY_EXCL | ANY_FILE_WR | FILE_LAZYIO | PIN
```

---

## 二、Capability 操作类型

### 2.1 操作枚举 (src/include/ceph_fs.h:897-911)

| 操作码 | 方向 | 说明 |
|--------|------|------|
| `CEPH_CAP_OP_GRANT` | MDS→Client | 授予 cap |
| `CEPH_CAP_OP_REVOKE` | MDS→Client | 撤销 cap |
| `CEPH_CAP_OP_TRUNC` | MDS→Client | 截断通知 |
| `CEPH_CAP_OP_UPDATE` | Client→MDS | 更新 (caps/dirty/wanted) |
| `CEPH_CAP_OP_FLUSH` | Client→MDS | 刷写脏数据 |
| `CEPH_CAP_OP_FLUSH_ACK` | MDS→Client | 刷写确认 |
| `CEPH_CAP_OP_RELEASE` | Client→MDS | 释放 cap |
| `CEPH_CAP_OP_RENEW` | Client→MDS | 续期请求 |

---

## 三、Capability 与分布式锁的关系

### 3.1 Inode 锁类型 (src/mds/CInode.h:1122-1148)

```
versionlock     -- inode 版本 (LocalLockC)
authlock        -- uid/gid/mode (SimpleLock)      → AUTH caps
linklock        -- nlink (SimpleLock)             → LINK caps
dirfragtreelock -- 目录碎片树 (ScatterLock)
filelock        -- 文件数据/size/mtime (ScatterLock) → FILE caps
xattrlock       -- 扩展属性 (SimpleLock)          → XATTR caps
snaplock        -- 快照状态 (SimpleLock)
nestlock        -- 嵌套 rstat (ScatterLock)
flocklock       -- POSIX 文件锁 (SimpleLock)
policylock      -- 导出策略 (SimpleLock)
```

### 3.2 锁状态

| 状态 | 说明 | 允许的 cap |
|------|------|-----------|
| `LOCK_SYNC` | 共享状态，多 MDS 可持有副本 | SHARED caps |
| `LOCK_LOCK` | 锁定，无副本 | 无 |
| `LOCK_EXCL` | 独占（一个 "loner" 客户端） | EXCL caps (loner) |
| `LOCK_XSYN` | 独占合成 | EXCL caps |
| `LOCK_MIX` | 混合状态 | 部分 SHARED |

### 3.3 Eval-Issue 链

```
锁状态变化
    ↓
Locker::eval()           [src/mds/Locker.cc:1501-1559]
    ↓
计算每个客户端允许的 cap
    ↓
Locker::issue_caps()     [src/mds/Locker.cc:2573-2700]
    ↓
MClientCaps(GRANT/REVOKE) 发送到客户端
```

---

## 四、Capability 状态机

### 4.1 MDS 端 Capability 类 (src/mds/Capability.h:33-398)

```c
class Capability {
    // 核心状态
    unsigned _wanted;   // 客户端理想情况下想要的 cap
    unsigned _pending;  // 已发送但客户端未确认的 cap
    unsigned _issued;   // 客户端实际持有的 cap (= _pending & ~_revokes)
    
    // 撤销追踪
    std::list<revoke_info> _revokes;  // 进行中的撤销
    
    // 序列号
    uint64_t last_sent;   // 最后发送的序列号
    uint64_t last_issue;  // 最后授权的序列号
    uint64_t mseq;        // 迁移序列号 (cap 在 MDS 间迁移)
    
    // 状态标志
    unsigned state;       // STATE_NEW, STATE_IMPORTING, STATE_NEEDSNAPFLUSH, etc.
};
```

### 4.2 关键方法

```c
// 授予新 cap (记录撤销)
unsigned issue(unsigned c, bool revoke = false);

// 授予 cap (不记录撤销，用于新 cap)
unsigned issue_norevoke(unsigned c);

// 客户端确认收到
void confirm_receipt(uint64_t seq, unsigned caps);

// 正在撤销的 bits
unsigned revoking() { return _issued & ~_pending; }
```

---

## 五、核心流程

### 5.1 Cap 授予流程 (MDS → Client)

```
客户端打开文件 (open/read/write)
    ↓
Server::handle_client_open()
    ↓
Locker::issue_new_caps()     [src/mds/Locker.cc:2434-2493]
    1. 计算 mode 对应的 wanted cap
    2. 创建 Capability 对象
    3. 设置 wanted bits, 标记 STATE_NEW
    ↓
Locker::eval()               [src/mds/Locker.cc:1501-1559]
    1. 选择 "loner" 客户端 (如果有)
    2. 评估每个锁的状态
    3. 计算每个客户端允许的 cap
    ↓
Locker::issue_caps()         [src/mds/Locker.cc:2573-2700]
    1. 对每个客户端 cap:
       allowed = get_allowed_caps()  // 基于锁状态
       if (pending & ~allowed): 需要撤销
         cap->issue(allowed, true)   // 记录撤销
         op = CEPH_CAP_OP_REVOKE
       else:
         cap->issue(allowed, false)
         op = CEPH_CAP_OP_GRANT
    2. 发送 MClientCaps 消息
```

### 5.2 Cap 撤销流程 (MDS → Client)

```
MDS 检测到冲突 (另一个客户端需要访问)
    ↓
Locker::issue_caps() 计算 allowed caps
    ↓
cap->issue(allowed, true)
    → 记录到 _revokes 列表
    → _pending = allowed (新 cap 集)
    → _issued = _pending & ~_revokes (客户端实际持有的)
    ↓
发送 MClientCaps(CEPH_CAP_OP_REVOKE)
    ↓
等待客户端确认 (FLUSH + RELEASE)
    ↓
客户端确认后清理 _revokes
```

### 5.3 Cap 刷写流程 (Client → MDS)

```
客户端收到 REVOKE (FILE_BUFFER 被撤销)
    ↓
客户端刷写脏数据到 OSD
    ↓
发送 MClientCaps(CEPH_CAP_OP_FLUSH)
    ↓
MDS Locker::handle_client_caps()   [src/mds/Locker.cc:3370-3579]
    1. 检查是否是重复刷写
    2. 如果是 FLUSHSNAP: 处理快照元数据
    3. 否则: 调用 _do_cap_update()
    ↓
Locker::_do_cap_update()           [src/mds/Locker.cc:4067-4240]
    1. 处理 max_size 变化
    2. 投影 inode 新值
    3. 强制 wrlock 相关锁
    4. 提交日志
    ↓
发送 MClientCaps(CEPH_CAP_OP_FLUSH_ACK)
```

### 5.4 Cap 释放流程 (Client → MDS)

```
客户端不再需要 cap (文件关闭)
    ↓
客户端发送 MClientCaps(CEPH_CAP_OP_RELEASE)
    ↓
MDS Locker::_do_cap_release()      [src/mds/Locker.cc:4290-4327]
    1. 清理旧的撤销历史
    2. remove_client_cap()
       - 清理 snapflush 状态
       - 使能锁缓存失效
       - 从 inode 移除 cap
    ↓
Locker::try_eval()                 // 重新评估锁状态
    → 可能触发新的 cap 授予
```

---

## 六、内核客户端 Cap 管理

### 6.1 Cap 引用计数 (linux/fs/ceph/caps.c:3186-3334)

```c
// 获取 cap 引用 (开始操作时)
ceph_get_cap_refs(cap, refs);

// 释放 cap 引用 (操作完成时)
__ceph_put_cap_refs(cap, refs);
  → 当最后一个引用释放时:
    → ceph_check_caps()  // 检查是否可以释放 cap 给 MDS
```

### 6.2 内核 Cap 状态检查 (linux/fs/ceph/caps.c:2010-2400)

```c
ceph_check_caps(inode, flags, session)
    1. 计算 wanted, used, issued, revoking
    2. 计算 retain 集合 (需要保留的 cap)
    3. 如果撤销了 CACHE cap:
       → 尝试页缓存失效
    4. 遍历每个 cap:
       → 如果撤销完成 (used & revoking == 0)
         → 发送 FLUSH 或 RELEASE
```

### 6.3 内核处理 GRANT/REVOKE (linux/fs/ceph/caps.c:3485-3794)

```c
handle_cap_grant(inode, newcaps, ...)
    1. 如果 CACHE 被撤销:
       → 尝试页缓存失效
    2. 如果 FILE_BUFFER 被撤销且正在使用:
       → 排队写回
    3. 更新 cap->issued = newcaps
    4. 更新 cap->implemented |= newcaps
    5. 排队写回和/或失效到单独线程
```

---

## 七、关键数据结构

### 7.1 MClientCaps 消息 (src/messages/MClientCaps.h)

```c
class MClientCaps : public Message {
    ceph_mds_request_head head;
    inodeno_t ino;           // inode 号
    snapid_t snapid;         // 快照 ID
    uint64_t client_follows; // 客户端跟随的快照序列
    uint64_t client_migrating; // 迁移中的 inode
    
    // Cap 状态
    uint32_t caps;           // 当前持有的 cap
    uint32_t wanted;         // 想要的 cap
    uint32_t dirty;          // 脏的 cap
    uint64_t flush_tid;      // 刷写事务 ID
    
    // 元数据 (用于 FLUSH)
    ceph_mds_caps cap;
    file_layout_t layout;
    uint64_t size, max_size, truncate_size;
    uint32_t truncate_seq;
    utime_t mtime, atime, ctime;
    ...
};
```

---

## 八、关键代码位置

| 功能 | 文件 | 行号 |
|------|------|------|
| Cap 位定义 | src/include/ceph_fs.h | 803-892 |
| Cap 操作枚举 | src/include/ceph_fs.h | 897-911 |
| Capability 类 | src/mds/Capability.h | 33-398 |
| issue_new_caps | src/mds/Locker.cc | 2434-2493 |
| issue_caps | src/mds/Locker.cc | 2573-2700 |
| get_allowed_caps | src/mds/Locker.cc | 2522-2571 |
| eval | src/mds/Locker.cc | 1501-1559 |
| handle_client_caps | src/mds/Locker.cc | 3370-3579 |
| _do_cap_update | src/mds/Locker.cc | 4067-4240 |
| _do_cap_release | src/mds/Locker.cc | 4290-4327 |
| MClientCaps 消息 | src/messages/MClientCaps.h | 1-383 |
| 内核 ceph_check_caps | linux/fs/ceph/caps.c | 2010-2400 |
| 内核 handle_cap_grant | linux/fs/ceph/caps.c | 3485-3794 |
| 内核 cap 引用计数 | linux/fs/ceph/caps.c | 3186-3334 |

---

## 九、常见问题与陷阱

### Q1: 为什么 cap 需要序列号 (seq, mseq)？

**A**: 防止消息乱序导致的竞态条件：
- `seq`: 跟踪 cap 授予顺序，确保客户端不会用旧的 cap 状态覆盖新的
- `mseq`: 跟踪 cap 在 MDS 间迁移的顺序，确保迁移后 cap 状态正确

### Q2: `_issued` 和 `_pending` 的区别？

**A**:
- `_pending`: MDS 已发送给客户端的 cap 集合
- `_issued`: 客户端实际持有的 cap 集合 (`= _pending & ~_revokes`)
- 区别在于 `_revokes` 中记录了 MDS 要求撤销但客户端尚未确认的 bits

### Q3: "Loner" 机制是什么？

**A**: 当只有一个客户端活跃访问某个 inode 时，MDS 将其标记为 "loner"，允许它持有 EXCL cap。这避免了每次操作都向 MDS 请求，大幅提升单客户端性能。当其他客户端需要访问时，MDS 撤销 loner 的 EXCL cap，降级为 SYNC 状态。
