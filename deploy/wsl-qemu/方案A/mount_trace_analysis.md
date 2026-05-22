# CephFS 挂载全链路追踪

操作: `mount → echo test_trace.txt → cat test_trace.txt → ls -l → umount`
日期: 2026-05-22 15:48:46

---

## 阶段 1: mount — 参数解析与 MON 连接

| dmesg 时间 | 集群时间 | 来源 | 关键日志 |
|-----------|---------|------|---------|
| [22224.427] | - | 内核 | `ceph_parse_source '127.0.0.1:40503:/'` — 解析 MON 地址 |
| [22224.427] | - | 内核 | `trying old device syntax` — 使用旧挂载语法 |
| [22224.427] | - | 内核 | `mount start 00000000fbf4a880` — 开始挂载 |
| [22224.429] | 15:48:46.220 | **MON** | `accept listen_fd=26 ... accept accepted incoming on sd 42` — MON 在 v2:40503 上收到连接 |
| [22224.429] | 15:48:46.223 | **MON** | `msgr2 ... start_server_banner_exchange` → `AUTH_ACCEPTING` — MSGR2 加密握手 |
| [22224.429] | - | 内核 | `mon0 (2)127.0.0.1:40503 session established` — 与 MON 建立会话 |

---

## 阶段 2: mount — 获取集群地图与 MDS 连接

| dmesg 时间 | 集群时间 | 来源 | 关键日志 |
|-----------|---------|------|---------|
| [22224.430] | - | 内核 | `handle_fsmap epoch 10` — 收到文件系统地图 |
| [22224.430] | - | 内核 | `client4617 fsid e20c1068-04d5-4745-a133-9853db2f5e8a` — 注册客户端 ID |
| [22224.430] | - | 内核 | `mdsmap_decode 1/1 4230 mds0.8 (2)127.0.0.1:6828 up:active` — 发现 myfs 的 MDS 地址 |
| [22224.430] | - | 内核 | `__choose_mds chose random mds0` → `open_session to mds0` |
| [22224.430] | 15:48:46.399 | **MDS** | `new session 0x59424ddc7900 for client.4617 127.0.0.1:0/662313906` — MDS 创建新会话 |

---

## 阶段 3: mount — MDS 会话打开与根 inode 查询

| dmesg 时间 | 集群时间 | 来源 | 关键日志 |
|-----------|---------|------|---------|
| [22224.435] | 15:48:46.399 | **MDS** | `client_session(request_open) from client.4617` — 内核请求打开会话 |
| [22224.430] | 15:48:46.399 | **MDS** | `ESession client.4617 ... open cmapv 14` — 会话写入日志（持久化） |
| [22224.435] | 15:48:46.403 | **MDS** | `New client session: addr="127.0.0.1:0/662313906",elapsed=0.002921,status="ACCEPTED",root="/"` |
| [22224.435] | 15:48:46.403 | **MDS** | `client_session(open cap_auths [MDSCapAuth(readable=1, writeable=1)])` — 回复会话打开，授予读写权 |
| [22224.435] | 15:48:46.403 | **MDS** | `client_request(client.4617:1 getattr p #0x1 ...)` — 内核发请求查根目录属性 |
| [22224.435] | - | 内核 | `__prepare_send_request ... tid 1 getattr (attempt 1) path '' r_parent = 0000000000000000` |
| [22224.435] | 15:48:46.403 | **MDS** | `reply to stat on client_request(client.4617:1 getattr p #0x1)` — MDS 返回根 inode |
| [22224.435] | 15:48:46.403 | **MDS** | `client_reply(???:1 = 0 (0) Success)` — 回复成功，带 cap |
| [22224.435] | - | 内核 | `handle_reply tid 1 result 0` |
| [22224.435] | - | 内核 | `add_cap inode ... cap 0000000091fa6ff9 pAsLsXs now pAsLsXs seq 1 mds0` — 获得 Pin/As/Ls/Xs cap |
| [22224.435] | - | 内核 | `000000006b5fd166 mode 040755 uid.gid 1000.1000` — 根目录属性 |
| [22224.435] | - | 内核 | `mount success` — 挂载完成 |
| [22224.435] | - | 内核 | `root ... inode 000000006b5fd166 ino 1.fffffffffffffffe` — 根 inode 号 |

