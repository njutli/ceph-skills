---
name: async-messenger
description: Ceph AsyncMessenger通信层专家。当用户询问AsyncMessenger架构、连接状态机、ProtocolV2协议格式、epoll事件循环、消息分发、RDMA/DPDK支持、加密握手时调用此技能。
---

# AsyncMessenger 通信层

## 〇、为什么需要 AsyncMessenger？

为什么需要 AsyncMessenger？Ceph 所有组件之间的通信都依赖消息层：

```
传统同步 I/O 的问题:
  每个连接需要一个线程阻塞等待 I/O
  1000 个 OSD 连接 = 1000 个线程
  → 线程切换开销巨大
  → 内存浪费 (每线程 8MB 栈)
  → 无法充分利用 epoll 的多路复用能力
```

AsyncMessenger 的核心设计是：**单线程 epoll 事件循环管理数千连接 + 优先级消息队列 + 双分发路径**。

```
AsyncMessenger 方案:
  Worker 线程: epoll 事件循环 (非阻塞 I/O)
  DispatchQueue: 优先级消息队列 + 两个分发线程
  ProtocolV2: 帧加密/解密、认证、压缩协商
  快速路径: 心跳等低延迟消息内联分发
  普通路径: 其他消息排队到 DispatchQueue
```

---

## 一、Messenger 架构

### 1.1 四层架构

```
Layer 1: Messenger.h (抽象接口)
         - 纯虚基类，定义 messenger 契约
         - 管理 dispatchers (优先级排序列表)、策略、认证

Layer 2: SimplePolicyMessenger (中间基类)
         - 在 Messenger 之上添加策略管理

Layer 3: AsyncMessenger (具体实现)
         - 生产环境异步 I/O messenger
         - 拥有: NetworkStack, Processors, DispatchQueue, Workers

Layer 4: 协议实现 (ProtocolV1, ProtocolV2, LoopbackProtocolV1)
         - 每个连接的线级协议处理
```

### 1.2 AsyncMessenger 组件关系 (src/msg/async/AsyncMessenger.h:96-251)

```
AsyncMessenger
  ├── NetworkStack (stack)          — 传输抽象层
  │     ├── Worker[]                — 工作线程 (epoll 事件循环)
  │     │     ├── EventCenter       — 事件中心 (每个 Worker 一个)
  │     │     └── ProtocolV2[]      — 连接协议处理
  │     └── PosixStack/RDMAStack/DPDKStack
  │
  ├── Processor[] (processors)      — 监听/接受连接
  ├── DispatchQueue                 — 消息分发队列
  │     ├── DispatchThread          — 普通消息分发线程
  │     └── LocalDeliveryThread     — 本地/回环消息线程
  │
  ├── conns (map)                   — 活跃连接映射
  ├── accepting_conns               — 正在接受的连接
  └── deleted_conns                 — 待删除连接
```

---

## 二、连接状态机

### 2.1 连接高层状态 (src/msg/async/AsyncConnection.h:156-163)

| 状态 | 值 | 说明 |
|------|-----|------|
| `STATE_NONE` | 0 | 初始/未初始化 |
| `STATE_CONNECTING` | 1 | 发起出站连接 |
| `STATE_CONNECTING_RE` | 2 | 等待非阻塞连接完成 |
| `STATE_ACCEPTING` | 3 | 接受入站连接 |
| `STATE_CONNECTION_ESTABLISHED` | 4 | TCP 已连接，准备协议握手 |
| `STATE_CLOSED` | 5 | 连接终止 |

### 2.2 ProtocolV2 详细状态 (src/msg/async/ProtocolV2.h:16-43)

```
客户端状态:
  START_CONNECT → BANNER_CONNECTING → HELLO_CONNECTING
    → AUTH_CONNECTING → AUTH_CONNECTING_SIGN
    → COMPRESSION_CONNECTING (可选)
    → SESSION_CONNECTING / SESSION_RECONNECTING
    → READY

服务端状态:
  START_ACCEPT → BANNER_ACCEPTING → HELLO_ACCEPTING
    → AUTH_ACCEPTING → AUTH_ACCEPTING_SIGN
    → COMPRESSION_ACCEPTING (可选)
    → SESSION_ACCEPTING
    → READY

数据传输状态:
  READY → THROTTLE_MESSAGE → THROTTLE_BYTES
    → THROTTLE_DISPATCH_QUEUE → THROTTLE_DONE
    → READ_MESSAGE_COMPLETE → READY

流控状态:
  STANDBY / WAIT / CLOSED
```

### 2.3 状态转换

