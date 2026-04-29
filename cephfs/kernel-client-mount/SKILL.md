---
name: kernel-cephfs-mount
description: 内核CephFS挂载流程专家。当用户询问内核CephFS挂载步骤、mount选项解析、MON连接、MDS会话建立、ceph_fs_client初始化、libceph客户端创建时调用此技能。
---

# 内核 CephFS 挂载流程

## 〇、为什么需要了解挂载流程？

挂载是内核 CephFS 客户端与集群建立连接的起点。理解挂载流程有助于：

1. **故障排查**: 挂载失败时定位是 MON 连接、认证、还是 MDS 会话问题
2. **性能调优**: 理解 mount 选项如何影响 I/O 路径
3. **安全配置**: 理解认证流程和权限模型
4. **多文件系统**: 理解 `mds_namespace` 如何区分不同 CephFS

---

## 一、挂载整体流程

```
mount -t ceph mon1:6789:/ /mnt -o name=admin

Phase 1: 模块初始化 (仅一次)
  init_ceph()
    → 注册 slab caches (inode, cap, dentry)
    → 注册文件系统 (ceph_fs_type)

Phase 2: FS Context 初始化
  ceph_init_fs_context()
    → 分配 ceph_options (libceph 层)
    → 分配 ceph_mount_options (文件系统层)
    → 设置默认选项

Phase 3: Mount 选项解析
  ceph_parse_mount_param() × N
    → 解析每个 -o 选项

Phase 4: 获取文件系统树
  ceph_get_tree()
    → create_fs_client()
    → ceph_mdsc_init()
    → sget_fc() → ceph_set_super()
    → ceph_real_mount()

Phase 5: 连接集群
  __ceph_open_session()
    → 连接 MON, 认证
    → 等待 OSDMap

Phase 6: 打开根目录
  open_root_dentry()
    → 建立 MDS 会话
    → 获取根 inode
```

---

## 二、关键数据结构

### 2.1 ceph_fs_client (linux/fs/ceph/super.h:146-190)

```c
struct ceph_fs_client {
    struct super_block *sb;              // VFS 超级块
    struct ceph_mount_options *mount_options;  // 挂载选项
    struct ceph_client *client;          // 共享 libceph 客户端
    struct ceph_mds_client *mdsc;        // MDS 客户端
    enum ceph_mount_state mount_state;   // MOUNTING/MOUNTED/UNMOUNTING/SHUTDOWN
    struct workqueue_struct *inode_wq;   // inode 工作队列
    struct workqueue_struct *cap_wq;     // cap 工作队列
    u64 max_file_size;                   // 最大文件大小 (来自 mdsmap)
    bool blocklisted;                    // 是否被集群 blocklist
};
```

### 2.2 ceph_mount_options (linux/fs/ceph/super.h:80-105)

```c
struct ceph_mount_options {
    unsigned int flags;                  // CEPH_MOUNT_OPT_* 位掩码
    unsigned int wsize;                  // 写缓冲区大小
    unsigned int rsize;                  // 读缓冲区大小
    unsigned int rasize;                 // 预读大小
    unsigned int caps_wanted_delay_min;  // cap 释放延迟最小值
    unsigned int caps_wanted_delay_max;  // cap 释放延迟最大值
    unsigned int max_readdir;            // 最大 readdir 条目数
    unsigned int max_readdir_bytes;      // 最大 readdir 字节数
    char *snapdir_name;                  // 快照目录名 (默认 ".snap")
    char *mds_namespace;                 // MDS 命名空间
    char *server_path;                   // 服务器路径
    bool new_dev_syntax;                 // 是否使用新语法
};
```

### 2.3 ceph_client (linux/include/linux/ceph/libceph.h:115-142)

```c
struct ceph_client {
    struct ceph_fsid fsid;               // 集群 FSID
    struct ceph_options *options;        // libceph 选项
    struct ceph_messenger msgr;          // 网络消息层
    struct ceph_mon_client monc;         // Monitor 客户端
    struct ceph_osd_client osdc;         // OSD 客户端
    struct mutex mount_mutex;            // 序列化挂载操作
    wait_queue_head_t auth_wq;           // 认证等待队列
    int (*extra_mon_dispatch)(...);      // FS 特定 MON 消息回调
};
```

### 2.4 ceph_mds_client (linux/fs/ceph/mds_client.h:442-551)

```c
struct ceph_mds_client {
    struct ceph_fs_client *fsc;          // 反向指针
    struct ceph_mdsmap *mdsmap;          // MDS 地图
    struct ceph_mds_session **sessions;  // MDS 会话数组
    int max_sessions;                    // 最大会话数
    struct rb_root request_tree;         // 待处理请求红黑树
    struct list_head cap_delay_list;     // cap 延迟列表
    struct list_head cap_dirty_list;     // 脏 cap 列表
    struct rw_semaphore snap_rwsem;      // 快照读写信号量
    struct ceph_metric metric;           // 性能指标
    struct delayed_work delayed_work;    // 定期工作 (keepalive, cap 管理)
};
```