---

## 阶段 4: echo "hello ceph trace" > /mnt/cephfs/test_trace.txt — 创建文件并写入

| dmesg 时间 | 集群时间 | 来源 | 关键日志 |
|-----------|---------|------|---------|
| [22224.455] | - | 内核 | `atomic_open 000000006b5fd166 dentry ... 'test_trace.txt' unhashed flags 33345 mode 0100666` — 试图创建+打开 |
| [22224.455] | - | 内核 | `do_request on 00000000a170d66e` → `tid 2 create` |
| [22224.455] | - | 内核 | `__prepare_send_request ... tid 2 create (attempt 1) dentry 1/test_trace.txt` — 向 MDS 发创建请求 |
| [22224.455] | 15:48:46.418 | **MDS** | `client_request(client.4617:2 create owner_uid=0, owner_gid=0 #0x1/test_trace.txt ...)` — MDS 收到创建请求 |
| [22224.455] | 15:48:46.418 | **MDS** | `open w/ O_CREAT on #0x1/test_trace.txt` — MDS 处理创建 |
| [22224.459] | - | 内核 | `handle_reply tid 2 result 0` — MDS 返回成功 |
| [22224.459] | - | 内核 | `get_inode on 1099511627777=10000000001.fffffffffffffffe got 0000000096c34c41 new 1` — 新文件的 inode |
| [22224.459] | - | 内核 | `add_cap ... cap 000000004562469c pAsxLsXsxFsxcrwb seq 1 mds0` — 获得全部 cap |
| [22224.459] | - | 内核 | `atomic_open result=0` — 文件创建+打开成功 |
| [22224.459] | - | 内核 | `aio_write 0000000096c34c41 10000000001.fffffffffffffffe 0~28` — 发起 28 字节异步写 |
| [22224.459] | - | 内核 | `get_cap_refs ... ret 1 got Fwb` — 获得文件写缓冲权限 |
| [22224.463] | - | 内核 | `write_end ... inode 0000000096c34c41 folio ... 0~28 (28)` — 页缓存写入完成 |
| [22224.463] | - | 内核 | `set_size 0000000096c34c41 0 -> 28` — 文件大小更新为 28 字节 |
| [22224.463] | - | 内核 | `dirty_folio ... head 0/0 -> 1/1` — 标记脏页，后续异步刷到 OSD |
| [22224.463] | - | 内核 | `release inode 0000000096c34c41 regular file` — 释放文件引用 |

> **注意**: aio_write 的 28 字节数据由内核客户端直接发 RADOS write 到 OSD 0/1/2（3副本，size=3），不经过 MDS。OSD 日志中因 memstore 写操作在 OSD 内部无明显日志行。

---

## 阶段 5: cat /mnt/cephfs/test_trace.txt — 读取文件

| dmesg 时间 | 集群时间 | 来源 | 关键日志 |
|-----------|---------|------|---------|
| [22224.479] | - | 内核 | `d_revalidate ... 'test_trace.txt' ... valid` — dentry 缓存有效 |
| [22224.479] | - | 内核 | `do_getattr inode ... mask As mode 0100644` — 检查文件属性（缓存命中） |
| [22224.479] | - | 内核 | `open inode ... fmode 1 want pFscr issued pAsxLsXsxFsxcrwb using existing` — 以读模式打开 |
| [22224.479] | - | 内核 | `aio_read ... 0~4194304` → `got cap refs on Fcr` — 获取文件读缓存权限 |
| [22224.479] | - | 内核 | `aio_read ... dropping cap refs on Fcr = 28` — 读返回 28 字节（文件内容） |
| [22224.479] | - | 内核 | `release inode 0000000096c34c41 regular file` |

> **关键**: 本次 cat 完全命中内核页缓存（之前 write 后 cache 还在），无任何网络请求。

---

## 阶段 6: ls -l /mnt/cephfs/ — 目录列表

