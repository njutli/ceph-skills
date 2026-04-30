具体过程过程如下：

1）调用proc Replica_log处理日志。通过日志的对比，获取该OSD上处于missing状态的对象列表。

2）如果peer MISSINGRequested为空，即所有的获取日志请求返回并处理。如果需要pg->need_up_thru，抛出NeedUpThru()事件。否则，直接调用函数post_event(Activate(pg->get_osmap()->get_epoch()))进入Activate状态。

函数 proc_replica_log 处理各个从 OSD 上发过来的日志。它通过比较该 OSD 的日志和本地权威日志，来计算该 OSD 上处于 missing 状态的对象列表。具体处理过程调用 pg_logproc_replica_log 来处理日志，输出为 omitting，也就是该 OSD 缺失的对象。

# 10.5.8 Active 操作

由上述可知，如果GetMissing处理成功，就跳转到Activate状态。到本阶段为止，可以说Peering主要工作已经完成，但还需要后续的处理，激活各个副本，如下所示：

```autohotkey
PG::RecoveryState::Active::Active(my_context ctx) 
```

状态 Activate 的构成函数里处理过程如下：

1）在构造函数里初始化了remote_shards_to_reserved_recovery和remote_shards_to_reserved_backfill，需要Recovery操作和Backfill操作的OSD。

2）调用函数pg->start Flush来完成相关数据的flush工作。

3）调用函数 pg->activate 完成最后的激活工作。

# 1. MissingLoc

类 MissingLoc 用来记录处于 missing 状态对象的位置，也就是缺失对象的正确版本分析在哪些 OSD 上。恢复时就去这些 OSD 上去拉取正确对象的数据：