| 从状态 | 事件 | 到状态 | 位置 |
|--------|------|--------|------|
| `STATE_NONE` | `connect()` 调用 | `STATE_CONNECTING` | AsyncConnection.cc:520 |
| `STATE_CONNECTING` | TCP socket 创建，epoll 注册 | `STATE_CONNECTING_RE` | AsyncConnection.cc:432 |
| `STATE_CONNECTING_RE` | 非阻塞连接完成 | `STATE_CONNECTION_ESTABLISHED` | AsyncConnection.cc:461 |
| `STATE_CONNECTING_RE` | 连接失败 | fault → 重试/退避 | AsyncConnection.cc:436-455 |
| `STATE_ACCEPTING` | Socket 接受，epoll 注册 | `STATE_CONNECTION_ESTABLISHED` | AsyncConnection.cc:467 |
| `READY` | 读取帧完成 | 继续读取下一帧 | ProtocolV2.cc:1181 |
| 任意状态 | 故障 | `STANDBY`(服务端) 或 `START_CONNECT`(客户端) | ProtocolV2.cc:322-420 |

---

## 三、事件循环架构

### 3.1 EventCenter (src/msg/async/Event.h:92-264)

每个 Worker 线程拥有恰好一个 EventCenter：

```cpp
class EventCenter {
    vector<FileEvent> file_events;           // 按 fd 索引的文件事件
    multimap<time_point, TimeEvent> time_events;  // 定时事件 (按过期排序)
    deque<EventCallbackRef> external_events;      // 跨线程事件注入
    EventDriver *driver;                          // OS 特定的事件驱动
    int notify_receive_fd, notify_send_fd;        // eventfd (Linux) 或 pipe
    vector<Poller*> pollers;                      // 自定义轮询器
};
```

### 3.2 平台特定的 EventDriver

| 驱动 | 文件 | 平台 | 关键细节 |
|------|------|------|---------|
| **EpollDriver** | EventEpoll.cc | Linux | `epoll_create` + `epoll_wait`，**EPOLLT** (边沿触发) |
| **KqueueDriver** | EventKqueue.cc | BSD/macOS | kqueue(2) |
| **SelectDriver** | EventSelect.cc | 回退/Windows | `select()` — 明确警告不用于生产 |
| **PollDriver** | EventPoll.cc | Solaris/Windows | poll(2) |
| **DPDKDriver** | dpdk/EventDPDK.h | DPDK | 用户态网络 (轮询) |

### 3.3 EpollDriver 细节 (src/msg/async/EventEpoll.cc)

```cpp
// init (line 28): epoll_create(1024), 设置 FD_CLOEXEC

// add_event (line 56):
//   EPOLL_CTL_ADD 或 EPOLL_CTL_MOD
//   始终设置 EPOLLET (边沿触发) (line 67)
//   EVENT_READABLE → EPOLLIN, EVENT_WRITABLE → EPOLLOUT

// event_wait (line 121):
//   epoll_wait() 带超时
//   将 EPOLLIN/EPOLLOUT/EPOLLERR/EPOLLHUP 翻译回 EVENT_READABLE/WRITABLE
```

### 3.4 事件循环处理 (src/msg/async/Event.cc:421-505)

```
process_events():
  1. 基于最近的定时事件计算超时
  2. 调用 driver->event_wait() 获取触发的文件事件
  3. 对每个触发事件，调用 read_cb 和/或 write_cb
  4. 通过 process_time_events() 处理过期的定时事件
  5. 排空 external_events (跨线程回调)
  6. 调用自定义 Poller::poll() 回调
```

---

## 四、ProtocolV2 线格式

### 4.1 连接建立序列

```
客户端                              服务端
  |                                   |
  |--- CEPH_BANNER_V2_PREFIX -------->|  (40 字节: "ceph v2\n")
  |--- payload_len (uint16) --------->|
  |--- supported_features (uint64) -->|
  |--- required_features (uint64) --->|
  |                                   |
  |<-- CEPH_BANNER_V2_PREFIX ---------|
  |<-- payload_len (uint16) ---------|
  |<-- supported_features (uint64) ---|
  |<-- required_features (uint64) ----|
  |                                   |
  |--- HelloFrame ------------------->|  (entity_type + peer_addr)
  |<-- HelloFrame --------------------|
  |                                   |
  |--- AuthRequestFrame ------------->|  (auth_method + preferred_modes + payload)
  |<-- AuthDoneFrame -----------------|  (global_id + con_mode + payload)
  |--- AuthSignatureFrame ----------->|  (预认证缓冲区的 HMAC-SHA256)
  |<-- AuthSignatureFrame ------------|  (服务端也签名)
  |                                   |
  |--- [CompressionRequestFrame] ---->|  (可选，如果支持 COMPRESSION)
  |<-- [CompressionDoneFrame] --------|
  |                                   |
  |--- ClientIdentFrame ------------->|  (addrs, target, gid, global_seq, features)
  |<-- ServerIdentFrame --------------|  (addrs, gid, global_seq, features)
  |                                   |
  |=== READY: 消息交换 ===============|
```

