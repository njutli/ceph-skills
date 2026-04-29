---
name: objecter-osd-client
description: Ceph Objecter (OSD客户端) 专家。当用户询问对象到OSD映射、op提交流程、OSDMap更新处理、ObjectCacher写回、Striper条带化计算、op重试逻辑时调用此技能。
---

# Objecter OSD 客户端

## 〇、为什么需要了解 Objecter？

Objecter 是 Ceph 客户端与 OSD 集群通信的核心引擎。所有高层库 (librados, libcephfs, librbd) 最终都通过 Objecter 发送 I/O：

```
应用层
  │
  ├── librados ──────────────────────┐
  ├── libcephfs → ObjectCacher ──────┤
  ├── librbd → ObjectCacher ─────────┤
  │                                  ▼
  │                              Objecter
  │                                  │
  │                              OSDMap + CRUSH
  │                                  │
  │                              MOSDOp 消息
  │                                  ▼
  └────────────────────────────── OSD 集群
```

理解 Objecter 有助于：
1. **I/O 路径调优**: 理解 op 提交、节流、超时机制
2. **读取策略**: 理解平衡读取、本地化读取
3. **缓存行为**: 理解 ObjectCacher 的写回机制
4. **故障排查**: 理解 op 重试、重定向、超时处理

---

## 一、对象到 OSD 的映射

### 1.1 映射管线

```
对象名 + Pool + Namespace
  │
  ▼ (object_locator_to_pg)
pg_t (Placement Group ID)
  │
  ▼ (pg_to_up_acting_osds — CRUSH)
up 集合 / acting 集合 / acting_primary
  │
  ▼ (读取策略选择)
目标 OSD
```

### 1.2 详细步骤

**Step 1: 对象定位器到 PG ID** (src/osdc/Objecter.cc:3018)

```cpp
osdmap->object_locator_to_pg(t->target_oid, t->target_oloc, pgid);
```

使用对象名 + pool + namespace 计算 PG ID。通过哈希对象名并应用 pool 的 `pg_num` 和 `pg_num_mask` (通过 `ceph_stable_mod`)。

**Step 2: PG 到 OSD 集合** (src/osdc/Objecter.cc:3040-3047)

```cpp
osdmap->pg_to_up_acting_osds(actual_pgid, &up, &up_primary,
                             &acting, &acting_primary);
```

OSDMap 内部运行 CRUSH 算法确定：
- **`up` 集合**: 根据当前 CRUSH 映射**应该**持有 PG 的 OSD
- **`acting` 集合**: **实际**负责 PG 的 OSD (恢复/Backfill 期间可能不同)
- **`up_primary` / `acting_primary`**: 每个集合中的主 OSD

**Step 3: 选择目标 OSD** (src/osdc/Objecter.cc:3162-3202)

| 操作 | 目标 OSD | 说明 |
|------|---------|------|
| **写入** | `acting_primary` | 始终写入主 OSD |
| **读取 (默认)** | `acting_primary` | 从主 OSD 读取 |
| **平衡读取** (`CEPH_OSD_FLAG_BALANCE_READS`) | 从 `acting` 集合随机选择 | 分散读负载 |
| **本地化读取** (`CEPH_OSD_FLAG_LOCALIZE_READS`) | 选择离客户端 CRUSH 位置最近的 OSD | 减少网络延迟 |

```cpp
// 平衡读取 (line 3169)
int p = rand() % t->acting.size();
t->osd = t->acting[p];

// 本地化读取 (line 3181)
osdmap->crush->get_common_ancestor_distance(client_loc, osd);
```

**Step 4: 分层缓存重定向** (src/osdc/Objecter.cc:2997-3009)

如果未设置 `CEPH_OSD_FLAG_IGNORE_OVERLAY`：
- 读取重定向到 `pi->read_tier` (缓存池)
- 写入重定向到 `pi->write_tier` (缓存池)

---

## 二、Op 提交流程

### 2.1 四个阶段

```
Stage 1: op_submit()              — 入口，尝试拆分 (EC/RMW)
  │
  ▼
Stage 2: _op_submit_with_budget() — 获取节流预算，设置超时
  │
  ▼
Stage 3: _op_submit()             — 计算目标，分配会话
  │
  ▼
Stage 4: _send_op()               — 网络传输
```