| dmesg 时间 | 集群时间 | 来源 | 关键日志 |
|-----------|---------|------|---------|
| [22224.482] | - | 内核 | `readdir 000000006b5fd166 file ... pos 0` — 对根目录发起 readdir |
| [22224.482] | - | 内核 | `__prepare_send_request ... tid 3 getattr` — 先查根目录属性 |
| [22224.483] | - | 内核 | `handle_reply tid 3 result 0` — 返回（部分缓存命中） |
| [22224.483] | - | 内核 | `readdir off 0 -> '.'` `readdir off 1 -> '..'` — readdir 内置 . 和 .. |
| [22224.483] | - | 内核 | `readdir fetching 1.fffffffffffffffe frag 0 offset '(null)'` — 需要向 MDS 请求 |
| [22224.483] | - | 内核 | `__prepare_send_request ... tid 4 readdir inode ... 1.fffffffffffffffe` |
| [22224.483] | 15:48:46.xxx | **MDS** | MDS 收到 readdir 请求，返回目录条目列表 |
| [22224.485] | - | 内核 | `handle_reply tid 4 result 0` |
| [22224.485] | - | 内核 | `parsed dir dname 'test.txt'` — 条目 1 |
| [22224.485] | - | 内核 | `parsed dir dname 'test_trace.txt'` — 条目 2 |
| [22224.485] | - | 内核 | `readdir_prepopulate 2 items under dn 000000006e08304f` — 预填充 2 个条目 |
| [22224.485] | - | 内核 | `readdir (0/2) -> ff0b61080000002 'test.txt'` |
| [22224.485] | - | 内核 | `readdir (1/2) -> ff647c0e0000002 'test_trace.txt'` |
| [22224.485] | - | 内核 | `readdir ... done` — readdir 完成 |

---

## 阶段 7: umount — 卸载

| dmesg 时间 | 集群时间 | 来源 | 关键日志 |
|-----------|---------|------|---------|
| [22224.506] | - | 内核 | `sync_fs (blocking)` — 开始同步刷盘 |
| [22224.506] | - | 内核 | `check_caps_flush ok, flushed thru 2` — cap 刷出完成 |
| [22224.506] | - | 内核 | `put_super` → `close_sessions` |
| [22224.506] | - | 内核 | `request_close_session mds0 state closing seq 1` — 请求关闭 MDS 会话 |
| [22224.509] | 15:48:46.xxx | **MDS** | MDS 返回 close 响应 |
| [22224.509] | - | 内核 | `handle_session mds0 close ... state closing seq 0` |
| [22224.509] | - | 内核 | `remove_session_caps on 000000009680d5d3` — 回收所有 cap |
| [22224.509] | - | 内核 | `put_cap ... 22 = 3 used + 0 resv + 19 avail` — 释放 cap 1 |
| [22224.509] | - | 内核 | `put_cap ... 22 = 2 used + 0 resv + 20 avail` — 释放 cap 2 |
| [22224.509] | - | 内核 | `put_cap ... 22 = 1 used + 0 resv + 21 avail` — 释放 cap 3 |
| [22224.509] | - | 内核 | `stopped` → `ceph_fs_debugfs_cleanup` → `destroy_fs_client done` |

---

## Cap 权限速查

| Cap 缩写 | 含义 |
|---------|------|
| **p** | Pin — 保留该 inode 在缓存中的权限 |
| **As** | Auth shared — 共享属性权限 (mode, uid, gid) |
| **Ls** | Link shared — 链路计数 |
| **Xs** | Excl shared — 共享执行/目录属性 |
| **Fs** | File shared — 文件大小 |
| **x** | 扩展属性权限 |
| **Fw** | File write — 文件写入 |
| **Fb** | File write buffer — 写缓冲 |
| **Fc** | File cache — 文件缓存读取 |
| **Fr** | File read — 文件读取 |
| **Fscr** | File shared cache read — 共享缓存读 |

## 文件路径

- `dmesg_full.log` — 内核客户端完整日志
- `mds_b_mount.log` — MDS b (myfs) 过滤日志
- `mon_a_mount.log` — MON a 过滤日志
- `osd_0_mount.log` — OSD 0 过滤日志
- `out/` — 完整原始日志 (mds.b.log, mon.a.log, osd.0.log 等)