### 4.2 帧标签 (src/msg/async/ProtocolV2.cc:1185-1216)

| 标签 | 说明 |
|------|------|
| `Tag::MESSAGE` | 正常消息载荷 |
| `Tag::HELLO` | Hello 交换 |
| `Tag::AUTH_REQUEST` | 认证发起 |
| `Tag::AUTH_DONE` | 认证完成 |
| `Tag::AUTH_SIGNATURE` | 预认证 HMAC 签名 |
| `Tag::CLIENT_IDENT` | 客户端标识 |
| `Tag::SERVER_IDENT` | 服务端标识 |
| `Tag::SESSION_RECONNECT` | 重连请求 |
| `Tag::KEEPALIVE2` | 带时间戳的保活 |
| `Tag::ACK` | 消息确认 (序列号) |
| `Tag::COMPRESSION_REQUEST` | 压缩协商 |

### 4.3 认证流程

1. **AuthRequest** (line 1758): 客户端调用 `auth_client->get_auth_request()` 获取初始认证 blob
2. **AuthDone** (line 1855): 服务端回复 global_id, con_mode, session_key
3. **Crypto setup** (line 1888): `session_stream_handlers = create_handler_pair()` — 创建 AES-GCM 或其他 cipher 处理器
4. **AuthSignature** (line 1893-1898): 双方交换预认证缓冲区的 HMAC-SHA256 签名以验证握手
5. **Pre-auth buffer cleared** (line 1896-1897): `pre_auth.enabled = false`，缓冲区清除

### 4.4 消息帧化 (src/msg/async/ProtocolV2.cc:528-591)

```
write_message(msg):
  1. out_seq++ (line 531)
  2. 构建 ceph_msg_header2 (line 541): seq, tid, type, priority, version, data_off, ack_seq, flags
  3. MessageFrame::Encode(header2, payload, middle, data) (line 549)
  4. 通过 append_frame() 追加到 outgoing_bl (line 554)
     → tx_frame_asm.get_buffer() 加密帧
  5. 调用 _try_send() 写入 socket
```

### 4.5 消息接收 (src/msg/async/ProtocolV2.cc:1402-1576)

```
handle_message():
  1. 从 segments 解码 MessageFrame (line 1407)
  2. 转换 ceph_msg_header2 → ceph_msg_header + ceph_msg_footer (lines 1423-1437)
  3. 调用 decode_message() 实例化具体 Message 子类 (line 1439)
  4. 验证序列号 (lines 1469-1487)
  5. 快速分发路径:
     ms_fast_preprocess() → ms_can_fast_dispatch() → dispatch_queue->fast_dispatch()
  6. 普通分发路径:
     dispatch_queue->enqueue() (line 1564)
  7. in_seq++ (line 1506)
  8. 如果不是 lossy 连接，发送 ACK (lines 1513-1516)
```

---

## 五、消息分发

### 5.1 DispatchQueue (src/msg/DispatchQueue.h/cc)

**数据结构**:
- `mqueue`: `PrioritizedQueue<QueueItem, uint64_t>` — 基于优先级的消息队列
- `marrival`: `multiset<double>` — 追踪消息到达时间用于年龄计算
- `dispatch_throttler`: `Throttle` — 防止分发积压

**两个线程**:
- **DispatchThread** (`ms_dispatch`): 处理主优先级队列
- **LocalDeliveryThread** (`ms_local`): 处理本地/回环消息

**QueueItem 类型**:
- 消息项 (`type == -1`): 实际要分发的消息
- 代码项 (`type != -1`): 连接生命周期事件
  - `D_CONNECT` (1): 连接建立
  - `D_ACCEPT` (2): 连接接受
  - `D_BAD_REMOTE_RESET` (3): 远端重置
  - `D_BAD_RESET` (4): 本地重置
  - `D_CONN_REFUSED` (5): 连接拒绝

### 5.2 两条分发路径 (src/msg/Dispatcher.h)

| 路径 | 用途 | 方法 |
|------|------|------|
| **快速分发** | 低延迟消息 (OSD 心跳、PG 通知) | `ms_fast_dispatch2()` — 在事件线程内联调用 |
| **普通分发** | 所有其他消息 | `ms_dispatch2()` — 排队到 DispatchQueue |