### 2.2 Stage 1: op_submit() (src/osdc/Objecter.cc:2362-2375)

```cpp
void Objecter::op_submit(Op *op, ceph_tid_t *ptid, int *ctx_budget)
{
    // 尝试拆分 op (用于 EC/RMW 操作)
    bool was_split = SplitOp::create(op, *this, rl, ptid, ctx_budget, cct);
    if (!was_split) {
        _op_submit_with_budget(op, rl, ptid, ctx_budget);
    }
}
```

### 2.3 Stage 2: _op_submit_with_budget() (src/osdc/Objecter.cc:2377-2409)

```cpp
void Objecter::_op_submit_with_budget(Op *op, ...)
{
    // 节流: 如果 in-flight op 的字节数/数量超过限制则阻塞
    if (!op->ctx_budgeted || (ctx_budget && (*ctx_budget == -1))) {
        int op_budget = _take_op_budget(op, sul);
    }
    // 设置超时回调
    if (osd_timeout > timespan(0)) {
        op->ontimeout = timer.add_event(osd_timeout,
            [this, tid]() { op_cancel(tid, -ETIMEDOUT); });
    }
    _op_submit(op, sul, ptid);
}
```

### 2.4 Stage 3: _op_submit() (src/osdc/Objecter.cc:2503-2615)

```cpp
void Objecter::_op_submit(Op *op, ...)
{
    // 1. 计算目标 OSD
    int r = _calc_target(&op->target);

    // 2. 获取会话 (可能需要升级锁)
    r = _get_session(op->target.osd, &s, sul);

    // 3. 统计 op (统计信息，性能计数器)
    _send_op_account(op);

    // 4. 分配到会话并发送
    _session_op_assign(s, op);

    // 5. 发送或排队等待映射检查
    if (need_send) {
        _send_op(op);
    } else {
        _send_op_map_check(op);  // pool 不存在时排队
    }
}
```

### 2.5 Stage 4: _send_op() (src/osdc/Objecter.cc:3454-3525)

```cpp
void Objecter::_send_op(Op *op)
{
    // 检查 backoff (OSD 可能要求我们暂停)
    auto p = op->session->backoffs.find(op->target.actual_pgid);
    if (p != op->session->backoffs.end()) {
        // 如果 backoff 适用于此对象，排队并返回
        return;
    }

    // 准备 MOSDOp 消息
    MOSDOp *m = _prepare_osd_op(op);

    // 通过连接发送
    op->session->con->send_message(m);
}
```

---

## 三、OSDMap 更新处理

### 3.1 入口: handle_osd_map() (src/osdc/Objecter.cc:1210-1420)

```cpp
void Objecter::handle_osd_map(MOSDMap *m)
{
    // 1. 验证 fsid
    // 2. 保存当前 pause/full 状态
    // 3. 应用增量或全量映射
    for (epoch_t e = osdmap->get_epoch() + 1; e <= m->get_last(); e++) {
        if (m->incremental_maps.count(e)) {
            osdmap->apply_incremental(...);  // 应用增量
        } else if (m->maps.count(e)) {
            osdmap = std::make_unique<OSDMap>();
            new_osdmap->decode(...);         // 应用全量映射
        } else {
            _maybe_request_map();  // 缺少 epoch，请求它
            break;
        }

        // 4. 每个 epoch 后，扫描所有未完成的请求
        _scan_requests(...);
    }

    // 5. 重新计算需要重新发送的 op 的目标
    for (auto p = need_resend.begin(); ...) {
        _calc_target(&op->target);
    }

    // 6. 实际重新发送 op
    for (auto p = need_resend.begin(); ...) {
        _session_op_assign(s, op);
        _send_op(op);
    }
}
```

### 3.2 _scan_requests() — 逐会话请求审查 (src/osdc/Objecter.cc:1081-1208)

对每个会话扫描三类请求：

| 类型 | 行号 | 处理 |
|------|------|------|
| **Linger ops** | 1098-1137 | 重新计算目标，处理 pool DNE/EIO |
| **Regular ops** | 1140-1168 | 重新计算目标，映射变化时移到 `need_resend` |
| **Command ops** | 1171-1198 | 重新计算命令目标 |

