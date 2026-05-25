# CephFS 挂载、文件打开与写入过程深度分析

> 基于 dmesg_full.log 与 Linux 内核源码 `/home/i_ingfeng/linux` 的联合分析

---

## 目录

1. [总览](#1-总览)
2. [第一阶段：挂载过程](#2-第一阶段挂载过程)
   - [2.1 参数解析](#21-挂载参数解析)
   - [2.2 创建文件系统客户端](#22-创建文件系统客户端)
   - [2.3 建立 MON 会话](#23-建立-mon-会话)
   - [2.4 获取 MDS Map](#24-获取-mds-map)
   - [2.5 打开 MDS 会话](#25-打开-mds-会话)
   - [2.6 打开根 inode](#26-打开根-inode)
   - [2.7 挂载完成](#27-挂载完成)
3. [第二阶段：文件创建与打开 (atomic_open)](#3-第二阶段文件创建与打开-atomic_open)
   - [3.1 getattr 获取目录属性](#31-getattr-获取目录属性)
   - [3.2 ACL 与扩展属性检查](#32-acl-与扩展属性检查)
   - [3.3 atomic_open 创建文件](#33-atomic_open-创建文件)
   - [3.4 创建请求提交到 MDS](#34-创建请求提交到-mds)
   - [3.5 处理 MDS 回复并填充 inode](#35-处理-mds-回复并填充-inode)
   - [3.6 打开文件与 cap 校验](#36-打开文件与-cap-校验)
4. [第三阶段：文件写入过程](#4-第三阶段文件写入过程)
   - [4.1 getattr 获取新文件属性](#41-getattr-获取新文件属性)
   - [4.2 安全属性检查](#42-安全属性检查)
   - [4.3 aio_write 启动写操作](#43-aio_write-启动写操作)
   - [4.4 Pool 权限检查](#44-pool-权限检查)
   - [4.5 Cap 获取 (Fwb)](#45-cap-获取-fwb)
   - [4.6 write_end 页面写入](#46-write_end-页面写入)
   - [4.7 标记 dirty 与 cap 延迟刷新](#47-标记-dirty-与-cap-延迟刷新)
   - [4.8 写完成与 cap 引用释放](#48-写完成与-cap-引用释放)
5. [第四阶段：flush 与卸载过程](#5-第四阶段flush-与卸载过程)
   - [5.1 预卸载处理](#51-预卸载处理)
   - [5.2 writepages 脏页回写](#52-writepages-脏页回写)
   - [5.3 Cap Flush 与 Flush ACK](#53-cap-flush-与-flush-ack)
   - [5.4 最终 sync 与销毁](#54-最终-sync-与销毁)
6. [附录：Cap 能力位含义](#附录cap-能力位含义)
7. [附录：关键数据结构](#附录关键数据结构)

---

## 1. 总览

本次操作经过三个阶段：
- **挂载**: 挂载 CephFS 到本地（设备 `127.0.0.1:40503:/`，MDS `127.0.0.1:6828`）
- **文件操作**: 使用 `atomic_open` 创建并打开文件 `test_trace.txt`，然后通过 `aio_write` 写入 28 字节
- **卸载**: 卸载文件系统，在此过程中触发脏页回写与 cap flush

整个过程中 Ceph 客户端使用以下内部打印函数输出调试信息：
- `doutc(cl, ...)` — 定义于 `include/linux/ceph/ceph_debug.h:22-27`，带客户端上下文 `[fsid global_id]`
- `dout(...)` — 定义于 `include/linux/ceph/ceph_debug.h:18-21`，通用调试输出

日志中的 `[22224.xxxxxx]` 前缀是内核时间戳（`printk` 默认时间）。

---

## 2. 第一阶段：挂载过程

### 2.1 挂载参数解析

```
日志第1-8行:
[22224.427468] ceph:  ceph_parse_mount_param fs_parse 'source' token 12
[22224.427472] ceph:  ceph_parse_source '127.0.0.1:40503:/'
[22224.427473] ceph:  device name '127.0.0.1:40503'
[22224.427474] ceph:  server path '/'
[22224.427474] ceph:  trying new device syntax
[22224.427474] ceph:  separator '=' missing in source
[22224.427475] ceph:  trying old device syntax
[22224.427478] ceph:  ceph_parse_mount_param fs_parse 'mds_namespace' token 10
```

**代码调用栈：**

```
VFS mount
  └── ceph_fs_type.mount() / .init_fs_context()
        └── ceph_init_fs_context()                     [super.c:1435]
              └── ceph_context_ops.parse_param()
                    └── ceph_parse_mount_param()       [super.c:404]
                          ├── fs_parse()               解析 token 'source' (token 12)
                          ├── ceph_parse_source()      解析设备名和路径
                          │     ├── 提取 "127.0.0.1:40503" 为设备名
                          │     ├── 提取 "/" 为服务器路径
                          │     ├── 尝试新语法 (key=value)，该例中无 '='
                          │     └── 尝试旧语法 (device:path)，解析成功
                          └── fs_parse()               解析 token 'mds_namespace' (token 10)
```

**源码分析：**

`ceph_parse_mount_param()` ([super.c:404](https://)) 使用内核 `fs_parse()` 机制解析挂载参数。`ceph_mount_parameters` 数组定义于 [super.c:193](https://)，包含所有支持的参数。

`ceph_parse_source()` 支持两种语法：
- **新语法**: `key=value` 格式（如 `source=addr:port:/path`）
- **旧语法**: `addr:port:/path` 格式（直接解析 `:` 分隔符）

本日志中使用的是旧语法格式。

---

### 2.2 创建文件系统客户端

```
日志第9-11行:
[22224.427483] ceph:  ceph_get_tree
[22224.427898] ceph:  set_super 0000000074c64218
[22224.427909] ceph:  get_sb using new client 00000000fbf4a880
```

**代码调用栈：**

```
ceph_get_tree()                                    [super.c:1299]
  ├── create_fs_client()                           [super.c:807]
  │     ├── 分配 struct ceph_fs_client
  │     ├── ceph_options_new()                     解析选项
  │     ├── ceph_create_client()                   创建 libceph 客户端
  │     └── ceph_mdsc_init()                       初始化 MDS 客户端
  ├── sget_fc()                                    查找/分配超级块
  │     ├── ceph_set_super()                       初始化 super_block [super.c:1204]
  │     └── ceph_compare_super()                   检查是否可复用已有超级块 [super.c:1239]
  └── ceph_real_mount()                            [super.c:1148]
```

**关键数据结构：**
- `struct ceph_fs_client *fsc` = `00000000fbf4a880` — Ceph 文件系统客户端实例
- `struct super_block *s` = `0000000074c64218` — VFS 超级块，通过 `sget_fc()` 分配

`ceph_set_super()` ([super.c:1204](https://)) 设置超级块的操作函数表（`sop`、`export_ops`、最大文件大小等）。

---

### 2.3 建立 MON 会话

```
日志第12-15行:
[22224.427992] ceph:  mount start 00000000fbf4a880
[22224.429310] libceph: mon0 (2)127.0.0.1:40503 session established
[22224.430178] ceph:  handle_fsmap epoch 10
[22224.430242] libceph: client4617 fsid e20c1068-04d5-4745-a133-9853db2f5e8a
```

**代码调用栈：**

```
ceph_real_mount()                                  [super.c:1148]
  └── ceph_open_session()                          [net/ceph/ceph_common.c:843]
        └── __ceph_open_session()                  [net/ceph/ceph_common.c:790]
              ├── ceph_monc_open_session()          连接到 MON 127.0.0.1:40503
              ├── 等待 monmap（MON 集群拓扑）
              ├── 等待 osdmap（OSD 集群拓扑）
              ├── 认证（cephx 协议）
              └── ceph_fs_debugfs_init()            初始化调试文件系统 [debugfs.c]
```

**关键日志解读：**
- `mon0 (2)127.0.0.1:40503 session established` — MON 会话成功建立。`(2)` 表示 mon 的 rank 编号。
- `handle_fsmap epoch 10` — 收到文件系统 map epoch 10，包含当前 MDS 的拓扑信息。
- `client4617` — libceph 分配的客户端全局 ID。
- `fsid e20c1068-...` — Ceph 集群的 FSID（文件系统唯一标识）。

---

### 2.4 获取 MDS Map

```
日志第26-31行:
[22224.430554] ceph:  handle_map epoch 9 len 1210
[22224.430559] ceph:  mdsmap_decode 1/1 4230 mds0.8 (2)127.0.0.1:6828 up:active
[22224.430560] ceph:  mdsmap_decode m_enabled: 1, m_damaged: 0, m_num_laggy: 0
[22224.430561] ceph:  mdsmap_decode success epoch 9
[22224.430561] ceph:  check_new_map new 9 old 0
[22224.430563] ceph:   wake request 00000000633789f0 tid 1
```

**MDS Map 解析：**

`ceph_mdsmap_decode()` ([mdsmap.c:118](https://)) 解析 MDS map 消息：

| 字段 | 值 | 含义 |
|------|-----|------|
| `m_max_mds` | 1 | 最大活跃 MDS 数为 1 |
| `m_num_active_mds` | 1 | 当前活跃 MDS 数为 1 |
| `mdsmap_epoch` | 9 | MDS map epoch 版本 |
| `mds0.8` | rank 0, incarnation 8 | MDS 实例标识 |
| `(2)127.0.0.1:6828` | 地址族 2(AF_INET), IP:端口 | MDS 网络地址 |
| `up:active` | 状态 | MDS 已上线且活跃 |
| `m_enabled` | 1 | MDS 已启用（非 degraded 模式） |
| `m_damaged` | 0 | MDS 无损坏 |
| `m_num_laggy` | 0 | 无滞后 MDS |

MDS map epoch 从 0 更新到 9 后，之前因缺少 map 而**等待的请求被唤醒** (`wake request tid 1`)。

---

### 2.5 打开 MDS 会话

```
日志第32-45行:
[22224.430564] ceph:  __choose_mds 0000000000000000 is_hash=0 (0x0) mode 0
[22224.430565] ceph:  __choose_mds chose random mds0
[22224.430567] ceph:  register_session: realloc to 1
[22224.430567] ceph:  register_session: mds0
[22224.430568] ceph:  do_request mds0 session 000000009680d5d3 state new
[22224.430569] ceph:  open_session to mds0 (up:active)
```

**代码调用栈：**

```
__do_request()                                     [mds_client.c:3486]
  ├── __choose_mds()                               选择目标 MDS
  │     ├── is_hash=0 (0x0), mode=0 → 根 inode 无哈希
  │     └── 随机选择 mds0 为根 inode 的 MDS
  │
  ├── register_session()                           注册/复用 MDS 会话
  │     └── struct ceph_mds_session *session = 000000009680d5d3
  │
  └── __open_session()                             打开 MDS 会话 [mds_client.c:1707]
```

**MDS 选择策略（`__choose_mds()` [mds_client.c:?]）：**
- 根 inode `0x0` 没有哈希值（`is_hash=0`），选择模式为 `USE_ANY_MDS`（`mode=0`）
- 在活跃 MDS 列表中**随机选择**一个 MDS（此处仅 mds0 活跃）
- 如果已有 auth cap，则选择 cap 所属的 MDS

```
日志第38-44行:
[22224.435183] ceph:  handle_session mds0 open 000000009680d5d3 state opening seq 0
[22224.435187] ceph:  renewed_caps mds0 ttl now 4300121668, was fresh, now stale
[22224.435190] ceph:  do_request mds0 session 000000009680d5d3 state open
[22224.435191] ceph:  __prepare_send_request 00000000633789f0 tid 1 getattr (attempt 1)
```

**MDS 会话状态转换：**

```
state: new → opening → open
```

当 MDS 返回 session open ACK 后：
- `handle_session` 处理会话建立确认
- `renewed_caps` 标记所有已有 cap 为 stale（因为之前无 cap）
- 请求被重新唤醒，`__choose_mds` 使用 `resend_mds mds0` 保证重试到同一 MDS
- `__prepare_send_request` 准备 `CEPH_MDS_OP_GETATTR` 请求

---

### 2.6 打开根 inode

```
日志第46-73行:
[22224.435723] ceph:  handle_reply 00000000633789f0
[22224.435726] ceph:  __unregister_request 00000000633789f0 tid 1
[22224.435727] ceph:  handle_reply tid 1 result 0
...
[22224.435802] ceph:  open_root_inode success
[22224.435830] ceph:  open_root_inode success, root dentry is 000000006e08304f
[22224.435832] ceph:  mount success
[22224.435833] ceph:  root 000000006e08304f inode 000000006b5fd166 ino 1.fffffffffffffffe
```

**代码调用栈：**

```
open_root_dentry()                                 [super.c:1055]
  └── ceph_mdsc_do_request()                       发送 GETATTR 请求获取根 inode
        │
        ├── __do_request()                          [mds_client.c:3486]
        │     ├── __prepare_send_request()          [mds_client.c:3349]
        │     │     └── 请求类型: getattr (CEPH_MDS_OP_GETATTR)
        │     │     └── 路径: "" (空路径 = 根目录)
        │     │
        │     └── 等待 MDS 回复...
        │
        ├── handle_reply()                          [mds_client.c:3898]
        │     ├── __unregister_request()            [mds_client.c:?] 注销 tid=1
        │     ├── handle_reply tid 1 result 0      (result=0 表示成功)
        │     ├── alloc_inode()                     分配 VFS inode [super.c:?]
        │     ├── ceph_get_inode()                  查找/创建 ceph inode
        │     │     └── ino 1.fffffffffffffffe → inode 000000006b5fd166
        │     ├── ceph_update_snap_trace()          更新快照追踪
        │     ├── ceph_create_snap_realm()          创建快照域
        │     ├── build_snap_context()              构建快照上下文 seq=1 (0 snaps)
        │     ├── fill_trace()                      填充 trace 信息 [inode.c:?]
        │     └── ceph_fill_inode()                 填充 inode 属性 [inode.c:?]
        │           ├── mode 040755 (目录)
        │           ├── uid.gid 1000.1000
        │           ├── truncate_size → 18446744073709551615 (ULLONG_MAX, 无截断)
        │           └── ceph_add_cap(): cap 8 pAsLsXs seq 1
        │
        └── ceph_unreserve_caps()                   释放预留的 cap
```

**根 inode 属性总结：**

| 属性 | 值 | 含义 |
|------|-----|------|
| ino | `1.fffffffffffffffe` | Ceph inode 编号 (ino=1, snap_id=CEPH_NOSNAP) |
| mode | `040755` | 目录，rwxr-xr-x |
| uid.gid | `1000.1000` | 所有者和组 |
| version | `v 18` | inode 版本号 |
| initia cap | `pAsLsXs` seq 1 | 能力位见附录 |

根 dentry 为 `000000006e08304f`，根 inode 为 `000000006b5fd166`。

---

### 2.7 挂载完成

```
日志第75-77行:
[22224.435832] ceph:  mount success
[22224.435833] ceph:  root 000000006e08304f inode 000000006b5fd166 ino 1.fffffffffffffffe
[22224.435882] ceph:  destroy_mount_options 0000000000000000
```

**代码调用栈：**

```
ceph_get_tree() 返回
  │
  ├── ceph_real_mount()                             成功返回 root dentry
  │     └── d_set_d_op(root, &ceph_dentry_ops)      设置 dentry 操作
  │
  └── destroy_mount_options()                       释放临时挂载选项（已被移入 fs_client）
```

此时 VFS 层完成挂载，文件系统可被用户空间访问。

---

## 3. 第二阶段：文件创建与打开 (atomic_open)

### 3.1 getattr 获取目录属性

```
日志第78-80行:
[22224.455169] ceph:  do_getattr inode 000000006b5fd166 mask As mode 040755
[22224.455173] ceph:  __ceph_caps_issued_mask ino 0x1 cap 0000000091fa6ff9 issued pAsLsXs (mask As)
[22224.455174] ceph:  __touch_cap 000000006b5fd166 cap 0000000091fa6ff9 mds0
```

**代码调用栈：**

```
VFS stat / open
  └── ceph_getattr()                                [inode.c:?]
        ├── mask = As (STATX_MODE)
        ├── __ceph_caps_issued_mask()               检查 cap 是否满足 As 需求 [caps.c:?]
        │     └── 已持有 pAsLsXs → 满足 As（A=Auth, s=shared）
        ├── __touch_cap()                           更新 cap touch 时间 [caps.c:869]
        └── 本地满足，直接返回（无需 MDS 请求）
```

当 cap 已足够时（`__ceph_caps_issued_mask` 返回非零），`do_getattr` 直接使用本地缓存的属性，无需与 MDS 通信。Cap 检查使用 `__touch_cap()` 更新时间戳以保持 cap 活跃。

---

### 3.2 ACL 与扩展属性检查

```
日志第81-90行:
[22224.455176] ceph:  getxattr 000000006b5fd166 name 'system.posix_acl_access' ver=1 index_ver=0
[22224.455198] ceph:  __build_xattrs() len=0
[22224.455199] ceph:  __get_xattr system.posix_acl_access: not found
[22224.455199] ceph:  __ceph_destroy_xattrs p=0000000000000000
[22224.455216] ceph:  atomic_open 000000006b5fd166 dentry 000000001fc8d320 'test_trace.txt' unhashed flags 33345 mode 0100666
```

**代码调用栈：**

```
ceph_atomic_open()                                 [file.c:793]
  │
  ├── ceph_do_getattr(dir, CEPH_STAT_CAP_INODE_ALL, ...)  → 日志第78行 getattr
  │
  ├── ceph_getxattr(dir, "system.posix_acl_access")         ← ACL 检查
  │     └── __build_xattrs()                                [xattr.c:?]
  │           └── len=0 → 无扩展属性
  │     └── __get_xattr → "not found"
  │     └── __ceph_destroy_xattrs → 释放 xattrs 缓冲区
  │
  └── ceph_getxattr(dir, "system.posix_acl_default")        ← 默认 ACL 检查 (日志93行)
        └── 同样: not found
```

**atomic_open 参数：**
- `dentry 000000001fc8d320 'test_trace.txt'` — 目标文件
- `unhashed` — dentry 尚未哈希（新创建或不在 dcache 中）
- `flags 33345` = `0x8241` = `O_CREAT|O_WRONLY|O_TRUNC|O_LARGEFILE`（打开标志）
- `mode 0100666` — 文件权限 0666（受 umask 影响后）

同时（日志第93-97行）检查 `system.posix_acl_default` 以决定是否继承默认 ACL。

---

### 3.3 atomic_open 创建文件

```
日志第99-116行:
[22224.455233] ceph:  unused open flags: 8000
[22224.455234] ceph:  do_request on 00000000a170d66e
...
[22224.455272] ceph:  encode_inode_release 000000006b5fd166 mds0 used|dirty p drop AxXxFs unless Fx
[22224.455273] ceph:  encode_inode_release 000000006b5fd166 cap 0000000091fa6ff9 pAsLsXs (noop)
```

**代码调用栈：**

```
ceph_atomic_open()                                 [file.c:793]
  │
  ├── ceph_mdsc_do_request()                       发送 CREATE 请求
  │     │
  │     ├── __register_request(tid=2)               [mds_client.c:1209]
  │     │
  │     ├── __choose_mds()                          [mds_client.c:?]
  │     │     ├── is_hash=1 (0xf6647c0e)           test_trace.txt 的哈希
  │     │     ├── choose_frag(f6647c0e) = 0        选择 dirfrag 0
  │     │     └── 使用 auth cap 对应的 mds0
  │     │
  │     └── __prepare_send_request()               [mds_client.c:3349]
  │           ├── 操作类型: create (CEPH_MDS_OP_CREATE)
  │           ├── dentry: 1/test_trace.txt
  │           └── encode_inode_release()
  │                 ├── 父目录 used|dirty p → 释放 p (pin) cap
  │                 ├── drop AxXxFs unless Fx     需要 AxXxFs，除非已持有 Fx
  │                 └── 当前持有 pAsLsXs → noop   (无需修改，因为已满足)
```

**unused open flags: 8000** — 日志提示 `O_TMPFILE` (0x400000) 相关的标志未在此 `atomic_open` 中使用。内核上层传下的标志中包含不被 Ceph 识别的位但无影响。

---

### 3.4 处理 MDS 回复并填充 inode

```
日志第117-157行:
[22224.459295] ceph:  handle_reply 00000000a170d66e
[22224.459300] ceph:  handle_reply tid 2 result 0
[22224.459328] ceph:  get_inode on 1099511627777=10000000001.fffffffffffffffe got 0000000096c34c41 new 1
[22224.459354] ceph:  0000000096c34c41 mode 0100644 uid.gid 0.0
[22224.459356] ceph:  size 0 -> 0
[22224.459357] ceph:  ceph_fill_file_size truncate_seq 0 -> 1
[22224.459357] ceph:  ceph_fill_file_size truncate_size 0 -> 18446744073709551615, encrypted 0
[22224.459357] ceph:  max_size 0 -> 4194304
```

**代码调用栈：**

```
handle_reply(tid=2)                                [mds_client.c:3898]
  │
  ├── result 0 (成功)
  ├── fill_trace()                                  解析 MDS 回复中的 trace
  │     │
  │     ├── ceph_fill_inode(父目录)                 更新父目录 inode
  │     │     ├── ino 1.fffffffffffffffe v 20 had 18 (版本 18→20)
  │     │     └── 维持 cap pAsLsXs seq 2
  │     │
  │     └── ceph_fill_inode(test_trace.txt)         填充新文件的 inode
  │           ├── ino 10000000001.fffffffffffffffe   新 inode 编号
  │           ├── mode 0100644                       普通文件, rw-r--r--
  │           ├── uid.gid 0.0                        创建者 root
  │           ├── size 0 → 0                         初始大小为 0
  │           ├── truncate_seq 0 → 1                 截断序列号
  │           ├── truncate_size → ULLONG_MAX          无截断限制
  │           ├── max_size 0 → 4194304               最大文件大小 4MB (wsize 的默认值)
  │           └── ceph_add_cap(): cap 1 pAsxLsXsxFsxcrwb seq 1
  │
  ├── dn attached: dentry 000000001fc8d320 → inode 0000000096c34c41
  ├── update_dentry_lease(duration=30000ms)          31秒 dentry 租约
  ├── dentry_lease_touch                            更新租约时间戳
  └── fill_trace done err=0
```

**新文件 cap 分析：**

新建文件获得完整的 **`pAsxLsXsxFsxcrwb`** cap 组合，各 bit 含义：

| Bit | 含义 |
|-----|------|
| p | Pin (固定) |
| As | 授权共享 |
| x | 排他标志 |
| Ls | 链接共享 |
| Xs | 扩展属性共享 |
| Fs | 文件布局共享 |
| x | (第二个) 排他标志 |
| c | 缓存 |
| r | 读取 |
| w | 写入 |
| b | 缓冲 |

但实际生效的只有 **pAsxLsXsxFsxcrwb**，其中 `Ax` 双写是因为 MDS 给出了非空的 "wanted" 集合。日志 `147` 行显示 "queueing" — 表示 MDS 想要的 cap 与当前 issued 的 cap 不匹配，需要延迟处理。

---

### 3.5 打开文件与 cap 校验

```
日志第158-167行:
[22224.459404] ceph:  atomic_open finish_open on dn 0000000000000000
[22224.459409] ceph:  open inode 0000000096c34c41 ino 10000000001.fffffffffffffffe file 00000000cd7a597b flags 33281 (33345)
[22224.459411] ceph:  open 0000000096c34c41 fmode 2 want pAsxXsxFxwb issued pAsxLsXsxFsxcrwb using existing
[22224.459412] ceph:  ceph_init_file_info 0000000096c34c41 00000000cd7a597b 0100644 (regular)
[22224.459419] ceph:  atomic_open result=0
```

**代码调用栈：**

```
ceph_atomic_open()                                 [file.c:793]
  │
  ├── finish_open()                                 调用 VFS 的 finish_open
  │
  ├── ceph_open()                                   [file.c:378]
  │     ├── flags 33281 = O_WRONLY|O_LARGEFILE     (过滤后的标志)
  │     ├── fmode 2 = FMODE_WRITE | FMODE_PWRITE    打开模式 → want pAsxXsxFxwb
  │     ├── __ceph_caps_issued()                    持有 pAsxLsXsxFsxcrwb
  │     ├── ceph_cap_string(want) = "pAsxXsxFxwb"
  │     ├── 已持有足够 cap，使用 "using existing"   无需额外 cap 请求
  │     └── ceph_init_file_info()                   [file.c:219]
  │           ├── 分配 struct ceph_file_info
  │           ├── fmode = 2 (WRITE)
  │           ├── flags = 3841 (O_WRONLY|O_LARGEFILE)
  │           └── 初始化 rw_contexts 列表 (用于恢复写)
  │
  ├── ceph_put_cap_refs("p")                        释放 pin cap 引用
  └── atomic_open result=0                          成功
```

---

## 4. 第三阶段：文件写入过程

### 4.1 getattr 获取新文件属性

```
日志第168-170行:
[22224.459420] ceph:  do_getattr inode 0000000096c34c41 mask As mode 0100644
[22224.459421] ceph:  __ceph_caps_issued_mask ino 0x10000000001 cap 000000004562469c issued pAsxLsXsxFsxcrwb (mask As)
[22224.459422] ceph:  __touch_cap 0000000096c34c41 cap 000000004562469c mds0
```

VFS 内核在 write 之前先调用 `ceph_getattr` 检查文件属性。由于已持有包含 `As` 的 cap，直接从本地返回，无需 MDS 请求。

---

### 4.2 安全属性检查

```
日志第171-176行:
[22224.459434] ceph:  getxattr 0000000096c34c41 name 'security.capability' ver=1 index_ver=0
[22224.459437] ceph:  __get_xattr security.capability: not found
```

内核 `security_file_permission()` 或类似函数触发了对文件 capabilities 的检查。文件无 `security.capability` 扩展属性（无额外的 file capabilities 设置）。

---

### 4.3 aio_write 启动写操作

```
日志第177行:
[22224.459438] ceph:  aio_write 0000000096c34c41 10000000001.fffffffffffffffe 0~28 getting caps. i_size 0
```

**代码调用栈：**

```
VFS write() / pwrite()
  └── ceph_write_iter()                             [file.c:2387]
        ├── 参数: offset=0, length=28, i_size=0
        ├── 检查只读/快照
        ├── 检查是否 O_DIRECT
        └── aio_write 路径（非 direct I/O）
              └── 输出日志: "0~28 getting caps"
```

`ceph_write_iter()` ([file.c:2387](https://)) 是 CephFS 的 `.write_iter` 实现。它首先获取写入所需的能力，然后根据写入模式（缓冲/直接/同步）路由到不同路径。

---

### 4.4 Pool 权限检查

```
日志第178-179行:
[22224.459439] ceph:  __ceph_pool_perm_get pool 5 no perm cached
[22224.463211] ceph:  __ceph_pool_perm_get pool 5 result = 3
```

**代码调用栈：**

```
ceph_write_iter()
  └── ceph_pool_perm_check()                        检查 pool 写入权限
        └── __ceph_pool_perm_get()
              ├── pool 5 → 检查缓存中是否有权限记录 → "no perm cached"
              ├── 向 MON 查询 pool 权限
              └── result = 3 (有写权限)
```

Pool 权限检查确保客户端对目标 pool 有写入权限。`result = 3` 表示权限检查通过（非零 = 有权限）。

---

### 4.5 Cap 获取 (Fwb)

```
日志第180-185行:
[22224.463215] ceph:  get_cap_refs 0000000096c34c41 need Fw want Fb
[22224.463216] ceph:  __ceph_caps_issued 0000000096c34c41 cap 000000004562469c issued pAsxLsXsxFsxcrwb
[22224.463217] ceph:  get_cap_refs 0000000096c34c41 have pAsxLsXsxFsxcrwb but not Fb (revoking -)
[22224.463219] ceph:  ceph_take_cap_refs 0000000096c34c41 wb 0 -> 1 (?)
[22224.463220] ceph:  get_cap_refs 0000000096c34c41 ret 1 got Fwb
[22224.463222] ceph:  aio_write 0000000096c34c41 10000000001.fffffffffffffffe 0~28 got cap refs on Fwb
```

**代码调用栈：**

```
ceph_write_iter()
  └── ceph_get_caps()                               [caps.c:3173]
        └── __ceph_get_caps()                       [caps.c:3040]
              ├── need=Fw, want=Fb                  需要 Fw (写权限), 想要 Fb (缓冲写)
              ├── __ceph_caps_issued() = pAsxLsXsxFsxcrwb   当前持有
              ├── 检查 Fw: 已持有 (crwb 包含 w, 但分开检查)
              │     检查 Fb: 未持有 (issued 中无 b)
              │     → "have pAsxLsXsxFsxcrwb but not Fb"
              │
              └── ceph_take_cap_refs()              尝试获取 Fwb 引用计数
                    ├── wb refs 0 → 1               增加 write+buffer 引用
                    ├── ret = 1 (成功)
                    └── got Fwb                      获取到 Fwb 能力
```

**Cap 获取机制详解：**

`ceph_get_caps()` / `__ceph_get_caps()` ([caps.c:3040-3171](https://)) 是 Ceph 能力管理的核心：

1. **need vs want**:
   - `need=Fw`: 必须持有的 cap（文件写入）
   - `want=Fb`: 希望持有的 cap（缓冲写入，允许页面缓存）

2. **cap 检查逻辑**:
   - 所有 cap 检查必须持有 `i_ceph_lock`
   - `__ceph_caps_issued()` 返回 inode 当前 issued 的 cap 集合
   - 如果 `need` 不满足 → 需要向 MDS 请求更多 cap
   - 如果 `want` 不满足 → 使用 need-only 模式

3. **引用计数**: 每个 cap 类别有独立的引用计数。`ceph_take_cap_refs()` 原子地增加引用计数。

---

### 4.6 write_end 页面写入

```
日志第186-188行:
[22224.463228] ceph:  write_end file 00000000cd7a597b inode 0000000096c34c41 folio 000000000828fb14 0~28 (28)
[22224.463230] ceph:  set_size 0000000096c34c41 0 -> 28
[22224.463247] ceph:  0000000096c34c41 dirty_folio 000000000828fb14 idx 0 head 0/0 -> 1/1 snapc 00000000d8b781fd seq 1 (0 snaps)
```

**代码调用栈：**

```
ceph_write_iter()
  └── generic_perform_write() / pagecache_write()
        ├── 分配 / 查找 folio (页面缓存页)
        │     └── folio 000000000828fb14 (第 0 页)
        │
        ├── iov_iter_copy_from_user_atomic()         从用户空间复制 28 字节到 folio
        │
        └── ceph_write_end()                         [addr.c:1901]
              │
              ├── set_page_dirty() → ceph_dirty_folio()  [addr.c:82]
              │     ├── 设置 folio dirty 标志
              │     ├── 更新 dirty_folio 计数: head 0/0 → 1/1
              │     └── 关联 snapc (快照上下文)
              │           └── snapc seq 1 (0 snaps) → 没有活跃快照
              │
              └── set_size 0 → 28
```

**`ceph_write_end()` ([addr.c:1901-1937](https://))** 是标准 VFS write_end 回调：
- 标记 folio 为脏
- 更新 inode 的 i_size
- 返回已写入的字节数

---

### 4.7 标记 dirty 与 cap 延迟刷新

```
日志第189-193行:
[22224.463250] ceph:  __mark_dirty_caps 0000000096c34c41 Fw dirty - -> Fw
[22224.463251] ceph:   inode 0000000096c34c41 now dirty snapc 00000000d8b781fd auth cap 000000004562469c
[22224.463252] ceph:  __cap_delay_requeue 0000000096c34c41 flags 0x38 at 4300121693
[22224.463253] ceph:  __cap_set_timeouts 0000000096c34c41 14997
```

**代码调用栈：**

```
ceph_dirty_folio()                                 [addr.c:82]
  └── __mark_dirty_caps()                           [caps.c:?]
        ├── 标记 Fw (File Write) 为 dirty
        │     └── dirty caps: - → Fw
        │
        ├── inode now dirty                         标记 inode 脏状态
        │     └── snapc = 00000000d8b781fd seq 1
        │
        ├── __cap_delay_requeue()                   [caps.c:518]
        │     ├── flags 0x38 = CHECK_CAPS_NODELAY | FLUSH | SQUEUE
        │     └── 设置 cap 刷新延迟定时器
        │
        └── __cap_set_timeouts()                    设置 14997 jiffies (~15s) 超时
```

**Cap Dirty 状态管理：**
- `dirty` 跟踪哪些 cap 位有未通知 MDS 的更改
- `__cap_delay_requeue()` 将 inode 排入延迟 cap check 队列
- 超时时间通常为 `mdsmap->m_session_timeout / 2` (约 15s)
- 到期后 `ceph_check_delayed_caps()` 会处理

---

### 4.8 写完成与 cap 引用释放

```
日志第193-197行:
[22224.463254] ceph:  aio_write 0000000096c34c41 10000000001.fffffffffffffffe 0~28  dropping cap refs on Fwb
[22224.463255] ceph:  put_cap_refs 0000000096c34c41 wb 1 -> 0 (?)
[22224.463255] ceph:  put_cap_refs 0000000096c34c41 had Fwb last put
...
[22224.463259] ceph:   mds0 cap 000000004562469c used Fcb issued pAsxLsXsxFsxcrwb implemented pAsxLsXsxFsxcrwb revoking -
```

**代码调用栈：**

```
ceph_write_iter() → 返回后
  └── ceph_put_cap_refs(Fwb)                       [caps.c:3326]
        ├── wb refs 1 → 0                           减少写/缓冲引用
        ├── had Fwb last put                         最后一次释放 Fwb
        └── __ceph_caps_issued() = pAsxLsXsxFsxcrwb 仍持有这些 cap
  │
  └── ceph_check_caps()                             检查 cap 状态 [caps.c:2010]
        ├── file_want pAsxXsxFxwb                    文件想要
        ├── used Fcb                                 当前使用中
        ├── dirty Fw                                 脏
        ├── flushing -                               无正在刷新
        ├── issued pAsxLsXsxFsxcrwb                  已发行
        └── retain pAsxLsxXsxFsxcrwbl
```

写操作完成后的状态总结：

| 状态 | 值 | 说明 |
|------|-----|------|
| file_want | `pAsxXsxFxwb` | 文件打开的想要 cap |
| used | `Fcb` | 正在使用中: F(ile), c(ache), b(uffer) |
| dirty | `Fw` | 脏: F(ile) w(rite) — 需要通知 MDS |
| flushing | `-` | 空闲，无正在刷新 |
| issued | `pAsxLsXsxFsxcrwb` | MDS 授权的完整集合 |
| revoking | `-` | 无正在回收的 cap |

此时数据已写入页面缓存（page cache），但尚未刷新到 OSD。

---

## 5. 第四阶段：Flush 与卸载过程

### 5.1 预卸载处理

```
日志第460-467行:
[22224.502516] ceph:  statfs
[22224.503402] ceph:  kill_sb 0000000074c64218
[22224.503404] ceph:  pre_umount
[22224.503406] ceph:  request mdlog flush to mds0 (open)s seq 0
[22224.503408] ceph:  flush_dirty_caps
```

**代码调用栈：**

```
ceph_kill_sb()                                     [super.c:1580+]
  ├── ceph_pre_umount()
  │     ├── request mdlog flush to mds0             请求 MDS 日志刷新
  │     └── ceph_flush_dirty_caps()                 [caps.c:4689]
  │           └── 遍历所有脏 inode，触发 cap flush
  │
  └── shutdown_cache() / kill_anon_super()
```

`ceph_flush_dirty_caps()` ([caps.c:4689](https://)) 遍历 MDS 客户端的所有脏 inode，为每个脏 inode 触发 `ceph_check_caps(ci, CHECK_CAPS_FLUSH)`。

```
日志第464-474行:
[22224.503409] ceph:  flush_dirty_caps 10000000001.fffffffffffffffe
[22224.503411] ceph:  __ceph_caps_issued 0000000096c34c41 cap 000000004562469c issued pAsxLsXsxFsxcrwb
[22224.503412] ceph:  check_caps ... FLUSH
[22224.503414] ceph:  flushing dirty caps
[22224.503415] ceph:  __mark_caps_flushing flushing Fw, flushing_caps - -> Fw
[22224.503415] ceph:   inode 0000000096c34c41 now !dirty
[22224.503418] ceph:  encode_cap_msg update 1 10000000001 caps pAsxXsxFsxcrwb wanted pAsxXsxFsxcrwb dirty Fw seq 2/2 tid 2/2 mseq 0 follows 1 size 28/0
```

**Cap Flush 过程：**

```
ceph_check_caps(FLUSH)                             [caps.c:2010]
  ├── check_caps 10000000001.fffffffffffffffe
  │     ├── file_want pAsxXsxFsxcrwb
  │     ├── dirty Fw
  │     └── → "FLUSH" 标志触发脏 cap 刷新
  │
  └── __mark_caps_flushing()                        [caps.c:1909]
        ├── flushing_caps: - → Fw                   标记 Fw 为正在刷新
        ├── inode now !dirty                        清除脏标记 (dirty_caps: Fw → -)
        └── encode_cap_msg()                        编码 CAP UPDATE 消息
              ├── caps pAsxXsxFsxcrwb (移除了 Ls 和部分)
              ├── dirty Fw (通知 MDS Fw 已刷新)
              ├── size 28 (新文件大小)
              └── mtime/atime (文件时间戳)
```

**encode_cap_msg 内容解读：**
- `caps pAsxXsxFsxcrwb`: 客户端当前使用的 cap
- `wanted pAsxXsxFsxcrwb`: 客户端希望继续持有的 cap
- `dirty Fw`: 本次 flush 的脏 cap 位
- `size 28/0`: 新大小 28 字节，max_size 仍为 0（该进程不需要更多空间）

---

### 5.2 writepages 脏页回写

```
日志第477-489行:
[22224.503468] ceph:  writepages_start 0000000096c34c41 (mode=NONE)
[22224.503470] ceph:   head snapc 00000000d8b781fd has 1 dirty pages
[22224.503513] ceph:  0000000096c34c41 will write page 000000000828fb14 idx 0
[22224.503528] ceph:  writepages got pages at 0~28
[22224.503538] ceph:  writepages dend - startone, rc = 0
```

**代码调用栈：**

```
sync_inodes_sb() → writeback_inodes_sb()
  └── ceph_writepages_start()                       [addr.c:1642]
        ├── mode=NONE                                非回收模式
        ├── head snapc has 1 dirty pages             头部快照有 1 个脏页
        ├── 扫描脏页
        │     ├── 旧快照: not cyclic, 0 to 2251799813685247
        │     └── pagevec_lookup_range_tag()         找到 1 个脏页
        │           └── folio 000000000828fb14 idx 0
        │
        ├── 构建 OSD 写入请求
        │     └── writepages got pages at 0~28       0 偏移写入 28 字节
        │
        └── 提交 OSD 请求
              └── dend - startone, rc = 0            提交完成，返回 0
```

`ceph_writepages_start()` ([addr.c:1642](https://)) 的回写流程：
1. 查找脏页并组装成连续的写入范围
2. 为每个范围构建 `ceph_osd_request`
3. 提交 OSD 请求（异步写入到 Ceph 对象存储）
4. 注册 `writepages_finish()` 作为完成回调

---

### 5.3 Cap Flush 与 Flush ACK

```
日志第514-519行:
[22224.506610] ceph:  handle_caps from mds0
[22224.506613] ceph:   op flush_ack ino 10000000001.fffffffffffffffe inode 0000000096c34c41
[22224.506614] ceph:   mds0 seq 1 cap seq 2
[22224.506616] ceph:  handle_cap_flush_ack inode 0000000096c34c41 mds0 seq 2 on Fw cleaned Fw, flushing Fw -> -
[22224.506617] ceph:   inode 0000000096c34c41 now !flushing
[22224.506617] ceph:   inode 0000000096c34c41 now clean
```

**代码调用栈：**

```
handle_caps (from mds0)                            [caps.c:3811?]
  └── handle_cap_flush_ack()                        [caps.c:3811]
        ├── op flush_ack                            操作类型: Flush 确认
        ├── mds0 seq 1 cap seq 2                    MDS cap seq=1, client cap seq=2
        ├── cleaned Fw                              MDS 确认 Fw 已清理
        ├── flushing Fw → -                         清除 flushing 状态
        ├── inode now !flushing                     刷新完成
        └── inode now clean                         所有 dirty/flushing 均已清除
```

Cap flush 确认后，inode 回到完全干净状态：

| 状态 | 之前 | 之后 |
|------|------|------|
| dirty | Fw | - |
| flushing | Fw | - |
| issued | pAsxLsXsxFsxcrwb | pAsxLsXsxFsxcrwb |

---

### 5.4 最终 sync 与销毁

```
日志第522-576行 (删除 + 销毁):
[22224.506637] ceph:  ceph_d_prune test.txt 00000000cebf0021
[22224.506639] ceph:  evict_inode 00000000de2b9de4 ino 10000000000.fffffffffffffffe
[22224.506661] ceph:  ceph_d_prune test_trace.txt 000000001fc8d320
[22224.506662] ceph:  evict_inode 0000000096c34c41 ino 10000000001.fffffffffffffffe
[22224.506669] ceph:  ceph_d_prune / 000000006e08304f
[22224.506670] ceph:  evict_inode 000000006b5fd166 ino 1.fffffffffffffffe
...
[22224.506692] ceph:  request_close_session mds0 state closing seq 1
[22224.509326] ceph:  __unregister_session mds0 000000009680d5d3
[22224.509358] ceph:  stopped
[22224.509500] ceph:  destroy_fs_client 00000000fbf4a880
[22224.509546] ceph:  destroy_mount_options 00000000d5c8dd09
[22224.509686] ceph:  destroy_fs_client 00000000fbf4a880 done
```

**代码调用栈：**

```
ceph_kill_sb()
  └── kill_anon_super()
        ├── generic_shutdown_super()
        │     ├── ceph_d_prune(各个 dentry)          清理 dcache [dir.c?]
        │     │     ├── "test.txt" 00000000cebf0021
        │     │     ├── "test_trace.txt" 000000001fc8d320
        │     │     └── "/" 000000006e08304f
        │     │
        │     ├── evict_inode(每个 inode)             驱逐 inode [inode.c?]
        │     │     ├── 10000000000.fffffffffffffffe (test.txt)
        │     │     ├── 10000000001.fffffffffffffffe (test_trace.txt)
        │     │     └── 1.fffffffffffffffe (根目录)
        │     │
        │     └── ceph_d_release(每个 dentry)         释放 dentry 内存
        │
        ├── ceph_sync_fs(sb)                          同步文件系统 [super.c?]
        │     ├── sync (blocking)                     
        │     ├── ceph_flush_dirty_caps()             
        │     └── flush_mdlog_and_wait_mdsc_unsafe_requests()
        │           └── 等待所有 unsafe MDS 请求完成
        │
        ├── ceph_put_super(sb)                        释放超级块 [super.c?]
        │     └── close_sessions()
        │           ├── request_close_session(mds0)   请求关闭 MDS 会话
        │           ├── waiting for sessions to close 等待 MDS 确认关闭
        │           └── handle_session(close)         处理关闭确认
        │                 ├── __unregister_session    注销会话
        │                 ├── cleanup_session_requests 清理未完成请求
        │                 ├── remove_session_caps      移除与该会话关联的 cap
        │                 ├── iterate_session_caps     迭代清理每个 cap
        │                 ├── dispose_cap_releases     处理延迟的 cap 释放
        │                 └── kick_requests            踢醒等待中的请求
        │
        └── destroy_fs_client()                       销毁客户端 [super.c:886]
              ├── ceph_fs_debugfs_cleanup()            清理 debugfs
              ├── ceph_mdsc_destroy()                  销毁 MDS 客户端 [mds_client.c]
              ├── ceph_debugfs_client_cleanup()
              ├── ceph_destroy_client()                销毁 libceph 客户端
              ├── destroy_mount_options()              释放挂载选项
              └── stopped                              完成
```

---

## 附录：Cap 能力位含义

Ceph 能力位定义于 `include/linux/ceph/ceph_fs.h:651-670`：

| 位符号 | 值 | 含义 |
|--------|-----|------|
| `CEPH_CAP_GSHARED` | 1 | Pin (p) - 共享 pin |
| `CEPH_CAP_GEXCL` | 2 | Auth (A) - 排他授权 |
| `CEPH_CAP_GCACHE` | 4 | Cache (c) - 缓存 |
| `CEPH_CAP_GRD` | 8 | Read (r) - 文件读取 |
| `CEPH_CAP_GWR` | 16 | Write (w) - 文件写入 |
| `CEPH_CAP_GBUFFER` | 32 | Buffer (b) - 缓冲 I/O |
| `CEPH_CAP_GWREXTEND` | 64 | Write Extend (x) - 扩展写入 |
| `CEPH_CAP_GLAZYIO` | 128 | Lazy I/O (l) - 懒 I/O |
| `CEPH_CAP_SAUTH` | 256 | Authoritative (A) - 授权 |
| `CEPH_CAP_SLINK` | 512 | Link (L) - 链接 |
| `CEPH_CAP_SXATTR` | 1024 | Xattr (X) - 扩展属性 |
| `CEPH_CAP_SFILE` | 2048 | File (F) - 文件布局/大小 |
| `CEPH_CAP_SFLOCK` | 4096 | Flock (F) - 文件锁 |

日志中使用的缩写映射：

| 缩写 | 对应 cap 位 |
|------|-----------|
| p | GSHARED (pin) |
| A | GEXCL (auth) + SAUTH (authoritative) |
| s | shared flag (区分共享/排他) |
| x | exclusive flag (排他标志) |
| L | SLINK |
| X | SXATTR |
| F | SFILE |
| c | GCACHE |
| r | GRD |
| w | GWR |
| b | GBUFFER |
| l | GLAZYIO |

---

## 附录：关键数据结构

### struct ceph_fs_client (super.h)

挂载点上下文，包含：
- `struct ceph_mount_options *mount_options` — 挂载选项
- `struct ceph_client *client` — libceph 客户端
- `struct ceph_mds_client *mdsc` — MDS 客户端
- `struct super_block *sb` — VFS 超级块
- `struct dentry *debugfs_*` — debugfs 目录
- `struct ceph_options *options` — 客户端选项

### struct ceph_mds_request (mds_client.h)

一次 MDS 请求的上下文，包含：
- `tid` — 事务 ID（唯一标识）
- `r_op` — 操作类型（GETATTR, CREATE, LOOKUP, etc）
- `r_dentry` / `r_inode` — 目标 dentry/inode
- `r_session` — 目标 MDS 会话
- `r_caps_reservation` — Cap 预留

### struct ceph_inode_info (super.h)

每个 Ceph inode 的私有数据，包含：
- `i_vino` — {ino, snap_id} 唯一标识
- `i_caps` — 能力列表
- `i_snap_realm` — 快照域
- `i_snap_caps` — 快照相关 cap
- `i_dirty_caps` / `i_flushing_caps` — 脏/刷新中的 cap 位
- `i_ceph_lock` — 保护上述字段的自旋锁

---

## 完整时间线总结

```
时间偏移(ms)    事件
────────────────────────────────────────────────────────────────
+0.000          开始解析挂载参数 (source=127.0.0.1:40503:/)
+0.009          ceph_get_tree 创建 fs_client
+0.419          sget_fc 分配超级块
+0.440          set_super 初始化超级块
+0.512          mount start — 开始挂载流程
+1.832          mon0 session established (MON 会话建立)
+2.689          handle_fsmap epoch 10 (收到 FS map)
+2.742          client4617 fsid e20c1068... (分配客户端 ID)
+2.766          open_root_inode opening '' (开始打开根 inode)
+3.071          handle_map epoch 9 (收到 MDS map)
+3.072          mdsmap_decode: mds0.8 (2)127.0.0.1:6828 up:active
+3.073          wake request tid 1 (唤醒等待的请求)
+7.694          handle_session mds0 open (MDS 会话打开)
+7.698          renewed_caps (刷新 cap TTL)
+7.699          __prepare_send_request tid 1 getattr (准备 GETATTR 请求)
+8.233          handle_reply tid 1 result 0 (收到根 inode 属性)
+8.252          ceph_fill_inode: mode 040755, uid.gid 1000.1000
+8.262          add_cap pAsLsXs seq 1 (添加根 inode cap)
+8.312          open_root_inode success (根 inode 打开成功)
+8.341          mount success (挂载成功！)

────────────────────────────────────────────────────────────────
+27.678         do_getattr (根目录 getattr)
+27.688         getxattr (ACL 检查)
+27.726         atomic_open test_trace.txt flags 33345 mode 0666
+27.763         do_request tid 2 create (发送 CREATE 请求)
+31.805         handle_reply tid 2 result 0 (MDS 返回成功)
+31.839         ceph_fill_inode test_trace.txt: mode 0100644
+31.869         add_cap pAsxLsXsxFsxcrwb seq 1 (新文件 cap)
+31.880         dn attached to inode (dentry 关联到 inode)
+31.921         ceph_open fmode 2 want pAsxXsxFxwb (打开文件)
+31.922         atomic_open result=0 (atomic_open 成功！)

────────────────────────────────────────────────────────────────
+31.930         do_getattr (新文件 getattr)
+31.948         aio_write 0~28 getting caps (开始写入 28 字节)
+35.721         __ceph_pool_perm_get pool 5 result = 3 (pool 权限 OK)
+35.725         get_cap_refs got Fwb (获取 Fwb cap)
+35.732         write_end folio 0~28 (写入 28 字节到页面缓存)
+35.740         set_size 0 → 28 (更新文件大小)
+35.755         dirty_folio (标记 folio 为脏)
+35.758         __mark_dirty_caps Fw (标记 Fw cap 为脏)
+35.760         __cap_delay_requeue (排队延迟 cap flush)
+35.761         aio_write dropping cap refs on Fwb (释放 cap 引用)
+35.766         ceph_check_caps (检查 cap 状态)
+35.770         file release (关闭文件 fd)

────────────────────────────────────────────────────────────────
+75.026         statfs (文件系统统计)
+75.912         kill_sb 卸载触发
+75.913         pre_umount
+75.915         flush_dirty_caps (开始刷新脏 cap)
+75.920         check_caps FLUSH (触发 cap flush)
+75.922         __mark_caps_flushing Fw
+75.926         encode_cap_msg size 28/0 (通知 MDS 文件大小 28)
+75.976         writepages_start (开始页面回写)
+75.977         head snapc has 1 dirty pages
+76.022         will write page idx 0 (回写第 0 页)
+76.036         writepages got pages at 0~28 (回写范围 0~28)
+76.046         writepages dend rc=0 (回写完成)
+79.119         handle_caps flush_ack from mds0 (MDS 确认 cap flush)
+79.121         handle_cap_flush_ack cleaned Fw
+79.122         inode now clean (inode 回到干净状态)
+79.134         sync_fs (blocking) done
+79.145         ceph_d_prune (清理 dcache)
+79.146         evict_inode (驱逐 inode)
+79.200         request_close_session mds0 (关闭 MDS 会话)
+81.835         __unregister_session (注销 MDS 会话)
+81.866         stopped
+82.010         destroy_fs_client (销毁客户端)
+82.055         destroy_fs_client done

────────────────────────────────────────────────────────────────
总耗时: 约 82ms (从 mount 开始到 destroy_fs_client done)
其中:
  mount 耗时: ≈8.34ms
  atomic_open 耗时: ≈4.15ms (从请求到回复)
  write 耗时: ≈0.01ms (仅页面缓存写入)
  unmount 耗时: ≈6.14ms
```

---

*分析基于 Linux 内核源码 `/home/i_ingfeng/linux` 和 dmesg 日志 `/home/i_ingfeng/ceph-skills/deploy/wsl-qemu/方案A/dmesg_full.log`*