**Messenger 分发迭代** (Messenger.h:744-760):
```cpp
for each dispatcher in priority order:
    result = dispatcher->ms_dispatch2(m)
    if HANDLED: return
    if ACKNOWLEDGED: remember, continue
```

---

## 六、完整的发送/接收流程

### 6.1 发送流程

```
应用代码
  │
  ▼
Messenger::send_to(msg, type, addrs)
  │
  ▼
AsyncMessenger::connect_to()
  ├─ 如果本地: 返回 local_connection
  ├─ 在 conns 映射中查找现有连接
  └─ 如果未找到: create_connect()
       ├─ 从 NetworkStack 获取 Worker (负载最少)
       ├─ 创建 AsyncConnection + ProtocolV2
       └─ conn->connect(addrs, type, target)
  │
  ▼
AsyncConnection::send_message(msg)
  ├─ 设置 header.src, m->set_connection(this)
  ├─ 如果回环: dispatch_queue->local_delivery()
  └─ 否则: protocol->send_message(m)
  │
  ▼
ProtocolV2::send_message(m)
  ├─ 可选准备 (编码) 消息
  ├─ 锁定 write_lock
  ├─ 排队到 out_queue[priority]
  └─ 如果 can_write 且 !write_in_progress:
       dispatch_event_external(write_handler)
  │
  ▼
ProtocolV2::write_event()
  ├─ 循环:
  │    _get_next_outgoing()
  │    如果未准备: prepare_send_message()
  │    write_message(m)
  │      ├─ out_seq++
  │      ├─ MessageFrame::Encode()
  │      ├─ append_frame() → tx_frame_asm 加密
  │      └─ _try_send() → cs.send()
  └─ 如果 ack_left > 0: 发送 AckFrame
  │
  ▼
ConnectedSocket::send()
  └─ ::sendmsg() 带 MSG_MORE (如果 more=true)
```

### 6.2 接收流程

```
epoll_wait() 检测到可读 fd
  │
  ▼
EpollDriver::event_wait()
  │
  ▼
EventCenter::process_events()
  └─ 对每个触发事件:
       event->read_cb->do_request(fd)
       = AsyncConnection::process()
  │
  ▼
AsyncConnection::process()
  ├─ 更新 last_active 时间戳
  └─ 根据状态:
       STATE_CONNECTION_ESTABLISHED → protocol->read_event()
  │
  ▼
ProtocolV2::read_event()
  └─ READY → read_frame()
  │
  ▼
ProtocolV2::read_frame()
  ├─ READ(preamble_len, handle_read_frame_preamble_main)
  │    (异步读取带延续)
  │
  ▼
handle_read_frame_preamble_main()
  ├─ rx_frame_asm.disassemble_preamble()
  ├─ 如果 Tag::MESSAGE: 转到 THROTTLE_MESSAGE
  └─ 否则: read_frame_segment()
  │
  ▼
throttle_message → throttle_bytes → throttle_dispatch_queue
  ├─ 如果节流器已满则等待 (带重试定时器)
  └─ THROTTLE_DONE → read_frame_segment()
  │
  ▼
read_frame_segment()
  ├─ 读取每个 segment (payload, middle, data)
  ├─ 读取 epilogue (auth tag)
  └─ rx_frame_asm.disassemble_segments()  // 解密 + 验证
  │
  ▼
handle_read_frame_dispatch()
  └─ 根据 next_tag:
       Tag::MESSAGE → handle_message()
       Tag::ACK → handle_message_ack()
       Tag::KEEPALIVE2 → handle_keepalive2()
  │
  ▼
handle_message()
  ├─ MessageFrame::Decode()
  ├─ decode_message() → 具体 Message*
  ├─ 验证 seq 号
  ├─ ms_fast_preprocess(message)
  ├─ 如果可以快速分发:
  │    dispatch_queue->fast_dispatch(message)  [内联]
  └─ 否则:
       dispatch_queue->enqueue(message, priority, conn_id)
       → 唤醒 dispatch_thread
  │
  ▼
DispatchQueue::entry()
  ├─ 从 mqueue 出队 (最高优先级优先)
  └─ ms_deliver_dispatch(m)
       └─ 按优先级迭代 dispatchers:
            dispatcher->ms_dispatch2(m)
```

---

## 七、RDMA 和 DPDK 支持

### 7.1 传输选择 (src/msg/async/Stack.cc:63-77)