### 2.5 ceph_mds_session (linux/fs/ceph/mds_client.h:214-254)

```c
struct ceph_mds_session {
    int s_mds;                           // MDS rank
    enum ceph_mds_session_state s_state; // NEW/OPENING/OPEN/HUNG/RESTARTING...
    struct ceph_connection s_con;        // MDS 连接
    struct ceph_auth_handshake s_auth;   // 认证握手
    struct list_head s_caps;             // 该 MDS 授予的所有 cap
    struct list_head s_cap_dirty;        // 脏 cap
    struct list_head s_cap_flushing;     // 刷写中的 cap
    struct xarray s_delegated_inos;      // 委托的 inode 号
};
```

---

## 三、Mount 选项解析

### 3.1 默认选项 (linux/fs/ceph/super.h:49-52)

```c
#define CEPH_MOUNT_OPT_DEFAULT      (CEPH_MOUNT_OPT_DCACHE | \
                                     CEPH_MOUNT_OPT_NOCOPYFROM | \
                                     CEPH_MOUNT_OPT_ASYNC_DIROPS)
```

### 3.2 选项解析入口 (linux/fs/ceph/super.c:403-619)

```c
ceph_parse_mount_param(fc, param)
    |
    +-- ceph_parse_param()           // libceph 通用选项
    |     name, secret, mon_addr, mount_timeout
    |
    +-- fs_parse()                   // 文件系统特定选项
          snapdirname, mds_namespace, wsize, rsize, rasize,
          dirstat, rbytes, asyncreaddir, dcache, ino32,
          fscache, poolperm, quotadf, copyfrom, wsync,
          pagecache, sparseread, acl, recover_session
```

### 3.3 源地址解析 (linux/fs/ceph/super.c:252-318)

支持两种语法：

**新语法**: `name@fsid.fsname=/path`
```c
ceph_parse_new_source("admin@abc123.cephfs=/")
    → entity_name = "admin"
    → fsid = "abc123"
    → fs_name = "cephfs"
    → server_path = "/"
```

**旧语法**: `host1:port1,host2:port2:/path`
```c
ceph_parse_old_source("mon1:6789,mon2:6789:/")
    → 解析 MON 地址列表
    → server_path = "/"
    → new_dev_syntax = false
```

---

## 四、核心挂载函数

### 4.1 ceph_get_tree (linux/fs/ceph/super.c:1291-1370)

```c
ceph_get_tree(fc)
    |
    +-- create_fs_client(fsopt, opt)
    |     |
    |     +-- kzalloc(ceph_fs_client)
    |     +-- ceph_create_client(opt, fsc)     // 创建 libceph 客户端
    |     |     |
    |     |     +-- ceph_messenger_init()      // 网络层初始化
    |     |     +-- ceph_monc_init()           // MON 客户端初始化
    |     |     +-- ceph_osdc_init()           // OSD 客户端初始化
    |     |
    |     +-- ceph_monc_want_map(monc, CEPH_SUB_MDSMAP, ...)  // 订阅 MDSMap
    |     +-- alloc_workqueue("ceph-inode")
    |     +-- alloc_workqueue("ceph-cap")
    |
    +-- ceph_mdsc_init(fsc)                    // MDS 客户端初始化
    |
    +-- sget_fc(fc, ceph_compare_super, ceph_set_super)
    |     |
    |     +-- 检查是否有兼容的已有 superblock
    |     +-- 如果没有: 调用 ceph_set_super()
    |           +-- 设置 s_maxbytes, s_op, s_magic
    |           +-- set_anon_super_fc()
    |
    +-- ceph_setup_bdi(sb, fsc)                // 设置 BDI (ra_pages, io_pages)
    |
    +-- ceph_real_mount(fsc, fc)               // 实际挂载 (见 4.2)
    |
    +-- fc->root = fsc->sb->s_root
```

### 4.2 ceph_real_mount (linux/fs/ceph/super.c:1140-1194)

