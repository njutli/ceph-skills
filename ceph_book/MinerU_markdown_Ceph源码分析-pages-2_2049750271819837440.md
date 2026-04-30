# 6.4 读写流程代码分析

在介绍了上述的数据结构和基本的流程之后，下面将从服务端接收到消息开始，分三个阶段具体分析读写的过程。

# 6.4.1 阶段1：接收请求

读写请求都是从OSD::ms_fast_dispatch开始，它是接收读写消息message的入口。下面从这里开始读写操作的分析。本阶段所有的函数是被网络模块的接收线程调用，所以理论上应该尽可能的简单，处理完成后交给后面的OSD模块的OpWQ工作队列来处理。

# 1. ms_fastishly

```cpp
void OSD::ms_fast_dispatch(Message *m) 
```

函数ms_fast_dispatch为OSD注册了网络模块的回调函数，其被网络的接收线程调用，具体实现过程如下：

1）首先检查 service，如果已经停止了，就直接返回。

2）调用函数 op_tracking.create_request 把 Message 消息转换为 OpRequest 类型，数据结构 OpRequest 包装了 Message，并添加了一些其他信息。

3）获取nextmap（也就是最新的osdmap）和session，类Session保存了一个Connection的相关信息。

4）调用函数update_waiting_for_pg来更新session里保存的OSDMap信息。

5）把请求加入waiting_on_map的列表里。

6）调用函数dispatch_session_waiting处理，它循环调用函数dispatch_op_fast处理请求。

7）如果session->waiting_on_map不为空，说明该session里还有等待osdmap的请求，把该session加入到session Waiting_for_map队列里。

# 2. dispatch_op_fast

```cpp
bool OSD::dispatch_op_fast(OpRequestRef& op, OsDMapRef& osdmap) 
```

该函数检查OSD目前的epoch是否最新：

1）检查变量 is_stopping 如果 true，就直接返回 true 值。

2）调用函数 op_required_epoch(op)，从 OpRequest 中获取 msg 带的 epoch，进行比较，如果该值大于 OSD 最新的 epoch，则调用函数 osdmmap Subscribe 更新 epoch，返回 flase 值。

3）否则，根据消息类型，调用相应的消息处理函数，本章只关注处理函数 handle_op 相关的流程。

# 3. handle_op

void OSD::handle_op(OpRequestRef& op, OsDMapRef& osdmap) 

该函数处理OSD相关的操作，其处理流程如下：

1）首先调用op_is_discardable，检查该OP是否可以丢弃。

2）构建share_map结构体，获取client_session，从client_session获取last_sent_epoch，调用函数service.should_share_map来设置share_map.should_send标志，该函数用于检查是否需要通知对方更新epoch值。这里和dispatch_op_fast的处理区别是，上次是更新自己，这里是通知对方更新。需要注意是，client和OSD的epoch不一致，并不影响读写，只要epoch的变化不影响本次读写PG的OSD list变化。

3）从消息里获取_pgid，从_pg_id里获取pool。

4）调用函数osdmap->raw_pg_to_pg，最终调用pg_pool_t::raw_pg_to_pg函数，对PG做了调整。

5）调用osdmap->get_primary_shard函数，获取该PG的主OSD。

6）调用函数get_pg_or_queue_for_pg，通过pgid获取PG类指针。如果获取成功，就调用函数enqueue_op处理请求。

7）如果PG类的指针没有获取成功，做一些错误检查：

a）seng map 为空，client 需要重试。

b）客户端的osdmap里没有当前的pool。

c）当前OSD的osdmap没有该pool，或者当前OSD不是该PG的主OSD。

总结，这个函数主要检查了消息的源端 epoch 是否需要 share，最主要的是获取读写请求相关的 PG 类后，下面就进入 PG 类的处理。

# 4.queue_op

void PG::queue_op(OpRequestRef& op) 

该函数的实现如下：

1）加 map_lock 锁，该锁保护 waiting_for_map 列表，判断 waiting_for_map 列表不为空，就把当前 OP 加入该列表，直接返回。waiting_for_map 列表不为空，说明有操作在等待 osdmap 的更新，说明当前 osdmap 不信任，不能继续当前的处理。

2）函数 op.must_wait_for_map 判断当前的 epoch 是否大于 OP 的 epoch，如果是，则必须加入 waiting_for_map 等待，等待更新 PG 当前的 epoch 值。