```cpp
if (t == "posix")
    stack.reset(new PosixNetworkStack(c));
#ifdef HAVE_RDMA
else if (t == "rdma")
    stack.reset(new RDMAStack(c));
#endif
#ifdef HAVE_DPDK
else if (t == "dpdk")
    stack.reset(new DPDKStack(c));
#endif
```

### 7.2 传输对比

| 特性 | Posix | RDMA | DPDK |
|------|-------|------|------|
| **文件** | PosixStack.h/cc | rdma/RDMAStack.h/cc | dpdk/DPDKStack.h/cc |
| **事件驱动** | EpollDriver | EpollDriver (完成事件) | DPDKDriver (轮询) |
| **监听表** | 共享 (内核) | 每 Worker | 每 Worker |
| **Worker 数** | 1 (或 ms_async_op_threads) | ms_async_op_threads | ms_async_op_threads |

**RDMA**: 使用 Verbs API 实现零拷贝、内核旁路 RDMA (InfiniBand/RoCE)。每个 Worker 管理自己的队列对 (QPs) 和完成队列。

**DPDK**: 使用 DPDK 的用户态网络栈和自己的事件驱动 (`EventDPDK.h`)。完全绕过内核以实现最大吞吐量。

---

## 八、关键代码位置

| 功能 | 文件 | 行号 |
|------|------|------|
| Messenger 抽象接口 | src/msg/Messenger.h | 96-864 |
| AsyncMessenger 类 | src/msg/async/AsyncMessenger.h | 96-450 |
| AsyncMessenger::send_to | src/msg/async/AsyncMessenger.cc | 912-949 |
| AsyncConnection 类 | src/msg/async/AsyncConnection.h | 56-256 |
| 连接状态枚举 | src/msg/async/AsyncConnection.h | 156-163 |
| AsyncConnection::process() | src/msg/async/AsyncConnection.cc | 384-499 |
| ProtocolV2 类 | src/msg/async/ProtocolV2.h | 14-283 |
| ProtocolV2 状态枚举 | src/msg/async/ProtocolV2.h | 16-43 |
| ProtocolV2::read_event() | src/msg/async/ProtocolV2.cc | 486-511 |
| ProtocolV2::write_event() | src/msg/async/ProtocolV2.cc | 645-768 |
| ProtocolV2::handle_message() | src/msg/async/ProtocolV2.cc | 1402-1576 |
| ProtocolV2::_fault() | src/msg/async/ProtocolV2.cc | 322-420 |
| ProtocolV2 认证流 | src/msg/async/ProtocolV2.cc | 1758-1924 (客户端), 2242-2395 (服务端) |
| EventCenter | src/msg/async/Event.h | 92-264 |
| EventCenter::process_events() | src/msg/async/Event.cc | 421-505 |
| EpollDriver | src/msg/async/EventEpoll.cc | 28-143 |
| NetworkStack | src/msg/async/Stack.h | 356-412 |
| DispatchQueue | src/msg/DispatchQueue.h | 41-241 |
| DispatchQueue::entry() | src/msg/DispatchQueue.cc | 158-216 |
| Dispatcher 接口 | src/msg/Dispatcher.h | 33-257 |
| Message 基类 | src/msg/Message.h | 263-556 |

---

## 九、常见问题与陷阱

### Q1: 快速分发和普通分发的区别？

**A**:
- **快速分发**: 在事件线程中内联调用，零排队延迟。用于 OSD 心跳、PG 通知等对延迟极其敏感的消息。代价是占用事件线程时间。
- **普通分发**: 排队到 DispatchQueue，由专用分发线程处理。用于大多数消息，避免阻塞事件循环。

### Q2: ProtocolV2 的加密是如何工作的？

**A**: 在认证握手期间，双方协商加密算法并建立 `session_stream_handlers` (通常是 AES-GCM)。之后所有帧通过 `tx_frame_asm.get_buffer()` 加密，通过 `rx_frame_asm.disassemble_segments()` 解密和验证。加密在帧级别进行，不是消息级别。

### Q3: 为什么使用边沿触发 (EPOLLET) 而不是水平触发？

**A**: 边沿触发只在状态变化时通知一次，避免了水平触发可能的重复通知开销。但要求代码必须读取/写入直到 EAGAIN，AsyncMessenger 通过循环处理确保了这一点。

### 参考文献

1. **消息层开发者文档**: https://docs.ceph.com/en/latest/dev/msg-layer/
2. **ProtocolV2 规范**: `src/msg/async/ProtocolV2.h`
3. **源码**: `src/msg/async/AsyncMessenger.cc`, `src/msg/async/AsyncConnection.cc`
4. **epoll 手册**: `man 7 epoll"