```c
ceph_real_mount(fsc, fc)
    |
    +-- __ceph_open_session(fsc->client)
    |     |
    |     +-- ceph_monc_open_session(&client->monc)
    |     |     |
    |     |     +-- 订阅 MONMAP, OSDMAP
    |     |     +-- __open_session(monc)
    |     |           |
    |     |           +-- pick_new_mon(monc)     // 随机选择一个 MON
    |     |           +-- ceph_con_open()        // 打开 TCP 连接
    |     |           +-- 发送认证请求
    |     |
    |     +-- 等待循环 (auth_wq):
    |     |     - have_monmap (收到并解码 monmap)
    |     |     - have_osdmap (收到并解码 osdmap)
    |     |     - 或 auth_err / timeout / signal
    |     |
    |     +-- pr_info("client%llu fsid %pU\n")
    |     +-- ceph_debugfs_client_init()
    |
    +-- ceph_fscache_register_fs()         // FSCache 注册 (可选)
    +-- ceph_apply_test_dummy_encryption() // 测试加密
    +-- ceph_fs_debugfs_init(fsc)          // DebugFS 初始化
    +-- open_root_dentry(fsc, path, started)  // 打开根目录 (见 4.3)
    +-- fsc->sb->s_root = dget(root)
    +-- fsc->mount_state = CEPH_MOUNT_MOUNTED
```

### 4.3 open_root_dentry (linux/fs/ceph/super.c:1047-1091)

```c
open_root_dentry(fsc, path, started)
    |
    +-- ceph_mdsc_create_request(mdsc, CEPH_MDS_OP_GETATTR, USE_ANY_MDS)
    +-- req->r_path1 = kstrdup(path)
    +-- req->r_ino1.ino = CEPH_INO_ROOT     // 根 inode = 1
    +-- req->r_ino1.snap = CEPH_NOSNAP
    +-- req->r_args.getattr.mask = CEPH_STAT_CAP_INODE
    +-- req->r_num_caps = 2
    +-- ceph_mdsc_do_request(mdsc, NULL, req)
    |     |
    |     +-- __choose_mds()                // 选择 MDS
    |     +-- register_session(mdsc, mds)   // 创建 MDS 会话
    |     |     |
    |     |     +-- 分配 ceph_mds_session
    |     |     +-- ceph_con_init(&s->s_con)
    |     |     +-- ceph_con_open(&s->s_con, CEPH_ENTITY_TYPE_MDS, mds, addr)
    |     |
    |     +-- __open_session(mdsc, session) // 发送 OPEN 请求
    |     |     |
    |     |     +-- 状态 = CEPH_MDS_SESSION_OPENING
    |     |     +-- create_session_full_msg()
    |     |     |     包含: hostname, kernel_version, entity_id,
    |     |     |          root path, supported features, metric spec
    |     |     +-- ceph_con_send()
    |     |
    |     +-- 等待回复，从 trace 中填充 inode
    |
    +-- d_make_root(inode)
    +-- 返回根 dentry
```

---

## 五、MON 连接与认证

### 5.1 MON 客户端初始化 (linux/net/ceph/mon_client.c:1161-1236)

```c
ceph_monc_init(monc, cl)
    |
    +-- build_initial_monmap(monc)      // 从 mount 选项构建初始 monmap
    +-- ceph_auth_init(name, key, con_modes)  // 认证子系统
    +-- monc->auth->want_keys = MON|OSD|MDS|AUTH
    +-- 分配消息: m_subscribe_ack, m_subscribe, m_auth_reply, m_auth
    +-- ceph_con_init(&monc->con, ...)  // 连接初始化
    +-- monc->cur_mon = -1              // 尚未连接
    +-- INIT_DELAYED_WORK(&monc->delayed_work, delayed_work)
```

### 5.2 MON 会话建立 (linux/net/ceph/mon_client.c:527-536)

```c
ceph_monc_open_session(monc)
    |
    +-- 订阅 MONMAP (CEPH_SUB_MONMAP)
    +-- 订阅 OSDMAP (CEPH_SUB_OSDMAP)
    +-- __open_session(monc)
          |
          +-- pick_new_mon(monc)         // 随机选择 MON
          +-- ceph_con_open()            // 打开 TCP 连接
          +-- 发送认证 hello 消息
```

### 5.3 认证握手 (linux/net/ceph/mon_client.c:1283-1331)

```
客户端 → MON: CEPH_MSG_AUTH (hello, entity_name, nonce)
MON → 客户端: CEPH_MSG_AUTH_REPLY (challenge)
客户端 → MON: CEPH_MSG_AUTH (response)
MON → 客户端: CEPH_MSG_AUTH_REPLY (success, global_id, con_secret)

finish_auth():
    → 设置 client->entity_type = CEPH_ENTITY_TYPE_CLIENT
    → 设置 client->global_id (从认证回复中)
    → 发送订阅请求 (MONMAP, OSDMAP)
```

### 5.4 MON 狩猎 (Hunting)

如果连接失败，客户端会尝试其他 MON：

```c
mon_fault() → reopen_session()
    |
    +-- pick_new_mon(monc)              // 随机选择新 MON
    +-- hunt_mult *= 2                  // 退避倍增
    +-- delayed_work 定期重试
```

---

## 六、MDS 会话管理

### 6.1 会话状态机