![image](https://cdn-mineru.openxlab.org.cn/result/2026-04-30/8e4d380f-113d-46f2-8e2b-b0978fb78f78/dc1b7cc32d942de8ca0627d24bc42750b7e1591e208001249e63118baed319a0.jpg)


这里的 osdmap 的 epoch 的判断，是一个 PG 层的 epoch 的判断。和前面的判断不在一个层次，这里是需要等待的。

3）最终，把请求加入OSD的op_wq处理队列里。

总结，这个函数做在PG类里，做PG层面的相关检查，如果ok，就加入OSD的op_wq工作队列里继续处理。

# 6.4.2 阶段2：OSD的op_wq处理

op_wq 是一个 ShardedWQ 类型的工作队列。以下操作都是在 op_wq 对应的线程池里调用做相应的处理。这里着重分析读写流程。

# 1. dequeue_op

void OSD::deque_op(PGRef pg, OpRequestRef op, ThreadPool::TPHandle &handle) 

1）检查如果 op->send_map_update 为 true，也就是如果需要更新 osdmap，就调用函数 service.share_map 更新源端的 osdmap 信息。在函数 OSD::handle_op 里，只在 op->send_map_update 里设置了是否需要 share_map 的标记，在这里才真正去发消息实现 share 操作。

2）检查如果 pg 正在删除，就把本请求丢弃，直接返回。

3）调用函数pg->do_request(op, handle)处理请求。

总之，本函数主要实现了使请求源端更新 osdmap 的操作，接下来在 PG 里调 do_request 来处理。

# 2. do_request

本函数进入ReplicatedPG类来处理：

void ReplicatedPG::do_request( RequestRef& op, ThreadPool::TPHandle &handle) 

处理过程如下：

1）调用函数 can_discard_request 检查 op 是否可以直接丢弃掉。

2）检查变量 flushes_in_progress 如果还有 flush 操作，把 op 加入 waiting_for_peered 队列里，直接返回。

3）如果PG还没有peered，调用函数can_handle_whileinactive检查pgfrontend能否处理该请求，如果可以，就调用pgfrontend->handle_message处理；否则加入waiting_for_peered队列，等待PG完成peering后再处理。

![image](https://cdn-mineru.openxlab.org.cn/result/2026-04-30/8e4d380f-113d-46f2-8e2b-b0978fb78f78/25d2b7e668d1db2c65e9579d865585c869897941daa99beee8d62b74aeab2670.jpg)


# 注意

PG处于inactive状态，pgfrontend只能处理MSG_OSD_PG_PULL类型的消息。这种情况可能是：本OSD可能已经不在该PG的acting osd列表中，但是可能在上一阶段该PG的OSD列表中，所以PG可能含有有效的对象数据，这些对象数据可以被该PG当前的主OSD拉取以修复当前PG的数据。

4）此时PG处于peered并且flushes_in_progress为0的状态下，检查pgfrontend能否处理该请求。pgfrontend可以处理数据恢复过程中的PULL和PUSH请求，以及主副本发的从副本的更新相关SUBOP类型的请求。

5）如果是CEPH_MSG_OSD_OP，检查该PG的状态，如果处于非active或者replay状态，则把请求添加到waiting_for.active等待队列。

6）检查如果该pool是cache pool，而该操作没有带CEPH_FEATURE_OSD_CACHE-PPOOL的feature标志，返回EOPNOTSUPP错误码。

7）根据消息的类型，调用相应的处理函数来处理

本函数开始进入ReplicatedPG层面来处理，主要检查当前PG的状态是否正常，是否

可以处理请求。

# 3. do_op

函数do_op数比较复杂，处理读写请求的状态检查和一些上下文的准备工作。其中大量的关于快照的处理，本章在遇到快照处理时只简单介绍一下。

```cpp
void ReplicatedPG::do_op(OpRequestRef& op) 
```

具体处理过程如下：

1）调用函数 $\mathrm{m->finish\_decode}$ ，把消息带的数据从bufferlist中解析出相关的字段。

2）调用osd->osd->init_op_flags初始化op->rmw_flags，函数init_op_flags根据flag来设置rmw_flags标志。

3）如果是读操作：

a）如果消息里带有CEPH_OSD_FLAG_BALANCE_READS（平衡读）或者CEPH_OSD_FLAG_LOCALIZE_READS（本地读）标志，表明主从副本都允许读。检查本OSD必须是该PG的primary或者replica之一。

b）如果没有上述标志，读操作只能读取主副本，本OSD必须是该PG的主OSD。

4）如果里面含有 includes_pg_op 操作，调用 pg_op.must_wait 检查该操作是否需要等待，如果需要等待，加入 waiting_for_all MISSING 队列；如果不需要等待，调用 do_pg_op 处理 PG 相关的操作。这里的 PG 操作，都是 CEPH_OSD_OP_PGLS 等类似的 PG 相关的操作，需要确保该 PG 上没有需要修复的对象，否则 ls 列出的对象就不准确。

5）调用函数op_has_sufficient_caps检查是否有相关的操作权限。

6）检查对象的名字是否超长。

7）检查操作的客是户端是否在黑名单（blacklist）中。

8）检查磁盘空间是否满。

9）检查如果是写操作，并且是snap，返回-EINVALID，快照不允许写操作。如果写操作带的数据大于osd_max_write_size(如果设置了)，直接返回-OSD_WRITETOOOBIG错误。

可以看到，以上完成基本的与操作相关的参数检查。

10）构建要访问对象的head对象（head对象和快照对象的概念可查看后面介绍快照

的章节）。

11）如果是顺序写，调用函数scrubber.writeBlocked_by_scrub检查：如果head对象正在进行scrub操作，就加入waiting_for.active队列，等待scrub操作完成后继续本次请求的处理。

12）检查head对象是否处于缺失状态（missing）需要恢复，调用函数wait_for_unwearable_object把当前请求加入相应的队列里等待恢复完成。

13）如果是顺序写，检查head对象是否is_degraded_or_backfilling_object，也就是正在恢复状态，需要调用wait_for_degraded_object加入相应的队列等待。

14）检查head对象的特殊情况：

a）检查队列objectsBlocked_on_degraded snap里如果保存的head对象，就需要等待。该队列里保存的head对象在rollback到某个版本的快照时，该版本的snap对象处于缺失状态，必须等待该snap对象恢复，从而完成rollback操作。因此该队列的head对象目前处于缺失状态。

b）队列objectsBlocked_on_snapshot_promotion里的对象表示head对象rollback到某个版本的快照时，该版本的快照对象在Cachepool层没有，需要到Data pool层获取。

如果head对象在上述的两个队列中，head对象都不能执行写操作，需要等待获取快照对象，完成rollback后才能写入。

可知，以上 $10\sim 14$ 步骤是构建并检查head对象的状态是否正常。

15）如果是顺序写操作，检查该对象是否在objectsBlocked_on_cache_full队列中，该队列中的对象因Cache pool层空间满而阻塞写操作。

![image](https://cdn-mineru.openxlab.org.cn/result/2026-04-30/8e4d380f-113d-46f2-8e2b-b0978fb78f78/7c8ea7414bf27c57fabb09a342a961fd0f60c037a19d532b49bbd15633e31731.jpg)


# 注意

当head对象被删除时，系统自动创建一个snapdir对象用来保存快照相关的信息。head对象和snapdir对象只能有一个存在，其都可以用来保存快照相关的信息。

16）检查该对象的snapdir对象（如果存在）是否处于missing状态。

17）检查snapdir对象是否可读，如果不能读，就调用函数wait_for_unreadable_object等待。

18）如果是写操作，调用函数 is_degraded_or_backfilling_object 检查 snapdir 对象是否缺失。

19）检查如果是CEPH_SNAPDIR类型的操作，只能是读操作。Snapdir对象只能读取。

（20）检查是否是客户端 replay 操作。

21）构建对象 oid，这才是实际要操作的对象，可能是 snap 对象也可能是 head 对象。

22）调用函数检查 maybe await.blocked_snapshot 是否被 block，检查该对象缓存的 ObjectContext 如果设置为 blocked 状态，该 object 有可能正在 flush，或者 copy（由于 Cache Tier），暂时不能写，需要等待。

23）调用函数 find_object_context 获取 object_context，如果获取成功，需要检查 oid 的状态。

24）如果hit_set不为空，就需要设置hit_set.hit_set和agent_state都是Cache tier的机制，hit_set记录cachepool中对象是否命中，暂时不深入分析。

25）如果 agent_state 不为空，就调用函数 agent_choose_mode 设置 agent 的状态，调用函数 maybe_handle_cache 来处理，如果可以处理，就返回。

26）获取 object locator，验证是否和 msg 里的相同。

27）检查该对象是否被阻塞。

28）获取 src_abc，也就是 src_oid 对应的 ObjectContext：同样的方法，对 src_oid 做各种状态检查，然后调用 find_object_context 函数获取 ObjectContext。

29）如果是操作对象 snapdir 对象，相关的操作就需要所有的 clone 对象，获取 clone 对象的 objectContext。对每一个 clone 对象，构建 objectContext，并把它加入的 src_obs 中。

30）创建 opContext。

31）调用executectx(ctx)。

总之，do_op主要检查相关对象的（head对象、snapdir对齐、src对象等）状态是否正常，并获取ObjectContext、OpContext相关的上下文信息。

# 4. get_object_context

本函数获取一个对象的 ObjectContext 信息。

ObjectContextRef ReplicatedPG::get_object_context(const hobject_t&soid，//soid要获取的对象

bool can_create, //是否允许创建新的 ObjectContext map<string, bufferlist> *attrs) //attrs 对象的属性

关键是从属性OI_ATTR中获取object_info_t信息。具体过程如下：

1）首先从LRU缓存object_contexts的中获取该对象的ObjectContext，如果获取成功，就直接返回结果。

2）如果从LRUcache里没有查找到：

a）如果参数attrs值不为空，就从attrs里获取OI_ATTR的属性值。

b）否则调用函数pgfrontend->objects_get_attr获取该对象的OI_ATTR属性值。如果获取失败，并且不允许创建，就直接返回ObjectContextRef()的空值。

3）如果成功获取OI_ATTR属性值，就从该属性值中decode后获取object_info_t的值。

4）调用get_snapshot_context获取SnapSetContext。

5）调用相关函数设置obc相关的参数，并返回obc。

# 5. get_snapshot_context

本函数获取对象的snapshot_context结构，其过程和函数get_object_context类似。具体实现如下：

1）首先从 LRU 缓存 snapshot_contexts 获取该对象的 snapshot_context，如果成功，直接返回结果。

2）如果不存在，并且 can_create，就调用 pgBackend->objects_get_attr 函数获取 SS_ATTR 属性。只有 head 对象或者 snapdir 对象保存有 SS_ATTR 属性，如果 head 对象不存在，就获取 snapdir 对象的 SS_ATTR 属性值，根据获得的值，decode 后获的 SnapshotContext 结构。

# 6. find_object_context

本函数查找对象的 object_context，这里需要理解 snapshot 相关的知识。根据 snap_seq 正确与否获取相应的 clone 对象，然后获取相应的 object_context。

int ReplicatedPG::find_object_context(const hobject_t& oid, //要查找的对象 ObjectContextRef *pobc, //输出对象的 ObjectContext

```txt
bool can_create, //是否需要创建  
bool map_snapid_toClone, //映射snapid到clone对象  
hobject_t *pmissing //如果对象不存在，返回缺失的对象
```

参数 map_snapid_toClone 指该 snap 是否可以直接对应一个 clone 对象，也就是 snap 对象的 snap_id 在 SnapSet 的 clones 列表中。

1）如果是head对象，就调用函数get_object_context获取head对象的ObjectContext，如果失败，设置head对象为pmissing对象，返回-ENOENT；如果成功，返回0。

2）如果是snapdir对象，先获取head对象的ObjectContext，如果失败，继续获取snapdir对象的ObjectContext，如果失败，返回返回-ENOENT；如果成功，返回0。

3）如果非 map_snapid_toClone 并且该 snap 已经标记删除了，就直接返回 -ENOENT, pmissing 为空，意味着该对象确实不存在。

4）调用函数get_snapshot_context来获取SnapSetContext，如果不存在，设置pmissing为head对象，返回-ENOENT。

5）如果是 map_snapshot_toClone：

a）如果 oid snap 大于 ssc->snapset seq，说明该 snap 是最新做的快照，osd 端还没有完成相关的信息更新，直接返回 head 对象 object_context，如果 head 对象存在，就返回 0，否则返回 -ENOENT。

b）否则，直接检查SnapSet的clones列表，如果没有，就直接返回-ENOENT。

c）如果找到，检查对象如果处于missing，pmissing就设置为该clone对象，返回-EAGAIN。如果没有，就获取该clone对象的object_context。

6）如果不是 map_snapid_toClone，就不能从 snap_id 直接获取 clone 对象，需要根据 snaps 和 clones 列表，计算 snap_id 对应的 clone 对象：

a）如果 oidsnap $>$ ssc->snapshot seq，获取 head 对象的 ObjectContext。

b）计算 oid snap 首次大于 ssc->snapshot.clones 列表中的 clone 对象，就是 oid 对应的 clone 对象。

c）检查该clone对象如果missing，设置pmissing为该clone对象，返回-EAGAIN。

d）获取该clone对象的ObjectContext。

e）最后检查该clone对象是如果first和last之间，这是合理的情况，返回0；否

则就是异常情况，返回-ENOENT。

本函数是获取实际对象的 ObjectContext，如果不是 head 对象，就需要获取快照对象实际对应的 clone 对象的 ObjectContext。

# 7. executectx

在do_op函数里，做了大量的对象状态的检查和上下文相关信息的获取，本函数开始执行相关的操作。

```cpp
void ReplicatedPG::executectx(开玩笑*ctx)
```

处理过程如下：

1）首先在OpContext中创建一个新的事务，该事务为pg骨架定义的事务。

```txt
ctx->op_t = pgBackend->get_transaction() 
```

2）如果是写操作，更新ctx->snapc值。ctx->snapc值保存了该操作的客户端附带的快照相关信息：

a）如果是给整个pool的快照操作，就设置ctx->snapc等于pool.snapc的值。

b）如果是用户特定快照（目前只有rbd实现），ctx->snapc值就设置为消息带的相关信息：

```txt
ctx->snapc seq = m->get_snapshot_seq();  
ctx->snapc.naps = m->get_snapshot(); 
```

c）如果设置了CEPH_OSD_FLAG_ORDERSNAP标志，客户端的snap_seq比服务端的小，就直接返回-EOLDSNAPC错误码。

3）如果是read操作，该对象的ObjectContext加ondisk_read_lock锁；对于源对象，无论读写操作，都需要加ondisk_read_lock锁。

![image](https://cdn-mineru.openxlab.org.cn/result/2026-04-30/8e4d380f-113d-46f2-8e2b-b0978fb78f78/53c96797339399ca5be5569984025c15444855bd615e9233496ca082bc3ab917.jpg)


# 提示

所谓的源对象，就是一个操作中带两个对象，比如copy操作，源对象会有读操作。

4）调用函数prepare_transaction把相关的操作封装到ctx->op_t的事务中。如果是读操作，对于replicate类型，该函数直接调用pgfrontend->objects_read_sync同步读取数据。如果是EC，就把请求加入pending_async_reads完成异步读取操作。

5）解除操作3中加的相关的锁。

6）如果是读操作，并且ctx->pending_async_reads为空，说明是同步读取，调用complete_readCtx完成读取操作，给客户端返回应答消息。如果是异步读取，就调用函数ctx->start_async_reads完成异步读取。读操作到这里就结束。后续都是写操作的流程。

7）调用calc_trim_to，计算需要trim的pglog的版本。

8）调用函数issue_repop向各个副本发送同步操作请求。

9）调用函数eval_repop，检查发向各个副本的同步操作是否已经reply成功，做相应的操作。

从上可以看出，executectx操作把相关的操作打包成事务，并没有真正的对对象的数据做修改。

# 8. calc_trim_to

本函数用于计算是否应该将旧的 pg log 日志进行 trim 操作：

```txt
void ReplicatedPG::calc_trim_to() 
```

处理过程如下：

1）首先计算target值：target值为最少保留的日志条数，默认设置为配置项cct->conf->osd_min_pg_log_entries的值。如果pg处于degraded，或者正在修复的状态，target值为cct->conf->osd_max_pg_log_entries（默认10000条）。

2）变量min_last_COMPLETE_ondisk为本pg在本osd上的完成最后一条日志记录的版本。如果它不为空，且不等于pg_trim_to，当前pg log的size大于target值，就计算需要trim掉的日志的条数：

a）num_to_trim为日志总数目减去target，如果它小于日志一次trim的最小值cct->conf->osd_pg_log_trim_min，就返回。

b）否则，从日志头开始计算最新的 pg_trim_to 版本。

# 9. prepare_transaction

本函数用于把相关的更新操作打包为事务，包括比较复杂的部分为对象的 snapshot 的处理：

```cpp
int ReplicatedPG::prepare_transaction(OpContext *ctx) 
```

处理过程如下：

1）首先调用函数ctx->snapc.is_valid()来验证SnapSet的有效性。

2）调用函数do_osdOps打包请求到ctx->op_t的transaction中。

3）如果事务为空，或者没有修改操作，就直接返回 result。

4）检查磁盘空间是否满。

5）如果该对象是head对象，就有相关快照对象COW机制的操作，需要调用函数make_writeable来完成，在关于快照的介绍中会详细介绍到。

6）调用函数finishctx来完成后续处理，该函数主要完成了快照相关的处理。如果head对象存在，就删除snapdir对象；如果不存在，就创建snapdir对象，用来保存快照相关的信息。后文会进一步介绍。

# 10. issue_repop

```cpp
void ReplicatedPG::issue_repop(RepGather *repop, OpContext *ctx) 
```

本函数的处理过程如下：

1）首先更新actingbackfill的osd对应的peer_info的相关信息：如果pinfo.last_update和pinfo.last_COMPLETE二者相等，说明该peer的状态处于clean状态，就同时更新二者，否则只更新pinfo.last_update值。

2）对该对象的 ObjectContext 的 ondisk_write_lock 加写锁，如果有 clone 对象，对该 clone 对象的 ObjectContext 的 ondisk_write_lock 加写锁。如果 snapshot_abc 不为空，也就是可能创建或者删除 snapdir 对象，对该 ObjectContext 的 ondisk_write_lock 加锁。

3）如果pool是可以rollback的（也就是ErasureCode模式），检查pg log也应该支持rollback操作。

4）分别设置三个回调 Context，调用函数 pgfrontend->submit_transaction 来完成事务向从 osd 的发送。

本函数调用pgfrontend的submit_transaction函数向从osd开始发送操作日志。

# 6.4.3 阶段3：PGBackend的处理

PGBackend 为 PG 的更新操作增加了一层与 PG 类型相关的实现。对于 Replica 类型的 PG 由类 ReplicaBackend 实现。其核心处理过程是把封装好的事务分发到该 PG 对应的其他从 OSD 上；对于 ErasureCode 类型的 PG 由类 ECBackend 实现，其核心处理过程为主 chunk 向各个分片 chunk 分发数据的过程。下面着重介绍 Replica 的处理方式。

ReplicatedBackend::submit_transaction函数最终调用网络接口，把更新的请求发送给从OSD，其处理过程如下：

1）首先构建InProgressOp请求记录。

2）调用函数ReplicatedBackend::issue_op把请求发送出去：对于该PG中的每一个从OSD：

a）调用函数 generate_subop 生成 MSG_OSD_REPOP 类型的请求。

b）调用函数get_parent() $\rightarrow$ send_message_osd_cluster把消息发送出去。

3）最后调用 parent $\rightarrow$ queue_transactions 函数来完成自己，也就是该 PG 的主 OSD 上本地对象的数据修改。

# 6.4.4 从副本的处理

当PG的从副本OSD接收到MSG_OSD_REPOP类型的操作，也就是主副本发来的同步写的操作时，处理流程和上述流程都一样。在函数sub_op_modify Impl里，对本地存储应用相应的事务，完成本地对象的数据写入。

# 6.4.5 主副本接收到从副本的应答

当PG的主副本接收到从副本的应答消息MSG_OSD_REPOPREPLY时，处理流程和上述类似，不同之处在于，在函数ReplicatedPG::do_request里调用了函数ReplicatedBackend::handle_message，在该函数里调用了ReplicatedBackend::sub_op_modify-reply函数处理该请求。

sub_op_modify_reply函数的处理过程如下：

1）首先在in_progressOps中查找到该请求。

2）如果是ondisk的ACK，也就是事务已经应答，就在ip_op.waiting_for_commit删除该OSD，该事务已经应答，那么必定已经提交了，那么从ip_op.waiting_for_applied删除该OSD。

3）如果只是事务提交到日志中的ACK，就从ip_op.waiting_for_applied删除。

![image](https://cdn-mineru.openxlab.org.cn/result/2026-04-30/8e4d380f-113d-46f2-8e2b-b0978fb78f78/0bb90e937927f323948739110584d2d6f51cb1047a66c8fb4e6002d07e65d4d7.jpg)


# 注意

这里特别说明的是，从副本需要给主副本发送两次 ACK，一次是事务提交到日志中，并没有应答到实际的对象数据中，一次是完成应答操作返回的 ACK。

4）最后检查，如果ip_op.waiting_for_applied为空，也就是所有从OSD的请求都返回来了，并且ip_op.on_applied不为空，就调用该Context的complete函数。同样，检查ip_op.waiting_for_commit为空，就调用该Context的函数的complete函数。

下面看一下，in_progressOps注册的回调函数。其回调函数是在ReplicatedPG::issue_repop函数调用里注册的。

```c
Context *on_all_commit = new C_OSD_RepopCommit(this, repop);  
Context *on_all_applied = new C_OSD_RepopApplied(this, repop);  
Context *onapplied_sync = new C_OSD_OndiskWriteUnlock(repop->obc, repop->ctx->clone_abc, unlock_snapshot_abc ? repop->ctx->snapshot_abc : ObjectContextRef()); 
```

回调函数都最终调用了函数ReplicatedPG::eval_repop，其最终向client发送应答消息。这里强调的是，主副本必须等所有的处于up的OSD都返回成功的ACK应答消息，才向客户端返回请求成功的应答。

# 6.5 本章小结

本章介绍了OSD读写流程核心处理过程。通过本章的介绍，可以了解读写流程的主干的流程，并对一些核心概念和数据结构的处理做了介绍。当然读写流程是Ceph文件系统的核心流程，其实现细节比较复杂，还需要读者对照代码继续研究。目前在这方面的工作，许多都集中在提供Ceph的读写性能。其基本的方法更多的就是优化读写流程的关键路径，通过减少锁来提供并发，同时简化一些关键流程。

![image](https://cdn-mineru.openxlab.org.cn/result/2026-04-30/8e4d380f-113d-46f2-8e2b-b0978fb78f78/4efa7e00ba851c3f19c5750a498ba40ba87ce714e834d9092592945094e05662.jpg)


# 第7章

# Chapter 7

# 本地对象存储

本地对象存储模块完成了数据如何原子地写入磁盘，这就涉及事务和日志的概念。对象如何在本地文件系统中组织的代码实现在 src/os 中。本章将介绍在单个 OSD 上数据如何写入磁盘中。

目前有4种本地对象存储实现：

- FileStore：这是目前比较稳定，生产环境上使用的主流对象存储引擎，也是本章重点介绍的对象存储引擎。

- BlueStore：这是目前社区在实现的一个新版本，社区丢弃了本地文件系统，自己写了一个简单的，专门支持RADOS用户态的文件系统。

□ KStore：这是以本地KV存储系统实现的对象存储，它基于RODOS的框架用来实现一个分布式的KV存储系统。

□Memstore：它把数据和元数据都保存在内存中，用来测试和验证使用。

KStore和Memtore两种存储引擎比较简单，这里就不介绍了。BlueStore社区还正在开发之中，这里也暂时不介绍。本章将详细介绍目前在生产环境中使用的FileStore存储的实现。

# 7.1 基本概念介绍

RADOS 本地对象存储系统（也称为对象存储引擎）基于本地文件系统实现，目前默认的文件系统为 XFS。一个对象包含数据和元数据两种数据。对应本地文件系统里，一个对象就是一个固定大小（默认 4MB）的文件，其元数据保存在文件的扩展属性或者本地独立的 KV 存储系统中。

# 7.1.1 对象的元数据

对象的元数据就是用于对象描述信息的数据，它以简单的key-value（键值对）形式存在，在RADOS存储系统中有两种实现：xattrs和omap：

- xattrs 保存在对象对应文件的扩展属性中，这要求支持对象存储的本地文件系统支持扩展属性。

□omap就是objectmap的简称，是一些键值对，保存在本地文件系统之外的独立的key-value存储系统中，例如leveldb、rockdb等。

有些本地文件系统可能不支持扩展属性，有些虽然也支持扩展属性但对key或者value占用空间的大小有限制，或者扩展属性占的总的空间大小有限制。对于leveldb等本地键值存储系统基本没有这样的限制，但是它的写性能优越于读性能。所以一般情况下，xattrs保存一些比较小而经常访问的信息。omap保存一些大而不是经常访问的数据。

# 7.1.2 事务和日志的基本概念

假设磁盘正在执行一个操作，此时由于发生磁盘错误，或者系统宕机，或者断电等其他原因，导致只有部分数据写入成功。这种情况就会出现磁盘上的数据有一部分是旧数据，部分是新写入的数据，使得磁盘数据不一致。

当一个操作要么全部成功，要么全部失败，不允许只有部分操作成功，就称这个操作具有原子性。引入事务和日志，就是为了实现操作的原子性，解决数据的不一致性问题。

引入日志后，数据写入变为两步：1）先把要写入的数据全部封装成一个事务，其整体作为一条日志，先写入日志文件并持久化的磁盘，这个过程称为日志的提交（journal

submit)。2）然后再把数据写入对象文件中，这称为日志的应用（journal apply）。

当系统在日志提交的过程中出错，系统重启后，直接丢弃不完整的日志条目即可，该条日志对应的实际对象数据并没有修改，数据可以保持一致。当在日志应用的过程中出错，由于数据已经写入并回刷到日志盘中，系统重启后，重放（replay）日志，就可以保证新数据重新完整写入，保证了数据的一致性。

这个机制需要确保所有的更新操作都是幂等操作。所谓幂等操作，就是数据的更新可以多次写入，不会产生任何副作用。对象存储的操作一般都具有幂等性。

在事务的提交过程中，一条日志记录可以对应一个事务。为了提高日志提交的性能，一般都允许多条事务并发提交，一条日志记录可以对应多个事务，批量提交。所以事务的提交过程，一般和日志的提交过程是一个概念。

日志有三个处理阶段，对应过程分别为：

□日志提交（journal submit）：日志写入日志磁盘。

□日志的应用（journal apply）：日志对应的修改更新到对象的磁盘文件中。这个修改不一定写入磁盘，可能缓存在本地文件系统的页缓存（page cache）中。

□日志的同步（Jouranl sync或者journal commit)：当确定日志对应的修改操作已经刷回到磁盘中，就可以把相应的日志记录删除，释放出所占的日志空间。

# 7.1.3 事务的封装

ObjectStore 的内部类 Transaction 用来实现相关的事务。它有两种封装形式，一种是 use_tbl，事务把操作的元数据和数据都封装在 bufferlist 类型的 tbl 中：

```txt
bool use_tbl; //标记是否使用tbl来encode/decode bufferlist tbl;
```

另一种是不使用 tbl，把元数据操作以 struct Op 的结构体，封装在 op_bl 中，把操作相关的数据封装在 dataz_bl 中：

```txt
bufferlist data_bl;  
bufferlist op_bl;  
bufferptr op_ptr; //操作临时指针
```

由于结构体 op 里保存的是 coll_id 和 object_id，所以需要保存 coll_t 到 coll_id 的映射关系，以及 ghobject_t 和 object_id 的映射关系。从这种存储方式就可以看出，这种方式和前一种的区别，是在数据封装上实现了一种压缩。当事务中多个操作有相同的对象时，只保存一次 ghobject_t 结构体，其他情况只保存 index 来索引。

```cpp
map<coll_t, _le32> coll_index; //coll_t -> coll_id的映射关系  
map<ghobject_t, _le32, ghobject_t::BitwiseComparator> object_index; //ghobject_t -> object_id的映射关系  
_1e32 coll_id; //当前分配的coll_id的最大值  
_1e32 object_id; //当前分配的object_id的最大值
```

数据结构 TransactionData 记录了一个事务中有关操作的统计信息：

```txt
structTransactionData{ _le64ops; //本事务中操作数目 //以下记录了事务中的带的数据最大的操作的： _le32largest_data_len; //最大的数据长度 _le32largest_data_off; //在对象中的偏移 _le32largest_data_off_in_tbl; //在tbl中的偏移 _le32fadvise_flags; //一些标志 }
```

一个事务中，对应如下三类回调函数，分别在事务不同的处理阶段调用。当事务完成相应阶段工作后，就调用相应的回调函数来通知事件完成。注意每一类都可以注册多个回调函数：

list<Context $\star >$ on_commit;   
list<Context $\star >$ on_applied;   
list<Context $\star >$ on_applied_sync; 

on_commit 是事务提交完成之后调用的回调函数；on_applied_sync 和 on_applied 都是事务应用完成之后的回调函数。前者是被同步调用执行，后者是在 Finisher 线程里异步调用执行。

# 7.2 ObjectStore对象存储接口

下面通过对象存储的接口说明和代码示例，可以了解对象存储的基本功能及如何使用这些功能，从而对对象存储有一个概要了解。

# 7.2.1 对外接口说明

类 ObjectStore 是对象存储系统抽象操作接口。所有的对象存储引擎都有继承并实现它定义的接口。下面列出一些有代表性的函数接口。

- objectStore 初始化相关的函数：

- mkfs 创建 objectstore 相关的系统信息。

- mount 加载 objectsotre 相关的系统信息。

- statfs 获取 objectstore 系统信息。

□ 获取属性已经 collection 相关的信息：

- getattr 获取对象的扩展属性 xattr。

- omap_get 获取对象的 omap 信息。

- queue_transactions 是所有 ObjectStore 更新操作的接口。更新相关的操作（例如创建一个对象，修改属性，写数据等）都是以事务的方式提交给 ObjectStore，该函数被重载成各种不同的接口。其参数为：

- Sequencer *osr 用于保证在同一个 Sequencer 的操作是顺序的。

- list<Transaction*>& tls 要提交的事务，或者事务的列表。

- Context *onreadablehe 和 Context *onreadable_sync 这个函数分别对应事务的 on_apply 和 on_apply_SYNC，当事务 apply 完成之后，修改后的数据可以被读取了。

- Context *ondisk 这个回调函数就是事务进行 on_commit 后的回调函数。

# 7.2.2 ObjectStore代码示例

下面通过一段 ObjectStore 的测试代码，来展示 ObjectStore 的基本功能和接口使用：

1）首先调用create方法创建一个ObjectStore实例。参数分别为配置选项CephContext，对象存储的类型:filestore。对象存储的目录名、日志文件名如下：

ObjectStore \*store_ $=$ ObjectStore::create(g_ceph_context, string("filestore"), string("store_test_temp_dir"), string("store_test_temp_journal")); 

2）创建并加载 ObjectStore 本地对象存储：

```txt
store->mkfs(); 
```

```txt
store->mount(); 
```

3）创建了一个 collection，并写数据到对象中。对象的任何写操作，都先调用事务的相关写操作。事务把相关操作的元数据和数据封装，然后调用 store 的 apply_transaction 来真正实现数据的更改。

ObjectStore::Sequencer osr("test"); coll_t cid; ghobject_t hoid(hobject_t(object_t("Object 1", CEPH_NOSNAP)); bufferlist bl; bl.append("1234512345"); int r; ObjectStore::Transaction t; //创建一个事务 t.create.collection(cid,0); //创建一个collection t.write(cid,hoid,0，bl.length(),bl); //写对象数据 $\mathbf{r} =$ store->apply_transaction(&osr，std::move(t)); //事务的应用，真正实现数据的写入操作 store->umount(); $\mathbf{r} =$ store->mount();

# 7.3 日志的实现

在本地对象中，日志是实现操作一致性的机制。在介绍Filestore之前，首先需要了解Journal的机制。本节首先介绍Journal的对外接口，通过对外提供的功能接口，可以了解日志的基本功能。然后详细介绍FileJournal的实现。

# 7.3.1 Jouanol 对外接口

通过Ceph中的test_filejournal.cc的一段测试程序，可以比较清楚地了解Journal的外部接口，通过这些外部接口可以了解Journal的功能，以及如何使用Jouarnal：

```txt
FileJournal j(fsid, finisher, &sync_cond, path, directio, aio);  
j.create();  
j.make_writeable();  
bufferlist bl;  
bl.append("small");  
j.submit_entry(1, bl, 0, new C_SafeCond(&wait_lock, &cond, &done));  
wait();  
j.close(); 
```

通过上述代码可以了解到，日志的基本使用方法如下所示：

1）创建一个FileJournal类型的日志对象，其构造函数path为日志文件的路径。

2）调用日志的create方法创建日志，并调用函数make writeable设置为可写状态。

3）在bufferlist中添加一段数据作为测试的日志数据，调用submit_entry提交一条日志。

4）等待日志提交完成

5）关闭日志。

# 7.3.2 FileJournal

类FileJournal以文件（包括块设备文件看做特殊文件）做为日志，实现了Journal的接口。下面先介绍FileJournal的数据结构和日志的格式，然后介绍日志的三个阶段：日志的提交（journal submit）、日志的应用（journal apply）、日志的同步（journal sync或者commit、trim）及其具体的实现过程。

# 1. FileJournal 数据结构

FileJournal数据结构如下：

```cpp
struct write_item {
    uint64_t seq; //日志的序号
    bufferlist bl; //日志的内容
    uint32_t orig_len; //日志的原始长度
    TrackedOpRef tracked_op; //操作的跟踪记录
    write_item( uint64_t s, bufferlist & b, int ol, TrackedOpRef opref) :
        seq(s), orig_len(ol), tracked_op(opref) {
            bl_claim(b, buffer::list::CLAIM_OPEN_NONSHAREABLE);
        }
    write_item() : seq(0), orig_len(0) {}
};
list<write_item> writeq; //保存write_item的队列
Mutex writeq_lock; //writeq相关的锁
Cond writeq_cond; //writeq相关的条件变量
```

结构体 write_item 封装了一条提交的日志，seq 为提交日志的序号，bl 保存了提交的日志数据，也就是提交的事务或者事务的集合序列化后的数据。Journal 并不关心日志数据的内容，它认为一条日志就是一段写入的数据，只负责把数据正确写入日志盘。orig_

len 记录日志的原始长度，由于日志字节对齐的需要，最终写入的数据长度可能会变化。  
tracked_op 用于跟踪记录一些统计信息。

所有提交的日志，都先保存到writeq这个内部的队列中，writeq_lock和writeq_cond是writeq相应的锁和条件变量：

```c
struct completion_item {
    uint64_t seq; //日志的序号
    Context *finish; //完成后的回调函数
    utime_t start; //提交时间
    TrackedOpRef tracked_op;
    completion_item uint64_t o, Context *c, utime_t s, TrackedOpRef opref);
    : seq(o), finish(c), start(s), tracked_op(opref) {}
    completion_item() : seq(0), finish(0), start(0) {}
};
```

结构体 completion_item 记录准备提交的 item，保存在列表 completions 中。由锁 completions_lock 保护。

```c
off64_t write_pos; //日志的写的开始位置  
off64_t read_pos; //日志读取的位置  
uint64_t writing_seq; //本次正在写入的最大 seq  
uint64_t last_committee_seq; //最后同步完成的 seq  
bool write_stop; //写线程是否停止的标记  
enum {  
    FULL_NOTFULL = 0,  
    FULL_FULL = 1,  
} full_state; //日志的磁盘空间的状态  
//日志头部  
struct header_t {  
    enum {  
        FLAG_CRC = (1 << 0),  
    };  
}  
uint64_t flags; //日志的一些标志  
uuid_d fsid; //文件系统的 id  
_u32_block_size; //日志文件或者设备的块大小  
_u32 Alignment; //对齐
```

```c
int64_t max_size; //日志的最大size  
int64_t start; //offset of first entry日志条目的起始地址  
uint64_t committed_up_to; //已经提交的日志的seq，等同于journaled_seq  
uint64_t start_seq; //等同于日志的起始，last_commited_seq + 1  
} header;
```

结构体header_t是日志文件的头信息，保存日志相关的全局信息。

deque<pair<uint64_t,off64_t $\rightharpoondown$ journalq; //seq->end用于记录日志在日志文件中的结束偏移，用于日志删除时可以知道偏移 uint64_t writing_seq; //当前正在写入的最大的seq bool must_write_header; //是否必须写入一个header，当日志同步完成后，就设置本标准，必须写入一个header，以持久化保存应日志同步而变化了的header结构

```txt
Mutex write_lock; //特别需要指出，以上数据结构都受write_lock的保护  
Mutex finisher_lock; Cond finisher_cond; uint64_t journaled_seq; //已经提交成功的日志的最大seq
```

# 2.日志的格式

日志的格式如下：

```txt
entry_header_t + journal data + entry_header_t 
```

每条日志数据的头部和尾部都添加了 entry_header_t 结构。此外，日志在每次同步完成的时候设置 must_write_header 为 true 时会强制插入一个日志头 header_t 的结构，用于持久化存储 header 中变化了的字段。日志的结构如图 7-1 所示。

![image](https://cdn-mineru.openxlab.org.cn/result/2026-04-30/8e4d380f-113d-46f2-8e2b-b0978fb78f78/f2c67160f39ced0c401bd0073d1db3b9bae92f6129dcab460437e95b15a6aee9.jpg)



图7-1 日志的结构图


日志记录1到4为已经由日志同步时trim掉的日志（逻辑删除），last_committed_seq记录了最后一个trim的日志序号，当前值为4。日志记录5到11为已经提交成功的日志。journaled_seq记录已经提交成功的日志最大序号，当前值为11。日志记录12到14为正在提交，还没有完成的日志。writing_seq记录正在提交的最大日志序号，当前值为14。

header_start_seq记录提交成功的第一条有效日志，当前显示为5，header commited_up_to为提交成功的最后一条日志，当前值为11。

# 3. 日志的提交

函数FileJournal::submit_entry用于提交日志。

```cpp
submit_entry  
FileJournal::submit_entry( uint64_t seq, bufferlist& e, uint32_t orig_len, Context *oncommit, TrackedOpRef osd_op) 
```

其处理过程如下：

1）其分别加writeq lock锁和completions lock锁。

2）然后构造 completion_item 和 write_item 两个数据结构体，并分别加入到 completions 队列和 writeq 队列中。

3）如果writeq队列为空，就通过writeq_cond Signals（如果writeq不为空，说明线程一直在忙碌处理请求，不会睡眠，就没必要再次触发了）。

write_thread 线程是 FileJournal 内部的一个工作线程。该线程的处理函数 write_thread_entry 实现了不断地从 writeq 队列中取出 write_item 请求，并完成把日志数据写入日志磁盘的工作。

write_thread_entry 函数调用 prepareMULTI_write 函数，把多个 write_item 合并成到一个 bufferlist 中打包写入，可以提高日志写入磁盘的性能。配置项 g_conf->journal_max_write_entries 为一次写入的最大日志条目数量，配置项 g_conf->journal_max_write_bytes 为一次合并写入的最大字节数。

prepare-multi_write函数调用batch_pop_write函数，从writeq里一次获取所有的write_item，对每一个write_item调用函数prepare_single_write处理。

prepare_single_write 函数实现了把日志数据封装成日志格式的数据：

1）在数据的前端和后端都添加了一个 entry header t 的数据结构。

2）journalq记录了该条日志在日志文件（或者设备）中的结束偏移量，它在日志同步时，用于逻辑删除该条日志：

```javascript
journalq.push_back(pair<uint64_t,off64_t>(seq，queue_pos)); 
```

3）更新writing_seq的值为当前日志的seq，并更新当前日志写入的queue_pos的值。当queue_pos达到日志的最大值时，需要从日志的头开始：

```txt
writing_seq = seq;  
queue_pos += size;  
if (queue_pos >= header.max_size)  
queue_pos = queue_pos + get_top() - header.max_size; 
```

当调用函数 preparemulti_write 把日志封装成需要的格式后，就需要把数据写入日志磁盘。目前有两种实现方式：一种是同步写入；另一种是 Linux 系统提供的异步 IO 的方式。

同步写入方式直接调用do_write函数实现，该函数具体过程如下：

1）首先判断是否需要写入日志的header头部：日志在每隔配置选项设定的g_conf>journal_write_header_freqency日志条数后或者must_write_header设置时插入一个header头。

2）其次计算如果日志数据超过了日志文件最大长度，就需要把该条日志的数据分两次写入：尾端写入一部分，然后绕到头部写入一部分。

3）如果不是directIO，就需要调用函数fsync把日志数据刷到磁盘上。

4）加 finisher_lock，更新 journaled_seq 的值。

5）如果日志为不满的状态，并且 plug_journal_completions 没有设置，就调用函数 queue_completions_thru 来把 completion_item 的 callback 函数加入 Finisher。最后删除该 completion_item 项。

异步IO的方式写入是日志写入的另一种方式，是使用了Linux操作系统内核自带的异步（aio）。Linux内核实现的aio，必须以directIO的方式写入。aio由于内核实现，数据不经过缓存直接落盘，性能要比同步的方式高很多。

异步IO相关的数据结构如下：

```c
struct aio_info {
    struct iocb iocb;
    bufferlist bl;
    struct iovec *iov;
    bool done;
    uint64_t off, len; //< these are for debug only
    uint64_t seq; //aio的序号
};
```

```c
io_context_t aioCtx; //异步io的上下文  
list<aoi_info> aio_queue; //已经提交的aoi请求队列  
Mutex aio_lock; //aio队列的锁  
Cond aio_cond; //控制inflight的aoi不能过多，否则需要等待  
Cond write_finish_cond; //aio完成，触发完成线程去处理  
int aio_num, aio_bytes; //正在进行的aoi的数量和数据量
```

异步IO的方式写日志，调用了do_aio_write函数，基本过程和do_write相似，只是写入的方式调用了write_aio_bl函数。

函数 write_aio_bl 实现了调用 Linux 的 aio 的 API 异步写入磁盘。实现过程如下：

1）首先把数据转换成iovec的格式。

2）构造aio_info数据结构，该结构保存了aio的上下文信息，并把它加入aio_queue队列里。

3）调用函数io Prep_pwritev和函数io_submit，异步提交请求：

```txt
io Prep_pwritev(&aio.iocb, fd, aio.iov, n, pos)  
int r = io_submit(aioCtx, 1, &piocb); 
```

4）触发write_finish_thread检查是异步IO是否完成：

```txt
aio_lock.Lock();  
write_finish_condSignal();  
aio lock.Unlock(); 
```

函数 write_finish_thread_entry 为线程 write_finish_thread 的执行函数，用来检查 aio 是否完成。其具体实现如下：

1）调用函数io_getevents来检查是否有aio事件完成的：

```txt
io_event event[16];  
int r = io_getevents(aioCtx, 1, 16, event, NULL); 
```

2）对每个完成的事件，获取相应的aio_info结构。检查写入的数据是否是aio的数据长度，然后设置ai $\rightarrow$ done为true，标记该aio已经完成。

3）调用函数 check_ao_completion 检查 aio_queue 队列的完成情况。由于 aio 的并发提交，多条日志可能并发完成，为了保证日志是按照 seq 的顺序完成，必须从头按顺序检查 aio_queue 的完成情况。

4）更新journaled_seq的值，调用queue_completions_thru完成后续工作，这和同步实现工作一样。

# 4. 日志的同步

日志的同步是在日志已经成功应用完成，对应的日志数据已经确保写入磁盘的日志后，就删除相应的日志，释放出日志空间的过程。

函数 committed_thru 用来实现日志的同步。它只是完成了日志的同步功能。日志同步时，必须确保日志应用完成，这个逻辑在 Filestore 里完成。

函数 committed_thru 具体实现如下：

1）确保当前同步的 seq 比上次 last_committee sequ 大。

2）用seq更新last_committeeq的值。

3）把journalq 中小于等于 seq 的记录删除掉，这些记录已经无用了。

4）删除相应的日志。这里只是逻辑上的删除，只是修改header.start指针就可以了。当journalq为空，就删除到设置write_pos位置。如果journalq不为空，就设置为队列的第一条记录的偏移值，这是最新日志的起始地址，也就是上一条日志的结尾。

5）设置must_write_header为true，由于header更新了，日志必须把header写入磁盘。

6）日志可能因日志空间满而阻塞写线程，此时由于释放出更多空间，就调用commit_condition.Signal()唤醒写线程。

# 7.4 FileStore的实现

在介绍完本地对象存储的概念、对外接口和日志实现后，本节介绍FileStore的实现。首先看一下FileStore的静态类图，如图7-2所示。

FileStore的静态类图说明如下：

□ ObjectStore 是对象存储的接口。FileStore 继承了 JournalingObjectSore 类。

□ JournalngObjectStore实现了和日志之间的交互。

- FileStoreBackend 定义了本地文件系统支持 Filestore 的一些接口，不同的文件系统会增加自己支持特性实现了不同的 FileStoreBackend，例如 BtrfsFileStoreBackend 和 XfsFilestoreBackend 分别对应 XFS 和 BTRFS 两种文件系统的实现。

□ Journal类定义了日志的抽象接口，FileJournal实现了以本地文件或者本地设备为存储的日志系统。

![image](https://cdn-mineru.openxlab.org.cn/result/2026-04-30/8e4d380f-113d-46f2-8e2b-b0978fb78f78/9b4005cafd07c8705574ee0b83a7fbed4a4110cfae9c0353da800094c6e6d05f.jpg)



图7-2FileStore的静态类图


# 7.4.1 日志的三种类型

在FileStore里，根据日志提交方式的不同，有三种类型的日志：

□ journal writeahead：日志数据先提交并写入日志磁盘上，然后再完成日志的应用（更新实际对象数据）。这种方式适合XFS、EXT4等不支持快照的本地文件系统使用这种方式。

□ journal parallel：日志提交到日志磁盘上和日志应用到实际对象中并行进行，没有先后顺序。这种方式适用于BTRFS和ZFS等实现了快照操作的文件系统。由于具有文件系统级别快照功能，当日志的应用过程出错，导致数据不一致的情况下，文件系统只需要回滚到上一次快照，并replay从上次快照开始的日志就可以了。显然，这种方式比writeahead方式性能更高。但是由于btrfs和zfs目前在Linux都不

稳定，这种方式很少用。

□不使用日志的情况，数据直接写入磁盘后才返回客户端应答。这种方式目前FileStore也实现了，但是性能太差，一般不使用。

本节只介绍FileStore默认的方式：journal writeahead方式的日志实现。

# 7.4.2 JournalingObjectStore

类 JournalingObjectStore 实现了 ObjectStore 和 FileJournal 的一些交互管理。它通过类 SubmitManager 和类 ApplyManager 分别实现了日志提交和日志应用的管理。下面分别介绍这两个类。

# 1. SubmitManager

类 SubmitManager 实现了日志提交的管理，准确地说是日志序号的管理。函数 op submit_start 给提交的日志分配一个序号。函数 op submit_finish 在日志提交完成后验证：当前的 op 的 seq 等于上次提交的序号 op submit_finish 加 1。最后更新 op submit_finish 的值。

```txt
class SubmitManager {
    Mutex lock;
    uint64_t op_seq; //日志最新的序号
    uint64_t op_submit_finish; //日志上次提交的序号
}
```

# 2. ApplyManager

ApplyManager 负责日志应用的相关处理：

```txt
class ApplyManager {
    Journal *&journal;
    Finisher &finisher;
    ...
    ...
    ...
    ...
    ...
    ...
    ...
    ...
    ...
    ...
    ...
    ...
    ...
    ...
    ...
    ...
    ...
    ...
    ...
    ...
    ...
    ...
    ...
    ...
    ...
    ...
    ...
    ...
    ...
    ...
    ...
    ...
    ...
    ...
    ...
    ...
    ...
    ...
    ...
    ...
    ...
    ...
    ...
    ...
    ...
    ...
    ...
    ...
    ...
    ...
    .
    ...
    ...
    ...
    ...
    ...
    ...
    ...
    ...
    ...
    ...
    ...
    ...
    ...
    ...
    ...
    ...
    ...
    ...
    ...
    ...
    ...
    ...
    ...
    ...
    ...
    ...
    ...
    ...
    ...
    ...
    ...
    ...
    ...
    ...
    ...
    ...
    ...
    ...
    ...
    ...
    ...
    ...
    ...
    ...
    ...
    ...
    ...
    ...
.
} 
```

```txt
//日志完成后的回调函数，用于没有日志的类型  
uint64_t committingting_seq, //正在应用的日志的seq  
committed_seq; //已经应用完成的seq
```

函数 ApplyManager::op_apply_start 在日志应用前调用，其实现如下操作：

1）给加apply_lock锁。

2）当blocked设置时，就等待，暂停日志的应用，这个后面日志同步时会说明。

3）统计值open ops加1。

函数 ApplyManager::op_apply_finish 在日志应用完成后调用，其实现如下操作：

1）加apply_lock锁。

2）统计值open ops自减。

3）如果 blocked 为 true，就调用 blocked_condSignal() 发通知。

4）更新max_applied_seq的值为应用完成的最大日志号。

函数 ApplyManager::commit_start 用于在日志同步时计算目前完成的最大的日志 seq。

ApplyManager::commit_started 表示日志开始同步，commit_finish 函数在日志同步结束时调用。

# 7.4.3 Filestore的更新操作

先介绍数据结构。结构Op代表了一个ObjectStore的操作的上下文信息在FileStore里的封装：

```txt
struct Op {
utime_t start; //日志应用的开始时间
uint64_t op; //日志的 seq
list<Transaction*> tls; //事务列表
Context *onreadable, *onreadable_sync; //事务应用完成之后的回调函数和同步回调函数
uint64_t ops, bytes; //操作数目和字节数，这里的ops指的是对象的基本操作例如create,
write, delete等基本操作，一个Op带多个Transaction，可
能带多个基本操作。
TrackedOpRef osd_op;
};
```

类OpSequencer用于实现请求的顺序执行。在同一个Sequencer类的请求，保证执

行的顺序，包括日志commit的顺序和apply的顺序。一般情况下，一个PG对应一个Sequencer类。所以一个PG里的操作都是顺序执行。

classOpSequencer{   
Mutexqlock;//本来所保护队列q，在flush时，peek和deguee也被该锁保护 list<Op $\text{十}$ q; //操作序列 list<uint64_t>jq; //日志序号 list<pair<uint64_t,Context $\text{十}$ >flush_commit_waiters; Cond cond;   
public: Sequencer\*parent;   
Mutexapply_lock; //日志应用的互斥锁 int id;

# 1. 更新实现

FileStore 的更新操作分两步：首先把操作封装成事务，以事务的形式整体提交日志并持久化到日志磁盘，然后再完成日志的应用，也就是修改实际对象的数据。

```cpp
int FileStore::queue_transactions(Sequencer *posr, vector<Transaction>& tls, TrackedOpRef osd_op, ThreadPool::TPHandle *handle) 
```

FileStore更新的入口函数为queue_transactions函数。参数posr用于确保执行的顺序；参数tls是Transaction的列表，一次可以提交多个事务；TrackedOpRef用于跟踪信息。

具体处理流程如下：

1）首先调用collect_context函数把tls中所有事务的on_applied类型的回调函数收集在一个list<Context $\ast >$ 结构里，然后调用list_to_context函数把Context列表变成了一个Context类。这里实际就是包装成一个回调函数。on_commit和on_applied_sync都做了类似的工作。

2）构建 op 对象。把相关的操作封装到 op 对象里：

```txt
Op \*o = build_op(tls, onreadable, onreadable_sync, osd_op); 
```

3）调用函数prepare_entry把所有日志的数据封装到tbl里。

```txt
int orig_len = journal->prepare_entry(o->tls, &tbl); 
```

4）分配一个日志的序号，并设置在op结构里。

```txt
uint64_t op_num = submitmanager.op_submit_start();  
o->op = op_num; 
```

5）如果日志是m_filestore_journal_parallel模式，就调用函数_op_journal_transactions开始提交日志。然后调用函数_queue_op，它把请求添加到op_wq里同时开始日志的应用。

6）如果日志是m_filestore_journal_writeahead模式，就把该op添加到osr中，调用_op_journal_transactions函数提交日志。

7）调用函数submitmanager.op Submit_finish(op_num)，通知submitmanager日志提交完成。

# 2. writeahead 和 parallel 的处理方式的不同

函数 op_journal_transactions 用于提交日志。当有日志并且日志可写时，就调用 journal 的 submit_entry 函数提交日志。当日志提交成功后，就会调用 op_journal_transactions 注册的回调函数 onjournal。

函数 queue_op 用于把操作添加的 Filestore 内部的线程池对应队列中 OpWq 中。OpWq 的调用 do_op 函数完成日志的应用，也就是完成实际的操作。

对于日志 parallel 方式，日志的提交和应用并发进行：

```javascript
_op_journal_transactions(tb1,orig_len，o->op，ondisk，osd_op); //加入到提交队列里 queue_op(osr,o);
```

对于writeahead方式，先调用函数_op_journal_transactions提交日志，其注册的回调函数onjournal为C_JournaledAhead：

```c
osr->queue_journal(o->op);  
_op_journal_transactions(osr, origin, o->op, new C_JournaledAhead(this, osr, o, ondisk), osd_op); 
```

在回调函数的 C_JournaledAhead 类里，其 finish 函数调用 _journaled Ahead，该函数调用 queue_op(osr, o) 把相应请求添加到 op_wq.queue(osr) 工作队列中。工作队列的处理线程实现了日志 apply 操作，完成实际对象数据的修改。

从上可以看出，writeahead是先完成了日志的提交，然后才开始日志的应用。Parallel方式是同时完成日志的提交和应用。

如果日志是第三种类型，即没有日志的形式，就调用函数applymanager.add_waiter把Context添加到commit_waiters中。

# 7.4.4 日志的应用

操作队列OpWq用于完成日志的应用。处理函数_process调用_do_op函数来完成应用日志，实现真正的修改操作。

函数_do_op函数实现操作如下：

1）首先调用osr->apply_lock.Lock()加锁，通过该锁，实现了同一时刻，osr里只有一个操作在进行。

2）调用op_apply_start，通知applymanager类开始日志的应用。

3）调用_do_transactions函数，用于完成日志的应用。它对每一个事务调用_do_transaction函数，解析事务，执行相应的操作。

4）调用op_apply_finish(o->op)通知applymanager完成。

# 7.4.5 日志的同步

在FileStore内部会创建sync线程，用来定期同步日志，该线程的入口函数为：

```txt
void FileStore::sync_entry() 
```

函数 sync_entry 定期执行同步操，处理过程如下：

1）调用函数tp.pause()，用来暂停FileStore的op_wq的线程池，等待正在应用的日志完成。

2）然后调用fsync同步内存中的数据到数据盘，当同步完成后，就可以丢弃相应的日志，释放相应的日志空间。

# 7.5 omap的实现

omap 的静态类图 7-3 所示。类 ObjectMap 定义了omap 的抽象接口。类 DBObjectMap 实现了以 KeyValueDB 的本地存储实现的 ObjectMap 的接口。

![image](https://cdn-mineru.openxlab.org.cn/result/2026-04-30/8e4d380f-113d-46f2-8e2b-b0978fb78f78/2084e2a29216c716d5a97c2dd9ca219bc3ef7aa5ca62ca964f857f0cae5ad2b0.jpg)



图7-3 omap静态类图


目前实现 KeyValueDB 的本地存储分别为：Facebook 开源的 levelDB 存储，对应类 LevelDBStore；Google 开源的 RocksDB 存储，对应类 RocksDBStore 实现；KineticStore 存储对应的类 kineticStore。默认采用 LevelDB 实现。

表7-1是LevelDB是一个key-value的存储系统，它是一维的flat模式的KV存储。表7-2是对象存储，多个不同对象的多个不同属性的二维存储模式。

二者如何映射呢？由于对象的名字是全局唯一的，属性在对象内也是唯一的，所以在 LevelDB 层面就可以用 Object 名字和属性的名字联合作为在 LevelDB 的 key，这是能想到的比较直观的解决方式。

例如：object1 的属性（key1，value1）的保存在 LevelDB 如下：

```txt
(object1_name + key1, value1) 
```

这种方法的一个缺点：当一个对象有多个KV值时，Object1的name多次作为key存储，由于Object的name一般比较长，这样存储方式浪费空间比较大。于是就提出了一种压缩的存储方法，也就是目前omap的存储方式。


表 7-1 levelDB 的 flat 的 KV 存储模式


<table><tr><td>Key</td><td>Value</td></tr><tr><td>Key1</td><td>Value1</td></tr><tr><td>Key2</td><td>Value2</td></tr><tr><td>......</td><td>......</td></tr><tr><td>KeyN</td><td>Value N</td></tr></table>


表 7-2 对象的 omap 存储模式


<table><tr><td rowspan="3">Object1</td><td>Key1</td><td>Value1</td></tr><tr><td>Key2</td><td>Value2</td></tr><tr><td>......</td><td>......</td></tr><tr><td rowspan="3">Object2</td><td>Key1</td><td>Value1</td></tr><tr><td>Key2</td><td>Value2</td></tr><tr><td>......</td><td>......</td></tr><tr><td>......</td><td></td><td></td></tr></table>

# 7.5.1 omap存储

目前omap的在leveldb中存储分两步：

1）在LevelDB中，保存键值对：

```txt
key: HOBJECT_TO_SEQ + ghobject_key(oid)  
value: header 
```

HOBJECT_TO_SEQ是固定的前缀标识字符串，函数ghobject_key获取对应的对象唯一的key字符串。

header 保存对象在 LevelDB 中的唯一标识 seq，以及支持快照的父对象的信息，同时保存了对象的 collection 和 oid（这里冗余保存，因为藏 key 信息里就可以获得）。

```c
struct {_Header{ uint64_t seq; //自己在leveldb中的序号 uint64_t parent; //父对象的序号 uint64_t num_child; //子对象的数量 coll_t c; //对象所在的collection ghobject_t oid; //对象标识 SequencerPosition spos; //保存了当前的日志的序号(op_seq，日志内多个事务的序号，每个事务内操作的序号，}；
```

2）对象的属性保存以下格式的键值对：

```txt
Key: USER_PREFIX + header_key(header->seq) + XATTR_PREFIX + key  
Value: value(omap 的值
```

综上所述，设置和获取对象的属性，需要两步：先根据对象的oid，构造键（HOBJECT_TO_SEQ + ghobject_key(oid)），获取header；根据对象的header中的seq，拼接在levelDB中的key值（USER_PREFIX + header_key(header->seq) + XATTR_PREFIX + key)，获取value值。

变量state用于保存KeyValueDB的全局状态，目前只有seq信息。

```c
struct State {
    __u8 v; //版本
    uint64_t seq; //全局分配的seq
} state;
```

函数 write_state 用于每次分配 seq 后，把 state 信息写入 LevelDB 中，保存：

```txt
SYS_PREFIX + GLOBAL_STATE_KEY -> sate 
```

# 7.5.2 omap的克隆

omap的克隆机制的实现如下所示：

oid $\rightarrow$ header  
oid_new $\rightarrow$ new_header 

□当克隆一个新的对象old_new时，仅创建一个对应的new_header，并不是把该对象的所有属性都在leveldb中拷贝一遍，同时在new_header的parent字段保存header的seq号，从而建立了它们之间的父子联系。

□当读取一个子对象的属性时，如果子对象不存在该属性，需要去父对象获取。

可以看出，omap的clone机制也实现了copy-on-write机制。

# 7.5.3 部分代码实现分析

# 1. SequencerPosition

类SequencerPosition用来验证操作的顺序性，当操作执行时，后面的操作序号必须大于之前的操作序号。

```c
struct SequencerPosition {
    uint64_t seq; 日志的序号
    uint32_t trans; //一个日志内多个事务的序号
    uint32_t op; //一个事务内多个op的序号
```

# 2. lookup_create_map_header

本函数用于获取对象的header:

1）首先调用函数 lookup_map_header 查找对象的 header:

a）首先在 caches 里查找是否缓存。

b）调用底层KeyValueDB查找header

```c
int r = db->get(HOBJECT_TO_SEQ, map_header_key(old), &out); 
```

c）如果查找成功，就返回header对象，如果不成功，返回一个新创建的header对象。

2）调用函数 generate_new_header 来设置 Header 的字段，并调用函数 write_state 写入全部变量 state:

```c
header->seq = state seq++;
if (parent) {
    header->parent = parent->seq;
    header->spos = parent->spos;
} 
header->num_children = 1;
header->oid = oid; 
```

3）调用函数 set_map_header，把新的 header 设置到 LevelDB 中。

# 3. get_xattrs

本函数用于获取对象的属性，实现如下：

1）首先获取对应的Header头部。

2）调用db设置具体的数据：

Header header $=$ lookup_map_headerhl,oid); if(!header) return -ENOENT; return db->get(xattr_prefix(header)，to_get，out); 

# 4. set_keys

本函数用于设置对象的属性，实现如下：

1）获取 KeyValueDB::Transaction 的一个事务。

```cpp
KeyValueDB::Transaction t = db->get_transaction(); 
```

2）先获取对象的Header：

```txt
MapHeaderLock hl(this, oid);  
Header header = lookup_create_map_header.hl, oid, t);  
if (!header)  
return -EINVALID;  
if (check_spos(oid, header, spos))  
return 0; 
```

3）先调用事务 set 函数设置属性：

$\mathsf{t}\rightarrow \mathsf{set}$ (user_prefix(header)，set); 

4）调用db提交事务：

```txt
return db->submit_transaction(t) 
```

# 7.6 CollectionIndex

Collection 的概念对应到本地文件系统中就是一个目录，用于存储一个 PG 里的所有的对象。

一个 collection 对应本地文件系统的一个目录，一个 PG 对应于一个 Collection，该 PG 的所有对象都保存在这个目录里，定义在类 coll_t 中：

class coll_t{enum type_t{TYPE_meta $= 0$ ，TYPE_LEGACY_TEMP $= 1$ ，//这个类型不再使用TYPE_PG $= 2$ ，TYPE_PG_TEMP $= 3$ ·};type_t type; 类型meta，pg，tempspg_t pgid; 对应的pgiduint64_t removal_seq; //这个字段不再使用，没有编码持久化存储string_str; //缓存的字符串}；

collection有三种不同的类型：TYPE_Meta类型表示这个PG里保存的是元数据（meta）相关的对象，TYPE_PG表示该collection保存的是PG相关的数据，TYPE_PG_TEMP保存临时对象。

当一个PG的对象数量比较多时，就会在一个目录里保存大量的文件。对于底层文件系统来说，如果一个目录里保存大量文件，当达到一定的程度后，性能会急剧下降。那么就需要一个collection里对应多个层级的子目录来存储大量文件，从而提高性能。

图7-4为CollectionIndex的静态类图。IndexManager类为管理CollectionIndex的实现。指数Index实现了LFNIndex，LFNIndex实现了CollectionIndex接口。

![image](https://cdn-mineru.openxlab.org.cn/result/2026-04-30/8e4d380f-113d-46f2-8e2b-b0978fb78f78/5a7ad7c80a87986ca7798d6f174b486ce2e3dd6e532e96b36160c565b2416444.jpg)



图7-4CollectionIndex静态类图


# 7.6.1 CollectIndex 接口

CollectionIndex 使一个 Collection 里的对象保存在多层子目录中。类 CollectionIndex 是对象在文件系统中多层目录存储的接口。

通过接口说明，就可以看到其能提供的功能：

□查找一个对象，返回对象对应文件的存储路径：

```c
virtual int lookup(const ghobject_t &oid, //输入参数，要查找的对象  
IndexedPath *path, //输出参数，对象的路径  
int *hardlink  
) = 0;
```

□根据对象的路径，创建一个对象：

```typescript
virtual int created(
    const ghobject_t &oid, // 创建对象名
    const char *path // 创建对象路径
) = 0;
```

删除一个对象：

```typescript
virtual int unlink(
    const ghobject_t &oid // 要删除的对象
) = 0;
```

□分裂目录，当上次目录里保存的文件或者子目录达到一定数量，就需要分裂成两个目录：

```c
virtual int split( uint32_t match, //输入参数：符合本子目录的bit值 uint32_t bits, //输入参数：要检查的bit CollectionIndex* dest //输入参数：目标index集合 } { assert(0); return 0; }
```

□ 根据对象的hash值，按序列出对象：

```c
virtual int collection_list_partial(
const ghobject_t &start, //输入参数：对象的起始位置
const ghobject_t &end, //输入参数：对象的结尾
bool sort_bitwise, //输入参数：对象排序的顺序
int max_count, //输入参数：最大对象数目
vector<ghobject_t> *ls, //输出参数：对象列表
ghobject_t *next //输出参数：下一次的对象起始
) = 0;
```

从上述接口介绍可知，CollectionIndex 提供了对象到其对应文件保存的目录路径映射管理。

# 7.6.2 HashIndex

HashMap 是 CollectionIndex 的一个实现。HashMap 实现了用对象的 Hash 值做为对象存储的目录。

# 1. 对象保存目录方式

以对象的 HASH 值为基准，从低位到高位十六进制的字符保存。

例如对象 ghobject_t("object", CEPH_NO_SNAP, 0xA4CEE0D2) 的 Hash 值是: 0xA4CEE0D2, 如果当前保存的目录层级是 2 级, 那么该对象保存的目录为:

```txt
root/DIR_2/DIR_D 
```

如果是3级目录，那么该对象保存的目录就是：

```txt
root/DIR_2/DIR_D/DIR_0 
```

# 2. 目录层级

如何确定当前保存目录的层级呢？何时创建一个新的子目录呢？当一个目录中的对象数目超过如下值：

```txt
abs(merge_threshold)) * 16 * splitmultiplier 
```

就重新创建一个子目录，原来的子目录要分裂为两个目录。其中 merge_threshold 由配置选项 g_ceph_context->conf->filestore_merge_threshold 设置。split-multiplier 由 g_conf->filestore_split_multiple 设置。

一个目录保存对象的统计信息保存在目录的扩展属性中，数据结构 subdir_info_s 定义了相关的属性；

```c
struct subdir_info_s {
    uint64_t objs; //该目录中对象的数目
    uint32_t subdirs; //该目录中子目录的数目
```

```txt
uint32_t hash_level; //子目录的hash层级数
```

# 7.6.3 LFNIndex

HashMap 继承了 LFIndex 接口。LFIndex 是 Long File Name Index 的缩写。从名称就可以知道，LFIndex 用来处理如下情况：当对象名太长，超过了本地文件系统支持的长度时，LFXIndex 实现把超出的部分文件名保存到文件的扩展属性中。有可能保存到扩展属性的多个 key-value 存储对中。

# 7.7 本章小结

本章介绍本地对象存储的基本概念，以及 ObjectStore 对象存储接口，通过它可以了解对象存储如何调用。然后介绍了 Journal 的对外接口和 Filejournal 的实现，以及 Filestore 的更新操作。最后介绍了对象 omap 实现以及对象在本地文件系统中的组织方式。

目前对象存储是研究的热点。通过上述介绍可知，FileStore的日志方式的写操作都需要数据的两次写入：一次写日志；另一次写数据对象。对于S3接口的对象存储等应用场景，双写是没有必要的。社区推出了新的存储引擎BlueStore，其解决了某些场景下的避免双写，同时提高了性能。但目前BlueStore还不稳定，不能用于生产环境。

# 第8章

Chapter 8 

# Ceph纠删码

本章介绍Ceph纠删码（Erasure Code，EC）的实现。首先介绍纠删码的原理，并对几种不同的编码原理进行分析，然后介绍纠删码的具体实现。其实现过程大部分逻辑和副本类似，这里着重介绍能完成数据回滚操作的不同实现点。

# 8.1 EC的基本原理

纠删码（EC）是最近一段时间存储领域（特别是在云存储领域）比较流行的数据冗余存储的方法，它的原理和传统的RAID类似，但是比RAID方式更灵活。

它将写入的数据分成N份原始数据，通过这N份原始数据计算出M份效验数据。把 $\mathrm{N + M}$ 份数据分别保存在不同的设备或者节点中，并通过 $\mathrm{N + M}$ 份中的任意N份数据块还原出所有数据块。

EC包含了编码和解码两个过程：将原始的N份数据计算出M份效验数据称为编码过程；通过这 $\mathrm{N + M}$ 份数据中的任意N份数据来还原出原始数据的过程称为解码过程。EC可以容忍M份数据失效，任意小于等于M份的数据失效能通过剩下的数据还原出原始数据。

目前一些主流的云存储厂商都采用EC编码方式。Google GFS II中采用了最基本的RS（6，3）编码，Facebook的HDFS RAID的早期编码方式为RS（10，4）编码。微软的云存储系统Azure使用了的LRC（12，2，2）编码。

# 8.2 EC的不同插件

Ceph 支持以插件的形式来指定不同的 EC 编码方式。各种编码的不同点，实质就是在 ErasureCode 的三个指标之间折中的结果，这个三指标是：空间利用率、数据可靠性和恢复效率。

# 8.2.1 RS编码

目前应用最广泛的纠错码是 ReedSolomon 编码，简称 RS 码。这种编码在 1960 年出现了，过了很多年以后才提出了快速算法并广泛应用于各种通信系统中，直到近几年才逐渐应用于各种存储系统中。下面介绍 RS 编码的几个实现。

# 1. Jerasure

Jerasure 是一个 ErausreCode 开源实现库，它实现了 EC 的 RS 编码。目前 Ceph 中默认的编码就是 Jerasure 方式。

# 2.ISA

ISA是Intel提供的一个EC库，只能运行在IntelCPU上，它利用了Intel处理器本地指令来加速EC的计算。

RS编码的不足之处在于：在 $\mathrm{N} + \mathrm{K}$ 个数据块中有任意一块数据失效，都需要读取N块数据来恢复丢失数据。在数据恢复的过程中引起的网络开销比较大。因此，LRC编码和SHEC编码分别从不同的角度做了相关优化。

# 8.2.2 LRC编码

LRC编码的核心思想为：将校验块（parity block）分为全局校验块（global parity）和

局部校验块（local reconstruction parity），从而减少恢复数据的网络开销。其目标在于解决当单个磁盘失效后恢复过程的网络开销。

LRC（M,G,L）的三个参数分别为：

□M是原始数据块的数量。

□G为全局校验块的数量。

□L为局部校验块的数量。

编码过程为：把数据分成M个同等大小的数据块，通过该M个数据块计算出G份全局效验数据块。然后把M个数据块平均分成L组，每组计算出一个本地数据效验块，这样共有L个局部数据校验块。

下面以 Azure 的 LRC（12,2,2）和 Facebook 的 HDFS RAID 的早期编码方式 RS（10,4）为例来比较 LRC 和 RS 编码在恢复过程的开销。参见表 8-1.


表 8-1 LRC 编码举例及其与 RS 的比较


<table><tr><td>LRC (12,2,2)</td><td>RS (12+4)</td></tr><tr><td>D1 D2 D3 D4 D5 D6 D7 D8 D9 D10 D11 D12</td><td>D1 ~ D12 P1 ~ P4</td></tr><tr><td>L1 L2</td><td></td></tr><tr><td>G1 G2</td><td></td></tr></table>

表8-1所示对应LRC编码：总共有12个数据块，分别为 $\mathrm{D}_1\sim \mathrm{D}_{12}$ 。有两个本地数据校验块 $\mathbf{L}_1$ 和 $\mathbf{L}_2$ ， $\mathbf{L}_1$ 为通过第一组数据块 $\mathrm{D}_1\sim \mathrm{D}_6$ 计算而得的本地效验数据块； $\mathbf{L}_2$ 为第二组数据块 $\mathrm{D}_7\sim \mathrm{D}_{12}$ 计算而得的本地效验数据块。有2个全局数据效验块 $\mathbf{G}_1$ 和 $\mathbf{G}_2$ ，它是通过所有数据块 $\mathrm{D}_1\sim \mathrm{D}_{12}$ 计算而来。对应RS编码，数据块为 $\mathrm{D}_1\sim \mathrm{D}_{12}$ ，计算出的效验块为 $\mathrm{P_1}\sim \mathrm{P_{4}}$ 

不同情况下的数据恢复开销：

□如果数据块 $\mathrm{D}_1\sim \mathrm{D}_{12}$ 只有一个数据块损坏，LRC只需要读取6个额外的数据块来恢复。而RS需要读取12个其他的数据块来修复。

□如果 $\mathbf{L}_1$ 或者 $\mathbf{L}_2$ 其中一个数据块损坏，LRC需要读取6个数据块。如果 $\mathbf{G}_1$ ， $\mathbf{G}_2$ 其中一个数据损坏，LRC仍需要读取12个数据块来修复。

最大允许失效的数据块：

□ RS 允许数据块和校验块中任意的小于等于 4 个数据的失效。

□ 而对于LRC：

- 数据块中，只允许任意的小于等于2个数据块失效。

- 允许所有的效验块 $(\mathrm{G}_1, \mathrm{G}_2, \mathrm{L}_1, \mathrm{L}_2)$ 同时失效。

- 允许至多两个数据块和两个本地效验块同时失效。

综上分析可知：对于只有一个数据块失效，或者一个本地数据效验块失效的情况下，在恢复该数据块时，LRC比RS可以减少一半的磁盘IO和网络带宽。所以LRC重点在单个磁盘失效后恢复的优化。但是对于数据可靠性来说，通过最大允许失效的数据块个数的讨论可知，LRC会有一定的损失。

# 8.2.3 SHEC编码

SHEC 编码方式为 SHEC（K，M，L），其中 K 代表 data chunk 的数量，M 代表 parity chunk 的数量，L 代表计算 parity chunk 时需要的 data chunk 的数量。其最大允许失效的数据块为：ML/K。这样恢复失效的单个数据块只需要额外读取 L 个数据块。

下面以SHEC（10，6，5）为例，其最大允许失效的数据块为：

$$
M (6) * L (5) / K (1 0) = 3
$$

如图8-1所示的， $\mathrm{D}_1 \sim \mathrm{D}_{10}$ 位数据块， $\mathrm{P}_1$ 为数据块 $\mathrm{D}_1 \sim \mathrm{D}_5$ 计算出的校验块。 $\mathrm{P}_2$ 位 $\mathrm{D}_3 \sim \mathrm{D}_7$ 计算出的校验块。其他校验块的计算如图所示。当一个数据块失效时，只读取5个数据块就可以恢复。

![image](https://cdn-mineru.openxlab.org.cn/result/2026-04-30/8e4d380f-113d-46f2-8e2b-b0978fb78f78/688546753c5d8e6c415d7eb51c5b303f83b8107b801841a7be439f77bb99d6d5.jpg)



图8-1 SHEC编码示意图


# 8.2.4 EC和副本的比较

在 Ceph 中可以设置一个 pool 为 EC 类型，并可以设置 N 和 M 的参数。副本和各种 RC 的编码比较如表 8-2 所示。


表 8-2 副本和各种 RC 的编码比较


<table><tr><td></td><td>三副本</td><td>RS(10,4)</td><td>LRC(10,6,5)</td><td>SHEC(10,6,5)</td></tr><tr><td>数据容量开销</td><td>3X</td><td>1.4X</td><td>1.8X</td><td>1.6X</td></tr><tr><td>数据恢复开销(单个数据块失效)</td><td>1X</td><td>10X</td><td>5X</td><td>5X</td></tr><tr><td>可靠性</td><td>高</td><td>中</td><td>中</td><td>中下</td></tr></table>

说明如下：

□在三副本的情况下，恢复效率和可靠性都比较高，缺点就是数据容量开销比较大。

□对于EC的RS编码，和三副本比较，数据开销显著降低，以恢复效率和可靠性为代价。

□LRC编码以数据容量开销略高的代价，换取了数据恢复开销的显著降低。

□ SHEC编码用可靠性换代价，在LRC的基础上进一步降低了容量开销。

# 8.3 Ceph中EC的实现

# 8.3.1 Ceph中EC的基本概念

下面介绍一些EC的基本概念。注意，这里stripe的概念是指RADOS系统定义的，可能与其他系统的定义不同。

□ chunk：一个数据块就叫data chunk，简称chunk，其大小为chunk_size设置的字节数。

□ stripe：用来计算同一个校验块的一组数据块，称为 data stripe，简称 stripe，其大小为 stripe_width，参与的数据块的数目为 stripe_size，这几个概念的关系如下：

$$
\text {s t r i p e} = \text {c h u n k}
$$

如图8-2所示为一个EC（4+2）示例：stripe_size为4，chunk_size的大小为1K，那

么stripe_width的大小就为4K。在Ceph系统中，默认的stirpe_width就为4K。

![image](https://cdn-mineru.openxlab.org.cn/result/2026-04-30/8e4d380f-113d-46f2-8e2b-b0978fb78f78/752bd9f6fa9e7df4078cea14caac01e5b2da8f04fece589a08f7939d61914c1f.jpg)



图8-2 EC的分片示意图


# 8.3.2 EC支持的写操作

目前Ceph的EC方式的写入操作是有一定限制的，其目前只支持如下操作：

□ create object: 创建对象。

□remove object：删除对象。

□ write full: 写整个对象。

□ append write（stripe width aligned）：追加写入（限定追加操作的起始偏移以 stripe_width 对齐）。

目前 Ceph 只支持上述操作，而不支持 overwrite 操作，其主要有如下两个条件的限制：

由于编码和解码的过程都以stripe width 整块数据计算。

□EC在特殊场景需要回滚的机制。

所以，目前EC只支持append写操作中，写操作的起始偏移offset以stripe_width对齐的情况，如果end不是以stripe_width对齐，就补0对齐即可。

目前不支持以下情况：

□ 情况1：append写操作，写操作的起始偏移offset没有以stripe_width对齐。

□情况2：overwrite写操作，offset和end都不以stripe_width对齐。

由于计算数据校验块需要读取整个stripe的数据块。所在情况1和情况2都需要读取该stripe缺失的数据块，来计算效验块。由于性能的原因，目前不支持。

□情况3：overwrite写操作，写操作的起始偏移offset和结束位置end都以stripe_width对齐。

情况3目前也不支持，其原因是由EC的回滚的机制导致。下面将介绍EC的回滚机制。

# 8.3.3 EC的回滚机制

依据EC的原理可知，EC（N+M）的写操作如果有小于等于M个OSD失效，不会导致数据丢失，数据可恢复。EC在理论上就最多只能容忍M个OSD失效。如果OSD失效的数量大于M，这种情况超出了理论设计的范畴，系统无法处理这种情况。可以说这是合理的。

但是对于所有的存储系统，必须应对一种特殊的情况：整个机房或者整个数据中心全局断电，系统重启后可恢复，并且数据不丢失。

当存储系统全局断电时，其数据的写入状态就有可能出现：小于N个磁盘的数据成功写入，而其他磁盘没有写成功的情况。

以图8-2所示的EC（4+2）为例，假设写操作只有3个OSD写成功了，其他3个OSD没有来得及把数据写入磁盘。这种情况下，不但导致新数据写入失败，而且导致旧数据也无法读取成功。这就需要EC支持回滚机制，回滚到最后一次成功写入的旧数据版本。

Ceph目前支持的EC操作都是回滚比较容易实现的，实现机制如下：

□ create object 操作的回滚实现比较简单，删除该对象即可。

□对于remove object操作，在执行时并不删除该对象，而是暂时保留该对象；如果需要回滚，就可以直接恢复。

□writeFull操作：暂时保留旧的对象，创建一个新的对象完成写操作。当需要回滚时，恢复旧的数据对象。

□ append 操作：记录 append 时的 size 到 PG 日志中；当回滚时，对该对象做 truncate 操作即可。

# 8.4 EC的源代码分析

对应EC的上述三种更新操作，其本地回滚的信息都记录在对应的PG日志记录的mod_desc里：

```c
struct pg_log_entry_t {
    ......
    ObjectModDesc mod_desc;
}; 
```

在函数ReplicatedPG::do_osdOps中实现操作的事务封装，下面着重分析一下EC的写操作和write_full操作的实现。

# 8.4.1 EC的写操作

操作步骤如下：

1）首先验证如果是EC类型，写操作的offset必须以stripe_width对齐，否则不支持。

case CEPH_OSD_OP_WRITE: if (pool.inforequires_aligned_apply(& & (opextent-offset $\%$ pool.inforequired_apply() $! = 0$ ）{ result $=$ -EOPNOTSUPP; break; 

2）如果对象不存在，就在mod_desc中添加创建的信息，否则在mod_desc中添加old size的信息：

```javascript
ctx->mod_desc.create(); 
```

否则就是追加写：

```txt
ctx->mod_desc.append(oi.size); 
```

3）最后把写操作添加到事务中：

```javascript
if (pool.inforequire_rollback())
    t->append(soid, opextent-offset, opextent.length, osd_op.indata, op.flags); 
```

# 8.4.2 EC的write_full

操作步骤如下：

1）如果对象已经存在，调用函数ctx->mod_desc.rmobject，如果返回flase，说明已经记录了信息，直接删除；如果返回true，就调用stash保存旧的对象数据，用来恢复：

```lisp
case CEPH_OSD_OP_WRITEFULL:  
......  
if (obs.exits) {  
    if (ctx->mod_desc.rmobject(ctx->at_version(version))) {  
        t->stash(soid, ctx->at_version(version));  
    } else {  
        t->remove(soid);  
    }  
} 
```

2）在事务中写入数据：

```javascript
t->append(soid, 0, op.length.length, osd_op.indata, op.flags); 
```

# 8.4.3 ECBackend

类ECBackend实现了EC的读写操作。ECUtil里定义了编码和解码的函数实现。  
ECTransaction定了EC的事务。相关的代码都比较清晰，这里就不详细介绍了。

# 8.5 本章小结

本章介绍 Ceph 纠删码的编码原理，以及目前支持的操作和目前尚不支持其他操作的原因，并简单介绍了 EC 的代码实现。目前纠删码的研究是一个热点。它可以极大地提供存储利用率，降低存储成本。目前研究都在着力研究纠删码如何直接支持块存储，也就是随机 overwrite 操作的能力。

# Ceph 快照和克隆

本章介绍Ceph的高级数据功能：快照和克隆，它们在企业级的存储系统中是必不可少的。这里首先介绍Ceph中快照和克隆的基本概念，其次介绍快照实现相关的数据结构，然后介绍快照操作的原理，最后分析快照的读写操作的源代码实现。

# 9.1 基本概念

下面介绍快照和克隆的基本概念，以及二者之间的区别。

# 9.1.1 快照和克隆

快照是一个RBD在某一时刻全部数据的只读镜像。克隆是在某一时刻全部数据的可写镜像。快照和克隆都是某一时间点的镜像，区别在于快照只能读，而克隆可以写。

Ceph 支持两种类型的快照：一种 pool 级别的快照（pool snap），是给 pool 整体做一个快照；另一种是用户管理的快照（self managed snap）。目前 RBD 快照的实现就属于后者。用户的写操作必须自己提供 SnapContext 信息。注意，这两种快照是互斥的，两种快照不能同时存在。也就是说，如果对 pool 整体做了快照操作，就不能对该 pool 中的 RBD 做快

照操作。

无论是pool级别的快照，还是RBD的快照，其实现的基本原理都是相同的。都是基于对象的COW（copy-on-write）机制。Ceph可以完成秒级的快照操作和克隆操作。

这里需要特别指出的是，对象的clone操作指的是快照对应的克隆操作，是RADOS在OSD服务端实现的对象的拷贝。RBD的clone操作是RBD的客户端实现的RBD层面的克隆。它们俩不是一个概念，希望读者区分开来。

在具体的实现过程中，克隆依赖快照的实现，克隆是在一个快照的基础上实现了可写功能。

图9-1是快照和克隆的示意图，其生成过程如下所示：

1）首先创建一个RBD块设备：rbd1。

2）对该块设备rbd1创建一个快照snap1。

3）调用rbdprotect来保护该快照，则该快照就不能被删除。

4）从该快照中克隆出一个新的 image，其名字为 clone1。

![image](https://cdn-mineru.openxlab.org.cn/result/2026-04-30/8e4d380f-113d-46f2-8e2b-b0978fb78f78/1103c05332b07fc6bd05cf057cc61118e073196b73a9f1587c7f1ebd47181df3.jpg)



图9-1 快照和克隆示意图


一个 image 的数据对象和快照对象都在同一个 pool 中，每个 image 的对象和对应的快照对象都在相同 OSD 上的相同 PG 中。快照的对象拷贝都是在 OSD 本地进行。

# 9.1.2 RBD的快照和克隆比较

RBD的快照和克隆在实现层面完全不同。快照是RADOS支持的，基于OSD服务端的COW机制实现的。而RBD的克隆操作完全是RBD客户端实现的一种COW机制，对于OSD的Server端是无感知的。

怎么理解RBD的克隆操作是由RBD的客户端实现的？如图9-1所示的快照和克隆，对克隆的image的读写过程如下：

1）当对克隆 image，也就是 clone1 发起写操作时，客户端对应的 OSD 发送正常的写请求。

2）OSD返回给客户端应答，表明该OSD上对应的对象不存在。

3）客户端要发读请求到给克隆 image 的父 image，读取对应 snap 1 上的数据返回给客户端。

4）客户端把该快照数据写入克隆 image 中。

5）客户端给克隆 image 发送写操，写入实际要写入的数据。

由以上过程可知，克隆的拷贝操作是由客户端控制完成，OSD的Sever端配合完成普通的读写操作。

当克隆的层级比较多时，需要客户端不断递归到其父image上去读取对应的快照对象，这会严重影响克隆的性能。

如图9-2所示，当读写clone2时，客户端首先给clone2发送写请求，如果对象不存在，就需要向clone1发送读请求，读取clone1对应的snap2的快照数据，并写入clone2，然后完成实际的写入操作。

![image](https://cdn-mineru.openxlab.org.cn/result/2026-04-30/8e4d380f-113d-46f2-8e2b-b0978fb78f78/7407fdbaac0c855a04be2752901e04023e6a411bd8c5a7fa94ed49c071c35152.jpg)



图9-2 多层级的克隆


如果clone1上的对象不存在，同样，客户端继续递归，不断给给父image发送读请

求，读取snap1的快照数据，并写入clone1中。

如果层级过多就会影响克隆操作的性能。因此系统提供了RBD的flattern操作，可直接把父image的快照对象拷贝给克隆image，这样以后就不需要去向父image查找对象数据了，从而提高了性能。

# 9.2 快照实现的核心数据结构

快照的核心数据结构如下：

□ head对象：也就是对象的原始对象，该对象可以进行写操作。

□ snap 对象：对某个对象做快照后，通过 cow 机制 copy 出来的快照对象只能读，不能写。

□ snap_seq 或者 seq: 快照序号, 每次做 snapshot 操作系统都分配一个相应快照序号, 该快照序号在后面的写操作中发挥重要作用。

- snapdir 对象：当 head 对象被删除后，仍然有 snap 和 clone 对象，系统自动创建一个 snapdir 对象，来保存 SnapSet 信息。head 对象和 snapdir 对象只有一个存在，其属性都可以保存快照相关的信息。这主要用于文件系统的快照实现。

# 1. SnapContext

在文件 librados/IoCtxImpl.h 中定义了 snap 相关的数据结构：

```txt
struct SnapContext {
    snapid_t seq; //最新的快照序号
    vector<snapid_t> snaps; //当前存在的快照序号，降序排队
```

SnapContext数据结构用来在客户端（RBD端）保存snap相关的信息。这个结构持久化存储在RBD的元数据中：

```cpp
struct librados::IoCtxImpl {
    ...
    snapid_t snap_seq;
    ::SnapContext snapc;
} 
```

其中：

□ seq 为最新的快照序号。

□ snaps降序保存了该RBD的所有的快照序号。

数据结构 IoCtxImpl 里的 snap_seq 一般也称为快照的 id（snap_id）。当打开一个 image 时，如果打开的是一个卷的快照，那么 snap_seq 的值就是该 snap 对应的快照序号。否则 snap_seq 就为 CEPH_NOSNAP (-2)，来表示操作的不是卷的快照，而是卷自身。

# 2. SnapSet

数据结构 DataSet 用于保存 Server 端（也就是 OSD 端）与快照相关的信息：

```txt
struct SnapSet {
    snapid_t seq; //最新的快照序号
    bool head_existss; //head对象是否存储
    vector<snapid_t> snaps; //所有的快照序号列表（降序排列）
    vector<snapid_t> clones; //所有的clone对象序号列表（升序排列）
    map<snapid_t, interval_set<uint64_t> > clone_overlap;
    //和上次clone对象之间overlap的部分
    map<snapid_t, uint64_t> clone_size;
    //clone对象的size
}
```

下面是其中一些数据字段介绍：

□ seq 保存最新的快照序号。

□ head_exists 保存 head 对象是否存在。

□ snaps 保存所有的快照序号。

□ clones 保存所有快照后的写操作需要 clone 的对象记录。

这里特别强调的是 clones 和 snaps 的区别。由于不是每次做快照操作后，都要拷贝对象。只当快照操作后有写操作，才会触发相关对象的 clone 操作复制出一份新的对象，该对象是 clone 出来的，其快照序号记录在 clones 队列中，称为 clone 对象。

□clone_overlap保存本次clone对象和上次clone对象（或者head对象）的overlap的部分，也就是重叠的部分。clone操作后，每次写操作，都要维护这个信息。这个信息用于在数据恢复阶段对象恢复的优化。

□clone_size保存每次clone后的对象的size。

SnapSet数据结构持久化保存在head对象的xattr的扩展属性中：

□在Head对象的xattr中保存key为snapshot，value为Snapshot结构序列化后的值。

□在snap对象的xattr中保存key为user.cephos seq的snap_seq值。

# 9.3 快照的工作原理

# 9.3.1 快照的创建

RBD 快照创建的基本步骤如下：

1）向Monitor发送请求，获取一个最新的快照序号snap_seq的值。

2）把该次快照的snap_name和snap_seq的值保存到RBD的元数据中。

在RBD的元数据里保存了所有快照的名字和对应的snap_seq号，并不会触发OSD端的数据操作，所以非常快。

# 9.3.2 快照的写操作

当对一个 image 做了一次快照后，该 image 写入数据时，由于快照的存在需要启动 copy-on-write（cow）机制。下面将介绍 cow 机制的具体实现。

客户端的每次写操作，消息中都必须带数据结构SnapContext信息，它包含了客户端认为的最新快照序号seq，以及该对象的所有快照序号snaps的列表。在OSD端，对象的Snap相关信息保存在SnapSet数据结构中，当有写操作发生时，处理过程按照如下规则进行。

# 规则1

如果写操作所带SnapContext的seq值小于SnapSet的seq值，也就是客户端最新的快照序号小于OSD端保存的最新的快照序号，那么直接返回-EOLDSNAP错误。

Ceph 客户端始终保持最新的快照序号。如果客户端不是最新的快照序号，可能的情

况是：在多个客户端的情形下，其他客户端有可能创建了快照，本客户端有可能没有获取到最新的快照序号。

Ceph有一套 Watcher 回调通知机制来实现快照序号的更新。如果其他客户端对一个卷做了快照，就会产生了一个最新的快照序号。OSD 端接收到最新快照序号变化后，通知相应的连接客户端更新最新的快照序号。如果有客户端没有及时更新，也没有太大的问题，OSD 端会返回客户端 -EOLDSNAP，客户端会主动更新为最新的快照序号，重新发起写操作。

# 规则2

如果head对象不存在，创建该对象并写入数据，用SnapContext相应的信息更新SnapSet的信息。

# 规则3

如果写操作所带SnapContext的seq值等于SnapSet的seq值，做正常的读写。

# 规则4

如果写操作所带SnapContext的seq值大于SnapSet的seq值：

1）对当前head对象做copy操作，clone出一个新的快照对象，该快照对象的snap序号为最新的序号，并把clone操作记录在clones列表里，也就是把最新的快照序号加入到clones队列中。

2）用SnapContext的seq和snaps值更新SnapSet的seq和snaps值。

3）写入最新的数据到head对象中。

# 9.3.3 快照的读操作

快照读取数据时，输入参数为RBD的名字和快照的名字。RBD的客户端通过访问RBD的元数据，来获取快照对应的snap_id，也就是快照对应的snap_seq值。

在OSD端，获取head对象保存的SnapSet数据结构。然后根据SnapSet中的snaps

和 clones 值来计算快照所对应的正确的快照对象。

# 9.3.4 快照的回滚

快照的回滚，就是把当前的head对象回滚到某个快照对象。具体操作如下：

1）删除当前head对象的数据。

2）拷贝相应的snap对象到head对象。

其源代码的实现在ReplicatedPG::_rollback_to里。

# 9.3.5 快照的删除

删除快照时，直接删除rbd的元数据中保存的Snap相关快照信息，然后给Monitor发快照删除信息。Monitor随后给相应的OSD发送删除的快照序号，然后由OSD控制删除本地相应的快照对象。该快照是否被其他快照对象共享。

由上可知，Ceph的快照删除是延迟删除，并不是直接立即删除。

# 9.4 快照读写操作源代码分析

下面着重分析快照的写操作和读操作相关的源代码实现。

# 9.4.1 快照的写操作

在结构体OpContext的上下文中，保存了快照相关的信息：

```txt
struct OpContext {
    const SnapSet *snapshot; //旧的SnapSet，也就是OSD服务端保存的快照信息。
    ObjectState new_obs;
    SnapSet new_snapshot; //新的SnapSet
    SnapContext snapc; //写操作带的，也就是客户端的SnapContext信息
};
```

在读写的关键流程中，有关快照的处理如下：

1）在OSD写操作的流程中，在函数ReplicatedPG::executeCtx中，把消息带的SnapContext信息保存在了OpContext的snapc中：

```c
ctx->snapc seq = m->get_snap_seq();  
ctx->snapc.naps = m->get_snaps(); 
```

2）在OpContext的构造函数里，用结构snapshot字段初始化了结构new_snapshot的相关字段。当前new_snapshot保存的就是OSD服务端的快照信息：

```txt
if (obc->ssc) {
    new_snapshot = abc->ssc->snapshot;
    snapshot = &obc->ssc->snapshot;
} 
```

3）在函数ReplicatedPG::prepare_transaction里调用了函数ReplicatedPG::make_writeable来完成快照相关的操作。

# 9.4.2 make_writeable 函数

函数make_writeable处理快照相关的写操作，其处理流程如下：

1）首先判断，如果服务端的最新快照序号大于客户端的快照序号，就用服务端的快照信息更新客户端的快照信息：

```cpp
if (ctx->new_snapshot seq > snapc seq) {
    snapc seq = ctx->new_snapshot seq;
    snapc.saps = ctx->new_snapshot.saps;
    cout(10) << " using newer snapc << snapc << endl;
} 
```

在数据读写的流程中可知，在函数ReplicatedPG::executeCtx里已经判断了：客户端的最新快照序号不能小于服务端的快照序号，否则就直接返回-EOLDSNAPC错误码客户端更新快照序号后重试。所以笔者认为这段代码不会进入，所以是无用的代码。

2）调用函数filter_snapshot把已经删除的快照过滤掉。

3）如果head对象存在，并且snaps的size不为空（有快照），并且客户端的最新快照序号大于服务端的最新快照序号，在这种情况下要克隆对象，实现对象数据的拷贝了：

a）构造clone对象coid，其coid snap为最新的客户端seq值。

b）计算snaps列表，也就是本次克隆对象对应的所有快照。

c）构造clone_abc，也就是克隆对象coid的ObjectContext。特别需要指出的是该克隆对象的object_info_t中的snaps信息，就是在上一步中计算出的snaps列表。

d）调用函数.make Clone实现对象的克隆操作。此时克隆操作都先封装在新创建的事务t中：

```cpp
PGBackend::PGTransaction \*t = pgfrontend->get_transaction(); _make Clone(ctx, t, ctx->clone_abc, soid, coid, snap_oi); t->append(ctx->op_t); delete ctx->op_t; ctx->op_t = t; 
```

注意，之前的写操作封装在事务ctx->op_t中，把该事务追加到事务t的尾部，然后删除ctx->op_t事务，事务t赋值给ctx->op_t。这样在事务应用时，就是先做克隆操作，然后才完成写操作。

4）最后把该克隆对象添加到ctx->new_snapshot.clones中，并添加clone_size记录和clone_overlap记录。

5）根据当前的写操作修改范围 modifiedRanges，来计算修改 clone_overlap 的记录，也就是当前 head 对象和上次克隆对象的重叠区域，该信息用来优化快照对象的恢复。

6）更新服务端的快照信息为客户端的快照记录信息。

下面举例说明。

# 例9-1 快照写操作见表9-1。


表 9-1 写操作示例


<table><tr><td>操作序列</td><td>操作</td><td>RBD的元数据信息</td><td>OSD端对应的对象</td></tr><tr><td>1</td><td>write data1snapContext{seq=0,snaps={}}</td><td></td><td>SnapSet{seq=0,head_exists=true,snaps={},clones={},}obj1_head(data1)</td></tr><tr><td>2</td><td>create snapshotname="snap1"</td><td>("snap1",1)</td><td></td></tr><tr><td>3</td><td>write data2snapContext{seq=1,snaps={1}}</td><td></td><td>SnapSet{seq=1,head_exists=true,snaps={1},clones={1},{}obj1_1(data1)obj1_head(data2)</td></tr><tr><td>4</td><td>create snapshotname="snap3"</td><td>("snap1",1)( "snap3",3)</td><td></td></tr><tr><td>5</td><td>create snapshotname="snap6"</td><td>("snap1",1)( "snap3",3)( "snap6",6)</td><td></td></tr><tr><td>6</td><td>write data3snapContext{seq=6,snaps={6,3,1}}</td><td></td><td>SnapSet{seq=6,head_exists=true,snaps={6,3,1},clones={1,6},{}obj1_1(data1)obj1_6(data2)obj1_head(data3)</td></tr></table>

# 说明如下：

1）在操作1里为第一次写操作，写入的数据为data1，SnapContext的初始seq为0，snaps列表为空。按规则2，OSD端创建对象并写入对象数据，用SnapContext的数据更新SnapSet中的数据。

2）在操作 2 里，创建了该 RBD 一个快照，名字为 snap1，并向 Monitor 申请分配一个快照序号，其值为 1。在该卷的元数据里添加了快照的名字和对应的快照序号。

3）操作3里，写入数据data2，写操作所带SnapContext中的seq值为1，snap列表为{1}。在OSD端处理，此时SnapContext的seq大于SnapSet的seq，操作按照规则4：

a）更新SnapSet中的seq为1，snaps列表更新为 $\{1\}$ 值。

b）创建快照对象obj1_1，拷贝当前head对象的数据data1到快照对象obj1_1中（快照对象名字下划线后面为快照序号，Ceph目前快照对象的名字中含有快照序号）。此时快照对象obj1_1的数据为data1，并在clones中添加clone操作记

录，clone列表的值为 $\{1\}$ 

c）向head对象obj1_head中写入数据data2中。

4）操作4和操作5连续做了两次快照操作，快照的名字分别为snap3和snap6，分配的快照序号分别为3,6（在Ceph里，快照序号是由Monitor分配的，全局唯一，所以单个RBD的快照序号不一定连续）。

5）操作6写入数据data3，此时写操作所带SnapContext中的seq值为6，snaps值为 $\{6,3,1\}$ 共三个快照。此时SnapSet的seq为1，操作按规则4处理过程如下：

a) 更新 SnapSet 结构中的 seq 值为 6, snaps 值为 \{6,3,1\}。

b）创建快照对象obj1_6，拷贝当前head对象的数据data2到快照对象obj1_6中，并把本次克隆操作记录添加到clone队列中。更新后的clone队列的值为{1,6}。

c）向head对象obj1_head中写入数据data3。

# 9.4.3 快照的读操作

快照的读取操作核心在函数ReplicatedPG::find_object_context里实现，其原理是根据读对象的快照序号，查找实际对应的克隆对象的ObjectContext。基本的步骤如下：

1）如果对象的快照序号 oidsnap 大于服务端的最新快照序号 ssc->snapshot seq，获取 head 对象就是该快照对应的实际数据对象。

2）计算 oid snap 首次大于 ssc->snapshot.clones 列表中的克隆对象，就是 oid 对应的克隆对象。

例如在例9-1中，最后的SnapSet为：

```txt
SnapSet = {
    seq=6,
    snaps={6,3,1},
    clones={1,6},
    ...
} 
```

这时候读取 seq 为 3 的快照，由于 seq 为 3 的快照并没有写入数据，也就没有对应的克隆对象，通过计算可知，seq 为 3 的快照和 snap 为 1 的快照对象数据是一样的，所以就读取 obj1_1 对象数据。

# 9.5 本章小结

Ceph 的基于 Copy-on-Write 的机制实现了秒级别的快照，其高效率的核心原理在于做快照操作时不会直接拷贝数据，而是只做了快照的记录，当只有实际的写操作发生时，才实现对象的拷贝操作。实质就是把整个卷的拷贝操作开销分散到后续每次写操作过程中，这样就实现了快照操作只是增加了新的快照记录，所以快照操作可以在秒级实现。

# Ceph Peering 机制

本章介绍Ceph中比较复杂的模块：Peering机制。该过程保障PG内各个副本之间数据的一致性，并实现PG的各种状态的维护和转换。本章首先介绍boost库的statechart状态机基本知识，Ceph使用它来管理PG的状态转换。其次介绍PG的创建过程以及相应的状态机创建和初始化。然后详细介绍Peering机制三个具体的实现阶段：GetInfo、GetLog、GetMissing。

# 10.1 statechart状态机

Ceph 在处理 PG 的状态转换时，使用了 boost 库提供的 statechart 状态机。因此先简单介绍一下 statechart 状态机的基本概念和涉及的相关知识，以便更好地理解 Peering 过程中 PG 的状态机转换流程。下面在举例时截取了 PG 状态机的部分代码。

# 10.1.1 状态

在statechart里，一个状态的定义方式有两种：

□没有子状态情况下的状态定义：

struct Reset : boost::statechart::state< Reset, RecoveryMachine > 

这里定义了状态 Reset，它需要继承 boost::statechart::state 类。该类的模板参数中，第一个参数为状态自己的名字 Reset，第二个参数为该状态所属状态机的名字，表明 Reset 是状态机 RecoveryMachine 的一个状态。

□有子状态情况下的状态定义：

struct Started : boost::statechart::state< Started, RecoveryMachine, Start > 

状态 Started 也是状态机 RecoveryMachine 的一个状态，模板参数中多一个参数 Start，它是状态 Started 的默认初始子状态，其定义如下：

struct Start : boost::statechart::state< Start, Started > 

这里定义的 Start 是状态 Started 的子状态。第一个模板参数是自己的名字，第二个模板参数是该子状态所属父状态的名字。

综上所述，一个状态，要么属于一个状态机，要么属于一个状态，成为该状态的子状态。其定义的模板参数，第一个参数是自己，第二个参数是拥有者，第三个参数是它的起始子状态。

# 10.1.2 事件

状态能够接收并处理事件。事件可以改变状态，促使状态发生转移。在boost库的statechart状态机中定义事件的方式如下所示：

struct QueryState : boost::statechart::event<QueryState> 

QueryState 为一个事件，需要继承 boost::statechart::event 类，模板参数为自己的名字。

# 10.1.3 状态响应事件

在一个状态内部，需要定义状态机处于当前状态时可以接受的事件以及如何处理这些事件的方法：

```c
struct Initial : boost::statechart::state<Initial, RecoveryMachine>, NamedState {
    typedef boost::mpl::list < 

```cpp
boost::statechart::transition< Initialize, Reset >,
boost::statechart::custom Reaction< Load >,
boost::statechart::custom Reaction< NullEvt >,
boost::statechart::transition< boost::statechart::event_base, Crashed >
> reactions;
boost::statechart::result react(const Load&);
boost::statechart::result react(const MNotifyRec&);
boost::statechart::result react(const MInfoRec&);
boost::statechart::result react(const MLogRec&);
boost::statechart::result
    react(const boost::statechart::event_base&)
return discard_event();
} 
```

上述代码列出了状态RecoveryMachine/Initial可以处理的事件列表和处理对应事件方法：

□通过boost::mpl::list定义该状态可以处理多个事件类型。在本例中可以处理Initialize、Load、NullEvt以及event_base事件。

□简单事件处理：

```rust
boost::statechart::transition< Initialize, Reset > 
```

定义了状态 Initial 接收到事件 Initialize 后，无条件直接跳转到 Reset 状态。

用户自定义事件处理：当接收到事件后，需要根据一些条件来决定状态如何转移这个逻辑需要用户自己定义实现：

```cpp
boost::statechart::custom_reaction< Load > 
```

custom_reaction 定义了一个用户自定义的事件处理方法，必须有一个 react 的处理函数处理对应该事件。状态转移的逻辑需要用户自己在 react 函数里实现：

```txt
boost::statechart::result react(const Load&) 
```

- NullEvt事件用户自定义处理，但是没有实现react函数来处理，最终事件匹配了boost::statechart::event_base事件，直接调用函数 discard_event把事件丢弃掉。

# 10.1.4 状态机的定义

RecoveryMachine 为定义的状态机，需要继承 boost::statechart::state-machine 类。

```txt
class RecoveryMachine : public boost::statechart::state-machine<RecoveryMachine, Initial> 
```

模板参数第一个参数为自己的名字，第二个参数为状态机默认的初始状态 Initial。

状态机的基本操作有两个：

□ 状态机的初始化：

```txt
machine.initiate() 
```

□函数process_event用来向状态机投递事件，从而触发状态机接收并处理该事件：

```javascript
machine.process_event(evt); 
```

# 10.1.5 context函数

context是状态机的一个比较有用的函数，它可以获取当前状态的所有祖先状态的指针。通过它可以获取父状态以及祖先状态的一些内部参数和状态值。

例如状态 Start 是 RecoveryMachine 的一个状态，状态 Started 是 Start 状态的一个子状态，那么如果当前状态是 Started，就可以通过该函数获取它的父状态 Start 的指针：

```txt
Start* parent = context<Start>(); 
```

同时也可以获取其祖先状态RecoveryMachine的指针：

RecoveryMachine\* grandparent $=$ context<RecoveryMachine $\rightharpoondown$ ( 

综上所述，context函数为获取当前状态的祖先状态上下文提供了一种方法。

# 10.1.6 事件的特殊处理

事件除了在状态转移列表中触发状态转移，或者进入用户自定义的状态处理函数，还可以有下列特殊的处理方式：

□在用户自定义的函数里，可以直接调用函数transit来直接跳转到目标状态。例如：

```txt
transit<WaitRemoteBackfillReserved>() 
```

可以直接跳转到状态WaitRemoteBackfillReserved。

□在用户自定义的函数里，可以调用函数post_event直接产生相应的事件，并投递给状态机。

□在用户自定义的函数里，调用函数 discard_event可以直接丢弃事件，不做任何处理。

□在用户自定义的函数里，调用函数forward_event可以把当前事件继续投递给状态机。

# 10.2 PG状态机

在类PG的内部定义了类RecoveryState，该类RecoveryState的内部定义了PG的状态机RecoveryMachine和它的各种状态。

在每个PG对象创建时，在构造函数里创建一个新的RecoveryState类的对象，并创建相应的RecoveryMachine类的对象，也就是创建了一个新的状态机。每个PG类对应一个独立的状态机来控制该PG的状态转换。

图10-1为PG状态机的总体状态转换图，相对比较复杂，在介绍相关的内容模块时再逐一详细介绍。

# 10.3 PG的创建过程

在PG的创建过程中完成了PG对应的状态机的创建和状态机的初始化操作。一个PG的创建过程会由其所在OSD在PG中担任的角色不同，创建的机制也不相同。

# 10.3.1 PG在主OSD上的创建

当创建一个 Pool 时，通过客户端命令行给 Monitor 发送创建 Pool 的命令，Monitor 对该 Pool 的每一个 PG 对应的主 OSD 发送创建 PG 的请求。

函数OSD::handle_pg_create用于处理Monitor发送的创建PG的请求，其消息类型为MOSDPGCreate的数据结构：

```txt
struct MOSDPGCreate : public Message {
    version_t epoch;
    map<pg_t,pg_create_t> mkpg; // 要创建的PG列表，一次可以创建多个PG
```

![image](https://cdn-mineru.openxlab.org.cn/result/2026-04-30/8e4d380f-113d-46f2-8e2b-b0978fb78f78/3fe729cb6bcd0e7c8515091537a5a20dd9010a0980cf5fc6889979bd15790d12.jpg)



图10-1 PG状态机的总体状态转换图


```txt
map<pg_t,utime_t> ctimes; //对应PG的创建时间
```

数据结构 pg_create_t 包含了一个 PG 创建相关的参数：

```txt
struct pg_create_t {
    epoch_t created; //创建了 epoch pg
    pg_t parent; //如果 parent 不为空 (if != pg_t(), 本 PG 不是从 parent 中分裂出来的位
    __s32 split_bits;
}
```

函数 handle_pg_create 的处理过程如下：

1）首先调用函数 require_mon_peer 确保是由 Monitor 发送的创建消息。

2）调用函数 require_same_or_newer_map 检查 epoch 是否一致。如果对方的 epoch 比自己拥有的新，就更新自己的 epoch；否则就直接拒绝该请求。

3）对消息中mkpg列表里每一个PG，开始执行如下创建操作：

a）检查该PG的参数split_bits，如果不为0，那么就是PG的分裂请求，这里不做处理；检查PG的preferred，如果设置了，就跳过，目前不支持；检查确认该pool存在；检查本OSD是该PG的主OSD；如果参数up不等于acting，说明该PG有temp_pg，至少确定该PG存在，直接跳过。

b）调用函数 _have_pg 获取该 PG 对应的类。如果该 PG 已经存在，跳过。

c）调用函数PG::create在本地对象存储中创建相应的collection。

d）调用函数create_lock_pg初始化PG。

e）调用函数 pg->handle_create(&rctx) 给新创建 PG 状态机投递事件，PG 的状态发生相应的改变，后面会介绍。

f) 所有修改操作都打包在事务 rctx.transaction 中，调用函数 dispatch_context 将事务提交到本地对象存储中。

4）调用函数 maybe_update-heartbeat_peers 来更新 OSD 的心跳列表。

# 10.3.2 PG在从OSD上的创建

Monitor并不会给PG的从OSD发送消息来创建该PG，而是由该主OSD上的PG在Peering过程中创建。主OSD给从OSD的PG状态机投递事件时，在函数handle_pg

peeringevt中，如果发现该PG不存在，才完成创建该PG。

函数 handle_pg_peeringevt 是处理 Peering 状态机事件的入口。该函数会查找相应的 PG，如果该 PG 不存在，就创建该 PG。该 PG 的状态机进入 RecoveryMachine/Stray 状态。

# 10.3.3 PG的加载

当OSD重启时，调用函数OSD::init()，该函数调用load_pgs函数加载已经存在的PG，其处理过程和创建PG的过程相似。

# 10.4 PG 创建后状态机的状态转换

图10-2为PG总体状态转换图的简化版：状态Peering、Active、RelicaActive的内部状态没有添加进去。

通过该图可以了解PG的高层状态转换过程，如下所示：

1）当PG创建后，同时在该类内部创建了一个属于该PG的RecoveryMachine类型的状态机，该状态机的初始化状态为默认初始化状态Initial。

2）在PG创建后，调用函数pg->handle_create(&rctx)来给状态机投递事件：

```cpp
void PG::handle_create(RecoveryCtx *rctx) {
    cout(10) << "handle_create" << endl;
    Initialize EVT;
    recovery_state.handle_event(evt, rctx);
    ActMap EVT2;
    recovery_state.handle_event(evt2, rctx);
} 
```

由以上代码可知：该函数首先向RecoveryMachine投递了Initialize类型的事件。由图10-2可知，状态机在RecoveryMachine/Initial状态接收到Initialize类型的事件后直接转移到Reset状态。其次，向RecoveryMachine投递了ActMap事件。

3）状态Reset接收到ActMap事件，跳转到Started状态。

```cpp
boost::statechart::result PG::RecoveryState::Reset::react(const ActMap&){ 
```

```javascript
return transit< Started >(); 
```

![image](https://cdn-mineru.openxlab.org.cn/result/2026-04-30/8e4d380f-113d-46f2-8e2b-b0978fb78f78/d35674ec7bc8a4c4794a3b65b56fc4a16f0397bfdf2dbf93c057aae8b56f89f3.jpg)



图10-2 PG总体状态图的简化版


在自定义的 react 函数里直接调用了 transit 函数跳转到 Started 状态。

4）进入状态RecoveryMachine/Started后，就进入RecoveryMachine/Started的默认的子状态RecoveryMachine/Started/Start中：

PG::RecoveryState::Start::Start(my_context ctx) : my_base(ctx), NamedState(context<RecoveryMachine $\rightharpoondown$ ).pg->cct，"Start")   
{ context<RecoveryMachine $\rightharpoonup$ ).log-enter(state_name); PG \*pg $=$ context<RecoveryMachine $\rightharpoondown$ ).pg; if (pg->is_primary()) { cout(1) $<<$ "transitioning to Primary" $<<$ endl; post_event(MakePrimary()); } else { //is_stray cout(1) $<<$ "transitioning to Stray" $<<$ endl; post_event(MakeStray()); } 

由以上代码可知，在 Start 状态的构造函数中，根据本 OSD 在该 PG 中担任的角色不

同分别进行如下处理：

□如果是主OSD，就调用函数post_event，抛出事件MakePrimary，进入主OSD的默认子状态Primary/Peering中。

□如果是从OSD，就调用函数post_event，抛出事件MakeStray，进入Started/Stray状态。

对于一个PG的OSD处于Stray状态，是指该OSD上的PG副本目前状态不确定，但是可以响应主OSD的各种查询操作。它有两种可能：一种是最终转移到状态ReplicatActive，处于活跃状态，成为PG的一个副本。另一种可能的情况是：如果是数据迁移的源端，可能一直保持Stray状态，该OSD上的副本可能在数据迁移完成后，PG以及数据就都被删除了。

# 10.5 Ceph的Peering过程分析

在介绍了 statechar 状态机和 PG 的创建过程后，正式开始 Peering 过程介绍。Peering 的过程使一个 PG 内的 OSD 达成一个一致状态。当主从副本达成一个一致的状态后，PG 处于 active 状态，Peering 过程的状态就结束了。但此时该 PG 的三个 OSD 的数据副本上的数据并非完全一致。

PG在如下两种情况下触发Peering过程。

□当系统初始化时，OSD重新启动导致PG重新加载，或者PG新创建时，PG会发起一次Peering的过程。

□当有OSD失效，OSD的增加或者删除等导致PG的acting set发生了变化，该PG就会重新发起一次Peering过程。

# 10.5.1 基本概念

# 1. acting set 和 up set

acting set 是一个 PG 对应副本所在的 OSD 列表，该列表是有序的，列表中第一个 OSD 为主 OSD。在通常情况下，up set 和 acting set 列表完全相同。要理解它们的不同之

处，需要理解下面介绍的“临时PG”概念。

# 2. 临时PG

假设一个PG的acting set为[0,1,2]列表。此时如果osd0出现故障，导致CRUSH算法重新分配该PG的acting set为[3,1,2]。此时osd3为该PG的主OSD，但是osd3为新加入的OSD，并不能负担该PG上的读操作。所以PG向Monitor申请一个临时的PG，osd1为临时的主OSD，这时up set变为[1,3,2]，acting set依然为[3,1,2]，导致acting set和up set不同。当osd3完成Backfill过程之后，临时PG被取消，该PG的up set修复为acting set，此时acting set和up set都为[3,1,2]列表。

# 3. 权威日志

权威日志（在代码里一般简写为olog）是一个PG的完整顺序且连续操作的日志记录。该日志将作为数据修复的依据。

# 4.up_thru

引入up_thru的概念是为了解决特殊情况：当两个以上的OSD处于down状态，但是Monitor在两次epoch中检测到了这种状态，从而导致Monitor认为它们是先后宕掉。后宕的OSD有可能产生数据的更新，导致需要等待该OSD的修复，否则有可能产生数据丢失。

# 例10-1 up_thru处理过程

下图为初始情况：

<table><tr><td>epoch</td><td colspan="2">处于up的OSD</td></tr><tr><td>1</td><td>A</td><td>B</td></tr><tr><td>2</td><td></td><td>B</td></tr><tr><td>3</td><td></td><td></td></tr><tr><td>4</td><td>A</td><td></td></tr></table>

过程如下所示：

1）在 epoch1 时，一个 PG 中有 A、B 两个 OSD（两个副本）都处于 up 的状态。

2）在 epoch2 时，Monitor 检测到了 A 处于 down 状态，B 仍然处于 up 状态。由于 Monitor 的检测可能滞后，实际可能有两种情况：

情况1：此时B其实也已经和A同时宕了，只是Monitor没有检测到。此时PG不可能完成PG的Peering过程，PG没有新数据写入。

情况2：此时B确实处于up状态，由于B上保持了完整的数据，PG可以完成Peering过程并处于active的状态，可以接受新的数据写操作。

上述两种不同的情况，Monitor无法区分。

3）在 epoch3 时，Monitor 检测到 B 也宕了。

4）在 epoch4 时，A 恢复了 up 的状态后，该 PG 发起 Peering 过程，该 PG 是否允许完成 Peering 过程处于 active 状态，可以接受读写操作？

- 如果在 epoch2 时，属于情况 1：PG 并没有数据更新，B 上不会新写入数据，A 上的数据保存完整，此时 PG 可以完成 Peering 过程从而处于 active 状态，接受写操作。

- 如果在 epoch2 时，属于情况 2：PG 上有新数据更新到了 osd B，此时 osd A 缺失一些数据，该 PG 不能完成 Peering 过程。

为了使Monitor能够区分上述两种情况，引入了up_thru的概念，up_thru记录了每个OSD完成Peering后的epoch值。其初始值设置为0。

在上述情况2，PG如果可以恢复为active状态，在Peering过程，须向Monitor发送消息，Monitor用数组up_thru[osd]来记录该OSD完成Peering后的epoch值。

当引入up_thru后，上述例子的处理过程如下：

<table><tr><td>epoch</td><td colspan="2">处于up的OSD</td><td>monitor up_thru</td></tr><tr><td>1</td><td>A</td><td>B</td><td></td></tr><tr><td>2</td><td></td><td>B</td><td>up_thru[B]=0</td></tr><tr><td>3</td><td></td><td></td><td></td></tr><tr><td>4</td><td>A</td><td></td><td></td></tr></table>

情况1的处理流程如下：

1）在 epoch1 时，up_thru[B] 为 0，也就是说 B 在 epoch 为 0 时参与完成 Peering。

2）在 epoch2 时，Monitor 检查到 OSD A 处于 down 状态，OSD B 仍处于 up 状态（实际 B 已经处于 down 状态），PG 没有完成 Peering 过程，不会向 Monitor 上报更新 up_thru 的值。

3）epoch3时，A和B两个OSD都宕了。

4）epoch4时，A恢复up状态，PG开始Peering过程，发现up_thru[B]为0，说明在epoch为2时没有更新操作，该PG可以完成Peering过程，PG处于active状态。

情况2的处理如下所示：

<table><tr><td>epoch</td><td colspan="2">处于up的OSD</td><td>monitor up_thru</td></tr><tr><td>1</td><td>A</td><td>B</td><td></td></tr><tr><td>2</td><td></td><td>B</td><td>up_thru[B]=0</td></tr><tr><td>3</td><td></td><td>B</td><td>up_thru[B]=2</td></tr><tr><td>4</td><td></td><td></td><td></td></tr><tr><td>5</td><td>A</td><td></td><td></td></tr></table>

情况2的处理流程如下：

1）在 epoch1 时，up_thru[B] 为 0，也就是说 B 在 epoch 为 0 时参与完成 Peering 过程。

2）在 epoch2 时，Monitor 检查到 OSD A 处于 down 状态，OSD B 还处于 up 状态，该 PG 完成了 Peering 过程，向 Monitor 上报 B 的 up_thru 变为当前 epoch 的值为 2，此时 PG 可接受写操作请求。

3）在epoch4时，A和B都宕了，B的up_thru为2。

4）在 epoch5 时，A 处于 up 状态，开始 Peering 过程，发现 up_thru[B] 为 2，说明在 epoch 为 2 时完成了 Peering，有可能有更新操作，该 PG 需要等待 B 恢复。否则可能丢失 B 上更新的数据。

# 10.5.2 PG日志

PG日志（pglog）为一个PG内所有更新操作的记录（下文所指的日志，如不特别指出，都是指PG日志）。每个PG对应一个PG日志，它持久化保存在每个PG对应pgmeta_oid对象的omap属性中。

它有如下的特点：

□记录一个PG内所有对象的更新操作元数据信息，并不记录操作的数据。

□是一个完整的日志记录，版本号是顺序的且连续的。

# 1. pg_log_t

结构体 pg_log_t 在内存中保存了该 PG 的所有操作日志，以及相关的控制结构。

```txt
struct pg_log_t {
    eversion_t head; //日志的头，记录最新的日志记录
    eversion_t tail; //日志的尾，记录最旧的日志记录
```

```txt
eversion_t can_rollback_to; //用于EC，指示本地可以回滚的版本，可回滚的版本都大于版本can_rollback_to的值
eversion_t rollback_info_trimmed_to; //在EC的实现中，本地保留了不同版本的数据。本数据段指示本PG里可以删除掉的对象版本
list<pg_log_entry_t> log; //所有日志的列表
```

需要注意的是，PG日志的记录是以整个PG为单位，包括该PG内所有对象的修改记录。

# 2. pg_log_entry_t

结构体 pg_log_entry_t 记录了 PG 日志的单条记录，其数据结构如下：

```c
struct pg_log_entry_t {
    __s32 op; // 操作的类型
    hobject_t soid; // 操作的对象
    eversion_t version, // 本次操作的版本
    prior_version, // 前一个操作的版本
    reverting_to; // 本次操作回退的版本（仅用于回滚操作）
ObjectModDesc mod_desc; // 用于保存本地回滚的一些信息，用于EC模式下的回滚操作
bufferlist snaps; // 克隆操作用于记录当前对象的snap列表
osd_reqid_t reqid; // 请求唯一标识（called + tid）
vector <pair<osd_reqid_t, version_t> > extra_reqids;
version_t user_version; // 用户的版本
utime_t mtime; // 这是用户本地时间
```

# 3. IndexedLog

类IndexedLog继承了类pg_log_t，在其基础上添加了根据一个对象来检索日志的功能，以及其他相关的功能。

# 4. 日志的写入

函数PG::add_log_entry添加pg_log_entry_t条目到PG日志中。同时更新了info.lastcomplete和info.last_update字段。

PGLog::write_log函数将日志写到对应的pgmeta_oid对象的kv存储中。在这里并没有直接写入磁盘，而是先把日志的修改添加到ObjectStore::Transaction类型的事务中，与数据操作组成一个事务整体提交磁盘。这样可以保证数据操作、日志更新及其pg info信息的更新都在一个事务中，都以原子方式提交到磁盘上。

# 5. 日志的trim操作

函数trim用来删除不需要的旧日志。当日志的条目数大于min_log_entries时，需要进行trim操作。

```txt
voidPGLog::trim(LogEntryHandler\*handler,  
eversion_t trim_to，pg_info_t&info) 
```

# 6. 合并权威日志

函数 merge_log 用于把本地日志和权威日志合并：

```txt
void PLog::merge_log(ObjectStore::Transaction& t, pg_info_t &oinfo, pg_log_t &olog, pg_shard_t fromosd, pg_info_t &info, LogEntryHandler *rollbacker, bool &dirty_info, bool &dirty/big_info) 
```

其处理过程如下：

1）本地日志和权威日志没有重叠的部分：在这种情况下就无法依据日志来修复，只能通过Backfill过程来完成修复。所以先确保权威日志和本地日志有重叠的部分：

```javascript
assert(log.head >= olog.tail && olog.head >= log.tail); 
```

2）本地日志和权威日志有重叠部分的处理：

- 如果 olog.tail 小于 log.tail，也就是权威日志的尾部比本地日志长。在这种情况下，只要把日志多出的部分添加到本地日志即可，它不影响 missing 对象集合。

- 本地日志的头部比权威日志的头部长，说明有多出来的divergent日志，调用函数rewinddivergentlog去处理。

- 本地日志的头部比权威日志的头部短，说明有缺失的日志，其处理过程为：把缺失的日志添加到本地日志中，记录 missing 的对象，并删除多出来的日志记录。

下面举例说明函数 merge_log 的不同处理情况。

# 例10-2 函数merge_log应用举例

情况1：权威日志的尾部版本比本地日志的尾部小，如下所示：

<table><tr><td>log(本地日志)</td><td></td><td></td><td>obj10(1,6)modify log tail</td><td>obj11(1,7)modify</td><td>obj13(1,8)modify log_head</td></tr><tr><td>olog(权威日志)</td><td>obj3(1,4)modify log tail</td><td>obj4(1,5)modify</td><td>obj10(1,6)modify</td><td>obj11(1,7)modify</td><td>obj13(1,8)modify log_head</td></tr></table>

本地log的log.tail为obj10（1,6），权威日志olog的log.tail为obj3（1,4）。

日志合并的处理方式如下所示：

<table><tr><td>log(本地日志)</td><td>obj3(1,4)modify logtail</td><td>obj4(1,5)modify</td><td>obj10(1,6)modify logtail</td><td>obj11(1,7)modify</td><td>obj13(1,8)modify</td></tr><tr><td>olog(权威日志)</td><td>obj3(1,4)modify logtail</td><td>obj4(1,5)modify</td><td>obj10(1,6)modify</td><td>obj11(1,7)modify</td><td>obj13(1,8)modify</td></tr></table>

把日志记录obj3（1,4）、obj4（1,5）添加到本地日志中，修改info.log.tail和log.tail指针即可。

情况2：本地日志的头部版本比权威日志长，如下所示：

<table><tr><td>olog</td><td>obj10(1,6)modify</td><td>obj11(1,7)modify</td><td>obj13(1,8)modifylog_head</td><td></td><td></td><td></td></tr><tr><td>log</td><td>obj10(1,6)modify</td><td>obj11(1,7)modify</td><td>obj13(1,8)modify</td><td>obj13(1,9)modify</td><td>obj11(1,10)modify</td><td>obj10(1,11)delete log_head</td></tr></table>

权威日志的log_head为obj13（1,8），而本地日志的log_head为obj10(1,11)。本地日志的log_head版本大于权威日志的log_head版本，调用函数rewind_divergent_log来处理本地有分歧的日志。

在本例的具体处理过程为：把对象 obj10、obj11、obj13 加入 missing 列表中用于修复。最后删除多余的日志，如下所示：

<table><tr><td>olog</td><td>obj10(1,6)modify</td><td>obj11(1,7)modify</td><td>obj13(1,8)modifylog_head</td><td></td><td></td><td></td></tr><tr><td>log</td><td>obj10(1,6)modify</td><td>obj11(1,7)modify</td><td>obj13(1,8)modifylog_head</td><td>obj13(1,9)modify</td><td>obj11(1,10)modify</td><td>obj10(1,11)delete log_head</td></tr><tr><td colspan="7">missing object: obj10 (1,6) obj11(1,7) obj13(1,8)</td></tr></table>

本例比较简单，函数rewind_divergent_log会处理比较复杂的一些情况，后面会介绍到。

情况3：本地日志的头部版本比权威日志的头部短，如下所示：

<table><tr><td>olog</td><td>obj10(1,6)modify</td><td>obj11(1,7)modify</td><td>obj13(1,8)modify</td><td>obj13(1,9)modify</td><td>obj11(1,10)modify</td><td>obj10(1,11)delete log_head</td></tr><tr><td>log</td><td>obj10(1,6)modify</td><td>obj11(1,7)modify</td><td>obj13(1,8)modifylog_head</td><td></td><td></td><td></td></tr></table>

权威日志的log_head为obj10（1,11），而本地日志的log_head为obj13(1,8)，即本地日志的log_head版本小于权威日志的log_head版本。

其处理方式如下：把本地日志缺失的日志添加到本地，并计算本地缺失的对象。最后把缺失的对象添加到 missing object 列表中用于后续的修复，处理结果如下所示：

<table><tr><td>olog</td><td>obj10(1,6) modify</td><td>obj11(1,7) modify</td><td>obj13(1,8) modify</td><td>obj13(1,9) modify</td><td>obj11(1,10) modify</td><td>obj10(1,11) delete log_head</td></tr><tr><td>log</td><td>obj10(1,6) modify</td><td>obj11(1,7) modify</td><td>obj13(1,8) modify</td><td>obj13(1,9) modify</td><td>obj11(1,10) modify</td><td>obj10(1,11) delete log_head</td></tr><tr><td colspan="3">missing object</td><td colspan="4">obj13(1,9) obj11(1,10)</td></tr></table>

# 7. 处理副本日志

函数 proc Replica_log 用于处理其他副本节点发过来的和权威日志有分叉（divergent）的日志。其关键在于计算 missing 的对象列表，也就是需要修复的对象，如下所示：

```cpp
void PLog:::proc_replica_log(ObjectStore::Transaction& t, pg_info_t &oinfo, const pg_log_t &olog, pg_missing_t& o missing, pg_shard_t from) const 
```

其参数都为远程节点的信息：

□ oinfo：远程节点的 pg_info_t。

□olog：远程节点的日志。

□omissing：远程节点的missing信息，为输出信息。

□ from: 来自远程节点。

函数 proc Replica_log 的具体处理过程如下：

1）如果日志不重叠，就无法通过日志来修复，需要进行Backfill过程，直接返回。

2）如果日志的head相同，说明没有分歧日志（divergent log），直接返回。

3）下面处理的都是这种情况：日志有重叠并且日志的head不相同，需要处理分歧的日志：

- 计算第一条分歧日志 first_non_divergent，从本地日志后往前查找小于等于 olog.head 的日志记录。

- 版本 lu 为分歧日志的边界。如果 first_non_divergent 没有找到，或者小于权威日志的 logTAIL，那么 lu 就设置为 logTAIL，否则就设置为 first_non_divergent 日志记录的版本。

- 把所有的分歧日志都添加到divergent队列里。

- 构建一个IndexedLog的对象folog，把所有没有分歧的日志添加到folog里。

- 调用函数 merge_divergent_entries 处理分歧日志。

- 更新oinfo的last_update为lu版本。

- 如果有对象 missing，就设置 last_COMPLETE 为小于 first_missing 的版本。

函数_merge_divergent_entries 处理所有的分歧日志，首先把所有分歧日志的对象按照对象分类，然后分别调用函数 merge_object_divergent_entries 对每个分歧日志的对象进行处理。

函数_merge_object_divergent_entries 用于处理单个对象的 divergent 日志，其处理过程如下：

1）首先进行比较，如果处理的对象hoid大于info.last_backfill，说明该对象本来就不存在，没有必要修复。

# 注意

这种情况一般发生在如下情景：该PG在上一次Peering操作成功后，PG还没有处理clean状态，正在Backfill过程中，就再次触发了Peering的过程。info.last_backfill为上次最后一个修复的对象。

在本PG完成Peering后就开始修复，先完成Recovery操作，然后会继续完成上次的Backfill操作，所以没有必要在这里检查来修复。

2）通过该对象的日志记录来检查版本是否一致。首先确保是同一个对象，本次日志记录的版本prior_version等于上一条日志记录的version值。

3）版本 first_divergent_update 为该对象的日志记录中第一个产生分歧的版本；版本 last_divergent_update 为最后一个产生分歧的版本；版本 prior_version 为第一个分歧产生的前一个版本，也就是应该存在的对象版本。布尔变量 object_not_in_STORE 用来标记该对象不缺失，且第一条分歧日志操作是删除操作。处理分歧日志的五种情况如下所示：

情况1：在没有分歧的日志里查找到该对象，但是已存在的对象的版本大于第一个分歧对象的版本。这种情况的出现，是由于在merge_log中产生权威日志时的日志更新，相应的处理已经做了，这里不做任何处理。

情况2：如果prior_version为eversion_t()，为对象的create操作或者是clone操作，那么这个对象就不需要修复。如果已经在missing记录中，就删除该missing记录。

情况3：如果该对象已经处于missing列表中，如下进行处理：

- 如果日志记录显示当前已经拥有的该对象版本 have 等于 prior_version，说明对象不缺失，不需要修复，删除 missing 中的记录。

- 否则，修改需要修复的版本 need 为 prior_version；如果 prior_version 小于等于 info.log_tail 时，这是不合理的，设置 new_divergent_prior 用于后续处理。

情况4：如果该对象的所有版本都可以回滚，直接通过本地回滚操作就可以修复，不需要加入missing列表来修复。

情况5：如果不是所有的对象版本都可以回滚，删除相关的版本，把prior_version加入missing记录中用于修复。

# 10.5.3 Peering的状态转换图

由10.4节分析可知，主OSD上PG对应的状态机RecoveryMachine目前已经处于Started/Primary/Peering状态。从OSD上的PG对应的RecoveryMachine处于Started/Stray状态。本节总体介绍Peering过程的状态转换。

如图10-3所示为Peering状态转换图，其过程如下：

1）当进入 Primary/Peering 状态后，就进入默认子状态 GetInfo 中。

2）状态 GetInfo 接收事件 GetInfo 后，转移到 GetLog 状态中。

3）如果状态GetLog接收到IsIncomplete事件后，跳转到Incomplete状态。

4）状态GetLog接收到事件GotLog后，就转入GetMissing状态。

5）状态GetMissing接收到事件Activate事件，转入状态active状态。

由上述Peering的状态转换过程可知，Peering过程基本分为如下三个步骤：

步骤1：GetInfo：PG的主OSD通过发送消息获取所有从OSD的pg_info信息。

步骤2：GetLog：根据各个副本获取的pg_info信息的比较，选择一个拥有权威日志的OSD（auth_log_shard）。如果主OSD不是拥有权威日志的OSD，就从该OSD上拉取权威日志。主OSD完成拉取权威日志后也就拥有了权威日志。

步骤3：GetMissing：主OSD拉取其他从OSD的PG日志（或者部分获取，或者全部获取FULL_LOG)。通过与本地权威日志的对比，来计算该OSD上缺失的object信息，作为后续Recovery操作过程的依据。

最后通过Active操作激活主OSD，并发送notify通知消息，激活相应的从OSD。

下面介绍这三个主要步骤。

![image](https://cdn-mineru.openxlab.org.cn/result/2026-04-30/8e4d380f-113d-46f2-8e2b-b0978fb78f78/3a2c0c17c3372813003dbf2ff05f883c42fa7e4b15272acaefd16c16ed823d7f.jpg)



图10-3 Peering状态转换图


# 10.5.4 pg_info 数据结构

数据结构 pg_info_t 保存了 PG 在 OSD 上的一些描述信息。该数据结构在 Peering 的整个过程，以及后续的数据修复中都发挥了重要作用，理解该数据结构的各个关节字段的含义可以更好地理解相关的过程。pg_info_t 数据结构如下：

```c
struct pg_info_t {
    spg_t pgid; //PG的id
    eversion_t last_update; //PG最后一次更新的版本
    eversion_t last_COMPLETE;
    epoch_t last_epoch_started;
    version_t last_user_version; //最后更新的用户版本号，用于分层存。
    eversion_t logTAIL; //日志的尾部版本
    hobject_t last_backfill; //上一次Backfill操作的对象指针。如果该OSD的Backfill操作没有完成，那么last_backfill和last_COMPLETE之间的对象就丢失。
}
interval_set<snapid_t> purged_snaps; //PG要删除的snap集合
pg_stat_t stats; //PG的统计信息
pg_history_t history; //PG的历史信息
pg_hit_set_history_t hit_set; //这是CacheTier用的hit_set
```

结构 pg_history_t 保存了 PG 的一些历史信息：

```txt
struct pg_history_t {
    epoch_t epoch_created; //PG创建的epoch值
    epoch_t last_epoch_started;
    epoch_t last_epoch_clean;
}
```

# 1. last_epoch_started 介绍

last_epoch_started字段有两个地方出现，一个是pg_info结构里的last_epoch_started，代表最后一次Peering成功后的epoch值，是本地PG完成Peering后就设置的。另一个是pg_history_t结构里的last_epoch_started，是PG里所有的OSD都完成Peering后设置的epoch值。

# 2. last Complete 和 last_Backfill 的区别

在这里特别指出last_update和last_COMPLETE、last_backfill之间的区别。下面通过一个例子来讲解，同时也可以大概了解PG数据恢复的流程。在数据恢复过程中先进行Recovery过程，再进行Backfill过程（可以参考第11章的详细介绍）。

情况1：在PG处于clean状态时，lastcomplete就等于last_update的值，并且等于PG日志中的head版本。它们都同步更新，此时没有区别。last_bacfill设置为MAX值。例如：下面的PG日志里有三条日志记录。此时last_update和last Complete以及pg_log.head都指向版本（1,2）。由于没有缺失的对象，不需要恢复，last_backfill设置为MAX值。示例如下所示：

<table><tr><td>obj1 (1,0)</td><td>obj2 (1,1)</td><td>obj3 (1,2)
last_update
last_COMPLETE
pg_log.head</td><td></td><td></td></tr></table>

情况2：当该osd1发生异常之后，过一段时间后又重新恢复，当完成了Peering状态后的情况。此时该PG可以继续接受更新操作。例如：下面的灰色字体的日志记录为该osd1崩溃期间缺失的日志，obj7为新的写入的操作日志记录。last_update指向最新的更新版本（1,7），last Complete依然指向版本（1,2）。即last_update指的是最新的版本，last Complete指的是上次的更新版本。过程如下：

<table><tr><td>obj1 ( 1,0 )</td><td>obj2 ( 1,1 )</td><td>obj3 ( 1,2 ) last Complete</td><td>obj4(1,3)</td><td>obj3(1,4)</td><td>obj5(1,5)</td><td>obj6(1,6)</td><td>obj7(1,7) last_update pg log .head</td></tr></table>

last Complete 为 Recovery 修复进程完成的指针。当该 PG 开始进行 Recovery 工作时，last Complete 指针随着 Recovery 过程推进，它指向完成修复的版本。例如：当

Recovery完成后last Complete指针指向最后一个修复的对象的版本（1,6），如下所示：

<table><tr><td>obj1 (1,0)</td><td>obj2 (1,1)</td><td>obj3 (1,2)</td><td>obj4(1,3)</td><td>obj3(1,4)</td><td>obj5(1,5)</td><td>obj6(1,6)
last Complete</td><td>obj7(1,7)
last_update
pg_log.head</td></tr></table>

last_backfill 为 Backfill 修复进程的指针。在 Ceph Peering 的过程中，该 PG 有 osd2 无法根据 PG 日志来恢复，就需要进行 Backfill 过程。last_backfill 初始化为 MIN 对象，用来记录 Backfill 的修复进程中已修复的对象。例如：进行 Backfill 操作时，扫描本地对象（按照对象的 hash 值排序）。last_backfill 随修复的过程中不断推进。如果对象小于等于 last_backfill，就是已经修复完成的对象。如果对象大于 last_backfill 且对象的版本小于 lastcomplete，就是处于缺失还没有修复的对象。过程如下所示：

<table><tr><td>Backfill 对象的列表</td><td>MIN</td><td>obj1 (1,0)</td><td>obj2 (1,1)</td><td>obj3 (1,4)</td><td>obj4 (1,3)</td><td>obj5 (1,5)</td><td>obj6 (1,6)</td></tr><tr><td></td><td>last_backfill</td><td></td><td></td><td></td><td></td><td></td><td>lastcomplete</td></tr></table>

当恢复完成之后，last_backfill设置为MAX值，表明恢复完成，设置lastcomplete等于last_update的值。

# 10.5.5 GetInfo

GetInfo过程获取该PG在其他OSD上的结构图pg_info_t信息（也称pg_info信息）。这里的其他OSD包括当前PG的活跃OSD，以及past interval期间该PG所有处于up状态的OSD。

由10.5.4节的介绍可知，当PG进入Primary/Peering状态后，就进入默认的子状态GetInfo里。其主要流程在构造函数里完成：

PG::RecoveryState::GetInfo::GetInfo(my_context ctx) :my_base(ctx), NamedState(context<RecoveryMachine $\succ$ ).pg->cct，"Started/Primary/Peering/ GetInfo") { 100 100 100 100 100 100 100 

在构造函数GetInfo里，完成了核心的功能，实现过程如下：

1）调用函数 generatepast_intervals 计算 past intervals 的值：

```txt
pg->generatepast_intervals(); 
```

2）调用函数 build_prior 构造获取 pg_info_t 信息的 OSD 列表：

```txt
pg->build_prior(prior_set); 
```

3）调用函数get_infos给参与的OSD发送获取请求：

```txt
get_infos(); 
```

由上述可知，GetInfo过程基本分三个步骤：计算past_interval的过程；通过调用函数build_prior来计算要获取pg_info信息的OSD列表；最后调用函数get_infos给相关的OSD发送消息来获取pg_info信息，并处理接收到的Ack应答。

# 1. 计算 past_interval

函数 past_interval 是 epoch 的一个序列。在该序列内一个 PG 的 acting set 和 up set 不会变化。current_interval 是一个特殊的 past_interval，它是当前最新的一个没有变化的序列。示例如下：

说明如下：

1）Ceph系统当前的epoch值为20，PG1.0的acting set和up set都为列表[0,1,2]。

2）osd3失效导致了osd map变化，epoch变为21。

3）osd5失效导致了osd map变化，epoch变为22。

4）osd6失效导致了osd map变化，epoch变为23。

上述三次 epoch 的变化都不会改变 PG1.0 的 acting set 和 up set。

<table><tr><td>epoch</td><td>20</td><td>21</td><td>22</td><td>23</td><td>24</td><td>25</td><td>26</td></tr><tr><td>失效 OSD</td><td></td><td>osd3</td><td>osd5</td><td>osd6</td><td>osd2</td><td>osd12</td><td>osd13</td></tr><tr><td rowspan="2">PG 1.0</td><td>[0,1,2]</td><td>[0,1,2]</td><td>[0,1,2]</td><td>[0,1,2]</td><td>[0,1,8]</td><td>[0,1,8]</td><td>[0,1,8]</td></tr><tr><td>[0,1,2]</td><td>[0,1,2]</td><td>[0,1,2]</td><td>[0,1,2]</td><td>[0,1,8]</td><td>[0,1,8]</td><td>[0,1,8]</td></tr><tr><td></td><td></td><td></td><td></td><td></td><td>last_epoch_started</td><td>last_epoch干净</td><td></td></tr><tr><td></td><td colspan="4">past_interval</td><td colspan="3">current_interval</td></tr></table>

5）osd2失效导致了osd map变化，epoch变为24；此时导致PG1.0的acting set和upset变为[0,1,8]，若此时Peering过程成功完成，则last_epoch_started为24。

6）osd12 失效导致了 osd map 变化，epoch 变为 25，此时如果 PG1.0 完成了 Recovery 操作，处于 clean 状态，last_epoch_clean 就为 25。

7）osd13 失效导致了 osd map 变化，epoch 变为 26。

epoch序列[20,21,22,23]就为PG1.0的一个past interval，epoch序列[24,25,26]就为PG1.0的current interval。

数据结构 pg_interval_t 用于保存 past_interval 的信息：

```c
struct pg_interval_t {
    vector<int32_t> up, acting; //在本interval阶段PG处于up和acting状态的OSD
    epoch_t first, last; //起始和结束的epoch
    bool maybe_went_rw; //在这个阶段是否有数据读写操作
    int32_t primary; //主OSD
    int32_t up_primary; //up状态的主OSD
};
```

上例中，past_interval对象的p值为：

```txt
p = {up = [0,1,2], acting = [0,1,2], first = 20, last = 23, maybe_went_rw = true, primary = 0, upPrimay = 0, 
```

函数 generatepast_intervals 用于计算 past_intervals 的值，计算的结果保存在 PG 中 past_intervals 的 map 结构里，map 的 key 值为 first epoch 的值：

```txt
map<epoch_t,pg_interval_t> past_intervals; 
```

具体计算过程如下：

1）调用函数 calcpast_interval_range 推测需要计算的 past_interval 的起始 epoch 值（start）和结束 epoch 值（end）。如果返回 false，说明不需要计算 past_interval，所有的 past_interval 已经计算好了。

2）从start到end开始计算past_interval。过程为调用函数check_new_interval比较两次epoch对应的osd map的变化。如果检查是一个新值，就创建一个新的past_interval对象。

```txt
bool PG::_calcpast_interval_range(epoch_t *start, epoch_t *end, epoch_t oldest_map) 
```

函数 calcpast_interval_range 用于计算 past_interval 的范围。参数 oldest_map 为 OSD 的 superblock 里保存的最老 osd map，输出为 start 和 end，分别为需要计算的 past_interval 的 start 和 end 值。具体实现过程如下。

计算end值如下所示：

1）变量end为当前osd map的epoch值，如果info-history同样的interval_since不为空，就设置为该值。该值表示和当前的osd map的epoch值在同一个interval中。

```c
if (info_history同样的_interval_since) {  
*end = info_history同样的_interval_since;  
} else {  
// 当前PG可能是新引入的，计算整个range期间interval  
*end = osdmap_ref->get_epoch();  
}
```

2）查看 past_intervals 里已经计算的 past_interval 的第一个 epoch，如果已经比 info_history.last_epoch_clean 小，就不用计算了，直接返回 false 值。否则设置 end 为其 first 值。

\*end $=$ past_intervals.begin()->first; 

计算 start 值如下所示：

1）start设置为info_history.last_epoch_clean，从最后一次last_epoch_clean算起。

2）当PG为新建时，从info_history_epoch_created开始计算。

3）oldest_map值为保存的最早osd map的值，如果start小于这个值，相关的osd map信息缺失，所以无法计算。

所以将 start 设置为三者的最大值：

\*start $=$ MAX(MAX(info-history_epoch_created, info-history.last_epoch_clean), oldest_map); 

下面举例说明计算 past_interval 的过程。


例10-3 past_interval计算示例


<table><tr><td>4</td><td>5</td><td>6</td><td>7</td><td>8</td><td>9</td><td>10</td><td>11</td><td>12</td><td>13</td><td>14</td><td>15</td><td>16</td></tr><tr><td colspan="5">past_interval 1</td><td colspan="3">past_interval 2</td><td colspan="2">past_interval 3</td><td colspan="3">current_interval</td></tr></table>

如上表所示：一个PG有4个interval。past_interval1，开始epoch为4，结束的epoch为8；past_interval2的epoch区间为（9,11）；past_interval3的区间为（12,13）；current_interval的区间为（14,16）。最新的epoch为16，info-history同样的interval_since为14，意指是从epoch14开始，之后的epoch值和当前的epoch值在同一个interval内。info-history.last_epoch_clean为8，就是说在epoch值为8时，该PG处于clean状态。

计算 start 和 end 的方法如下：

1）start的值设置为info-history.last_epoch_clean值，其值为8。

2）end值从14开始计算，检查当前已经计算好的past_interval的值。past_interval的计算是从后往前计算。如果第一个past interval的first小于等于8，也就是past_interval1已经计算过了，那么后面的past_interval2和past_interval3都已经计算过，就直接退出。否则就继续查找没有计算过的past_interval的值。

# 2.构建OSD列表

函数 build_prior 根据 past_intervals 来计算 probe_targets 列表，也就是要去获取 pg_info 的 OSD 列表。具体实现为：首先重新构造一个 PriorSet 对象，在 PriorSet 的构造函数中完成下列操作：

1）把当前PG的acting set和upset中的OSD加入到probe列表中。

2）检查每个past_intervals阶段：

a）如果interval.last小于info_history.last_epoch_started，这种情况下past_interval就没有意义，直接跳过。

b）如果该interval的act为空，就跳过。

c）如果该interval没有rw操作，就跳过。

d）对于当前interval的每一个处于acting状态的OSD进行检查：

- 如果该OSD当前处于up状态，就加入到up_now列表中。同时加入到probe

列表中，用于获取权威日志以及后续数据恢复。

- 如果该OSD当前不是up状态，但是在该past_interval期间还处于up状态，就加入up_now列表中。

- 否则加入 down 列表，该列表保存所有宕了的 OSD。

- 如果当前 interval 确实有宕的 OSD，就调用函数 pcontdec，也就是 PG 的 IsPGRecoverablePredicate 函数。该函数判断该 PG 在该 interval 期间是否可恢复。如果无法恢复，直接设置 pg_down 为 true 值。

![image](https://cdn-mineru.openxlab.org.cn/result/2026-04-30/8e4d380f-113d-46f2-8e2b-b0978fb78f78/4aef99af0ad0f55fe9ccf1a2daefe20e43da8a7b37de2af398b836e38f9b764f.jpg)


# 注意

这里特别强调的是，要确保每个interval期间都可以进行修复。函数IsPGRecoverable-Predicate实际上是一个类的运算符重载。对于不同类型的PG有不同的实现。对于ReplicatedPG对应的实现类为RPCRecPred，其至少保证有一个处于up状态的OSD；对应ErasureCode（ $\mathbf{n} + \mathbf{m}$ ）类型的PG，至少有n个处于up状态的OSD。

3）如果prior.pg_down设置为true，就直接设置PG为PG_STATE_DOWN状态。

4）检查是否需要need_up_thru设置。

5）用prior_set->probe设置probe_targets列表。

# 3. 获取 pg_info 信息

在上述过程中计算出了PG在past interval期间以及当前处于up状态的OSD列表，下面就发送请求给OSD来获取pg_info信息：

```autohotkey
void PG::RecoveryState::GetInfo::get_infos() 
```

函数get_infos向prior_set的probe集合中的每个OSD发送pg_query_t::INFO消息，来获取PG在该OSD上的pg_info信息。发送消息的过程调用RecoveryMachine类的send_query函数来进行：

context<RecoveryMachine $\rightharpoondown$ ).send_query( peer, pg_query_t(pg_query_t::INFO, it->shard，pg->pg_whoami.shard， pg->info-history, pg->get_osdmap()->get_epoch()) ）; peer_info_requested.insert(peer); 

```txt
pg->blocked_by.insert( peer.osd) 
```

数据结构 pg_notify_t 定义了获取 pg_info 的 ACK 信息：

```c
struct pg_notify_t {
    epoch_t query_epoch; // 查询时请求消息的 epoch
    epoch_t epoch_sent; // 发送时应对消息的 epoch
    pg_info_t info; // pg_info 的信息
    shard_id_t to; // 目标 OSD
    shard_id_t from; // 源 OSD
};
```

在主OSD收到pg_info的ACK消息后封装成MNotifyRec事件发送给该PG对应的状态机。在下列的事件处理函数中来处理MNotifyRec事件。

```txt
boost::statechart::result PG::RecoveryState::GetInfo::react(const MNotifyRec& infoevt) 
```

具体处理过程如下：

1）首先从peer_info_requested里删除该peer，同时从blocked_by队列里删除。

2）调用函数PG::proc Replica_info来处理副本的pg_info消息：

a）首先检查如果该OSD的pg_info信息，如果已经存在，并且last_update参数相同，则说明已经处理过，返回false值。否则保存该pg_info的值。

b）调用函数 has been up since 检查该 OSD 在 send epoch 时已经处于 up 状态。

c）确保自己是主OSD，把该OSD的pg_info信息保存到peer_info数组，并加入might have_unfound数组里。该数组里的OSD用于后续的数据恢复。

d）调用函数unreg next scrub使该PG不在scrub操作的队列中。

e）调用info_history.merge函数处理从OSD发过来的pg_info信息。处理方法是：更新为最新的字段，设置dirty info为true值。

f) 调用函数 reg_next_scrub 注册 PG 下一次的 scrub 的时间。

g）如果该OSD既不在up数组中也不在acting数组中，那就加入stray_set列表中。当PG处于clean状态时，就会调用purge_strays函数删除stray状态的PG及其上的对象数据。

h）如果是一个新的OSD，就调用函数update-heartbeat_peers更新需要heartbeat的OSD列表。

3）在变量 old_start 里保存了调用 proc Replica_info 前主 OSD 的 pg->info_history.last_epoch_started，如果该 epoch 值小于合并后的值，说明该值被更新了，从 OSD 上的 epoch 值比较新，需要进行如下操作：

a）调用pg->build_prior重新构建prior_set对象。

b）从peer_info_requested队列中去掉上次构建的prior_set中存在的OSD，这里最新构建上次不存在的OSD列表。

c）调用get_infos函数重新发送查询peer_info请求。

4）调用pg->applypeer_features更新相关的features值。

5）当peer_info_requested队列为空，并且prior_set不处于pg_down的状态时，说明收到所有OSD的peer_info并处理完成。

6）最后检查 past_interval 阶段至少有一个 OSD 处于 up 状态且非 incomplete 状态；否则该 PG 无法恢复，标记状态为 PG_STATE_DOWN 并直接返回。

7）最后完成处理，调用函数post_event(GotInfo())抛出GetInfo事件进入状态机的下一个状态。

在GetInfo状态里直接定义了当前状态接收到GotInfo事件后，直接跳转到下一个状态GetLog里：

```txt
struct GetInfo : boost::statechart::state< GetInfo, Peering >, NamedState {
    typedef boost::mpl::list < boost::statechart::custom_reaction< QueryState >,
        boost::statechart::transition< GotInfo, GetLog >,
        boost::statechart::custom_reaction< MNotifyRec >
> reactions;
} 
```

# 10.5.6 GetLog

当PG的主OSD获取到所有从OSD（以及past interval期间的所有参与该PG且目前仍处于active状态的OSD）的pg_info信息后，就跳转到GetLog状态。

```autohotkey
PG::RecoveryState::GetLog::GetLog(my_context ctx) 
```

然后在GetLog的构造函数里做相应的处理，其具体处理过程分析如下：

1）调用函数 pg->choose Acting(auth_log_shard) 选出具有权威日志的 OSD，并计算出 acting_backfill 和 backfill_targets 两个 OSD 列表。输出保存在 auth_log_shard 里。

2）如果选择失败并且want Acting不为空，就抛出NeedActingChange事件，状态机转移到Primary/WaitActingChang状态，等待申请临时PG返回结果。如果want Acting为空，就抛出IsIncomplete事件，PG的状态机转移到Primay/Peering/Incomplete状态。表明失败，PG就处于Incomplete状态。

3）如果auth_log_shard等于pg->pg_whoami的值，也就是选出的拥有权威日志的OSD为当前主OSD，直接抛出事件GotLog()完成GetLog过程。

4）如果 pg->info.last_update 小于权威 OSD 的 log.tail，也就是本 OSD 的日志和权威日志不重叠，那么本 OSD 无法恢复，抛出 IsIncomplete 事件。经过函数 choose Acting 的选择后，主 OSD 必须是可恢复的。如果主 OSD 不可恢复，必须申请一个临时 PG，选择拥有权威日志的 OSD 为临时主 OSD。

5）如果自己不是权威日志的OSD，则需要去拥有权威日志的OSD上去拉取权威日志，并与本地合并。

# 1. choose Acting

函数choose Acting用来计算PG的acting_backfill和backfill_targets两个OSD列表。acting_backfill保存了当前PG的acting列表，包括需要进行Backfill操作的OSD列表；backfill_targets列表保存了需要进行Backfill的OSD列表。其处理过程如下：

1）首先调用函数 find_best_info 来选举出一个拥有权威日志的 OSD，保存在变量 auth_log_shard 里。

2）如果没有选举出拥有权威日志的OSD，则进入如下流程：

a）如果up不等于acting，申请临时PG，返回false值。

b）否则确保want Acting列表为空，返回false值。

3）计算是否是compat_mode模式，检查是，如果所有的OSD都支持纠删码，就设置compat_mode值为true。

4）根据PG的不同类型，调用不同的函数，对应ReplicatedPG调用函数calc_replicated Acting来计算PG的需要列表：

```txt
set<pg_shard_t> want_backfill, want Acting_backfill; //want_backfill为该PG需要进行Backfill的pg_shard //want Acting backfill包括进行acting和Backfill的pg_shard pg_shard_t want_primary; //主OSD vector<int> want; //在compat_mode模式下，和want Acting backfill相同
```

5）下面就是对PG做的一些检查：

a）计算num_want Acting数量，检查如果小于min_size，进行如下操作：

- 如果对于 EC，清空 want acting，返回 false 值。

- 对于ReplicatePG，如果该PG不允许小于min_size的恢复，清空want Acting，返回false值。

b）调用IsPGRecoverablePredicate来判断PG现有的OSD列表是否可以恢复，如果不能恢复，清空want Acting，返回false值。

6）检查如果want不等于acting，设置want Acting为want：

a）如果wang acting等于up，申请empty为pg_temp的OSD列表。

b）否则申请want为pg_temp的OSD列表。

7）最后设置PG的actingbackfill为want Acting backfill，设置backfill_targets为want backfill，并检查backfill_targets里的pg_shard应该不在stray_set里面。

8）最终返回 true 值。

下面举例说明需要申请 pg_temp 的场景：

1）当前PG1.0，其acting列表和up列表都为[0,1,2]，PG处于clean状态。

2）此时，osd0崩溃，导致该PG经过CRUSH算法重新获得acting和up列表都为[3,1,2]。

3）选择出拥有权威日志的osd为1，经过calc_replicated Acting算法，want列表为[1,3,2]，acting_backfill为[1,3,2]，want_backfill为[3]。特别注意want列表第一个为主OSD，如果up primay无法恢复，就选择权威日志的OSD为主OSD。

4）want[1,3,2]不等于acting[3,1,2]时，并且不等于up[3,1,2]，需要向Monitor申请pg_temp为want。

（5）申请成功pg_temp以后，acting为[3,1,2],up为[1,3,2]，osd1做为临时的主OSD，

处理读写请求。当该PG恢复处于clean状态，pg_temp取消，acting和up都恢复为[3,1,2]。

# 2. find_best_info

函数 find_best_info 用于选取一个拥有权威日志的 OSD。根据 last_epoch_clean 到目前为止，各个 past_interval 期间参与该 PG 的所有目前还处于 up 状态的 OSD 上 pg_info_t 信息，来选取一个拥有权威日志的 OSD，选择的优先顺序如下：

1）具有最新的lastupdate的OSD。

2）如果条件1相同，选择日志更长的OSD。

3）如果1，2条件都相同，选择当前的主OSD。

代码实现具体的过程如下：

1）首先在所有OSD中计算max_last_epoch_started，然后在拥有最大的last_epoch_started的OSD中计算min last update acceptable的值。

2）如果min_last_update_acceptable为eversion t::max()，返回infos.end()，选取失败。

3）根据以下条件选择一个OSD：

a）首先过滤掉last_update小于min_last_updateizable，或者last_epoch_started小于max_last_epoch_startedfound，或者处于incomplete的OSD。

b）如果PG类型是EC，选择最小的last_update；如果PG类型是副本，选择最大的last_update的OSD。

c）如果上述条件都相同，选择longtail最小的，也就是日志最长的OSD。

d）如果上述条件都相同，选择当前的主OSD。

综上的选择过程可知：拥有权威日志的OSD特征如下：必须是非incomplete的OSD；必须有最大last_epoch_strated；last_update有可能是最大，但至少是min_last_update_acceptable，有可能是日志最长的OSD，有可能是主OSD。

# 3. calc_replicated Acting

本函数计算本PG相关的下列OSD列表：

□want_primary：主OSD，如果它不是up primaries，就需要申请pg_temp。

□ backfill：需要进行 Backfill 操作的 OSD。

□ acting backfill：所有进行 acting 和 Backfill 的 OSD 的集合。

want和acting backfill的OSD相同，前者类型是pg_shard_t，后者为int型。

具体处理过程如下：

1）首先选择want primay列表中的OSD：

a）如果up_primary处于非incomplete状态，并且last_update大于等于权威日志的log.tail，说明up_primary的日志和权威日志有重叠，可通过日志记录恢复，优先选择up_primary为主OSD。

b）否则选择auth log shard，也就是拥有权威日志的OSD为主OSD。

c）把主OSD加入到want和actingbackfill列表中。

2）函数的输入参数 size 为要选择的副本数，依次从 up、acting、all_info 里选择 size 个副本 OSD:

a）如果该OSD上的PG处于incomplete的状态，或者cur_info.last_update小于主OSD和auth_log_shard的最小值，则该PG副本无法通过日志修复，只能通过Backfill操作来修复。把该OSD分别加入backfill和acting backfill集合中。

b）否则就可以根据PG日志来恢复，只加入acting_backfill集和want列表中，不用加入到Backfill列表中。

# 4. 收到缺失的权威日志

如果主OSD不是拥有权威日志的OSD，就需要去拥有权威日志的OSD上拉取权威日志：

```bash
boost::statechart::result PG::RecoveryState::GetLog::react(const MLogRec& logevt) 

当收到权威日志后，封装成 MLogRec 类型的事件。本函数就用于处理该事件。它首先确认是从 auth_log_shard 端发送的消息，然后抛出 GotLog() 事件：

```bash
boost::statechart::result PG::RecoveryState::GetLog::react(const GotLog&) 

本函数捕获GetLog事件，处理过程如下：

1）如果msg不为空，就调用函数proc/master_log合并自己缺失的权威日志，并更新自己pg_info相关的信息。从此，做为主OSD，也是拥有权威日志的OSD。

2）调用函数pg->start Flush添加一个空操作。

3）状态转移到GetMissing状态。

经过GetLog阶段的处理后，该PG的主OSD已经获取了权威日志，以及pg_info的权威信息。

# 10.5.7 GetMissing

GetMissing的处理过程为：首先，拉取各个从OSD上的有效日志。其次，用主OSD上的权威日志与各个从OSD的日志进行对比，从而计算出各从OSD上不一致的对象并保存在对应的pg MISSING_t结构中，做为后续数据修复的依据。

主OSD的不一致的对象信息，已经在调用函数proc/master_log合并权威日志的过程中计算出来，所以这里只计算从OSD上的不一致的对象。

# 1. 拉取从副本上的日志

在GetMissing的构造函数里，通过对比主OSD上的权威pg_info信息，来获取从OSD上的日志信息。

```autohotkey
PG::RecoveryState::GetMissing::GetMissing(my_context ctx) 
```

其具体处理过程为遍历 pg->actingbackfill 的 OSD 列表，然后做如下的处理：

1）不需要获取PG日志的情况：

a）如果pi.is_empty()为空，没有任何信息，需要Backfill过程来修复，不需要获取日志。

b）pi.last_update小于pg->pg_log.getTAIL()，该OSD的pg_info记录中，last_update小于权威日志的尾部记录，该OSD的日志和权威日志不重叠，该OSD操作已经远远落后于权威OSD，已经无法根据日志来修复，需要Backfill过程来修复。

c）pi.last_backfill为hobject_t()，说明在past interval期间，该OSD标记需要Backfill操作，实际并没开始Backfill的工作，需要继续Backfill过程。

d）pi.last_update等于pi.last_COMPLETE，说明该PG没有丢失的对象，已经完成Recovery操作阶段，并且pi.last_update等于pg->info.last_update，说明日志和权威日志的最后更新一致，说明该PG数据完整，不需要恢复。

2）获取日志的情况：当pi.last_update大于pg->info.log.tail，该OSD的日志记录和权威日志记录重叠，可以通过日志来修复。变量sine是从last_epoch_started开始的版本值：

a）如果该PG的日志记录pi.log.tail小于等于版本值since，那就发送消息pg_query_t::LOG，从since开始获取日志记录。

b）如果该PG的日志记录pi.logtail大于版本值since，就发送消息pg_query_t::FULLLOG来获取该OSD的全部日志记录。

3）最后检查如果peer MISSINGRequested为空，说明所有获取日志的请求返回并处理完成。如果需要pg->need_up_thru，抛出post_event(NeedUpThru();；否则，直接调用post_event Activate(pg->get_osmap()->get_epoch()))进入Activate状态。

下面举例说明获取日志的两种情况：

<table><tr><td>权威日志</td><td>(9,10)</td><td>......</td><td>(10,0)sine</td><td>(10,1)</td><td>(10,2)</td><td>(10,3)</td><td>......</td><td>last_update</td></tr><tr><td>osd0 日志</td><td></td><td></td><td></td><td></td><td>log_tail</td><td></td><td></td><td></td></tr><tr><td>osd1 日志</td><td>log_tail</td><td></td><td></td><td></td><td></td><td></td><td></td><td></td></tr></table>

当前last_epoch_started的值为10，sine是last_epoch_started后的首个日志版本值当前需要恢复的有效日志是经过sine操作之后的日志，之前的日志已经没有用了。

对应osd0，其日志logtail小于since，全部拷贝osd0上的日志；对应osd1，其日志logtail小于since，只拷贝从sine开始的日志记录。

# 2. 收到从副本上的日志记录处理

当一个PG的主OSD接收到从OSD返回的获取日志ACK应答后，就把该消息封装成MLogRec事件。状态GetMissing接收到该事件后，在下列事件函数里处理该事件：

```bash
boost::statechart::result PG::RecoveryState::GetMissing::react(const MLogRec& logevt) 