```cpp
int r = _calc_target(&op->target);
switch(r) {
case RECALC_OP_TARGET_NO_ACTION:
    break;
case RECALC_OP_TARGET_NEED_RESEND:
    _session_op_remove(op->session, op);
    need_resend[op->tid] = op;
    break;
case RECALC_OP_TARGET_POOL_DNE:
    _check_op_pool_dne(op, &sl);
    break;
case RECALC_OP_TARGET_POOL_EIO:
    _check_op_pool_eio(op, &sl);
    break;
}
```

---

## 四、In-Flight Op 追踪和重试

### 4.1 追踪结构

**每会话 Op 映射** (src/osdc/Objecter.h:2484-2517):
```cpp
struct OSDSession {
    std::map<ceph_tid_t, Op*> ops;           // 常规 op
    std::map<uint64_t, LingerOp*> linger_ops;  // 监视 op
    std::map<ceph_tid_t, CommandOp*> command_ops;  // 命令 op
};
```

**Op 状态** (src/osdc/Objecter.h:1989-2156):
```cpp
struct Op : public RefCountedObject {
    OSDSession *session = nullptr;
    ceph_tid_t tid = 0;
    int attempts = 0;           // 重试计数
    bool should_resend = true;  // 映射变化时是否重新发送
};
```

### 4.2 回复处理 (src/osdc/Objecter.cc:3654-3866)

```cpp
void Objecter::handle_osd_op_reply(MOSDOpReply *m)
{
    ceph_tid_t tid = m->get_tid();
    Op *op = s->ops.find(tid)->second;

    // 检查重试尝试匹配
    if (m->get_retry_attempt() >= 0) {
        if (m->get_retry_attempt() != (op->attempts - 1)) {
            return;  // 旧尝试的陈旧回复，忽略
        }
    }

    // 处理重定向
    if (m->is_redirect_reply()) {
        op->target.target_oloc = m->get_redirect();
        _op_submit(op, sul, NULL);  // 重新提交到新目标
        return;
    }

    // 处理 -EAGAIN (副本读取弹回)
    if (rc == -EAGAIN && !op->target.force_shard) {
        op->target.flags &= ~(CEPH_OSD_FLAG_BALANCE_READS | CEPH_OSD_FLAG_LOCALIZE_READS);
        _op_submit(op, sul, NULL);  // 重新提交到主 OSD
        return;
    }

    // 完成并清理
    _finish_op(op, 0);
    Op::complete(std::move(onfinish), osdcode(rc), rc, ...);
}
```

### 4.3 重试逻辑总结

| 触发条件 | 处理 |
|---------|------|
| **OSDMap 变更** | `_scan_requests()` 将 op 移到 `need_resend`，重新发送 |
| **重定向回复** | 更新目标定位器，重新提交 |
| **-EAGAIN (副本读取)** | 去除读取平衡标志，重新提交到主 OSD |
| **Pool 不存在** | 等待最新映射，到达后重试 |
| **超时** | `op_cancel(tid, -ETIMEDOUT)` 在 `osd_timeout` 秒后触发 |
| **Backoff** | OSD 发送 `MOSDBackoff` 后，范围内 op 排队 |

---

## 五、ObjectCacher 写回机制

### 5.1 BufferHead 状态机 (src/osdc/ObjectCacher.h:107-116)

| 状态 | 值 | 说明 |
|------|-----|------|
| `STATE_MISSING` | 0 | 数据缺失 |
| `STATE_CLEAN` | 1 | 干净，已同步到 OSD |
| `STATE_ZERO` | 2 | 干净的零 (EOF 之外) |
| `STATE_DIRTY` | 3 | 脏，需要写回 |
| `STATE_RX` | 4 | 读取进行中 |
| `STATE_TX` | 5 | 写入进行中 (传输中) |
| `STATE_ERROR` | 6 | 读取错误 |

### 5.2 写入路径: writex() (src/osdc/ObjectCacher.cc:1749-1851)

```
1. 对每个 ObjectExtent，获取/创建 Object，调用 o->map_write()
2. 将用户数据复制到 BufferHead 的 bufferlist
3. 通过 mark_dirty(bh) 标记 BH 为 DIRTY
4. 调用 _wait_for_write() 处理背压
```