```
NEW → OPENING → OPEN
         ↓         ↓
       HUNG     RESTARTING → RECONNECTING → OPEN
         ↓         ↓
      CLOSED    CLOSING → CLOSED
         ↓
      REJECTED
```

### 6.2 会话 Keepalive

```c
mdsc->delayed_work 定期执行:
    → 对每个 OPEN 会话发送心跳
    → 检查会话超时 (session_timeout)
    → 如果超时: 状态 → HUNG, 使能 cap 和 lease
```

---

## 七、关键代码位置

| 功能 | 文件 | 行号 |
|------|------|------|
| FS 类型注册 | linux/fs/ceph/super.c | 1592-1598 |
| FS Context 初始化 | linux/fs/ceph/super.c | 1427-1473 |
| Mount 选项解析 | linux/fs/ceph/super.c | 403-619 |
| Mount 选项表 | linux/fs/ceph/super.c | 192-225 |
| 源地址解析 (新) | linux/fs/ceph/super.c | 271-318 |
| 源地址解析 (旧) | linux/fs/ceph/super.c | 252-269 |
| Get Tree | linux/fs/ceph/super.c | 1291-1370 |
| 创建 FS 客户端 | linux/fs/ceph/super.c | 806-877 |
| 实际挂载 | linux/fs/ceph/super.c | 1140-1194 |
| 打开根目录 | linux/fs/ceph/super.c | 1047-1091 |
| 创建 libceph 客户端 | linux/net/ceph/ceph_common.c | 706-756 |
| 打开会话 (等待) | linux/net/ceph/ceph_common.c | 790-841 |
| MON 客户端初始化 | linux/net/ceph/mon_client.c | 1161-1236 |
| MON 打开会话 | linux/net/ceph/mon_client.c | 527-536 |
| MON 认证握手 | linux/net/ceph/mon_client.c | 1283-1331 |
| MON Map 处理 | linux/net/ceph/mon_client.c | 539-574 |
| OSD 客户端初始化 | linux/net/ceph/osd_client.c | 5199-5267 |
| MDS 客户端初始化 | linux/fs/ceph/mds_client.c | 5533-5611 |
| MDS 会话注册 | linux/fs/ceph/mds_client.c | 964-1033 |
| MDS 会话打开 | linux/fs/ceph/mds_client.c | 1663-1687 |
| MDS 会话消息创建 | linux/fs/ceph/mds_client.c | 1539-1656 |
| 选择 MDS | linux/fs/ceph/mds_client.c | 1284-1437 |
| ceph_fs_client 结构 | linux/fs/ceph/super.h | 146-190 |
| ceph_mount_options 结构 | linux/fs/ceph/super.h | 80-105 |
| ceph_mds_client 结构 | linux/fs/ceph/mds_client.h | 442-551 |
| ceph_mds_session 结构 | linux/fs/ceph/mds_client.h | 214-254 |
| ceph_client 结构 | linux/include/linux/ceph/libceph.h | 115-142 |
| ceph_options 结构 | linux/include/linux/ceph/libceph.h | 47-70 |
| ceph_mon_client 结构 | linux/include/linux/ceph/mon_client.h | 70-105 |
| ceph_mdsmap 结构 | linux/include/linux/ceph/mdsmap.h | 24-49 |

---

## 八、常见问题与陷阱

### Q1: 挂载时 "timed out" 错误如何排查？

**A**: 按顺序检查：
1. MON 地址是否正确可达 (`telnet mon_ip 6789`)
2. 认证凭据是否正确 (`ceph auth get-key client.admin`)
3. 防火墙是否阻止了 MON/OSD/MDS 端口
4. 集群是否健康 (`ceph status`)

### Q2: `new_dev_syntax` 和 `old_dev_syntax` 的区别？

**A**:
- **新语法** (`name@fsid.fsname=/`): 明确指定 entity、FSID、文件系统名，支持多文件系统集群
- **旧语法** (`host:port:/`): 仅指定 MON 地址，不指定文件系统名，适用于单文件系统集群

### Q3: 多个 mount 到同一集群会共享连接吗？

**A**: 是的。`sget_fc()` 会检查是否有兼容的已有 superblock。如果 FSID、options 等匹配，多个 mount 会共享同一个 `ceph_client`（包括 MON 连接和 OSD 连接），但每个 mount 有独立的 `ceph_fs_client` 和 `ceph_mds_client`。

### 参考文献

1. **Linux 内核源码**: `fs/ceph/super.c`, `fs/ceph/mds_client.c`, `net/ceph/ceph_common.c`
2. **CephFS 挂载文档**: https://docs.ceph.com/en/latest/cephfs/mount-using-kernel-driver/
3. **Ceph 客户端认证**: https://docs.ceph.com/en/latest/rados/operations/user-management/
4. **"Understanding the Linux Kernel" Ch.12 (VFS)** — O'Reilly"
