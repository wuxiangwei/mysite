[toc]

# Recovery

## Primary、Replica分别进入Active状态

```
1. Active::Active() --> PG::activate()  // 发送MOSDPGLog消息给Replica节点
2. Stray::react(MLogRec) --> Stray::post_event(Activate) --> 
   ReplicaActive --> ReplicaActive::react(Activate) --> PG::activate()
```
**Primary进入Active状态、Replica进入ReplicaActive状态**。对Primary，Peering状态的GetMissing子状态接收到所有actingbackfill OSD的pg log后，投递Activate事件将状态机切换到**Active**状态。Primary进入Active状态后，向Replica发送*MOSDPGLog*请求，使Replica从Stray状态切换到**ReplicaActive**状态。

```C++
boost::statechart::result PG::RecoveryState::Stray::react(const MLogRec& logevt) {
    post_event(Activate(logevt.msg->info.last_epoch_started));
    return transit<ReplicaActive>();
}
```
注意boost状态机的一个特性：react中post_event的事件，只是将event放入事件主流程，react结束后执行。如此，Activate事件将被送入ReplicaActive::react。

```
PG
    |-- peer_activated: set<pg_shard_t>  // 已Active的Peer
    |-- blocked_by: set<int>  // 

// Replica通知Priamry本节点已Active
1. PG::activate() --> C_PG_ActivateCommitted --> PG::_activate_committed()
// Primary接收到消息，检查是否所有Replica均已Active
// 若全Active，添加AllReplicasActivated到Peer队列，由Peer工作线程开始恢复相关任务
2. Active::react(MInfoRec) --> PG::all_activated_and_committed() --> PG::queue_peering_event(AllReplicasActivated)
// 所有Replica都已Active，开始恢复。。。
3. Active::react(AllReplicasActivated) --> ReplicatedPG::on_activate()
```
**Priamry等待所有Replica进入ReplicaActive状态，启动恢复流程**。Primary使用peer_activated和blocked_by标记已经active的Peer。对Replica，_activate_committed通知Priamry本节点已Active。


```
1. PG::needs_recovery() // 检查是否增量恢复
2. ReplicatedPG::on_activate() --> PG::queue_peering_event(DoRecovery) --> WaitLocalRecoveryReserved
```
Primary检查是否要恢复。
1. 自己的missing列表非空，恢复；
2. actingbackfill中Peer的missing列表非空，恢复。
若要恢复，向Peer队列添加*DoRecovery*事件，将状态机切换到WaitLocalRecoveryReserved状态。

```
1. PG::needs_backfill() // 检查是否全量恢复
2. ReplicatedPG::on_activate() --> PG::queue_peering_event(RequestBackfill) --> WaitLocalBackfillReserved
```
Primary检查是否要backfill。检查每个backfill_targets中pg info，若last_backfill，全量恢复。

## DoRecovery

### 预留 Reserve

```
OSDService
    |-- remote_reserver: AsyncReserver<spg_t>
    |-- local_reserver: AsyncReserver<spg_t>
        |-- queues: map<unsigned, list<pair<T, Context*> > >  // Key为pg恢复的优先级，Value为List，List元素为<pgid, QueuePeeringEvt>
        |-- queue_pointers: map<T, pair<unsigned, typename list<pair<T, Context*> >::iterator > >  // Key为pgid，Value为queues[prio]
        |-- in_progress: set<T>  // 正在处理的请求
        |-- f: Finisher  // 调用queues中Context的队列
        |-- max_allowed: unsigned
        |-- min_priority: unsigned

1. WaitLocalRecoveryReserved::WaitLocalRecoveryReserved() --> AsyncReserver::request_reservation() -->
AsyncReserver::do_queues()  // 根据优先级从大到小出队列
2. QueuePeeringEvt::finish() --> PG::queue_peering_event(LocalRecoveryReserved)  // 由AsyncReserver的Finish线程调用
```
AsyncReserver从两个方面来预留，一方面是优先级，只有优先级高于min_priority的预留才会被处理，优先级越高的预留被处理地越早；另一方面是最大预留数目，默认为1，超过最大值后后续的预留将停留在队列内，等到预留数目下降后才会被处理。预留内容可以是Backfill或者Recovery，以Recover为例，如果OSD自己为Primary，使用local_reserver来预留，否则OSD自己为Replica，使用remote_reserver来预留。默认情况下，一个OSD最多允许2个PG执行Recovery，一个PG为Primary，一个PG为Replica。如果两个PG均为Primary或均为Replica，则无法同时Recovery。


| 配置项 | 默认值 | 说明 |
|:--|:--|:--|
| osd_recovery_priority | 5 | Recovery预留在AsyncReserver队列中的优先级 |
| osd_scrub_priority    | 5 | |
| osd_snap_trim_priority| 5 | |
| osd_client_op_priority| 63 | |
| osd_min_recovery_priority | osd_min_recovery_priority |AsyncReserver的最低优先级 |
| osd_max_backfills | 1 | 最大预留数 |


![rc_reserve.png](rc_reserve.png)

```
1 --> 2 --> 3 --> 4 --> 5 --> 6 -->
A: 3 --> 预留下个Replica
B: 7 --> 所有Replica都预留成功
```
Primary一个接一个地向Replica预留，只有前个Replica预留成功后才开始预留下个。Primary和Replica间的消息走Dispatch流程。

### 恢复 Recoverying