### 5.3 写回触发

**A. 脏数据阈值背压** (`_maybe_wait_for_writeback()`, lines 1873-1919):
```cpp
while (get_stat_dirty() + get_stat_tx() > max_dirty + get_stat_dirty_waiting()) {
    stat_dirty_waiting += len;
    stat_cond.wait(l);  // 阻塞直到 flusher 腾出空间
    stat_dirty_waiting -= len;
}
```

**B. Flusher 线程** (`flusher_entry()`, lines 1969-2057):

专用后台线程持续运行：
```
1. 如果 stat_dirty + stat_dirty_waiting > target_dirty: 刷新多余脏数据
2. 如果脏 BH 数超过 target_dirty_bh: 刷新多余 BH
3. 否则: 扫描脏 LRU 尾部，刷新老化 BH (last_write <= cutoff - max_dirty_age)
```

### 5.4 实际写回

**单个 BH 写回** (lines 1141-1179):
```cpp
void ObjectCacher::bh_write(BufferHead *bh, ...)
{
    ceph_tid_t tid = writeback_handler.write(bh->ob->get_oid(), ...);
    mark_tx(bh);  // 转换 DIRTY → TX
}
```

**分散/合并写回** (lines 1088-1139):

启用 `scattered_write` 时，同一对象上相邻的脏 BH 合并为单个 I/O：
```cpp
void ObjectCacher::bh_write_scattered(list<BufferHead*>& blist)
{
    // 收集所有 BH 数据到 io_vec
    ceph_tid_t tid = writeback_handler.write(ob->get_oid(), ..., io_vec, ...);
    for (BufferHead *bh : blist) {
        mark_tx(bh);
    }
}
```

### 5.5 写回完成 (lines 1181-1287)

```
写入完成:
  1. 找到匹配的 TX BufferHead (按 last_write_tid)
  2. 标记为 CLEAN (错误时重新标记 DIRTY)
  3. 更新 ob->last_commit_tid
  4. 如果整个 ObjectSet 现在干净，调用 flush_set_callback
  5. 唤醒 waitfor_commit[tid] 上的等待者
```

---

## 六、Striper 条带化计算

### 6.1 核心算法: file_to_extents() (src/osdc/Striper.cc:182-271)

条带器使用 `file_layout_t` 参数将连续文件字节范围转换为一组对象范围：
- `object_size`: 每个对象的总大小
- `stripe_unit`: 对象内每个条带段的大小
- `stripe_count`: 条带集合中的对象数

**每个字节位置的计算** (lines 212-220):
```cpp
uint64_t blockno = cur / su;                    // 哪个条带单元块
uint64_t stripeno = blockno / stripe_count;     // 哪个水平条带 (Y)
uint64_t stripepos = blockno % stripe_count;    // 集合中的哪个对象 (X)
uint64_t objectsetno = stripeno / stripes_per_object;  // 哪个对象集合
uint64_t objectno = objectsetno * stripe_count + stripepos;  // 最终对象 ID
```

**对象内的偏移** (lines 223-227):
```cpp
uint64_t block_start = (stripeno % stripes_per_object) * su;
uint64_t block_off = cur % su;
uint64_t x_offset = block_start + block_off;
```

### 6.2 反向映射: extent_to_file() (lines 273-308)

```cpp
uint64_t stripepos = objectno % stripe_count;
uint64_t objectsetno = objectno / stripe_count;
uint64_t stripeno = off / su + objectsetno * stripes_per_object;
uint64_t blockno = stripeno * stripe_count + stripepos;
uint64_t extent_off = blockno * su + off_in_block;
```

### 6.3 StripedReadResult (lines 386-539)

将来自多个对象的部分读取结果组装回连续缓冲区：
- `add_partial_result()`: 将对象数据映射到缓冲区偏移
- `add_partial_sparse_result()`: 处理稀疏读取 (空洞)
- `assemble_result()`: 连接所有部分结果，零填充间隙

---

## 七、架构总结