class MissingLoc{ map<Hobject_t,pg MISSING_t::item,hobject_t::BitwiseComparator> needs_ recovery_map; //缺失的对象 $\rightarrow$ item（现在版本，缺失的版本） map<Hobject_t,set<pg_shard_t>,hobject_t::BitwiseComparator $>$ missing_location; //缺失的对象----所在的OSD集合 set<pg_shard_t>missing_location_sources;

//所有缺失对象所在的OSD集合

```rust
PG \*pg; set<pg_shard_t> empty_set; public: boost::scoped_ptr<IsPGReadablePredicate> is_readable; boost::scoped_ptr<IsPGRecoverablePredicate> is_recoverable; 
```

下面介绍一些 MissingLog 处理函数，作用是添加相应的 missing 对象列表。其对应两个函数：add.active MISSING 函数和 add_source_info 函数。

add.active MISSING函数用于把一个副本中的所有缺失对象添加到MissingLoc的needs_recovery_map结构里：

void add.active MISSING(const pg MISSING_t &missing) 

add_source_info函数用于计算每个缺失对象是否在本OSD上：

```txt
PG::MissingLoc::add_source_info(pg_shard_t fromosd, const pg_info_t &oinfo, const pgmissing_t &omissing, bool sort_bitwise, ThreadPool::TPHandle* handle) 
```

具体实现如下：

遍历 needs_recovery_map 里的所有对象，对每个对象做如下处理：

1）如果 oinfo.last_update < need（所需的缺失对象的版本）大于 oinfo.last_update 的值，就跳过。

2）如果该PG正常的last_backfill指针小于MAX值，说明还处于Backfill阶段，但是sort_bitwise不正确，跳过。

3）如果该对象大于last_backfill，显然该对象不存在，跳过。

4）如果该对象大于last Complete，说明该对象或者是上次Peering之后缺失的对象，还没有来得及恢复；或者是新创建的对象。检查如果在missing记录已存在，就是上次缺失的对象，直接跳过；否则就是新创建的对象，存在该OSD中。

5）经过上述检查后，确认该对象在本OSD上，在missing_location添加该对象的location为本OSD。

# 2. Activate 状态

PG::activate函数是Peering过程的最后一步，该函数完成以下功能：

□更新一些pg_info的参数信息。

□给 replica 发消息，激活副本 PG。

□计算MissingLoc，也就是缺失对象分布在哪些OSD上，用于后续的恢复。

具体处理过程如下：

1）如果需要客户回答，就把PG添加到replay_queue队列里。

2）更新info.last_epoch_started变量，info.last_epoch_started指的是本OSD在完成目前Peering进程后的更新，而info_history.last_epoch_started是PG的所有的OSD都确认完成Peering的更新。

3）更新一些相关的字段。

4）注册C_PG_ActivateCommitted回调函数，该函数最终完成activate的工作。

5）初始化snap trimmed快照相关的变量。

6）设置info.last_COMPLETE指针：

- 如果 missing num missing() 等于 0，表明处于 clean 状态。直接更新 info.lastcomplete 等于 info.last_update，并调用 pg_log.reset_recovery_pointers() 调整 log的 complete_to 指针。

- 否则，如果有需要恢复的对象，就调用函数 pg_logactivate_not_COMPLETE(info)，设置 info.last_COMPLETE 为缺失的第一个对象的前一版本。

7）以下都是主OSD的操作，给每个从OSD发送MOSDPGLog类型的消息，激活该PG的从OSD上的副本。分别对应三种不同处理：

- 如果 pi.last_update 等于 info.last_update，这种情况下，该 OSD 本身就是 clean 的，不需要给该 OSD 发送其他信息。添加到 activator_map 只发送 pg_info 来激活从 OSD。其最终的执行在 PeeringWQ 的线程执行完状态机的事件处理后，在函数 OSD::dispatch_context 里调用 OSD::do_infos 函数实现。

- 需要 Backfill 操作的 OSD，发送 pg_info，以及 osd_min_pg_log_entries 数量的 PG 日志。

- 需要Recovery操作的OSD，发送pg_info，以及从缺失的日志。

8）设置 MissingLoc，也就是统计缺失的对象，以及缺失的对象所在的 OSD，核心就

是调用 MissingLoc 的 add_source_info 函数，见 MissingLoc 相关的分析。

9）如果需要恢复，把该PG加入到osd->queue_for_recovery(this)的恢复队列中。

10）如果PG的size小于act set的size，也就是当前的OSD不够，就标记PG的状态为PG_STATE_DEGRADED和PG_STATE_UNDERSED状态，最后标记PG为PG_STATE_ACTIVATING状态。

# 3. 收到从 OSD 的 MOSDPGLog 的应对

当收到从OSD发送的MOSDPGLog的ACK消息后，触发MInfoRec事件，下面这个函数处理该事件：

```cpp
boost::statechart::result PG::RecoveryState::Active::react(const MInfoRec& infoevt) 
```

处理过程比较简单：检查该请求的源OSD在本PG的actingbacfill列表中，以等待列表中删除该OSD。最后检查，当收集到所有的从OSD发送的ACK，就调用函数allActivated_and_committed触发AllReplicasActivated事件。

对应主OSD在事务的回调函数C_PG_ActivateCommitted里实现，最终调用activatecommitted加入peerActivated集合里。

# 4. AllReplicasActivated

这个函数处理 AllReplicasActivated 事件：

```cpp
boost::statechart::result PG::RecoveryState::Active::react(const AllReplicasActivated &evt) 
```

当所有的 replica 处于 activated 状态时，进行如下处理：

1）取消PG_STATE_ACTIVATING和PG_STATE Creatinging状态，如果该PG上acting状态的OSD数量大于等于Pool的min_size，设置该PG为PG_STATE.Active的状态；否则设置为PG_STATE_PEERED状态。

2）ReplicatedPG::check_local检查本地的 stray对象是否都被删除。

3）如果有读写请求在等待Peering操作，则把该请求添加到处理队列pg->requeueOps(pg->waiting_for_peered)。

4）调用函数ReplicatedPG::on Activate，如果需要Recovery操作，触发DoRecovery事件，如果需要Backfill操作，触发RequestBackfill事件；否则触发AllReplicasRecovered事件。

5）初始化CacheTier需要的hit_set对象。

6）初始化CacheTier需要的agent对象。

# 10.5.9 副本端的状态转移

当创建PG后，根据不同的角色，如果是主OSD，PG对应的状态机就进入了Primary状态。如果不是主OSD，就进入Stray状态。

# 1. Stray 状态

Stray 状态有两种情况。

情况1：只接收到PGINFO的处理：

```bash
boost::statechart::result PG::RecoveryState::Stray::react(const MInfoRec& infoevt) 

从PG接收到主PG发送的MInfoRec事件，也就是接收到主OSD发送的pg_info信息。其判断如果当前pg->info.last_update大于infoevt.info.last_update，说明当前的日志有divergent的日志，调用函数rewind_divergent_log清理日志即可。最后抛出Activate(infoevt.info.last_epoch_started事件，进入ReplicaActive状态。

情况2：接收到MOSDPGLog消息：

```bash
boost::statechart::result PG::RecoveryState::Stray::react(const MLogRec& logevt) 

当从PG接收到MLogRec事件，就对应着接收到主PG发送的MOSDPGLog消息，其通知从PG处于activate状态，具体处理过程如下：

1）如果msg->info.last_backfill为hobject_t()，需要Backfill操作的OSD。

2）否则就是需要Recovery操作的OSD，调用merge_log把主OSD发送过来的日志合并。

抛出 Activate(logevt.msg->info.last_epoch_started) 事件，使副本转移到 ReplicaActive 状态。

# 2. ReplicateActive 状态

ReplicateActive 状态如下：

```cpp
boost::statechart::result PG::RecoveryState::ReplicaActive::react(const Activate& actevt) 
```

当处于ReplicateActive状态，接收到Activate事件，就调用函数pg->activate，在函数 Activate_committed给主PG发送应答信息，告诉自己处于activate状态，设置PG为 activate状态。

# 10.5.10 状态机异常处理

在上面的流程介绍中，只介绍了正常状态机的转换流程。Ceph之所以用状态机来实现PG的状态转换，就是可以实现任何异常情况下的处理。下面介绍当OSD失效时导致相关的PG重新进行Peering的机制。

当一个OSD失效，Monitor会通过heartbeat检测到，导致osd map发生了变化，Monitor会把最新的osd map推送给OSD，导致OSD上的受影响PG重新进行Peering操作。

具体的流程如下：

1）在函数OSD::handle_osd_map处理osd map的变化，该函数调用consume_map，对每一个PG调用pg->queue_null，把PG加入到peering_wq中。

2）peering_wq的处理函数process_peering_events调用OSD::advance_pg函数，在该函数里调用PG::handleadvance_map给PG的状态机RecoveryMachine发送AdvMap事件：

```cpp
boost::statechart::result PG::RecoveryState::Started::react(const AdvMap& advmap) 
```

当处于 Started 状态，接收到 AdvMap 事件，调用函数 pg->should_restart_peering 检查，如果是 new_interval，就跳转到 Reset 状态，重新开始一次 Peering 过程。

# 10.6 本章小结

本章介绍了Ceph的Peering过程，其核心过程就是通过各个OSD上保存的PG日志选择出一个权威日志的OSD。以该OSD上的日志为基础，对比其他OSD上的日志记录，计算出各个OSD上缺失的对象信息。这样，PG就使各个OSD的数据达成了一致。

# Ceph 数据修复

当PG完成了Peering过程后，处于Active状态的PG就已经可以对外提供服务了。如果该PG的各个副本上有不一致的对象，就需要进行修复。Ceph的修复过程有两种：Recovery和Backfill。

Recovery是仅依据PG日志中的缺失记录来修复不一致的对象。Backfill是PG通过重新扫描所有的对象，对比发现缺失的对象，通过整体拷贝来修复。当一个OSD失效时间过长导致无法根据PG日志来修复，或者新加入的OSD导致数据迁移时，就会启动Backfill过程。

从第10章可知，PG完成Peering过程后，就处于activate状态，如果需要Recovery，就产生DoRecovery事件，触发修复操作。如果需要Backfill，就会产生RequestBackfill事件来触发Backfill操作。在PG的数据修复过程中，如果既有需要Recovery过程的OSD，又有需要Backfill过程的OSD，那么处理过程需要先进行Recovery过程的修复，再完成Backfill过程的修复。

本章介绍 Ceph 的数据修复的实现过程。首先介绍数据修复的资源预约的知识，然后通过介绍修复的状态转换图，大概了解整个数据修复的过程。最后分别详细介绍 Recovery 过程和 Backfill 过程的具体实现。

# 11.1 资源预约

在数据修复的过程中，为了控制一个OSD上正在修复的PG最大数目，需要资源预约，在主OSD上和从OSD上都需要预约。如果没有预约成功，需要阻塞等待。一个OSD能同时修复的最大PG数在配置选项osd_max_backfills中设置，默认值为1。

类AsyncReserver用来管理资源预约，其模板参数<T>为要预约的资源类型。该类实现了异步的资源预约。当成功完成资源预约后，就调用注册的回调函数通知调用方预约成功：

class AsyncReserver{ unsigned maxAllowed; //定义允许的最大资源数量，在这里指允许修复的PG的数量 unsigned minpriority; //最小的优先级 Finisher \*f; //当预约成功够，用来执行的回调函数 map<unsigned, list<pair<T, Context $\text{串}$ $>$ > queues; //优先级到待预约资源链表的映射，pair<T, Context $\text{串}$ $>$ 定义预约的资源和注册的回调函数 map<T, pair<unsigned, typename list<pair<T, Context $\text{串}$ $>$ ::iterator>> queue pointers; //资源在queue链表中的位置指针 set<T> in_progress; //预约成功，正在使用的资源

# 1. 资源预约

函数 request reservation 用于预约资源：

```txt
void request reservation( T item, //输入参数：申请的资源 Context \*on_reserved, //输入参数：申请成功后的回调函数
```

具体处理过程如下：

1）把要请求的资源根据优先级添加到queue队列中，并在queue pointers中添加其对应的位置指针：

```txt
queues[prio].push_back(make_pair(item, on_reserved));  
queue pointers.insert(make_pair(item, make_pair(prio, -- (quences[prio]).  
end()); 
```

2）调用函数do_queues用来检查queue中的所有资源预约申请：从优先级高的请求开始检查，如果还有配额并且其请求的优先级至少不小于最小优先级，就把资源授权给它。

3）在queue队列中删除该资源预约请求，并在queue pointers删除该资源的位置信息。把该资源添加到in_progress队列中，并把请求相应的回调函数添加到Finisher类中，使其执行该回调函数。最后通知预约成功。

# 2. 取消预约

函数cancel reservation用于释放拥有的不再使用的资源：

```txt
void cancel reservation( T item //输入参数：删除申请的资源
```

具体处理过程如下：

1）如果该资源还在queue队列中，就删除（这属于异常情况的处理）；否则在in progress队列中删除该资源。

2）调用do_queue函数把该资源重新授权给其他等待的请求。

# 11.2 数据修复状态转换图

如图11-1所示的是修复过程状态转换图。当PG进入Active状态后，就进入默认的子状态Activating。

![image](https://cdn-mineru.openxlab.org.cn/result/2026-04-30/bbb6afcc-af41-45b5-9598-d269ca9c27e9/0fb30085ec13df8bb298f271a495176a905c6f23a9c9dd9fc2049455fd0c10ce.jpg)



图11-1 修复过程状态转换图


数据修复的状态转换过程如下所示。

情况1：当进入Activating状态后，如果此时所有的副本都完整，不需要修复，其状态转移过程如下：

1）Activating状态接收到AllReplicatedRecovered事件，直接转换到Recovered状态。

2）Recovered状态接收到GoClean事件，整个PG转入Clean状态。

情况2：当进入Activating状态后，没有Recovery过程，只需要Backfill过程的情况：

1）Activating状态直接接收到RequestBackfill事件，进入WaitLocalBackfillReserved状态。

2）当WaitLocalBackfillReservede状态接收到LocalBackfillReserved事件后，意味着本地资源预约成功，转入WaitRemoteBackfillReserved状态。

3）所有副本资源预约成功后，主PG就会接收到AllBackfillsReserved事件，进入Backfilling状态，开始实际数据Backfill操作过程。

4）Backfilling状态接收Backfilled事件，标志Backfill过程完成，进入Recovered状态。

5）异常处理：当在状态WaitRemotBackfillReserved和Backfilling接收到RemoteReservationRejected事件时，表明资源预约失败，进入NotBackfilling状态，再次等待RequestBackfilling事件来重新发起Backfill过程。

情况3：当PG既需要Recovery过程，也可能需要Backfill过程时，PG先完成Recovery过程，再完成Backfill过程，特别强调这里的先后顺序。其具体过程如下：

1）Activating状态：在接收到DoRecovery事件后，转移到WaitLocalRecoveryReserved状态。

2）WaitLocalRecoveryReserved状态：在这个状态中完成本地资源的预约。当收到LocalRecoveryReserved事件后，标志着本地资源预约的完成，转移到WaitRemote-RecoveryReserved状态。

3）WaitRemoteRecoveryReserved状态：在这个状态中完成远程资源的预约。当接收到AllRemotesReserved事件，标志着该PG在所有参与数据修复的从OSD上完成资源预约，进入Recovering状态。

4）Recovering状态：在这个状态中完成实际的数据修复工作。完成后把PG设置为

PG_STATE_RECOVERING状态，并把PG添加到recovery_wq工作队列中，开始启动数据修复：

```c
pg->state_clear(PG_STATE_RECOVERY_WAIT);  
pg->state_set(PG_STATE_RECOVERING);  
pg->osd->queue_for_recovery(pg); 
```

5）在Recovering状态完成Recovery工作后，如果需要Backfill工作，就接收RequestBackfill事件，转入Backfill流程。

6）如果没有Backfill工作流程，直接接收AllReplicasRecovered事件，转入Recovered状态。

7）Recovered状态：到达本状态，意味着已经完成数据修复工作。当收到事件GoClean后，PG就进入clean状态。

# 11.3 Recovery过程

数据修复的依据是在Peering过程中产生的如下信息：

□主副本上的缺失对象的信息保存在pg_log类的pg MISSING_t结构中。

□各从副本上的缺失对象信息保存在OSD对应的peer MISSING中的pg MISSING_t结构中。

□缺失对象的位置信息保存在类MissingLoc中。

根据以上信息，就可以知道该PG里各个OSD缺失的对象信息，以及该缺失的对象目前在哪些OSD上有完整的信息。基于上面的信息，数据修复过程就相对比较清晰：

□对于主OSD缺失的对象，随机选择一个拥有该对象的OSD，把数据拉取过来。

□对于 replica 缺失的对象，从主副本上把缺失的对象数据推送到从副本上来完成数据的修复。

□对于比较特殊的快照对象，在修复时加入了一些优化的方法。

# 11.3.1 触发修复

Recovery过程由PG的主OSD来触发并控制整个修复的过程。在修复的过程中，先

修复主OSD上缺失（或者不一致）的对象，然后修复从OSD上缺失的对象。由数据修复状态转换过程可知，当PG处于Activate/Recovering状态后，该PG被加入到OSD的RecoveryWQ工作队列中。在recovery_wq里，其工作队列的线程池的处理函数调用do_recovery函数来执行实际的数据修复操作：

void OSD::do_recovery(PG *pg, ThreadPool::TPHandle &handle) 

函数do_recovery由RecoveryWQ工作队列的线程池的线程执行。其输入的参数为要修复的PG，具体处理流程如下：

1）配置选项osd_recovery_sleep设置了线程做一次修复后的休眠时间。如果设置了该值，每次线程开始先休眠相应的时间长度。该参数默认值为0，不需要休眠。

2）加 recovery_wq.lock() 锁，用来保护 recovery_wq 队列以及变量 recoveryOps.active。计算可修复对象的 max 值，其值为允许修复的最大对象数 osd_recovery_max.active 减去正在修复的对象数 recoveryOps.active，然后调用函数 recovery_wq.unlock() 解锁。

3）如果max小于等于0，即没有修复对象的配额，就把PG重新加入工作队列recovery_wq中并返回；否则如果max大于0，调用pg->lock_suspend_timeout(handle)重新设置线程超时时间。检查PG的状态，如果该PG处于正在被删除状态，或者既不处于peered状态，也不是主OSD，则直接退出。

4）调用函数 pg->start_recoveryOps 修复，返回值 more 为还需要修复的对象数目。输出参数 started 为已经开始修复的对象数。

5）如果 more 为 0，也就是没有修复的对象了。但是 pg->have_unfound() 不为 0，还有 unfound 对象（即缺失的对象，目前不知道在哪个 OSD 上能找到完整的对象），调用函数 discover_all MISSING 在 might_have_unfound 队列中的 OSD 上继续查找该对象，查找的方法就是给相关的 OSD 发送获取该 OSD 的 pg_log 的消息。

6）如果 rctx(query_map->empty() 为空，也就是没有找到其他 OSD 去获取 pg_log 来查找 unfound 对象，就结束该 PG 的 recover 操作，调用函数从 recovery_wq_.dequeue(pg) 删除 PG。

7）函数dispatch_context做收尾工作，在这里发送query_map的请求，把ctx transaction的事务提交到本地对象存储中。

由上过程分析可知，do_recovery函数的核心功能是计算要修复对象的max值，然后

调用函数 start_recovery ops 来启动修复。

# 11.3.2 ReplicatedPG

类ReplicatedPG用于处理Replicate类型PG的相关修复操作。下面分析它用于修复的start_recoveryOps函数及其相关函数的具体实现。

# 1. start_recoveryOps

函数 start_recoveryOps 调用 recovery_primary 和 recovery Replica 来修复该 PG 上对象的主副本和从副本。修复完成后，如果仍需要 Backfill 过程，则抛出相应事件触发 PG 状态机，开始 Backfill 的修复进程。

```cpp
bool ReplicatedPG::start_recoveryOps(int max, RecoveryCtx *prctx, ThreadPool::TPHandle &handle, int *ops_started) 
```

该函数具体处理过程如下：

1）首先检查OSD，确保该OSD是PG的主OSD。如果PG已经处于PG_STATE_RECOVERING或者PG_STATE_BACKFILL的状态则退出。

2）从 pg_log 获取 missing 对象，它保存了主 OSD 缺失的对象。参数 num MISSING 为主 OSD 缺失的对象数目；num_unfound 为该 PG 上缺失的对象却没有找到该对象其他正确副本所在的 OSD；如果 num MISSING 为 0，说明主 OSD 不缺失对象，直接设置 info.lastcomplete 为最新版本 info.last_update 的值。

3）如 num MISSING 等于 num_unfound，说明主 OSD 所缺失对象都为 unfound 类型的对象，先调用函数 recover Replica 启动修复 replica 上的对象。

4）如果started为0，也就是已经启动修复的对象数量为0，调用函数recover_primary修复主OSD上的对象。

5）如果started仍然为0，且num_unfound有变化，再次启动recover_re replicas修复副本。

6）如果started不为零，设置work_in_progress的值为true。

7）如果 recovering 队列为空，也就是没有正在进行 Recovery 操作的对象，状态为 PG_STATE_BACKFILL，并且 backfill_targets 不为空，started 小于 max，missing.num MISSING()

为0，的情况下：

a）如果标志get_osmap()->test_flag(CEPH_OSDMAP_NOBACKFILL)设置了，就推迟Backfill过程。

b）如果标志CEPH_OSDMAP_NOREBALANCE设置了，且是degrade的状态，推迟Backfill过程。

c）如果 backfill_reserved 没有设置，就抛出 RequestBackfill 事件给状态机，启动 Backfill 过程。

d）否则，调用函数 recover_fill 开始 Backfill 过程。

8）最后PG如果处于PG_STATE_RECOVERING状态，并且对象修复成功，就检查：如果需要Backfill过程，就向PG的状态机发送RequestBackfill事件；如果不需要Backfill过程，就抛出AllReplicasRecovered事件。

9）否则，PG的状态就是PG_STATE_BACKFILL状态，清除该状态，抛出Backfilled事件。

# 2. recover_primary

函数 recover_primary 用来修复一个 PG 的主 OSD 上缺失的对象：

```txt
int ReplicatedPG::recover primary(int max, ThreadPool::TPHandle &handle) 
```

其处理过程如下：

1）调用pgfrontend->open_recovery_op返回一个PG类型相关的PGBackend::RecoveryHandle。对于ReplicatedPG对应的RPGHandle，内部有两个map，保存了Push和Pull操作的封装PushhOp和PullOP：

```cpp
struct RPGHandle : public PGBackend::RecoveryHandle {
    map<pg_shard_t, vector<PushOp> > pushes;
    map<pg_shard_t, vector<PullOp> > pulls;
}; 
```

2）last_requested为上次修复的指针，通过调用low_bound函数来获取还没有修复的对象。

3）遍历每一个未被修复的对象：latest为日志记录中保存的该缺失对象的最后的一条日志，soid为缺失的对象。如果latest不为空：

a）如果该日志记录是pg_log_entry_t::CLONE类型，这里不做任何的特殊处理，直到成功获取snapshot 相关的信息SnapSet后再处理。

b）如果该日志记录类型为pg_log_entry_t::LOST_REVERT类型：该revert操作为数据不一致时，管理员通过命令行强行回退到指定版本，reverting_to记录了回退的版本号：

- 如果 item.have 等于 latest->reverting_to 版本，也就是通过日志记录显示当前已经拥有回退的版本，那么就获取对象的 ObjectContext，如果检查对象当前的版本 obc->obs.oi-version 等于 latest->version，说明该回退操作完成。

- 如果 item.have 等于 latest->reverting_to，但是对象当前的版本 obc->obs.oi.version 不等于 latest->version，说明没有执行回退操作，直接修改对象的版本号为 latest->version 即可。

- 否则，需要拉取该 reverting_to 版本的对象，这里不做特殊的处理，只是检查所有 OSD 是否拥有该版本的对象，如果有就加入到 missing_location 记录该版本的位置信息，由后续修复继续来完成。

c）如果该对象在 recovering 过程中，表明正在修复，或者其 head 对象正在修复，跳过，并计数增加 skipped；否则调用函数 recover MISSING 来修复。

4）调用函数pgfrontend->run_recovery_op，把PullOp或者PushOp封装的消息发送出去。

下面举例说明，当最后的日志记录类型为 LOST_REVERT 时的修复过程。

# 例11-1 日志修复过程。

PG日志的记录如下：每个单元代表一条日志记录，分别为对象的名字和版本以及操作，版本的格式为（epoch,version）。灰色的部分代表本OSD上缺失的日志记录，该日志记录是从权威日志记录中拷贝过来的，所以当前该日志记录是连续完整的。

<table><tr><td>obj2(1,3) modify</td><td>obj1(1,4) modify</td><td>obj2(1,5) modify</td><td>obj1(1,6) modify</td><td>obj1(1,7) modify</td><td>obj1(1,8) modify</td></tr></table>

情况1：正常情况的修复。

缺失的对象列表为 [obj1, obj2]。当前修复对象为 obj1。由日志记录可知：对象 obj1 被修改过三次，分别为版本 6,7,8。当前拥有的 obj1 对象的版本 have 值为 4，修复时只修复到最后修改的版本 8 即可。

情况2：最后一个操作为LOST_REVERT类型的操作。

<table><tr><td>obj2(1,3) modify</td><td>obj1(1,4) modify</td><td>obj2(1,5) modify</td><td>obj1(1,6) modify</td><td>obj1(1,7) modify</td><td>obj1(1,8)
lost_revert_
version = 8
prior_version=7
reverting_to=4</td></tr></table>

对于要修复的对象 obj1，最后一次操作为 LOST_REVERT 类型的操作，该操作当前版本 version 为 8，修改前的版本 prior_version 为 7，回退版本 reverting_to 为 4。

在这种情况下，日志显示当前已经有版本4，检查对象obj1的实际版本，也就是object_info里保存的版本号：

1）如果该值是8，说明最后一次revert操作成功，不需要做任何修复动作。

2）如果该值是4，说明LOST_REVERT操作就没有执行。当然数据内容已经是版本4了，只需要修改object_info的版本为8即可。

如果回退的版本 reverting_to 不是版本 4，而是版本 6，那么最终还是需要把 obj1 的数据修复到版本 6 的数据。Ceph 在这里的处理，仅仅是检查其他 OSD 缺失的对象中是否有版本 6，如果有，就加入到 missing_location 中，记录拥有该版本的 OSD 位置，待后续继续修复。

# 3. recover MISSING

函数 recovery MISSING 处理 snap 对象的修复。在修复 snap 对象时，必须首先修复 head 对象或者 snapdir 对象，获取 SnapSet 信息，然后才能修复快照对象自己。

```cpp
int ReplicatedPG::recover MISSING(const hobject_t &soid, eversion_t v, int priority, PGBackend::RecoveryHandle *h) 
```

具体实现如下：

1）检查如果对象soid是unfound，直接返回PULL_NON值。暂时无法修复处于unfound的对象。

2）如果修复的是snap对象：

a）查看如果对应的head对象处于missing，递归调用函数recover MISSING先修复head对象。

b）查看如果snapdir对象处于missing，就递归调用函数recover MISSING先修复snapdir对象。

3）从head对象或者snapdir对象中获取head_abc信息。

4）调用函数pgfrontend->recover_object把要修复的操作信息封装到PullOp或者PushOp对象中，并添加到RecoveryHandle结构中。

# 11.3.3 pg后台

pgfrontend封装了不同类型的Pool的实现。ReplicatedBackend实现了replicate类型的PG相关的底层功能，EC后台实现了Erasure code类型的PG相关的底层功能。

由11.3.2节的分析可知，需要调用pgfrontend的recovery_object函数来实现修复对象的信息封装。这里只介绍基于副本的。

函数 recovery_object 实现 pull 操作，调用 preparePull 操作把请求封装成 PullOp 结构。如果是 push 操作，就调用 start_pushes 把请求封装成 PushOp 的操作。

# 1. pull 操作

PreparePull函数把要拉取的object相关的操作信息打包成PullOp类信息，如下所示：

```cpp
void ReplicatedBackend::preparePull(  
eversion_t v, //要拉取对象的版本信息  
const hobject_t& soid, //要拉取的对象  
ObjectContextRef headctx, //拉取对象的 ObjectContext 信息  
RPGHandle *h) //封装后保存的 RecoveryHandle
```

难点在于snap对象的修复处理过程。下面先介绍PullOp数据结构。


PullOp数据结构如下：


```cs
struct PullOp {
    hobject_t soid; //需要拉取的对象
    ObjectRecoveryInfo recovery_info; //对象修复的信息
    ObjectRecoveryProgress recovery_progress; //对象修复进度信息
}
```

函数preparePull具体处理过程如下：

1）通过调用函数get_parent()来获取PG对象的指针。pg属于自己的是相应的PG对象。通过PG获取missing、peer MISSING、missing_location等信息。

2）从soid对象对应的missing_location的map中获取该soid对象所在的OSD集合。把该集合保存在shuffle这个向量中。调用random Shuffle操作对OSD列表随机排序，然后选择向量中首个OSD作为缺失对象来拉取源OSD的值。从这一步可知，当修复主OSD上的对象，而多个从OSD上有该对象时，随机选择其中一个源OSD来拉取。

3）当选择了一个源shard之后，查看该shard对应的peer MISSING来确保该OSD上不缺失该对象，即确实拥有该版本的对象。

4）确定拉取对象的数据范围：

a）如果是head对象，直接拷贝对象的全部，在copy Subset加入区间（0，-1），表示全部拷贝，最后设置size为-1：

recovery_info.copy Subset.insert(0， (uint64_t)-1);   
recovery_info.size $=$ ((uint64_t)-1); 

b）如果该对象是snap对象，确保head对象或者snapdir对象二者必须存在一个。如果headctx不为空，就可以获取SnapSetContext对象，它保存了snapshot相关的信息。调用函数calcclone_subsets来计算需要拷贝的数据范围。

5）设置PullOp的相关字段，并添加到RPGHandle中。

函数 calc Clone_subsets 用于修复快照对象。在介绍它之前，这里需要介绍 DataSet 的数据结构和 clone 对象的 overlap 概念。

在SnapSet结构中，字段clone_overlap保存了clone对象和上一次clone对象的重叠的部分：

```c
struct SnapSet {
    snapid_t seq;
    bool head_exists;
    vector<snapid_t> snaps; //序号降序排列
    vector<snapid_t> clones; //序号升序排列
    map<snapid_t, interval_set<uint64_t> > clone_overlap;
    //写操作导致的和最新的克隆对象重叠的部分
    map<snapid_t, uint64_t> clone_size;
};
```

下面通过一个示例来说明clone_overlap数据结构的概念。

例11-2 clone_overlap数据结构如图11-2所示：

![image](https://cdn-mineru.openxlab.org.cn/result/2026-04-30/bbb6afcc-af41-45b5-9598-d269ca9c27e9/d7a6aac8fee61cfd9efff2c4890fa42965c5082b27598bd0a889d3d88f6d49ea.jpg)



图11-2clone_overlap示意图


snap3从snap2对象clone出来，并修改了区间3和4，其在对象中范围的offset和length为（4,8）和（8,12）。那么在SnapSet的clone_overlap中就记录：

```txt
clone_overlap[3] = {4,8}, (8,12)} 
```

函数 calc_clone_subsets 用于修复快照对象时，计算应该拷贝的数据区间。在修复快照对象时，并不是完全拷贝快照对象，这里用于优化的关键在于：快照对象之间是有数据重叠，数据重叠的部分可以通过已存在的本地快照对象的数据拷贝来修复；对于不能通过本地快照对象拷贝修复的部分，才需要从其他副本上拉取对应的数据。

函数 calc_clone_subsets 具体实现如下：

1）首先获取该快照对象的size，把（0,size）加入到data Subset中：

data Subset.insert(0, size); 

2）向前查找（oldest snap）和当前快照相交的区间，直到找到一个不缺失的快照对象，添加到clone_subsets中。这里找的不重叠区间，是从不缺失快照对象到当前修复的快照对象之间从没有修改过的区间，所以修复时，直接从已存在的快照对象拷贝所需区间数据即可。

3）同理，向后查找（newset snap）和当前快照对象相重叠的对象，直到找到一个不缺失的对象，添加到clone_subsets中。

4）去除掉所有重叠的区间，就是需要拉取的数据区间：

data Subset subtraction(cloning); 

对于上述的算法，下面举例来说明。

例 11- 3 快照对象修复示例如图 11- 3 所示：

![image](https://cdn-mineru.openxlab.org.cn/result/2026-04-30/bbb6afcc-af41-45b5-9598-d269ca9c27e9/b487abb9094cab4a9dd73b3da5f53c8b8f13c062439c23528013cd868689e113.jpg)



图11-3 快照对象修复计算示例图


要修复的对象为snap4，不同长度代表各个clone对象的size是不同的，其中灰色的区间代表clone后修改的区间。snap2、snap3和snap5都是已经存在的非缺失对象。

算法处理流程如下：

1）向前查找和snap4重叠的区间，直到遇到非缺失对象snap2为止。从snap4到snap2一直重叠的区间为1,5,8三个区间。因此，修复对象snap4时，修复1,5,8区间的数据，可以直接从已经存在的本地非缺失对象snap2拷贝即可。

2）同理，向后查找和snap4重叠的区间，直到遇到非缺失对象snap5为止。snap5和snap4重叠的区间为1,2,3,4,7,8六个区间。因此，修复对象snap4时，直接从本地对象snap4中拷贝区间1,2,3,4,7,8即可。

3）去除上述本地就可修复的区间，对象snap4只有区间6需要从其他OSD上拷贝数据来修复。

# 2.push操作

函数 start pushes 获取 actingbackfill 的 OSD 列表，通过 peer MISSING 查找缺失该对象的 OSD，调用 prep.push_to Replica 打包 PushOp 请求。

函数 prep.push_to Replica 实现过程如下：

1）如果需要push的对象是snap对象：检查如果head对象缺失，调用prep.push推送head对象；如果是headdir对象缺失，则调用prep.push推送headdir对象。

2）如果是snap对象，调用函数calc_clone_subsets来计算需要推送的快照对象的数据区间。

3）如果是head对象，调用calc_head_subsets来计算需要推送的head对象的区间，其原理和计算快照对象类似，这里就不详细说明了。最后调用prep.push封装PushInfo信息，在函数build.push_op里读取要push的实际数据。

# 3. 处理修复操作

函数 run Recover_op 调用 send PUSH 函数和 send pulls 函数把请求发送给相关的 OSD，这个流程比较简单。

当主OSD把对象推送给缺失该对象的从OSD后，从OSD需要调用函数handle.push来实现数据写入工作，从而来完成该对象的修复。同样，当主OSD给从OSD发起拉取对

象的请求来修复自己缺失的对象时，需要调用函数 handle_pulls 来处理该请求的应对。

在函数ReplicatedBackend::handle.push里处理handle.push的请求，主要调用submit.push_data函数来写入数据。

handle pulls 函数收到一个 PullOp 操作，返回 PushOp 操作，处理流程如下：

1）首先调用store->stat函数，验证该对象是否存在，如果不存在，则调用函数prep.push_op_blank，直接返回空值。

2）如果该对象存在，获取 ObjectRecoveryInfo 和 ObjectRecoveryProgress 结构。如果 progress.first 为 true 并且 recovery_info.size 为 -1，说明是全拷贝修复：将 recovery_info.size 设置为实际对象的 size，清空 recovery_info.copy Subset，并把 (0, size) 区间添加到 recovery_info.copy Subset.insert(0, st.st_size) 的拷贝区间。

3）调用函数 build.push_op，构建 PullOp 结构。如果出错，调用 prep.push_op_ blank，直接返回空值。

函数 build.push_on 完成构建 push 的请求。具体处理如下：

1）如果progress.first为true，就需要获取对象的元数据信息。通过store->omap_get_header获取omap的header信息，通过store->getattrs获取对象的扩展属性信息，并验证oi.version是否为recovery_info(version；否则返回-EINVALID值。如果成功，new_progress.first设置为false。

2）上一步只是获取了omap的header信息，并没有获取omap信息。这一步首先判断progressomapcomplete是否完成，（初始化设置为false）如果没有完成，就迭代获取omap的（key, value）信息，并检查一次获取信息的大小不能超过cct->conf->osd_recovery_max_chunk设置的值（默认为8MB）。特别需要注意的是，当该配置参数的值小于一个对象的size时，一个对象的修复需要多次数据的push操作。为了保证数据的完整一致性，先把数据拷贝到PG的temp存储空间。当拷贝完成之后，再移动到该PG的实际空间中。

3）开始拷贝数据：检查 recovery_info.copy Subset，也就是拷贝的区间。

4）调用函数store->fiemap来确定有效数据的区间out_op->data INCLUDED的值，通过store->read读取相应的数据到data里。

（5）设置PullOp的相关的字段，并返回。

# 11.4 Backfill 过程

当PG完成了Recovery过程之后，如果backfill_targets不为空，表明有需要Backfill过程的OSD，就需要启动Backfill的任务，来完成PG的全部修复。下面介绍Backfill过程相关的数据结构和具体处理过程。

# 11.4.1 相关数据结构

数据结构 BackfillInterval 用来记录每个 peer 上的 Backfill 过程。其字段说明如下：

version：记录扫描对象列表时，当前PG对象更新的最新版本，一般为last_update，由于此时PG处于active状态，可能正在进行写操作。其用来检查从上次扫描到现在是否有对象写操作。如果有，完成写操作的对象在已扫描的对象列表中，进行Backfill操作时，该对象就需要更新为最新版本。

□ objects：扫描到的准备进行 Backfill 操作的对象列表。

□begin：当前处理的对象。

□ end：本次扫描对象的结束，用于作为下次扫描对象的开始：

```txt
struct BackfillInterval {
    // 一个peer的backfill_interval信息
    eversion_t version; //扫描时的最新对象版本
    map<Hobject_t, eversion_t, hobject_t::Comparator> objects;
    bool sort_bitwise;
    hobject_t begin; //当前处理的对象
    hobject_t end; //本次扫描对象的结束
}
```

# 11.4.2 Backfill的具体实现

函数 recovery_backfill 作为 Backfill 过程的核心函数，控制整个 Backfill 修复进程。其工作流程如下。

# 1）初始设置。

在函数on Activate里设置了PG的属性值new_backfill为true，设置了last_backfill_started为earliest_backfill()的值。该函数计算需要backfill的OSD中，peer_info信息里保存的last_backfill的最小值。

peer_backfill_info 的 map 中保存各个需要 Backfill 的 OSD 所对应 backfillInterval 对象信息。首先初始化 begin 和 end 都为 peer_info.last_backfill，由 PG 的 Peering 过程可知，在函数 activate 里，如果需要 Backfill 的 OSD，设置该 OSD 的 peer_info 的 last_backfill 为 hobject_t()，也就是 MIN 对象。

backfills_in_flight 保存了正在进行 Backfill 操作的对象，pending_backfill Updates 保存了需要删除的对象。

2）设置backfill_info.begin为last_backfill_started，调用函数update_range来更新需要进行Backfill操作的对象列表。

3）根据各个peer_info的last_backfill对相应的backfillInterval信息进行trim操作。根据last_backfill_started来更新backfill_info里相关字段。

4）如果 backfill_info.begin 小于等于 earliestpeer_backfill()，说明需要继续扫描更多的对象，backfill_info 重新设置，这里特别注意的是，backfill_info 的 version 字段也重新设置为（0,0），这会导致在随后调用的 update_scan 函数再调用 scan_range 函数来扫描对象。

5）进行比较，如果pbi.begin小于backfill_info.begin，需要向各个OSD发送MOSDPGScan::OP.scan_GET_DIGEST消息来获取该OSD目前拥有的对象列表。

6）当获取所有OSD的对象列表后，就对比当前主OSD的对象列表来进行修复。

7）check对象指针，就是当前OSD中最小的需要进行Backfill操作的对象：

a）检查check对象，如果小于Backfill_info.begin，就在各个需要Backfill操作的OSD上删除该对象，加入到to_remove队列中。

b）如果 check 对象大于或者等于 backfill_info.begin，检查拥有 check 对象的 OSD，如果版本不一致，加入 need_ver_targ 中。如果版本相同，就加入 keep_ver_targs 中。

c）那些begin对象不是check对象的OSD，如果pinfo.last_backfil小于backfill_info.begin，那么，该对象缺失，加入missing_targs列表中。

d）如果 pinfo.last_backfil 大于 backfill_info.begin，说明该 OSD 修复的进度已经超越当前的主 OSD 指示的修复进度，加入 skip_targs 中。

8）对于keep_ver_targs列表中的OSD，不做任何操作。对于need_ver_targs和missing_targs中的OSD，该对象需要加入到to.push中去修复。

9）调用函数 send_remove_op 给 OSD 发送删除的消息来删除 to_remove 中的对象。

10）调用函数prep_backfill_object.push把操作打包成PushOp，调用函数pgfrontend-run_recovery_op把请求发送出去。其流程和Recovery流程类似。

11）最后用 new_last_backfill 更新各个 OSD 的 pg_info 的 last_backfill 值。如果 pinfo.last_backfill 为 MAX，说明 backfill 操作完成，给该 OSD 发送 MOSDPGBackfill::OPBackFILL_finish 消息；否则发送 MOSDPGBackfill::OP_BACKFILL_progress 来更新各个 OSD 上的 pg_info 的 last_backfill 字段。

下面举例说明。

例11-4如图11-4所示，该PG分布在5个OSD上（也就是5个副本，这里为了方便列出各种处理情况），每一行上的对象列表都是相应OSD当前对应backfillInterval的扫描对象列表。osd5为主OSD，是权威的对象列表，其他OSD都对照主OSD上的对象列表来修复。

<table><tr><td>osd0</td><td>obj4(1,1)
last_backfill
peer_backfill_info[0].begin</td><td>obj5(1,4)</td><td>obj6(1,10)</td></tr><tr><td>osd1</td><td></td><td>obj5(1,3)
last_backfill
peer_backfill_info[1].begin</td><td></td></tr><tr><td>osd2</td><td>obj4(1,1)
last_backfill
peer_backfill_info[2].begin</td><td>obj5(1,4)</td><td></td></tr><tr><td>osd3</td><td></td><td>obj6(1,1)
last_backfill
peer_backfill_info[3].begin</td><td>obj7(1,8)</td></tr><tr><td>osd4</td><td></td><td>obj5(1,4)
last_backfill
peer_backfill_info[4].begin</td><td>obj6(1,10)</td></tr><tr><td>osd5
(主)</td><td></td><td>obj5(1,4)
backfill_info.begin</td><td>obj6(1,10)</td></tr><tr><td></td><td>last_backfill_started</td><td></td><td></td></tr></table>

图11-4 backfill修复过程（1）

下面举例来说明步骤7中的不同的修复方法：

1）当前check对象指针为主OSD上保存的peer_backfill_info中begin的最小值。图

中 check 对象应为 obj4 对象。

2）比较 check 对象和主 osd5 上的 backfill_info.begin 对象，由于 check 小于 obj5，所以 obj4 为多余的对象，所有拥有该 check 对象的 OSD 都必须删除该对象。故 osd0 和 osd2 上的 obj4 对象被删除，同时对应的 begin 指针前移。

<table><tr><td>osd0</td><td>obj4(1,1)
last_backfill</td><td>obj5(1,4)
peer_backfill_info[0].begin</td><td>obj6(1,10)</td></tr><tr><td>osd1</td><td></td><td>obj5(1,3)
last_backfill
peer_backfill_info[1].begin</td><td></td></tr><tr><td>osd2</td><td>obj4(1,1)
last_backfill</td><td>obj6(1,4)
peer_backfill_info[2].begin</td><td></td></tr><tr><td>osd3</td><td></td><td>obj6(1,1)
last_backfill
peer_backfill_info[3].begin</td><td>obj7(1,8)</td></tr><tr><td>osd4</td><td></td><td>obj5(1,4)
last_backfill
peer_backfill_info[4].begin</td><td>obj6(1,10)</td></tr><tr><td>osd5
(pirmary)</td><td></td><td>obj5(1,4)
backfill_info.begin</td><td>obj6(1,10)</td></tr><tr><td></td><td>last_backfill_started</td><td></td><td></td></tr></table>

图11-5 backfill修复过程（2）

3）当前各个OSD的状态如图11-5所示：此时check对象为obj5，比较check和backfill_info.begin的值：

a）对于当前begin为check对象的osd0、osd1、osd4：

- 对于 osd0 和 osd4，check 对象和 backfill_info.begin 对象都是 obj5，且版本号都为（1,4），加入到 keep_ver targs 列表中，不需要修复。

- 对于 osd1，版本号不一致，加入 need ver targs 列表中，需要修复。

b）对于当前begin不是check对象的osd2和osd3：

- 对于 osd2，其 last_backfill 小于 backfill_info.begin，显然对象 obj5 缺失，加入 missing	targs 修复。

- 对于 osd3，其 last backfill 大于 backfill info.begin，也就是说其已经修复到

obj6 了，obj5 应该已经修复了，加入 skip_targs 跳过。

4）步骤3处理完成后，设置last_backfill_started为当前的backfill_info.begin的值。backfill_info.begin指针前移，所有begin等于check对象的begin指针前移，重复以上步骤继续修复。

函数update_range调用函数scan_range更新BackfillInterval修复的对象列表，同时检查上次扫描对象列表中，如果有对象发生写操作，就更新该对象修复的版本。

具体实现步骤如下：

1）bi->version记录了扫描要修复的对象列表时PG最新更新的版本号，一般设置为last_update_applied或者info.last_update的值。初始化时，bi->version默认设置为（0,0），所以小于info.log.tail，就更新bi->version的设置，调用函数scan_range扫描对象。

2）检查如果bi->version的值等于info.last_update，说明从上次扫描对象开始到当前时间，PG没有写操作，直接返回。

3）如果bi->version的值小于info.last_update，说明PG有写操作，需要检查从bi->version到log_head这段日志中的对象：如果该对象有更新操作，修复时就修复最新的版本；如果该对象已经删除，就不需要修复，在修复队列中删除。

下面举例说明update_range的处理过程。

# 例11-5 update_range的处理过程

1）日志记录如下图所示：

<table><tr><td>obj1(1,2) 
modify</td><td>Obj1(1,3) 
modify</td><td>obj2(1,4) 
modify</td><td>obj3(1,5) 
modify</td><td>obj4(1,6) 
modify</td><td></td><td></td><td></td></tr></table>

扫描列表为：bi->objects [obj1(1,3), obj2(1,4) obj3(1,5) obj4(1,6)]

(begin) 

(end) 

BackfillInterval 的扫描的对象列表：bi->begin 为对象 obj1(1,3)，bi->end 为对象 obj6(1,6)，当前 info.last_update 为版本（1,6），所以 bi->version 设置为（1,6）。由于本次扫描的对象列表不一定能修复完，只能等下次修复。

2）日志记录如下图所示：

<table><tr><td>obj1(1,2) 
modify</td><td>Obj1(1,3) 
modify</td><td>obj2(1,4) 
modify</td><td>obj3(1,5) 
modify</td><td>obj4(1,6) 
modify</td><td>obj3(1,7) 
modify</td><td>obj4(1,8) 
delete</td><td></td></tr></table>

扫描列表为：bi->objects [obj1(1,3), obj2(1,4) obj3(1,5) obj4(1,6)]

(begin) 

(end) 

第二次进入函数 recover_backfill，此时 begin 对象指向了 obj2 对象。说明上次只完成了对象 obj1 的修复。继续修复时，期间有对象发生更新操作：

a）对象obj3有写操作，版本更新为（1,7）。此时对象列表中要修复的对象obj3版本（1,5），需要更新为版本（1,7）的值。

b）对象obj4发送删除操作，不需要修复了，所以需要从对象列表中删除。

综上所述可知，Ceph的Backfill过程是扫描OSD上该PG的所有对象列表，和主OSD做对比，修复不存在的或者版本不一致的对象，同时删除多余的对象。

# 11.5 本章小结

本章介绍了Ceph的数据修复的过程，有两个过程：Recovery过程和Backfill过程。Recovery过程根据missing记录，先完成主副本的修复，然后完成从副本的修复。对于不能通过日志修复的OSD，Backfill过程通过扫描各个部分上的对象来全量修复。整个Ceph的数据修复过程比较清晰，比较复杂的副本可能就是涉及快照对象的修复处理。

目前这部分代码是 Ceph 最核心的代码，除非必要，都不会轻易修改。目前社区也提出了修复时的一种优化方法。就是在日志里记录修改的对象范围，这样在 Recovery 过程中不必拷贝整个对象来修复，只修复修改过的对象对应的范围即可，这样在某些情况下可以减少修复的数据量。

![image](https://cdn-mineru.openxlab.org.cn/result/2026-04-30/bbb6afcc-af41-45b5-9598-d269ca9c27e9/770cc7ab405e26e2e9f4f17c0fa5db6f008b893602a9e84b866a0cbddd3a7c9e.jpg)


# 第12章

Chapter 12 

# Ceph 一致性检查

本章介绍Ceph的一致性检查工具Scrub机制。首先介绍数据校验的基本知识，其次介绍Scrub的基本概念，然后介绍Scrub的调度机制，最后介绍Scrub具体实现的源代码分析。

# 12.1 端到端的数据校验

在存储系统中可能会发生数据静默损坏（Silent Data Corruption），这种情况的发生大多是由于数据的某一位发生异常反转（Bit Error Rate）。

图12-1是一般存储系统的协议栈，数据损坏的情况会发生在系统的所有模块中：

□ 硬件错误，例如内存、CPU、网卡等。

□数据传输过程中的信噪干扰，例如SATA、FC等协议。

□固件bug，例如RAID控制器、磁盘控制、网卡等。

□软件bug，例如操作系统内核的bug，本地文件系统的bug，SCSI软件模块的bug等。

![image](https://cdn-mineru.openxlab.org.cn/result/2026-04-30/bbb6afcc-af41-45b5-9598-d269ca9c27e9/937ec670f0ea6c466337b5c8e7d0512444032f5f8b21239cbd99b24e5b8169d3.jpg)



图12-1 一般存储系统的协议栈


在传统的高端磁盘阵列中，一般采用端到端的数据校验实现数据的一致性。所谓端到端的数据校验，指客户端（应用层）在写入数据时，为每个数据块都计算一个CRC校验信息，并将这个校验信息和数据块发送至磁盘（Disk）。磁盘在接收到数据包之后，会重新计算校验信息，并和接收到的校验信息作对比。如果不一致，那么就认为在整个I/O路径上存在错误，返回I/O操作失败；如果校验成功，就把数据校验信息和数据保存在磁盘上。同样，在数据读取时，客户端再获取数据块和从磁盘读取校验信息时，也需要再次检查是否一致。

通过这种方法，应用层可以很明确地知道一次I/O请求的数据是否一致。如果操作成功，那么磁盘上的数据必然是正确的。

这种方式在不影响I/O性能或者影响比较小的情况下，可以提高数据读写的完整性。但这种方式也有一些缺点：

□无法解决目的地址错误导致的数据损坏问题

□ 端到端的解决方案需要在整个I/O路径上附加校验信息。现在的I/O协议栈涉及的模块比较多，每个模块都附加这种校验信息实现起来比较困难。

由于这种实现方式对 Ceph 的 I/O 性能影响比较大，所以 Ceph 并没有实现端到端的数据校验，而是实现 Ceph Scrub 机制，采用一种通过在后台扫描的方案来解决 Ceph 的一致

性检查。

# 12.2 Scrub概念介绍

Ceph在内部实现了数据一致性检查的一个工具：Ceph Scrub。其原理为：通过对比各个对象副本的数据和元数据，完成副本一致性检查。

这种方法的优点是在后台可以发现由于磁盘损坏而导致的数据不一致现象。缺点是发现的时机往往比较滞后。

Scrub 按照扫描的内容分为两种方式：

一种叫 Scrub，它仅仅通过对比对象各副本的元数据，来检查数据的一致性。由于只检查元数据，读取数据量和计算量都比较小，是一种比较轻度的检查。

□另一种叫deep-scrub，它进一步检查对象的数据内容是否一致，实现了深度扫描，几乎要扫描磁盘上的所有数据并计算crc32校验值，因此比较耗时，占用系统资源更多。

Scrub 按照扫描的方式分为两种：

□在线扫描：不影响系统正常的业务。

□ 离线扫描：需要整个系统暂停或者冻结。

Ceph的Scrub功能实现了在线检查，即不中断系统当前读写请求，客户端可以继续完成读写访问。整个系统并不会暂停，但是后台正在进行Scrub的对象要被锁定暂时阻止访问，直到该对象完成Scrub操作后才能解锁允许访问。

# 12.3 Scrub的调度

Scrub的调度解决了一个PG何时启动Scrub扫描机制。主要有以下方式：

□手动立即启动执行扫描。

□在后台设置一定的时间间隔，按照间隔的时间来启动。比如默认时间为一天执行一次。

□设置启动的时间段。一般设定一个系统负载比较轻的时间段来执行Scrub操作。

# 12.3.1 相关数据结构

在类PG里有下列与Scrub相关的数据结构：

```c
uint scribed_scrub_lock; //Scrub相关变量的保护锁
int scrubs_pending; //资源预约已经成功，正等待Scrub的PG
int scrubs.active; //正在进行Scrub的PG
set<ScrubJob> sched_scrub_pg; //PG对应的所有ScrubJob列表
```

结构体 ScrubJob 封装了一个 PG 的 Scrub 任务相关的参数：

```txt
struct ScrubJob{ spg_t pgid; //Scrub对应的PG utime_t sched_time; //Scrub任务的调度时间，如果当前负载比较高，或者当前的时间不在 设定的Scrub工作时间段内，就会延迟调度 utime_t deadline; //调度时间的上限，过了该时间必须进行Scrub操作，而不受系统负载和 Scrub时间段的限制   
}；
```

# 12.3.2 Scrub的调度实现

在OSD的初始化函数OSD::init中，注册了一个定时任务：

```txt
tick_timer_without_osd_lock.add_event_after(cct->conf->osd-heartbeat_interval, new C_Tick WithoutOSDLock(this)); 
```

该定时任务每隔osd-heartbeat_interval时间段（默认为1秒），就会触发定时器的回调函数OSD::tick_without_osd_lock()，处理过程的函数调用关系图如图12-2所示。

![image](https://cdn-mineru.openxlab.org.cn/result/2026-04-30/bbb6afcc-af41-45b5-9598-d269ca9c27e9/8154b268e428752ae7fd84c82e22657e0aa60a195bad074217f4c07205ea0b06.jpg)



图12-2 Scrub的触发函数调用关系


以上函数实现了PG的Scrub调度工作。下面将介绍处理过程中关键的两个函数OSD::sched_scrub和PG::sched_scrub函数的实现。

# 1.OSD::sched_scrub函数

本函数用于控制一个PG的Scrub过程启动时机，具体过程如下：

1）调用函数 can_ inc_scrubs_pending 来检查是否有配额允许 PG 开始启动 Scrub 操作。变量 scrubs_pending 记录了已经完成资源预约正在等待 Scrub 的 PG 的数量，变量 scrubs.active 记录了正在进行 Scrub 检查的 PG 数量，其二者的数量之和不能超过系统配置参数 cct->conf->osd_max_scrubs 的值。该值设置了同时允许 Scrub 的最大的 PG 数量。

2）调用函数scrub_time Permit检查是否在允许的时间段内。如果cct->conf->osd_scrub_begin_hour大于cct->conf->osd_scrub_end_hour，当前时间必须在二者设定的时间范围之间才允许。如果cct->conf->osd_scrub_begin_hour小于等于cct->conf->osd_scrub_end_hour，当前时间在二者设定的时间范围之外才允许。

3）调用函数scrub_loadbelow_threshold检查当前的系统负载是否允许。函数getloadavg获取最近1、5、15分钟的系统负载。

a）如果最近1分钟的负载小于cct->conf->osd_scrub_load_threshold的设定值，就允许执行。

b）如果最近1分钟的负载小于daily_loadavg的值，并且最近1分钟负载小于最近15分钟的负载，就允许执行。

4）获取第一个等待执行 Scrub 操作的 ScrubJob 列表，如果它的 scrub.sched_time 大于当前时间 now 值，说明时间不到，就跳过该 PG 先执行下一个任务。

5）获取该PG对应的PG对象，如果该PG的pgfrontend支持Scrub，并且处于active状态：

a）如果scrub.deadline小于now值，也就是已经超过最后的期限，必须启动Scrub操作。

b）或者此时timeisable并且load_is_low，也就是时间和负载都允许。

在上述两种情况下，调用函数 pg->sched_scrub() 来执行 Scrub 操作。

# 2.PG::sched_scrub函数

本函数实现了对执行 Scrub 任务时相关参数的设置，并完成了所需资源的预约。其处理过程如下：

1）首先检查PG的状态，必须是主OSD，并且处于active和clean状态，并且没有正在进行Scrub操作。

2）设置deep_scrub_interval值：如果该PG所在的pool选项中没有设置该值，就设置为系统配置参数cct->conf->osd_deep_scrub_interval的值。

3）检查是否启动deep_scrub，如果当前时间大于info_history.last_deep_scrub_stamp与deep_scrub_interval之和，就启动deep_scrub操作。

4）如果scrubber.must_scrub的值为true，为用户手动强制启动deep_scrub操作。如果该值为false，则需系统自动以一定的概率来启动deep_scrub操作，具体实现就是：自动产生一个随机数，如果该随机数小于cct->conf->osd_deep_scrub_randomize_ratio，就启动deep_scrub操作。

5）决定最终是否启动deep_scrub，在步骤3）和4）中只要有一个设置好，就启动deep_scrub操作。

6）如果 osdmap 或者 pool 中带有不支持 deep_scrub 的标记，就设置 time_for_deep 为 false，不启动 deep_scrub 操作。

7）如果 osdmap 或者 pool 中带有不支持 Scrub 的标记，并且也没有启动 deep-scrub 操作则返回并退出。

8）如果cct->conf->osd_scrub_AUTO repairing设置了自动修复，并且pg后台也支持，而且是deep_scrub操作，则进行如下判断过程：

a）如果用户设置了must repairing，或者must_scrub，或者must_deep_scrub，说明这次Scrub操作是用户触发的，系统尊重用户的选择，不会自动设置scrubber.auto repairing的值为true。

b）否则，系统就设置scrubber.auto repar的值为true来自动进行修复。

9）Scrub过程和Recovery过程类似，都需要耗费系统大量资源，需要去PG所在的OSD上预约资源。如果scrubber.reserved的值为false，还没有完成资源的预约，需进行如下操作：

a）把自己加入到scrubber.reserved_peers中。

b）调用函数scrub_reserved.replicas，向OSD发送CEPH_OSD_OP_SCRUB_RESERVE消息来预约资源。

c）如果scrubber.reserved_peers.size()等于acting.size(),表明所有的从OSD资源预约成功,把PG设置为PG_STATE_deep SCRUB状态。调用函数queue_

scrub 把该 PG 加入到工作队列 op_wq 中，触发 Scrub 任务开始执行。

# 12.4 Scrub的执行

Scrub的具体执行过程大致如下：通过比较对象各个OSD上副本的元数据和数据，来完成元数据和数据的校验。其核心处理流程在函数PG::chunky_scrub里控制完成。

# 12.4.1 相关数据结构

Scrub 操作相关的主要数据结构有两个，一个是 Scrubber 控制结构，它相当于是一次 Scrub 操作的上下文，控制一次 PG 的操作过程。另一个是 ScurbMap 保存需要比较的对象各个副本的元数据和数据的摘要信息。

# 1. Scrubber

结构体 Scrubber 用来控制一个 PG 的 Scrub 的过程：

```txt
struct Scrubber{ //元数据 set<pg_shard_t>reserved_peers; //资源预约的 shard bool reserved，reserve_failed; //是否预约资源，预约资源是否失败 epoch_t epoch_start; //开始Scrub操作的epoch   
//common to both scrubs bool active; //Scrub是否开始 bool queue_snapshot.Trim; //当PG有snap trimmed操作时，如果检查Scrubber处于active状态，说明正在进行Scrub操作，那么snap trimmed操作暂停，设置queue_snapshot_trim的值为true。当PG完成Scrub任务后，如果queue_snapshot_trim的值为true，就把PG添加到相应的工作队列里，继续完成snap trimmed操作。 intwaiting_on; //等待的副本计数 set<pg_shard_t>waiting_on_whom; //等待的副本 int shallow Errors; //轻度扫描错误数 intdeep errors; //深度扫描错误数 int fixed; //已经修复的对象数 Scrubber primary_scrrubmap; //主副本的ScrubMap map<pg_shard_t, ScrubMap>received_maps; //接收到的从副本的ScrubMap OpRequestRef active_rep_scrub;
```

```rust
utime_t scrub_reg_stamp; // stamp we registered for  
// flags to indicate explicitly requested scrubs (by admin)  
bool must_scrub, must_deep_scrub, must_repair;  
bool auto_repair; //是否自动修复  
// Maps from objects with errors to missing/inconsistent peers  
map<Hobject_t, set<pg_shard_t>, hobject_t::BitwiseComparator> missing;  
// 扫描出的缺失对象  
map<Hobject_t, set<pg_shard_t>, hobject_t::BitwiseComparator> inconsistent;  
// 扫描出的不一致对象  
// Map from object with errors to good peers  
map<Hobject_t, list<pair<ScrubMap::object, pg_shard_t>>, hobject_t::BitwiseComparator> authoritative;  
// 如果所有副本对象中有不一致的对象，authoritative记录了正确对象所在的OSD  
// digest updates which we are waiting on  
int num_digest Updates_pending; //等待更新digest的对象数目  
hobject_t start, end; //扫描对象列表的开始和结尾  
eversion_t subset_last_update; //扫描对象列表中最新的版本号  
bool deep; //是否为深度扫描  
uint32_t seed; //计算crc32校验码的种子  
list<Context*> callbacks;  
} scrubber;
```

# 2. ScrubMap


数据结构ScrubMap保存准备校验的对象以及相应的校验信息：


```txt
struct ScrubMap {
struct object {
map<string,bufferptr> attrs; //对象的属性
set<snapid_t> snapcolls; //该对象所有的snap序号
uint64_t size; //对象的size
_u32 omap_digest; //omap的crc32c校验码
_u32_digest; //对象数据的crc32c校验码
uint32_t nlinks; //snap对象（clone对象）对应的snap的数量
bool negative:1;
bool digest_present:1; //是否计算了数据的校验码标志
bool omap_digest_present:1; //是否有omap的校验码标志
bool read_error:1; //读对象的数据出错标志
bool stat_error:1; //调用stat获取对象的元数据出错标志
};
```

map<hoject_t,object,hobject_t::BitwiseComparator>objects; //需要校验的对象（hobject） $\rightarrow$ 校验信息（Object）的映射 eversion_t valid_through; eversion_t incr_since;

内部类object用来保存对象需要校验的信息，包括以下5个方面：

□对象的大小（size）

□对象的属性（attrs）

□对象omap的校验码（digest）

对象数据的校验码（digest）

□对象所有clone对象的快照序号

# 12.4.2 Scrub的控制流程

Scrub 的任务由 OSD 的工作队列 OpWq 来完成，调用对应的处理函数 pg->scrub(handle) 来执行。

PG::scrub函数最终调用PG::chunky_scrub函数来实现。PG::chunky_scrub函数控制了Scrub操作状态转换和核心处理过程。

具体过程分析如下所示：

1）Scrubber的初始状态为PG::Scrubber::INACTIVE，该状态的处理如下：

a）设置scrubber_epoch_start的值为info-history同样的intervalsince。

b）设置scrubber.active的值为true。

c）设置状态scrubber.state的值为PG::Scrubber::NEW_chunk。

d）根据peer_features，设置scrubber.seed的类型，这个seed是计算crc32的初始化哈希值。

2）PG::Scrubber::NEW_chunk状态的处理如下：

a）调用get_pgbackend()->objects_list_partial函数从start对象开始扫描一组对象，一次扫描的对象数量在如下两个配置参数之间：cct->conf->osd_scrub_chunk_

min（默认值为5）和cct->conf->osd_scrub_chunk_max（默认值为25）。

b）计算出对象的边界。相同的对象具有相同的哈希值。从列表后面开始查找哈希值不同的对象，从该地方划界。这样做的目的是把一个对象的所有相关对象（快照对象、回滚对象）划分在一次扫描校验过程中。

c）调用函数_range-available_for_scrub检查列表中的对象，如果有被阻塞的对象，就设置done的值为true，退出PG本次的Scrub过程。

d）根据 pg_log 计算 start 到 end 区间对象最大的更新版本号，这个最新版本号设置在 scrubber.subset_last_update 里。

e）调用函数_request_scrub_map 向所有副本发送消息，获取相应 ScrubMap 的校验信息。

f) 设置状态为 PG::Scrubber::WAIT_PUSHES。

3）PG::Scrubber::WAIT_PUSHES状态的处理如下：

a）如果 active.pushes 的值为 0，设置状态为 PG::Scrubber::WAIT LAST_UPDATE，进入下一个状态处理。

b）如果 active pushes 不为 0，说明该 PG 正在进行 Recovery 操作。设置 done 的值为 true，直接结束。在进入 chunky_scrub 时，PG 应该处于 CLEAN 状态，不可能有 Recovery 操作，这里的 Recovery 操作可能是上次进行 chunky_scrub 操作后的修复操作。

4）PG::Scrubber::WAIT LAST_UPDATE状态的处理如下：

a）如果last_update_applied的值小于scrubber.subset_last_update的值，说明虽然已经把操作写入了日志，但是还没有应用到对象中。由于Scrub操作后面的步骤有对象的读操作，所以需要等待日志应用完成。设置done的值为true结束本次PG的Scrub过程。

b）否则就设置状态为PG::Scrubber::BUILD_MAP。

5）PG::Scrubber::BUILD_MAP状态的处理如下：

a）调用函数 build_scrub_map_chunk 构造主.OSD 上对象的 ScrubMap 结构。

b）如果构造成功，计数scrubber.waiting_on的值减1，并从队列中删除scrubber.

waiting_on_whom，则相应的状态设置为PG::Scrubber::WAIT_REPLICAS。

6）PG::Scrubber::WAIT_REPLICAS状态的处理如下：

a）如果scrubber.waiting_on不为零，说明有replica请求没有回答，设置done的值为true，退出并等待。

b）否则，进入PG::Scrubber::COMPARE_maps状态。

7）PG::Scrubber::COMPARE_maps状态的处理如下：

a）调用函数scrub(compare_maps比较各副本的校验信息。

b）将参数scrubber.start的值更新为scrubber.end。

c）调用函数requeueOps，把由于Scrub而阻塞的读写操作重新加入操作队列里执行。

d）状态设置为PG::Scrubber::WAIT_DIGEST_UPDATE。

8）PG::Scrubber::WAIT_DIGEST_UPDATEs状态的处理如下：

a）如果有scrubber num_digest Updates pending等待，等待更新数据的digest或者omap的digest。

b）如果scrubber.end小于hobject_t::get_max()，本PG还有没有Scrub操作完成的对象，设置状态scrubber.state为PG::Scrubber::NEW_CHUNK，继续把PG加入到osd->scrub_wq中。

c）否则，设置状态为PG::Scrubber::FINISH值。

9）PG::Scrubber::FINISH状态的处理如下：

a）调用函数scrub_finish来设置相关的统计信息，并触发修复不一致的对象。

b）设置状态为PG::Scrubber::INACTIVE。

# 12.4.3 构建ScrubMap

构建ScrubMap有多个函数实现，下面分别介绍。

1. build_scrub_map_chunk 

函数 build_scrub_map_chunk 用于构建从 start 到 end 之间的所有对象的校验信息并保

存在 ScrubMap 结构中。

```rust
int PG::build_scrub_map_chunk( ScrubMap &map, hobject_t start, hobject_t end, bool deep, uint32_t seed, ThreadPool::TPHandle &handle) 
```

处理过程分析如下：

1）设置 map.valid through 的值为 info.last update。

2）调用get_pgbackend()->objects_list_range函数列出所有的start和end范围内的对象，ls队列存放head和snap对象，rollback_obs队列存放用来回滚的ghobject t对象。

3）调用函数get pgbackend()->be scan list扫描对象，构建ScrubMap结构。

4）调用函数_scan_rollback_obs 来检查回滚对象：如果对象的 generation 小于 last_rollback_info trimmed to applied 值，就删除该对象。

5）调用scan snaps来修复SnapMapper里保存的snap信息。

# 2. _scan_snets

函数_scan_snaps 扫描 head 对象保存的 snap 信息和 SnapMapper 中保存的该对象的 snap 信息是否一致。它以前者保存的对象 snap 信息为准，修复 snapMapper 中保存的对象 snap 信息。

```autohotkey
void PG::_scan_snap (ScrubMap &smap) 
```

具体实现过程为：对于ScrubMap里的每一个对象循环做如下操作：

1）如果对象的hoid.snap的值小于CEPH_MAXSNAP的值，那么该对象是snap对象，从o.attrs[OI_ATTR]里获取object_info_t信息。

2）检查oi的snaps。如果oi.snaps.empty()为0，设置nlinks等于1；如果oi.snaps.size()为1，设置nlinks等于2；否则设置nlinks等于3。

3）从oi获取oi_saps，从snap mapper获取cur_saps，比较两个snap信息，以oi的信息为准：

a）如果函数 snap Mapper.get_snapshot(hoid, &cur_snapshot) 的结果为 -ENOENT，就把信息该添加到 snap Mapper 里。

b）如果信息不一致，先删除snap Mapper里不一致的对象，然后把该对象的snap信息添加到snap Mapper里。

# 3.be_scan_list

函数 be_scan_list 用于构建 ScubMap 中对象的校验信息：

```txt
void PGBackend::be_scan_list( ScrubMap &map, const vector<Hobject_t> &ls, bool deep, uint32_t seed, ThreadPool::TPHandle &handle) 
```

具体处理过程就是循环扫描Is向量中的对象：

1）调用store->stat获取对象的stat信息：

a）如果获取成功，设置o.size的值等于st.st_size，并调用store->getattrs把对象的attr信息保存在o_attrs里。

b）如果stat返回结果r为-ENOENT，就直接跳过该对象（该对象在本OSD上可能缺失，在后面比较结果时会检查出来）。

c）如果stat返回结果r为-EIO，就设置o.stat_error的值为true。

2）如果deep的值为true，调用函数be_deep_scrub进行深度扫描，获取对象的omap和data的digest信息。

# 4. be_deep_scrub

函数be_deep_scrub实现对象的深度扫描：

```cpp
void ReplicatedBackend::be_deep_scrrub(
const hobject_t &poid, //深度扫描的对象
uint32_t seed, //crc32的种子
ScrubMap::object &o, //保存对应的校验信息
ThreadPool::TPHandle &handle)
```

实现过程分析如下：

1）设置 data 和 omap 的 bufferhash 的初始值都为 seed。

2）循环调用函数store->read读取对象的数据，每次读取长度为配置参数cct->conf->osd_deep_scrub_stride（512k），并通过bufferhash计数crc32校验值。如果中间出错（r==-EIO)，就设置o.read_error的值为true。最后设置o.digest为计算出crc32的校验值，设置o.digest_present的值为true。

3）调用函数store->omap_get_header获取header，迭代获取对象的omap的key-value值。计算header和kev-value的digest信息，并设置在o.omap_digest中，标记o.omap_digest_present的值为true。

综上可知，通过函数be_scan_list来获取对象的元数据信息，通过be_deep_scrub函数获取对象的数据和omap的digest信息保存在ScrubMap结构中。

# 12.4.4 从副本处理

当从副本接收到主副本发送来的MOSDRepScrub类型消息，用于获取对象的校验信息时，就调用函数replica_scrub来完成。

函数 replica_scrub 具体实现如下：

1）首先确保scrubber.active_rep_scrub不为空。

2）检查如果msg->map_epoch的值小于info-history同样的 interval_since的值就直接返回。在这里从副本直接丢弃掉过时的MOSDRepScrub请求。

3）如果last_update_applied的值小于msg->scrub_to的值，也就是从副本上完成日志应用的操作落后主副本scrub操作的版本，必须等待它们一致。把当前的op操作保存在scrubber.active_rep_scrub中等待。

4）如果 active pushes 大于 0，表明有 Recovery 操作正在进行，同样把当前的 op 操作保存在 scrubber.active_rep_scrub 中等待。

5）否则就调用函数 build_scrub_map_chunk 来构建 ScrubMap，并发送给主副本。

当等待的本地操作应用完成之后，在函数ReplicatedPG::op_applied检查，如果scrubber.active_rep_scrub不为空，并且该操作的版本等于msg->scrub_to，就会把保存的op操作重新放入osd->op_wq请求队列，继续完成该请求。

# 12.4.5 副本对比

当对象的主副本和从副本都完成了校验信息的构建，并保存在相应的结构 ScrubMap 中，下一步就是对比各个副本的校验信息来完成一致性检查。首先通过对象自身的信息来选出一个权威的对象，然后用权威对象和其他对象做比较来检验。下面介绍用于比较的函数。

# 1. scrub(compare_maps

函数 scrubCompare_maps 实现比较不同的副本信息是否一致，处理过程如下：

（1）首先确保 acting.size() 大于 1，如果该 PG 只有一个 OSD，则无法比较。

2）把actingbackfill对应OSD的ScrubMap放置到maps中。

3）调用函数 beCompare_scrubmaps 来比较各个副本的对象，并把对象完整的副本所在 shard 保存在 authoritative 中。

4）调用_scrub 函数继续比较 snap 之间对象的一致性。

# 2. be comparise_scrubmaps

函数beCompare_scrubmaps用来比较对象各个副本的一致性，其具体处理过程分析如下：

1）首先构建master set，也就是所有副本OSD上对象的并集。

（2）对master set中的每一个对象进行如下操作：

a）调用函数be_select_auth_object选择出一个具有权威对象的副本auth，如果没有选择出权威对象，变量shallow Errors加一来记录这种错误。

b）调用函数be comparc_scrub_objects，比较各个shard上的对象和权威对象：分别对data的digest、omap的omap_digest和属性attrs进行对比：

- 如果结果为 clean，表明该对象和权威对象的各项比较完全一致，就把该 shard 添加到 auth_list 列表中。

- 如果结果不为 clean，就把该对象添加到 cur_inconsistent 列表中，分别统计 shallowErrors 和 deep Errors 的值。

- 如果该对象在该 shard 上不存在，添加到 cur MISSING 列表中，统计 shallow errors 值。

c）检查该对象所有的比较结果：如果cur MISSING不为空，就添加到missing队列列；如果有cur_inconsistent对象，就添加到inconsistent对象里；如果该对象有不完整的副本，就把没有问题的记录放在authoritative中。

d）如果权威对象object_info里记录的data的digest和omap的omap_digest和实际扫描出数据计算的结果不一致，update的模式就设置为FORCE，强制修复。如果object_info里没有data的digest和omap的digest，修复的模式update设置为MAYBE。

e）最后检查，如果update的模式为FORCE，或者该对象存在的时间age大于配置参数g_conf->osd_deep_scrub_update_digest_min_age的值，就加入missing_digest列表中。

# 3. be_select_auth_object

函数be_select_auth_object用于在各个OSD上的副本对象中，选择出一个权威的对象：auth_obj对象。其原理是根据自身所带的冗余信息来验证自己是否完整，具体流程如下：

1）首先确认该对象的 read_error 和 stat_error 没有设置，也就在获取对象的数据和元数据的过程中没有出错，否则就直接跳过。

2）确认获取的属性OI_ATTRIBUTE值不为空，并将数据结构object_info_t正确解码，就设置当前的对象为auth_obj对象。

3）验证保存在 objec_info_t 中的 size 值和扫描获取对象的 size 值是否一致，如果不一致，就继续查找一个更好的 auth_obj 对象。

4）如果是replicated类型的PG，验证在object_info_t里保存的data和omap的digest值是否和扫描过程计算出的值一致。如果不一致，就继续查找一个更好的auth_obj对象。

5）如上述都一致，直接终止循环，当前已经找到满意的auth_obj对象。

由上述的选择过程可知，选中一个权威对象的条件如下：

□步骤1)，2）两点条件是基础，对象的数据和属性能正确读取。

（步骤3），4）是利用了objectinfo t中保存的对象size，以及omap和data的digest。

的冗余信息。用这些信息和对象扫描读取的数据计算出的信息比较来验证。

# 4. _scrub

函数_scrub 检查对象和快照对象之间的一致性：

```cpp
void ReplicatedPG::_scrub(ScrubMap &scrubmap, const map<hoject_t, pair<uint32_t, uint32_t> &missing_digest) 
```

如果pool有cachepool层，那么就允许拷贝对象有不一致的状态，因为有些对象可能还存在于cache pool中没有被刷回。通过函数pool.info.allow_incomplete_clones()来确定上述情况。

其实代码比较复杂，下面通过举例来说明其实现基本过程。

例12-1 _scrub函数实现过程举例，如表12-1所示。


表 12-1 _scrub 函数实现过程


<table><tr><td>步骤</td><td>对象列表</td><td>希望的对象</td></tr><tr><td>1</td><td>obj1 snap1</td><td>head/snapdir, unexpected obj1 snap 1</td></tr><tr><td>2</td><td>obj2 head</td><td>head/snapdir, head ok[snAPSHOT clones 6 4 2 1]</td></tr><tr><td>3</td><td>obj2 snap7</td><td>obj2 snap 6, unexpected obj2 snap 7</td></tr><tr><td>4</td><td>obj2 snap6</td><td>obj2 snap 6, match</td></tr><tr><td>5</td><td>obj2 snap4</td><td>obj2 snap 4, match</td></tr><tr><td>6</td><td>obj3 head</td><td>obj2 snap 2 (expected), obj2 snap 1 (expected),head ok[snAPSHOT clones 3 1]</td></tr><tr><td>7</td><td>obj3 snap3</td><td>obj3 snap 3 match</td></tr><tr><td>8</td><td>obj3 snap1</td><td>obj3 snap 1 match</td></tr><tr><td>9</td><td>obj4 snapdir</td><td>head/snapdir, snapdir ok[snAPSHOT clones 4]</td></tr><tr><td>10</td><td>EOL</td><td>obj4 snap 4, (expected)</td></tr></table>

_scurb 实现过程说明如下：

1）对象 obj1 snap1 为快照对象，就应该有 head 或者 snapdir 对象。但是该快照对象没有对应的 head 或者 snapdir 对象，那么该对象被标记为 unexpected 对象。

2）对象obj2head是一个head对象，这是预期的对象。通过head对象，获取snapshot

的 clones 列表为 [6,4,2,1]。

3）检查对象 obj2 snap7 并没有在对象 obj2 的 snapshot 的 clones 列表中，为异常对象。

4）对象obj2的快照对象snap6，在snapshot的clones列表中。

5）对象obj2的快照对象snap4，在snapshot的clones列表中。

6）当遇到obj3head对象时，预期的对象obj2中应该还有快照对象snap2和snap1为缺失的对象。继续获取snap3的snapshot的clones值为列表[3,1]。

7）对象 obj3 的快照对象 snap3 和预期对象一致

8）对象obj3的快照对象snap1和预期对象一致。

9）对象obj4的snapdir对象，符合预期。获取该对象的snapshot的clones值为列表[4]。

10）扫描的对象列表结束了，但是预期对象为obj4的快照对象snap4，对象obj4snap4缺失。

目前对于excepted的对象和unexcepted的对象，都只是在日志中标记出来，并不做进一步的处理。

最后对于那些其他都正确但对象的 digest 不正确的数据对象，也就是 missing_digest 中需要更新 digest 的对象，发送更新 digest 请求。

# 12.4.6 结束 Scrub 过程

scrub_finish函数用于结束Scrub过程，其处理过程如下：

1）设置了相关PG的状态和统计信息。

2）调用函数scrub_process_inconsistent用于修复scrubber里标记的missing和inconsistent对象，其最终调用repair_object函数。它只是在peer MISSING中标记对象缺失。

3）最后触发DoRecovery事件发送给PG的状态机，发起实际的对象修复操作。

# 12.5 本章小结

本章介绍了 Scrub 的基本原理，及 Scrub 过程的调度机制，然后介绍了校验信息的构建和比较验证的具体流程。

最后，总结一下Ceph一致性检查Scrub功能实现的关键点：

□如果做 Scrub 操作检查的对象范围 start 和 end 之间有对象正在进行写操作，就退出本次 Scrub 进程；如果已经开始启动了 Scrub，那么在 start 和 end 之间的对象写操作需要等待 Scrub 操作的完成。

□检查就是对比主从副本的元数据（size、attrs、omap）和数据是否一致。权威对象的选择是根据对象自己保持的信息（object info）和读取对象的信息是否一致来选举出。然后用权威对象对比其他对象副本来检查。

# 第13章

# Ceph 自动分层存储

本章介绍 Ceph 的 Cache Tier 模块，它实现了的自动分层存储技术。本章首先介绍自动分层存储的概念和原理，其次介绍 Ceph 的自动分层存储 Cache Tier 的读写模式，最后对 Cache Tier 的源代码进行详细的分析。

# 13.1 自动分层存储技术

分层存储在“信息生命周期管理”的基础上对数据进一步分层。这个概念的提出基于这样一个事实：在数据的不同生命周期里，其访问的频度是截然不同的，即使是处于同一生命周期的不同类型的数据，访问频率也会不同。

自动分层存储技术目标在于：把用户访问频率高的数据放置在高性能、小容量的存储介质中，把大量低频访问的数据放置在大容量、性能相对较低的存储介质中。它为用户提供的价值在于：在提高热点数据存储性能的同时，降低了存储成本。首先，数据可以自由安全地迁移到更低层的存储介质中，这样可以节约存储成本；其次热点数据可以自动的从低层迁移到高层存储层，提高访问热点数据的性能。

自动分层存储技术实现的核心点在于：

□数据访问行为的追踪、统计与分析：持续追踪与统计每个数据块的存取频率，并通过定期分析，识别出存取频率高的“热”区块，与存取频率低的“冷”区块。

□数据迁移：以存取频率为基础，定期执行数据迁移，将热点数据块迁移到高速存储层，把较不活跃的冷数据块迁移到低速存储层。

传统的磁盘阵列存储产品的自动分层存储技术中，数据访问行为统计和分析的粒度是以数据块为单位的。在Ceph中，数据访问行为统计、分析以及数据在各存储层之间迁移的粒度是以对象（默认为4MB）为基本单位的。

# 13.2 Ceph分层存储架构和原理

在 Ceph 里，Cache Tier 模块在 pool 层设置。可以设置一个 pool 为另一个 pool 的 cache 层。在这里，做缓存层的 pool 称为 cache pool（或者 cache tier、hot pool 等）；相对低性能的存储层称为 data pool（或者 base pool、cold pool 等）。

图13-1是CacheTier的存储架构图。在client层的librados对于CacheTier是透明的，它对CacheTier是无感知的。Objecter实现了CacheTier相关的逻辑处理。CacheTier层为高速存储层，热点数据都保存在CacheTier层，DataTier层为慢速设备存储层，所有数据最终都会保存在DataTier层。

![image](https://cdn-mineru.openxlab.org.cn/result/2026-04-30/bbb6afcc-af41-45b5-9598-d269ca9c27e9/cd0f97c4f3bdccbf5abd24ee0e8e55cb57c93292f370356f7297142034a27c24.jpg)



图13-1 CacheTier存储架构图


# 13.3 CacheTier的模式

在结构cache_mode_t的枚举类型里，定义cache_pool有几种模式：

write back 

read forward 

read proxy 

write proxy 

read only 

下面将详细介绍各种模式的不同应用场景。

# 1. write back 模式

在write back模式下，读写请求都直接发给cache pool，这种模式适合于大量修改数据的应用场景（例如图片视频编辑、事务处理OLTP类应用）。

在write back模式下，读操作时对象在缓存层不存在（cache miss）时，有read forward和read proxy两种特殊模式进行处理。写操作在缓存不命中的情况下有writeproxy的特殊模式进行处理。

# 2. read forward 模式

当进行 read 操作时，对象不在 cache pool 中，就直接返回，client 直接向 data pool 发送请求，数据直接返回给 client。

如图13-2所示，当客户端向cache pool发起读请求时，出现cache miss状态，就返回redirect信息给client,client根据返回的redirect信息，再次直接向base pool层发起读请求，并返回需要的数据。

![image](https://cdn-mineru.openxlab.org.cn/result/2026-04-30/bbb6afcc-af41-45b5-9598-d269ca9c27e9/c83506947a3d2fc52532257e0821b4a233b2047a9fdc7b0d6efdc2cf3db0a62f.jpg)



图13-2 readforward模式


# 3. Read proxy 模式

当进行 read 操作时，对象不在 cache pool 中，cache pool 层向 data pool 发送请求，返

回给cachepool层，cachepool层再返回给client数据。

如图13-3所示，当client向cachepool发起读请求，该对象在cachepool处于cache miss状态，cachepool层向base pool层发起读请求，返回数据给cache pool层，cache pool层再将数据返回给client。

![image](https://cdn-mineru.openxlab.org.cn/result/2026-04-30/bbb6afcc-af41-45b5-9598-d269ca9c27e9/30f5e79568b096e674befa88ed6e8d546a1d2d0a1bf0e50653ac54f51de200c5.jpg)



图13-3 read proxy模式


# 4. write proxy 模式

图13-4是write proxy模式示意图，当客户端先cache pool发起写操作时，处于cache miss的状态，cache pool并不等该对象从base pool中加载到cache pool中，然后写入，而是直接发请求给base pool，直接把数据写入base pool层。

![image](https://cdn-mineru.openxlab.org.cn/result/2026-04-30/bbb6afcc-af41-45b5-9598-d269ca9c27e9/b443d951a889c004ed4408723ddd032379241fb273c859c82f7d452bef0146aa.jpg)



图13-4 write proxy 模式


write proxy 模式中，当 cache pool 处于 cache miss 状态时，直接写入 base pool，降低了写操作的延迟，提高了写操作的性能。

# 5. read only 模式

read only 模式也称为: write-around 或者 read cache。读请求直接发送给 cache pool, 写请求并不经过 cache pool, 而是直接发送给 base pool。

这种方式下，cache pool 设置为单副本，极大地减少缓存空间的占用库。当 cache pool 层失效时，也不会有数据丢失。这种模式比较适合数据一次写入多次读取的应用场景。例如图片、视频、音频等的读写操作。

# 13.4 CacheTier的源码分析

CacheTier的代码分布在Ceph源代码的各个模块，其核心在对象的数据读写路径上。下面就着重分析相关的数据结构，并介绍其如何影响对象读写路径的。

# 13.4.1 pool中的CacheTier数据结构

数据结构pg_pool_t里有CacheTier相关数据字段，如下所示：

```txt
set<uint64_t> tiers;  
int64_t tier_of; 
```

这两个字段用来设置pool的属性：

□如果当前pool是一个cachepool，那么tierof记录了该cachepool的basepool层。

□如果当前pool是basepool，那么 tiers就记录该basepool的cachepool层，一个basepool可以设置多个cachepool层。

使用如下命令来设置 CacheTier 的两个存储层之间的关系：

```txt
ceph osd tier add {data_pool} {cache_pool} 
```

pool的信息都由Monitor来持久化存储。该命令行程序发送请求给Monitor，然后由Monitor相关pool设置上述属性值：

```txt
cache_mode_t cache_mode; // cache 的模式
```

cache_mode 设置了 cache 的模式，可以通过下面命令设置：

```txt
ceph osd tier cache-mode {cachepool} {cache-mode}
int64_t readTier;
int64_t writeTier; 
```

read_tier 和 write_tier 分别设置 base pool 的读缓存层和写缓存层。根据 Ceph 不同的 CacheTier 模式来设置。如果是 write_back 模式，那么该 cache pool 既是 read 层，又是 write 层。如果只是 read only 模式，那么设置的 cache pool 只是 read 层，并不是 write 层。

根据模式分别设置 read tier 和 write tier 字段的命令如下：

```txt
ceph osd tier set-overlay{data_pool} {cache_pool}  
uint64_t target_max_bytes;  
uint64_t target_max Objects; 
```

字段 target_max_bytes 设置了 cache_pool 的最大的字节数。target_maxobjects 设置了 cache_pool 的最大对象数量。

```txt
uint32_t cache_target_dirty_ratio_micro;  
uint32_t cache_target_dirty_high_ratio_micro;  
uint32_t cache_target_full_ratio_micro; 
```

上面这三项分别为：

□ 目标脏数据率：当脏数据率达到这个值时，后台 agent 开始 flush 数据

□ 高目标脏数据率：当脏数据率达到这个值时，后台 agent 开始高速 flush 数据

数据满的比率：当数据达到这个比率就认为cache处于满的状态

字段cache_min Flush_age设置了一个对象在cache中被返回base tier的最小时间。字段cache_min_evict_age设置了一个对象在cache中被evict操作的最小操作时间。

hit_set_parameters 是 hitset 相关的参数。字段 hit_set_period 用来设置每过该段时间，系统要重新产生一个新的 hit_set 对象来记录对象的缓存统计信息。hit_set_count 用来记录系统保存最近的多少个 hit_set 记录。use_gmt_hitset 表示 hitset archive 对象的命名规则。

# 13.4.2 HitSet

类 HitSet 用来跟踪和统计对象的访问行为。目前仅记录了对象是否在缓存中。它定义了一个缓存查找的抽象接口。目前的三种实现方式分别为：ExplicitObjectHitSet、ExplicitHashHitSet、BloomHitSet。下面分别介绍

# 1. ExplicitObjectHitSet

类 ExplicitObjectHitSet 用一个基于 hobject 的 set 来记录对象的命中：

```cpp
ceph::unordered_set<hoject_t> hits; 
```

这种方式实现简单，直观。但是缺点也很明显，要在内存中缓存数据结构hobject，其占用内存空间比较大。

可以粗略地统计一下，在假设对象的名字占40个字节，一个hobject大约占100字节那么大：

<table><tr><td>400G的SSD盘(卡)的占用空间</td><td>400G/4M×100=10M</td></tr><tr><td>4T的SSD盘(卡)的占用空间</td><td>4T/4M×100=100M</td></tr></table>

# 2. ExplicitHitHitSet

类 ExplicitHashSet 基于对象 32 位哈希值的 set 来记录的对象命中：

```cpp
ceph::unordered_set<uint32_t> hits; 
```

每个对象占用4字节的内存空间，占用内存空间如下：

<table><tr><td>400G的SSD盘(卡)的占用空间</td><td>400G/4M×4 = 400k</td></tr><tr><td>4T的SSD盘(卡)的占用空间</td><td>4T/4M×4 = 4M</td></tr></table>

这种方式占用空间相对较少。但缺点也比较明显：当知道一个对象的哈希值，去寻找对应的对象时，需要扫描所有的对象，计算对象的哈希值来对比查找。

# 3. BloomHitSet

类BloomHitSet采用压缩了的Bloomfilter的方式来记录对象是否在缓存中。其原理实现在这里就不详细介绍了，它进一步减少了占用的内存空间。

特别需要注意的是：在模式 CACHEDMODE_NON（没有 cache tier）和模式 read only 模式下，是不需要 HitSet 来记录缓存命中的。只有在以下模式 write back、read forward、read proxy 中才需要 HitSet 来记录缓存命中。

# 13.4.3 CacheTier的初始化

CacheTier初始化有两个入口，如下所示：

□ on.active：如果该pool已经设置为cache pool，在该cache pool的所有PG处于activave状态后初始化。

□ on_pool_change：当该pool的所有PG都已经处于active状态后，才设置该pool为cache pool，那么就等待Monitor通知osd map相关信息的变化，在on_pool_change函数里初始化。

1. hit_set_setup 

函数hit_set_setup用来创建并初始化HitSet对象：

```rust
void ReplicatedPG::hit_set_setup() 
```

具体实现如下：

1）首先检查如果PG既不处于active状态，也不处于primary状态，就调用函数hit_set_clear清除hit_set相关的记录，直接返回不做任何设置。

2）如果参数pool.info.hit_set_count为零，或者参数pool.info.hit_set_period为零，或者pool.info.hit_set.params参数为HitSet::TYPE_NONE，就调用hit_set_clear清空hit_set的记录，并调用函数hit_set_remove_all删除所有的HitSet对象。

3）调用函数hit_set_create根据类型创建相应的HitSet类。

4）调用函数hit_set_apply_log来添加从PG日志中获取的新对象记录，并添加到HitSet中。

# 2. agent_setup

函数 agent_setup 完成 agent 相关的初始化工作：

```txt
void ReplicatedPG::agent_setup() 
```

具体实现如下：

1）首先检查各种参数，如果PG处于下列情况之一：

- 不处于 active 状态。

- 非primary。

- 模式是 pg_pool_t::CACHEMODE_NONE。

- pool.info.tier_of没有设置。

- cache pool 不存在。

调用函数 agent_clear，停止 agent 线程，清除相关设置。

2）如果 agent_state 为空，就重新设置 agent_state 类，并设置相关的初始化参数。

3）调用函数 agent_choose_mode 设置 flush_mode 模式和 evict_mode 模式，并把 PG 添加到 agent 相应的工作队列中。

# 13.4.4 读写路径上的CacheTier处理

在OSD的正常读写路径上，如果该pool有CacheTier设置，处理逻辑就发生了变化。如下所示：

```cpp
void ReplicatedPG::do_op(OpRequestRef& op) {
......
bool in_hit_set = false;
if (hit_set) {
if (obc.get()) {
if (obc->obs.oi.soid != hobject_t() && hit_set->contains(obc->obs.oi.soid))
in_hit_set = true;
} else {
if (missing_oid != hobject_t() && hit_set->contains(missing_oid))
in_hit_set = true;
}
if (!op->hitset_inserted) {
hit_set->insert(oid);
op->hitset_inserted = true;
if (hit_set->is_full() ||
hit_set_start_stamp + pool.info.hit_set_period <= m->get_recv_stamp())) { 
```

```txt
hit_setpersist(); } } } if(agent_state){ if(agent_select_mode(false,op)) return; } if(maybe_handle_cache(op, write_ordered, abc, r, missing_oid, false, in_hit_set)) return; 
```

这里增加了相关的处理逻辑：

1）首先根据对象的上下文信息obc获取的情况，分两种情况调用函数hit_set->contains检查该对象是否命中，如果对象没有在hit_set中就调用hit_set->insert把该对象插入hit_set中。

2）如果hit_set已经满了，或者时间达到了pool.info.hit_set_period的值，就持久化保存该hit_set对象到磁盘上，创建一个新的hit_set对象。

3）调用函数 agent_choose_mode 对相应的 PG 设置 flush 模式和 evict 模式。

4）调用函数 maybe_handle_cache 来处理有关 cache 的读写请求。

下面分别介绍相应函数。

# 1. agent_choose_mode

函数 agent_choose_mode 用来计算一个 PG 的 flush_mode 和 evict_mode 的参数值：

```cpp
bool ReplicatedPG::agent_choose_mode(bool restart,OpRequestRef op) 
```

如果该函数返回值为 true，就表明该请求 Op 被重新加入请求队列里（由于 EvictMode 为 Full），其他情况都返回 false 值。

这个函数实现过程如下：

1）如果该 agent_state 处于 agent_state->delaying 状态，就返回 flase 值。

2）如果info.stats.stats_INVALID为true，表明当前的统计信息无效。agent模式的计算依赖这些统计信息，在这种情况下暂时跳过，暂不计算flush_mode和evict_mode的值。

3）计算 divisor 值，也就是 cache_pool 里的 PG 数量。

4）计算unflushable，也就是不能刷回的对象，并去掉HitSet相关的对象。

5）获取base_pool并检查如果base_pool不支持omap，就去掉所有需要omap支持的对象。

6）计算num_userobjects，其值为统计的对象数减去unflushable对象数。

7）计算num_user_bytes，用统计信息的字节数减去unflushable_bytes的字节数。

8）计算脏对象的数目 num_dirty 的值，如果 base_pool 不支持 omap，就去掉带 omap 的对象。

9）计算 dirty_micro 和 full_micro，分别是脏数据的比率和数据满的比率，单位为百万分之一：

a）如果设置了pool.info.target_max_bytes，就按照字节计算。首先计算每个对象的平均大小avg_size：

- dirty_micro = 脏对象数目 × 每个对象的平均大小 / 每个 PG 的平均字节数 × 1000000

- full_micro = 用户对象数目 × 每个对象平均大小 / 每个 PG 的平均字节数 × 1000000

b）如果设置了pool.info.target_maxobjects，就按照对象数目来计算：

- dirty_objects_micro = 脏对象数目 / 每个 PG 的平均对象数目 × 1000000

- full_objects_micro = 用户对象数目 / 每个 PG 的平均对象数目 × 1000000

最终选择两种计算方式中最大的一个为最终结果。

10）获取 flush_target 和 flush_high_target 参数。如果是 restart，或者当前的 flush_mode 为 IDLE，就用 flush_slop 对二者增加，否则就减少（通过 flush_slop 对比率做修正）。

11）根据脏数据的比例，设置flush_mode：

- 如果 dirty_micro 大于 flush_high_target，设置 flush_mode 为 TierAgentState::FLUSH_

MODE_HIGH。 

- 如果 dirty-micro 大于 flush_target，就设置 flush_mode 为 TierAgentState::FLUSH_MODE_LOW。

12）获取evict_target的值，用evict_slop做一点比例的修正。

13）如果 full_micro 大于 1000000，evict_mode 就设置为 EVICT_MODE_FULL 模式，evict_effort 设置为最大 1000000。

14）如果 full_mircor 大于 evict_target，就设置 evict_mode 为 EVICT_MODE(Some 类型。这时候需要计算 evict_effort 的值，evict_effort 为 PG 在 agent 工作队列里的优先级：

- over = full - evict_target 

- span = 1 - evict_target 

- evict_effort = over/ span，转换成单位为百万分之一

- 通过 g_conf->osd_agent quantize_effort 的修正，使得 evict_effort 的级别不会太多。

# 例13-1 举例说明evict_effort

例如：evict_target: $60\%$ 

full_micro: $80\%$ 

over $= 80\% -60\%$ 

$\mathrm{span} = 1 - 60\%$ 

evict_effort $= (80\% -60\%)$ 1-60%) $= 50\%$ 

本例子里为了便于理解，都使用了单位百分之一。

15）设置新计算的flush_mode值，并更新相关的统计信息。

16）设置新计算的 evict_mode 值，如果 evict_mode 是由 EVICT_MODE_FULL 变为其他类型，并且 PG 的状态为 is.active()，就需要把当前的 op 以及因 cache_full 而等待的操作都重新加入请求队列，设置返回值为 true。

17）根据模式，做相应的处理：

a）如果 idle，就调用函数 agent_disable_pg 把该 PG 从 agent_queue 中删除。

b）如果是retart，或者之前是idle，就调用函数agent_enable_pg把该PG重新加入。

agent_queue 处理队列中。

c）如果之前已经在队列中，就调用函数 agent_adjust_pg 来调整 evict_effort，也就是调整在队列中的优先级。

# 2. maybe_handle_cache_detail

函数 maybe_handle_cache 处理有关 cache 的逻辑，调用函数 maybe_handle_cache_detail 来完成，处理流程如下：

1）首先检查 op 的请求标志里如果有 CEPH_OSD_FLAGignite_CACHE 标志就直接返回。如果 pool.info-cache_mode 为 pg_pool_t::CACHEMODE_NON 也直接返回。

2）如果可以获取obc，并且是write_ordered，该对象被阻塞，就返回cache_result_t::NOOP值。

（3）如果该对象确实不存在，就返回cache_result_t::NOOP值。

4）如果可以获得obc，并且对象存在，则认为缓存命中，则直接返回cache_result_t::NOOP值。

5）检查如果操作里设置了op->need.skip_handle_cache()，就返回cache_result_t::NOOP值。

6）如果是 CACHEMODE_WRITEBACK 模式（下列情况都是缓存没有命中，cache pool 中没有需要的对象）：

情况1：当前agent_state的evict_mode为EVICT_MODE_FULL，说明当前的cache已经接近满了：

a）如果是只读操作：

```txt
if (!op->may_write() && !op->may_cache() &&!write_ordered &&!must_promote) 
```

如果可以 can proxy read，就调用函数 do proxy read 读取。否则调用 do_cache_REDIRECT 函数返回给客户端 redirect 信息，客户端再去访问 base pool，这种方式就是 forward 方式读取。

b）如果是write操作，就调用block_write_on_full_cache阻塞当前的操作。

情况2：不是EVICT_MODE_FULL模式：

a）如果hit_set为空：说明在这种模式下不需要hitset（可能使read only模式），调用promote_object函数去base pool读取该对象。

b）如果hit_set不为空，并且op->may_write()||op->may_cache()，也就是写操作：

- 如果允许 can proxy_write，就调用 do proxy_write 来以 proxy 的方式写入，否则调用 promote_object 操作，返回 cache_result_t::BLOCKED_promOTE

- 如果是以 proxy_write 方式写入，检查是否还需要调用函数 promote_object 操作。

c）否则，也就是read操作

- 如果允许 proxy_read，就调用函数 do proxy_read 以代理的方式来读取。

- 检查如果需要 promote，就调用函数 maybe_promote 来检查。

- 如果既没有 promote，又没有 proxy_read，就调用 do_cacheredirect 来读取。

7）如果是 CACHEMODE_FORWARD 模式，就直接调用函数 do_cacheredirect，以 forward 模式读取。

8）如果是 CACHEMODE_READONLY 模式：如果是读操作，就调用 promote_object 函数从 base pool 读取；如果是写操作，就以 forward 的方式写。

9）如果是CACHEMODE_READFORWARD模式

- 如果是 write 操作，evict 的模式为 TierAgentState::EVICT_MODE_FULL，就阻塞，否则就调用函数 promote_object 来读取该对象，然后写入。

- 如果是读，就以forward模式读取。

如果是CACHEMODE_READPROXY模式，对于写，和CACHEMODE_READ-FORWARD处理相同；对于读，就以proxy模式读取。

下面介绍上述处理过程中的，处理代理读、代理写、从数据层加载对象到缓存层的具体处理函数。

函数do proxy read给base pool发送请求消息，来读取cache pool没有的对象数据。函数finish proxy read完成了prox_read的回调函数，其对结果做检查，并清理等待列表。通过函数complete_readctx给client端发送读操作的返回消息。特别注意的是，函数并没有把对象的数据写入cache pool，所以后续还需要调用函数promote_object从base pool读

取该对象数据，然后写入cachepool中。

函数do proxy write直接写数据到base pool中。函数finish proxy write为do proxy write的回调函数，完成给客户端的发送ACK应对。同样，cache pool中并没有该数据对象，还需要后续调用promote_object函数把对象从base pool中读到cache pool中。

函数promote_object完成base pool读取相关的对象并将其写入cache pool中，处理过程如下：

1）首先检查scrubber.writeblockedby_scrub是否处于block状态。

2）构造PromoteCallback回调函数，然后调用函数start_copy拷贝数据。

函数 start copy 用于执行 copy 操作拷贝数据，其处理过程如下：

1）检查如果copy ops里有该对象，就调用函数cancel_copy取消正在进行的copy操作。

（2）构造CopyOp操作，并添加到copy ops的map中。

3）调用obc->start_block()，把该obc设置为blocked为true的状态。

4）调用函数_copy(Some 做相应的操作。

函数 `copy(Some)` 执行 copy 操作做部分数据量的拷贝，它是通过调用 osdc 的读操作来完成，其处理过程如下：

1）根据cop->flags设置各种flags标志。

2）构建C_GatherBuilder，它是一个Context的回调函数。

3）调用函数 cop->cursor.is_init() 判断是否为第一次操作，并且 cop->mirror_snapshot 为ture，就需要调用 list_snapps 获取snapSet 的信息，并把结果保存在 cop->results_snapshot 中，保存tid到cop->objecter_tid2中。

4）如果 cop->results.user_version 被设置了（在第一次操作中设置），在以后的操作中，需要执行 assert_version 操作。

5）如果该pool不支持omap（ec），在copyget_flags标志中，设置CEPH_OSDCOPY_GET_FLAG_NOTSUPP_OMAP标志。

6）调用 op.copy_get 操作，设置相关的操作，并调用函数 op.set_last_op_flags 设置最后一个 op 的 flags 标志。

（7）构造C_Copyfrom回调函数，调用函数gather.set_finisher设置gather的finisher。

函数。

8）调用函数osd->objecter->read把操作发送出去。

9）把tid设置为fin->tid，调用函数gatheractivate()使回调函数激活。

函数process_copy_chunk为Copyfrom的回调函数，完成了数据的部分写入。函数finish_promote为PromoteCallback的回调函数，完成后续promote操作。

# 13.4.5 cache的flush和evict操作

cache pool 空间不够时，需要选择一些脏对象回刷到数据层（即 flush 操作），并将一些 clean 对象从缓存层剔除（即 evict 操作），以释放更多的缓存空间。这两种操作都是在后代完成的。flush 操作和 evict 操作算法的好坏，决定了 Cache Tier 的缓存命令率。

# 1.OSDService中Agent相关的数据结构

在类OSDService中，定义了一个AgentThread的后台线程，用于完成两个任务：一是把dirty对象从cache pool层适时地回刷到base pool层，二是从cache pool层剔除掉一些不经常访问的clean对象。

```cpp
Class OSDService{....   
Mutex agent_lock; //agent线程锁，保护以下所有的数据结构 Cond agent_cond; //线程相应的条件变量   
map<uint64_t, set<PGRef> >agent_queue; //所有回刷或者剔除所需的PG集合，根据PG集合的优先级，保存在不同的map中   
set<PGRef>::iterator agent_queue_pos; //当前在扫描的PG集合的一个位置，只有agent_validIterator为true时，这个指针才有效， 否则从集合的起始处扫描 bool agent_validIterator;   
int agent ops; //所有正在进行的回刷和剔除操作 int flush_mode_high_count; //如果回刷模式为HIGH模式，该该值就增加 set<Hobject_t,hobject_t::BitwiseComparator>agent_oids; //所有正在进行的agent的操作（回刷或剔除）的对象 bool agent.active; //agent是否有效   
//agent线程 struct AgentThread:public Thread{
```

```c
OSDService \*osd; explicit AgentThread(OSDService \*o) : osd(o) {} void \*entry(）{ osd->agent_entry(); return NULL; } agent_thread; bool agent_stop_flag; //agent 停止的标志   
Mutex agent_timer_lock; SafeTimer agent_timer; //agent相关的定时器，该定时器用于：当扫描一个PG对象时，该对象既没有剔除操作，也没有回刷 操作，就停止PG的扫描，把该PG加入到定时器，5s后继续
```

# 2. 数据结构TierAgentState

在ReplicatedPG的内部，变量agent_state用来保存PG相关的agent信息。

```txt
struct TierAgentState {
    hobject_t position; //PG内扫描的对象位置
    int started; //PG里所有对象扫描完成后，所发起的所有agent操作数目。当PG扫描完成所有的对象，如果没有agent操作，就需要延迟一段时间
    hobject_t start; //本次扫描起始位置
    bool delaying; //是否延迟
    //历史的统计信息
    pow2_hist_t temp_hist;
    int hist_age;
    map<time_t, HitSetRef> hit_set_map; //HitSet的历史记录
    //最近处于clean的对象
    list<hoject_t> recent Cleaner;
}
```

# 3. agent_entry

agent_entry是agent_thread的入口函数，它在后台完成flush操作和evict操作，具体处理流程如下：

1）加 agent_lock 锁。该锁保护 OSDService 里所有和 agent 相关的字段。

2）如果 agent_stop_flag 为 true，就将 agent_lock 解锁并退出，否则继续以下操作。

3）扫描agent_queue队列，如果agent_queue为空，就在条件变量agent_cond上等待。

4）从队列agent_queue中取出级别最高的PG的集合top（始终从级别最高的取），如果top集合为空，就把该集合从agent_queue中删除，并使agent_valid.iterator集合内的PG指针设为false，使其无效。

5）检查如果正在进行的 agent 操作数 agentOps，如果该值大于所配置最大允许的 agent 操作的数量 g_conf->osd_agent_maxOps，或者非 agent.active，就等待。

6）如果agent_validIterator有效，就从agent_queue_pos处获取该PG set中的一个PG；否则就从该set的头开始，获取相应的PG指针。

7）获取 agent 操作的最大数目 max 值，以及 agent Flush_quota 的值。

8）调用函数 pg->agent_work，正常情况返回 true 值。如果它的返回值为 false，该 PG 处于 delay 状态，需要加入 agent_timer 定时器，在配置项 g_conf->osd_agent_delay_time 设定的秒数后，调用 agent_select_mode 重新设置模式。

# 4. agent_work

agent_work 完成一个 PG 内的 evict 操和 flush 操作，主要的流程如下：

1）调用函数 lock() 对 PG 加锁，agent_state 数据结构受它保护。

2）检查如果 agent_state 为空，或者 agent_state->is_idle()，就解锁返回 true 值。

3）调用函数 agent_load_hit_sets 加载 hit_set 的历史对象到内存中。

4）调用函数pgfrontend->objects_list_partial来扫描本PG的对象，从agent_state->position开始位置扫描，结果保存在对象向量ls中。最小扫描一个对象，最大扫描数是配置参数osd_pool_default_cache_max_evict_check_size设置的对象个数。

5）对扫描到的ls中的对象，做如下检查：

- 跳过 hitset 对象。

- 跳过 degraded 对象。

- 跳过 missing 对象。

- 跳过 object context 不存在的对象。

- 跳过对象的 obs 不存储。

- 跳过正在进行 Scrub 操作的对象。

- 跳过已经被阻塞的对象。

- 跳过有正在读写请求的对象。

- 如果 base_pool 不支持 omap，就跳过有 omap 的对象。

- 如果 agent_state->evict_mode != TierAgentState::EVICT_MODE_IDLE，调用agentmaybe_evict函数来剔除掉一些对象。

- 如果当前的 flush_mode 不是 IDLE 状态，就调用 agentmaybe Flush 函数来回刷一些对象。

- 如果started大于agent_max，也就是已经发起的agent操作数目大于最大允许的agent_max数目，就停止并退出。设置next的指针，也就是下次开始扫描的对象的起始位置。

6）更新 agent_state->hist_age 的值，如果 agent_state->hist_age 大于 g_conf->osd_agent_hist_halflife，就重置为 0，并调用函数 agent_state->temp_hist.decay() 把直方图的统计的对象减半。

7）比较扫描的指针，如果PG的对象扫描了一圈后，total_started值为0，也就是没有agent操作（flush操作或者evict操作），就设置need_delay值为true，需要延迟一段时间。

8）调用函数hit_set_in_memory_trim在内存中对一些旧的hitset对象做trim操作。

9）如果需要延迟，就调用函数 agent_delay 把该 PG 从 agent_queue 中删除。

10）调用agent_choose_mode重新计算agent的evict和flush的模式值。

# 5. agentmaybe_evict

函数 agentmaybe_evict 完成一个对象的 evict 操作，处理过程分析如下：

1）首先检查对象的状态：

- 如果不是 after flush，就需要确保对象处于 clean 状态，否则直接返回 false 值。

- 如果该对象还有watcher，说明还有客户端在使用该对象，返回false值。

- 对象如果处于 blocked 状态，返回 false 值。

- 如果对象处于 cache pinned 状态，返回 false 值。

- 验证对象的 clone 信息是否一致，如果不一致，返回 false 值。

2）如果evict_mode为EVICT_MODE(Some模式：

- 需要确保该对象处于 clean 的时间大于 pool.info-cache_min_evict_age 的长度。

- 调用函数 agent Estimate_temp 计算该对象的热度值 temp。

- 调用函数 get_position_micro 计算该对象的 temp 值在对象的统计直方图的位置值 emp_upper。

- 如果1000000减去temp_upper的值大于等于agent_state->evict_effort，该对象就不删除，直接返回false值。

3）如果 evict_mode 为 EVICT_MODE_FULL 模式，就调用函数 delete_oid 删除该对象，最后调用函数 simple_opc Submit 发起实践的删除请求。

6. agentmaybe Flush 

函数 agentmaybe Flush 完成一个对象的 flush 操作，处理过程如下：

1）检查如果对象不脏，就返回false值。

2）检查如果该对象处于cache_pinned，就返回false值。

3）如果当前 evict_mode 不是 evict_mode_full 状态，就检查该对象处于 dirty 的时间是否超过 pool.info-cache_min Flush_age 值。

4）检查对象是否处于activate状态。

（5）调用函数start Flush来完成对象的回刷操作。

7. start Flush 

函数 start Flush 完成实际的 flush 操作，具体处理过程如下：

1）首先调用obc->ssc->snapshot.get_filtered把已经删除的snap对象过滤掉。

（2）检查比当前clone对象版本更早的克隆对象：

- 如果还有版本更早的克隆对象处于 missing 状态，就返回 -ENOENT 错误码。

- 如果还有版本更早的克隆对象，获取该对象的 older_abc 值，并且标记该对象处于 dirty 状态，直接返回 EBUSY；如果该对象的 abc 不存在，就默认为 clean 状态。

例如：clones[1,4,5,8]，当前snap为5，就必须确保[1,4]处于clean状态。

3）如果blocking设置为true，就设置对象处于blocked状态。

4）检查如果该对象在 flushOps 中，也就是该对象已经在 flush 中，根据不同的情况分别处理。

（5）最后处理克隆对象的flush操作。

# 13.5 本章小结

本章介绍了Ceph的CacheTier的原理和架构，及其各种模式，以及因引入CacheTier而引起Ceph的读写路径上不同的处理。最后介绍了CacehTier后台的flush操作和evict操作的处理过程。

Ceph的CacheTier功能目前在对象的访问频率和热点统计上的实现都比较简单，有待后续实现更好的基于自学习的Cache算法。

# 推荐阅读

![image](https://cdn-mineru.openxlab.org.cn/result/2026-04-30/bbb6afcc-af41-45b5-9598-d269ca9c27e9/9cdbcbfa5f130d3bb121ca96bf3130a40aa2fb1e7371eb7779961d06b11bd7a6.jpg)


![image](https://cdn-mineru.openxlab.org.cn/result/2026-04-30/bbb6afcc-af41-45b5-9598-d269ca9c27e9/40ee7624aa90f1a97b377d652d481d963479f2c7f4fed2e2dc5bdf6f91ce1d6d.jpg)


# 深入理解大数据：大数据处理与编程实践

作者：黄宜华等ISBN:978-7-111-47325-1定价：79.00元

本书在总结多年来MapReduce并行处理技术课程教学经验和成果的基础上，与业界著名企业Intel公司的大数据技术和产品开发团队和资深工程师联合，以学术界的教学成果与业界高水平系统研发经验完美结合，在理论联系实际的基础上，在基础理论原理、实际算法设计方法以及业界深度技术三个层面上，精心组织材料编写而成。

作为国内第一本经过多年课堂教学实践总结而成的大数据并行处理和编程技术书籍，本书全面地介绍了大数据处理相关的基本概念和原理，着重讲述了Hadoop MapReduce大数据处理系统的组成结构、工作原理和编程模型，分析了基于MapReduce的各种大数据并行处理算法和程序设计的思想方法。适合高等院校作为MapReduce大数据并行处理技术课程的教材，同时也很适合作为大数据处理应用开发和编程专业技术人员的参考手册。

——中国工程院院士、中国计算机学会大数据专家委员会主任 李国杰

# 软件定义数据中心——技术与实践

作者：陈熹 孙宇熙 ISBN: 978-7-111-48317-5 定价：69.00元

国内首部系统介绍软件定义数据中心的专业书籍。

众多业界专家倾力奉献，揭秘如何实现软件定义数据中心。

理论与企业案例完美融合，呈现云计算时代的数据中心最佳解决方案。

有了以软件定义数据中心为基础的混合云，企业就可以进退有度，游刃有余，加上成功管理新的移动终端技术，可轻松进入“云移动”时代！这也是为什么软件定义数据中心最近获得大家注意的根本原因。EMC中国研究院编著的这本《软件定义数据中心：技术与实践》恰逢其时，它会给读者详细解说怎么实现软件定义数据中心。

VMware高级副总裁，EMC中国卓越研发集团创始人Charles Fan

# 推荐阅读

![image](https://cdn-mineru.openxlab.org.cn/result/2026-04-30/bbb6afcc-af41-45b5-9598-d269ca9c27e9/b16d70c052d54ee9d0c2730227e0003b5ad528c363de666e583247d8fe70f00d.jpg)


![image](https://cdn-mineru.openxlab.org.cn/result/2026-04-30/bbb6afcc-af41-45b5-9598-d269ca9c27e9/26bb213acecf85c7a6717848806ba9d66c78cc3b3831b6bc4452b008d4710a83.jpg)


# Spark大数据处理：技术、应用与性能优化

作者：高彦杰 ISBN:978-7-111-48386-1 定价：59.00元

# Hadoop核心技术

作者：翟周伟 ISBN:978-7-111-49468-3 定价：69.00元

![image](https://cdn-mineru.openxlab.org.cn/result/2026-04-30/bbb6afcc-af41-45b5-9598-d269ca9c27e9/5eabe6c6dd9268929424b2d20bed92775873d37c13fcfa35045c0a75480c72e2.jpg)


![image](https://cdn-mineru.openxlab.org.cn/result/2026-04-30/bbb6afcc-af41-45b5-9598-d269ca9c27e9/ceaa7ba209bbfdcf396ad5480368eb8575621efea449c64630bf592b711d410f.jpg)


# 大规模分布式系统架构与设计实战

作者：彭渊 ISBN:978-7-111-45503-5 定价：59.00元

# 大规模分布式存储系统：原理解析与架构实战

作者：杨传辉 ISBN:978-7-111-43052-0 定价：59.00元

# 推荐阅读

![image](https://cdn-mineru.openxlab.org.cn/result/2026-04-30/bbb6afcc-af41-45b5-9598-d269ca9c27e9/4994c111cc027c48c1ceb3f5517771385e716e2c78b2714d8a4dfa9e47cb4f97.jpg)


![image](https://cdn-mineru.openxlab.org.cn/result/2026-04-30/bbb6afcc-af41-45b5-9598-d269ca9c27e9/3eb1bba62b6781f372705cc6b678a3be2f1c1a4befed6272a059bd13bb72df66.jpg)


![image](https://cdn-mineru.openxlab.org.cn/result/2026-04-30/bbb6afcc-af41-45b5-9598-d269ca9c27e9/c3f0c8dab3434e716c8ac4506dae38dd343ff11d3cedd0a67158421471ee85e9.jpg)


![image](https://cdn-mineru.openxlab.org.cn/result/2026-04-30/bbb6afcc-af41-45b5-9598-d269ca9c27e9/ee9a3065ebda988836b9111744b7102285ae417833627e969b2f7355c88173fb.jpg)


![image](https://cdn-mineru.openxlab.org.cn/result/2026-04-30/bbb6afcc-af41-45b5-9598-d269ca9c27e9/5bb5ead4bf4d12414b2402b1544ccdfd6ca49a4b4e214295f64559e051318b39.jpg)


![image](https://cdn-mineru.openxlab.org.cn/result/2026-04-30/bbb6afcc-af41-45b5-9598-d269ca9c27e9/e482f267da78570d7634f9064297aa327198a7e59f6520fa5efdf23391c25d7e.jpg)


这本书是目前我所看到的从代码角度解读Ceph的最好作品，即使在全球范围内，都没有类似的书籍能够与之媲美。信每个Ceph爱好者都能从这本书中找到自己心中某些疑问的解答途径。

——王豪迈 XSKY公司C

看到常涛写的这本书很是欣慰，很高兴国内有了一本专业书供相关人员去接触和学习Ceph代码，这有助于增加国内发者在Ceph社区的代码提交量。另外Ceph中国社区也将计划组织Ceph Hackathon活动，以此来促进Ceph代码的性循环，取之于Ceph社区，回馈于Ceph社区。我们要让全世界开发者看到中国人不仅仅在OpenStack中贡献大，Ceph社区中的贡献同样显著。

— 耿航 Ceph中国社区联合创始人，XSKY公司市场技术专

# 本书主要内容包括

Ceph数据读写功能关键流程的实现。

Ceph如何做到数据强一致性，并可大规模部署。

Ceph数据分布式算法CRUSH如何实现只通过计算就可定位数据块。

Ceph如何实现存储的高级功能：如快照、克隆、纠删码、Scrub、CacheTier等。

Ceph数据修复的实现过程，包括Recovery和Backfill两个过程的具体实现。

Ceph如何实现集群伸缩时的数据迁移和数据均衡。

![image](https://cdn-mineru.openxlab.org.cn/result/2026-04-30/bbb6afcc-af41-45b5-9598-d269ca9c27e9/b65dd565bac9f17ba80e3f634e61876c0ce170f3ade906282e0aa9f8dde6481a.jpg)