```
PG
    |-- recovery_queued: bool  // 入队列时设置标记，出队列时清除标记

OSDService
    |-- awaiting_throttle: list<pair<epoch_t, PGRef> >
    |-- recovery_ops_reserved: uint64_t  // 预留的ops，进ShardOp队列的recovery请求数
    |-- recovery_ops_active: uint64_t
    |-- defer_recovery_until: utime_t  //  允许Recovery启动的时间
    |-- recovery_paused: bool  // 暂停Recovery，通过OSDMap来设置

Recovering::Recovering() --> PG::queue_recovery() --> OSDService::_maybe_queue_recovery() --> OSDService::_queue_for_recovery() --> ShardedWQ::queue(PGRecovery)
```
**入ShardOp队列**。

| 配置项 | 默认值 | 说明 |
|:--|:--|:--|
| osd_recovery_max_active | 3 | |
| osd_recovery_max_single_start | 1 | 一次Recovery的Object数量的最大值 |
| osd_recovery_delay_start      | 0 | 推迟Recovery开始时间 |


```
ReplicatedPG(PG)
    |-- recovery_ops_active: int  // 递增
    |-- recovering: map<hobject_t, ObjectContextRef, hobject_t::BitwiseComparator>  // 正在Recovery的Object
    |-- pushing: map<hobject_t, map<pg_shard_t, PushInfo>, hobject_t::BitwiseComparator> //  prep_push时设置

OSDService
    |-- recovery_ops_active: uint64_t  //  递增

RPGHandle
    |-- pushes: map<pg_shard_t, vector<PushOp> >  // Key为Relica OSD
        |-- recovery_info: ObjectRecoveryInfo

PGQueueable::RunVis::operator() --> OSD::do_recovery() --> ReplicatedPG::start_recovery_ops() --> 
ReplicatedPG::recover_replicas() -->
1. ReplicatedPG::prep_object_replica_pushes() --> ReplicatedBackend::recover_object() --> 
   ReplicatedBackend::start_pushes() --> ReplicatedBackend::prep_push_to_replica() --> 
   ReplicatedBackend::prep_push()
2. ReplicatedBackend::run_recovery_op() --> ReplicatedBackend::send_pushes() // 发送MOSDPGPush消息给Replica
```
**出ShardOp队列，恢复Replica**。


| 配置项 | 默认值 | 说明 |
|:--|:--|:--|
| osd_recovery_sleep | 0 | 出队列后先Sleep一段时间，拉长两个Recovery的时间间隔 |
| osd_recovery_op_priority | 3 | Recovery Op的优先级 |
| osd_max_push_cost | 8^20 | MOSDPGPush消息的大小 |
| osd_max_push_objects | 10 | MOSDPGPush消息允许的Object数量 |


```
// Replica处理MOSDPGPush消息
1. OSD::dispatch_op_fast() --> OSD::handle_replica_op() --> OSD::enqueue_op() -->
   PG::queue_op() --> ShardedWQ::queue()
2. ReplicatedPG::do_request() --> ReplicatedBackend::handle_message() -->
   ReplicatedBackend::do_push() --> ReplicatedBackend::_do_push()

// Primary处理MOSDPGPushReply消息
3. ReplicatedPG::do_request() --> ReplicatedBackend::handle_message() --> 
   ReplicatedBackend::do_push_reply() --> ReplicatedBackend::handle_push_reply() -->
   ReplicatedPG::on_global_recover() --> PG::finish_recovery_op() -->
   OSDService::finish_recovery_op() --> OSDService::_maybe_queue_recovery()
```
一个Recovery Object的所有Replica都恢复后，Primary重新向ShardOp队列投递PGRecovery请求，ShardOp线程开始下个Object的恢复。所有Object都恢复后ShardOp线程使用AllReplicasRecovered事件将状态机从Recovering状态切换到Recovered状态，同时释放Replica的预留。最后，如果所有节点都Active，则从Recovered状态切换到Clean状态。

MOSDPGPush消息在Recplica走Fast dispatch路径，入ShardOp队列。Replica将数据落盘后向Primary回复MOSDPGPushReply消息。

![recovery.png](recovery.png)

```
1 --> 2 --> 3 --> 4 --> 5 --> 6 --> 7 --> 8 -->9 --> 10 --> 11 --> 12 --> 13 --> 14 --> 
1. 1 --> 2 --> // Recovery下个Object
2. 15 --> 16 // 完成
```

## Backfill

### 预留（同DoRecovery）

```
MBackfillReserve
1. WaitLocalBackfillReserved::WaitLocalBackfillReserved() --> AsyncReserver::request_reservation(LocalBackfillReserved)
2. WaitRemoteBackfillReserved
```
同DoRecovery类似，WaitLocalBackfillReserved状态本地预留，本地预留成功后切换到WaitRemoteBackfillReserved状态。WaitRemoteBackfillReserved状态向逐个向backfill节点发送*MBackfillReserve*预留请求。所有backfill节点预留成功后通过*AllBackfillsReserved*事件，将状态机切换到Backfilling状态。

### Backfilling状态

```
Backfilling::Backfilling() --> PG::queue_recovery() --> OSDService::queue_for_recovery()
```
入ShardOp队列，同Recovery。

```
ReplicatedPG(PG)
    |-- new_backfill: bool
    |-- backfill_targets: set<pg_shard_t>
    |-- backfill_info: BackfillInterval
    |-- peer_backfill_info: map<pg_shard_t, BackfillInterval>


RunVis::operator(PGRecovery) --> OSD::do_recovery() --> ReplicatedPG::start_recovery_ops() -->
ReplicatedPG::recover_backfill()
```
出ShardOp队列。



## 基本数据结构

```
pg_shard_t
    |-- osd: int32_t // osd.1 
    |-- shard: shard_id_t  // EC Pool用到，参考PG::init_primary_up_acting函数
```

## 参考资料

1. <http://docs.ceph.com/docs/master/dev/peering/>