```
┌─────────────────────────────────────────────────┐
│                   应用层                          │
│  librados  │  libcephfs  │  librbd               │
├─────────────────────────────────────────────────┤
│                  ObjectCacher                    │
│  BufferHead 状态机 │ 写回 │ 脏数据背压 │ Flusher  │
├─────────────────────────────────────────────────┤
│                     Striper                      │
│  file_to_extents │ extent_to_file │ 读取组装      │
├─────────────────────────────────────────────────┤
│                    Objecter                      │
│  对象→OSD 映射 │ Op 提交 │ OSDMap 更新 │ 重试     │
├─────────────────────────────────────────────────┤
│                  MOSDOp 消息                     │
│              (通过 AsyncMessenger)                │
├─────────────────────────────────────────────────┤
│                    OSD 集群                       │
└─────────────────────────────────────────────────┘
```

---

## 八、关键代码位置

| 功能 | 文件 | 行号 |
|------|------|------|
| 对象到 OSD 映射 | src/osdc/Objecter.cc | 2959-3214 |
| OSDMap PG 映射 | src/osdc/Objecter.cc | 3040-3047 |
| Op 提交入口 | src/osdc/Objecter.cc | 2362-2375 |
| 预算/节流 | src/osdc/Objecter.cc | 2377-2409 |
| Op 分发 | src/osdc/Objecter.cc | 2503-2615 |
| 网络发送 | src/osdc/Objecter.cc | 3454-3525 |
| OSDMap 处理 | src/osdc/Objecter.cc | 1210-1420 |
| 请求重新扫描 | src/osdc/Objecter.cc | 1081-1208 |
| 回复处理 | src/osdc/Objecter.cc | 3654-3866 |
| Op 数据结构 | src/osdc/Objecter.h | 1989-2156 |
| 会话数据结构 | src/osdc/Objecter.h | 2484-2517 |
| ObjectCacher 写入 | src/osdc/ObjectCacher.cc | 1749-1851 |
| BH 写回 | src/osdc/ObjectCacher.cc | 1141-1179 |
| 分散写回 | src/osdc/ObjectCacher.cc | 1088-1139 |
| 写回完成 | src/osdc/ObjectCacher.cc | 1181-1287 |
| Flusher 线程 | src/osdc/ObjectCacher.cc | 1969-2057 |
| 脏数据背压 | src/osdc/ObjectCacher.cc | 1873-1919 |
| Striper 核心 | src/osdc/Striper.cc | 182-271 |
| 反向条带化 | src/osdc/Striper.cc | 273-308 |
| 读取结果组装 | src/osdc/Striper.cc | 386-539 |

---

## 九、常见问题与陷阱

### Q1: 平衡读取和本地化读取的区别？

**A**:
- **平衡读取**: 从 acting 集合中随机选择 OSD，分散读负载。适用于读取密集型工作负载。
- **本地化读取**: 选择离客户端 CRUSH 位置最近的 OSD，减少网络跳数。适用于跨机架/数据中心的部署。

### Q2: ObjectCacher 的"分散写回"是什么？

**A**: 当同一对象上有多个相邻的脏 BufferHead 时，分散写回将它们合并为单个 I/O 操作。这减少了 OSD 上的小写入数量，提高了写入吞吐量。通过 `scattered_write` 选项启用。

### Q3: Backoff 机制的作用？

**A**: 当 OSD 过载时，可以发送 `MOSDBackoff` 消息要求客户端暂停发送特定 PG 范围内的 op。Objecter 收到 backoff 后，将匹配的 op 排队而不立即发送，给 OSD 喘息的空间。这是一种细粒度的流控机制。

### 相关技能

- [CRUSH 算法](../crush/crush-algorithm/SKILL.md) — Objecter 使用 CRUSH 计算对象到 OSD 的映射
- [AsyncMessenger](../messenger/async-messenger/SKILL.md) — op 通过 Messenger 发送到目标 OSD

### 参考文献

1. **Objecter 源码**: `src/osdc/Objecter.cc`, `src/osdc/Objecter.h`
2. **RADOS API 文档**: https://docs.ceph.com/en/latest/rados/api/librados-intro/
3. **CRUSH 论文**: https://ceph.com/assets/pdfs/weil-crush-sc06.pdf
4. **Ceph 整体架构**: https://docs.ceph.com/en/latest/architecture/"
