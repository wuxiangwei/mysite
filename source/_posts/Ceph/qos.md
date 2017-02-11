title: "Ceph QoS 初探（下）" 
date: 2017-01-24 20:23:29 
categories: [Ceph]
tags: [Ceph]
---

作者：[吴香伟](www.wuxiangwei.cn) 发表于 2017/01/24
版权声明：可以任意转载，转载时务必以超链接形式标明文章原始出处和作者信息以及版权声明

---------------------- 

存储QoS是个可以做很大也可以做很小的特性。SolidFire认为将QoS归类为特性太儿戏，QoS应该是存储系统设计之初就要仔细考虑的架构问题。的确，分析了一众主流存储大厂后还是觉得它在这方面做得最细致最全面。同时也有些厂商做得比较简陋，只提供了带宽或者IOPS的限速功能。这或许在某些场景中已经够用，但我认为一个完整的QoS方案至少要包括对带宽、IOPS的预留、上限和优先级控制，如果再精细点还可以考虑IO的粒度、延迟、突发、空间局部性、系统内部IO、用户IO、缓存、磁盘等要素。

分布式存储都有很长的IO路径，简单的IOPS限速功能通常在路径的最前端实现。例如OpenStack Cinder默认使用QEMU完成存储块的限速功能，QEMU对存储来说已经属于客户端的角色了。

QoS的本质总结起来就四个字：**消此长彼**，它并不会提高系统整体处理能力，只是负责资源的合理分配。据此就可以提出一连串问题了：首先，如何知道什么时候该消谁什么时候该长谁？其次，该怎么消该怎么长？这两个问题QoS算法可以帮忙解决，可以参考我的另外一篇文章《聊聊dmclock算法》。在这两个问题之前还需要选择一块风水宝地，能够控制希望可以控制的IO，否则即使知道何时控制以及如何控制也鞭长莫及无能为力。风水宝地的选择可以参考我的另外一篇文章《拆开Ceph看线程和队列》。

对Ceph来说，OSD的ShardedOpWq队列是个不错的选择，因为几乎所有重量级的IO都会经过该队列。这些IO可以划分为两大类，一类是客户端过来的IO，包括文件、对象和块存储；另一类是系统内部活动产生的IO，包括副本复制、Scrub、Recovery和SnapTrim等。第一类IO由于涉及到一些敏感内容暂不考虑，本文主要分析第二类IO，这也是本文叫做下篇的原因。

<!--more-->


## Recovery

| 配置项 | 默认值 | 说明 |
|:--|
| osd_recovery_threads | 1 | Recovery线程池中线程的个数<br>**wipe_dmclock2分支已经禁用该线程池** |
| osd_max_backfills | 1 | 同时进行恢复的PG数目的最大值 |
| osd_min_recovery_priority | 0 | 优先级最高为255, 基数为230。可通过命令行配置PG的优先级 |
| osd_recovery_max_active | 3 | |
| osd_recovery_max_single_start | 1 | 一个PGRecovery对应的Object个数 |
| osd_recovery_delay_start | 0 | 推迟Recovery开始时间 |
| osd_recovery_sleep | 0 | 出队列后先Sleep一段时间，拉长两个Recovery的时间间隔|
| osd_recovery_op_priority | 3 | Recovery Op的优先级 |
| osd_max_push_cost | 8^20 | MOSDPGPush消息的大小 |
| osd_max_push_objects | 10 | MOSDPGPush消息允许的Object数量 |
| osd_recovery_cost | 20MB |入ShardOpWq队列时配置，**待补充**|
| osd_recovery_priority | 5 | 入ShardOpWq队列时配置，**待补充**|

Recovery自己已经具备了一些优先级控制的功能，上表给出了一些控制参数，下面一一介绍下每个参数的作用。

Ceph主线分支中Recovery拥有独立的工作队列和线程池，线程池的线程数目由配置项*osd_recovery_threads*指定，默认为1。Ceph wip_dmclock2分支取消了Recovery的工作队列和线程池，转而将Recovery Op入ShardOpWq队列。这样Recovery Op和其它类型Op在相同的队列，因此理论上会有更好的控制效果。

### 预留

```
OSDService
    |-- remote_reserver: AsyncReserver<spg_t>
    |-- local_reserver: AsyncReserver<spg_t>
        |-- queues: map<unsigned, list<pair<T, Context*> > >  // Key为pg恢复的优先级，Value为List，List元素为<pgid, QueuePeeringEvt>
        |-- queue_pointers: map<T, pair<unsigned, typename list<pair<T, Context*> >::iterator > >  // Key为pgid，Value为queues[prio]
        |-- in_progress: set<T>  // 正在处理的请求
        |-- f: Finisher  // 调用queues中Context的队列
        |-- max_allowed: unsigned  // osd_max_backfills配置项
        |-- min_priority: unsigned  // osd_min_recovery_priority配置项

1. WaitLocalRecoveryReserved::WaitLocalRecoveryReserved() --> AsyncReserver::request_reservation() --> AsyncReserver::do_queues()
2. QueuePeeringEvt::finish() --> PG::queue_peering_event(LocalRecoveryReserved)  // 由AsyncReserver的Finish线程调用
```
在开始恢复数据前Ceph会先进行预留，预留的其中一个目的是控制不同PG恢复的优先级。预留通过AsyncReserver类实现，该类包含了一个优先级队列queues，预留时先将PG入优先级队列，再根据PG的优先级从高到低的顺序出队列，优先级越高的PG越先恢复。虽然AsyncReserver以PG为单位进行优先级控制，但事实上用户以Pool为单位设置PG的优先级。Ceph的*ceph osd pool set recovery_priority*命令用于设置Pool的Recovery优先级，属于同个Pool的PG具有相同的优先级。

预留的另一个目的是控制OSD中同时进行恢复的PG数目。AsyncReserver::max_allowed限制PG出队列，若正在处理的PG数目超过max_allowed则后面的请求将留在队列内直到其它PG完成恢复后才出队列。预留会同时考虑Primary OSD和Replica OSD，Primary OSD通过local_reserver来预留，Replica OSD通过remote_reserver来预留，默认每个AsyncReserver同一个时刻只允许一个PG进行恢复。因为一个OSD同时为某些PG的Primary为另一些PG的Replica，所以一个OSD同一时刻只允许两个PG进行恢复。


### 恢复

```
PGRecovery
    |-- reserved_pushes: uint64_t

OSDService
    |-- awaiting_throttle: list<pair<epoch_t, PGRef> >
    |-- recovery_ops_reserved: uint64_t  // 预留的ops，进ShardOp队列的recovery请求数
    |-- recovery_ops_active: uint64_t
    |-- defer_recovery_until: utime_t  //  允许Recovery启动的时间
    |-- recovery_paused: bool  // 暂停Recovery，通过OSDMap来设置

// 状态机进入Recoverying状态，将PGRecovery Op入ShardOpWq队列
Recovering::Recovering() --> PG::queue_recovery(false) --> OSDService::_maybe_queue_recovery() --> OSDService::_queue_for_recovery() --> ShardedWQ::queue(PGRecovery)

RPGHandle(PGBackend::RecoveryHandle)
    |-- pushes: map<pg_shard_t, vector<PushOp>>  // pg_shard_t目标OSD，PushOp Object的详细内容
    |-- pulls: map<pg_shard_t, vector<PullOp>>  // pg_shard_t目标OSD，PullOp Object的详细内容

MOSDPGPush
    |-- pushes: vector<PushOp>

// 出ShardOpWq队列
PGQueueable::RunVis::operator() --> OSD::do_recovery() --> ReplicatedPG::start_recovery_ops() --> ReplicatedPG::recover_replicas() -->
1. ReplicatedPG::prep_object_replica_pushes() --> ReplicatedBackend::recover_object() --> ReplicatedBackend::start_pushes() --> ReplicatedBackend::prep_push_to_replica() --> ReplicatedBackend::prep_push()
2. ReplicatedBackend::run_recovery_op() --> ReplicatedBackend::send_pushes()
```
回顾下前端IO控制，在3副本情况下一个前端MOSDOp请求将衍生出两个额外的MOSDRepOp请求，而mClock队列只控制MOSDOp请求的速度，通过MOSDOp来间接控制MOSDRepOp请求。Recovery也采用类似的策略，mClock队列控制PGRecovery出队列的速度，而每个PGRecovery可能对应多个Object的恢复，通过PGRecovery间接控制Object的恢复。

除了在mClock队列中控制PGRecovery速度外，Ceph还提供了多种手段来控制PGRecovery到Object的映射关系。**首先**，限制每个PGRecovery对应的Object的数目，由*osd_recovery_max_single_start*配置决定，默认为1。也就说，每个PGRecovery默认只能恢复一个Object。**其次**，限制活动的Object恢复操作，由*osd_recovery_max_active*配置决定。Ceph将恢复Op分为两类：一类是Active恢复操作代表已经正在恢复的操作；另一类是预留的恢复操作，代表正在mClock队列等候的PGRecovery对应的恢复操作。当PGRecovery入mClock队列时，根据这两类操作数以及osd_recovery_max_active来限制PGRecovery允许的Object个数。**最后**限制恢复请求中对象的个数和大小，由*osd_max_push_cost*和*osd_max_push_objects*两个配置决定。


```
// Replica处理MOSDPGPush消息
1. OSD::dispatch_op_fast() --> OSD::handle_replica_op() --> OSD::enqueue_op() --> PG::queue_op() --> ShardedWQ::queue()
2. ReplicatedPG::do_request() --> ReplicatedBackend::handle_message() --> ReplicatedBackend::do_push() --> ReplicatedBackend::_do_push()

// Primary处理MOSDPGPushReply消息
3. ReplicatedPG::do_request() --> ReplicatedBackend::handle_message() --> ReplicatedBackend::do_push_reply() --> ReplicatedBackend::handle_push_reply() --> ReplicatedPG::on_global_recover() --> PG::finish_recovery_op() --> OSDService::finish_recovery_op() --> OSDService::_maybe_queue_recovery()
```
一个Recovery Object的所有Replica都恢复后，Primary重新向ShardOp队列投递PGRecovery请求，ShardOp线程开始下个Object的恢复。所有Object都恢复后ShardOp线程使用AllReplicasRecovered事件将状态机从Recovering状态切换到Recovered状态，同时释放Replica的预留。最后，如果所有节点都Active，则从Recovered状态切换到Clean状态。


## (Deep)Scrub

| 配置项 | 默认值 | 说明 |
|:--|
| osd_scrub_chunk_min | 5 | PGScrub对应的Object数目的最小值 |
| osd_scrub_chunk_max | 25 | PGScrub对应的Object数目的最大值 |
| osd_deep_scrub_interval | 1周 | Deep scrub周期 |
| osd_scrub_sleep | 0 | 两个PGScrub Op间休息一段时间 |
| osd_heartbeat_interval | 6 | 周期性执行OSD::sched_scrub函数 |
| osd_scrub_begin_hour | 0 | 允许触发Scrub的时间段的起始时间 |
| osd_scrub_end_hour | 0 | 允许触发Scrub的时间段的结束时间，结束时间可以小于起始时间|
| osd_scrub_auto_repair | false | 自动repair不一致Object，不支持副本池，只支持EC池|
| osd_max_scrubs | 1 | OSD允许同时运行的Scrub任务的最大数目 |
| osd_scrub_min_interval | 60\*60\*24 | 一天 |
| osd_scrub_max_interval | 7\*60\*60\*24 | 一周 |
| osd_scrub_interval_randomize_ratio | 0.5 | [min, min*(1+randomize_ratio)] |
| osd_scrub_during_recovery | true | 允许在OSD Recovery过程中执行Scrub任务 |
| osd_scrub_load_threshold | 0.5 | 只有负载低于该值时才允许触发Scrub |

同前端IO和Recovery一样，Ceph通过控制PGScrub来间接控制Scrub的所有IO优先级。

### 启动

```
OSDService
    |-- sched_scrub_pg: set<ScrubJob>  // 已注册的Scrub任务
        |-- sched_time: utime_t  // 任务开始执行的时间
        |-- deadline: utime_t

// 注册ScrubJob
PG::reg_next_scrub()  --> OSD::reg_pg_scrub() --> OSD::sched_scrub_pg

// 调度ScrubJob
OSD::init() --> C_Tick_WithoutOSDLock::finish() --> OSD::tick_without_osd_lock() --> OSD::sched_scrub() --> PG::sched_scrub() --> PG::queue_scrub() --> PG::requeue_scrub() --> OSD::queue_for_scrub()
```
Ceph以PG为单位执行Scrub操作，若要执行Scrub操作事先需要以ScrubJob的形式向OSD注册，OSD会定时检查注册的ScrubJob，若条件满足则开始执行Scrub操作。这涉及到两个问题：第一个问题是何时注册ScrubJob，第二个问题是何时执行ScrubJob。执行PGLog合并、PG分裂、Scrub相关命令时都会注册ScrubJob，每个ScrubJob包含一个任务开始时间(sched_time)和一个最终时间(deadline)。默认情况下，ScrubJob::sched_time小于当前时间+osd_scrub_min_interval，ScrubJob::deadline为当前时间+osd_scrub_max_interval，注册任务后正常情况在一天内执行，特殊情况一周内执行。

OSD进程启动时初始化C_Tick_WithoutOSDLock定时器，定时器默认每隔18秒检查一次OSD中已注册的ScrubJob，对满足条件的ScrubJob开始执行预留操作。检查的内容包括以下几项：

- 检查是否到达ScrubJob的开始时间（sched_time），没到达不执行；
- 在有Recovery Op的情况下，检查 osd_scrub_during_recovery配置是否允许同时执行Scrub任务；
- 检查PG是否为Active状态，非Active不允许执行；
- 检查ScrubJob是否超期(deadline)；
- 检查当前时间是否位于允许Scrub的时间段内并且系统负载也在允许范围内。

特别注意在ScrubJob已经超期的情况下将忽略最后一个限制条件强制执行Scrub任务。另外值得一提的是Scrub时间段，它由osd_scrub_begin_hour和osd_scrub_end_hour两个配置项控制。osd_scrub_begin_hour可以小于也可以大于osd_scrub_end_hour，它们两的取值范围都是0到24。当osd_scrub_begin_hour小于osd_scrub_end_hour时，允许时间段为[osd_scrub_begin_hour, osd_scrub_end_hour]；当osd_scrub_begin_hour大于osd_scrub_end_hour时，允许的时间段为[osd_scrub_begin_hour, 24]和[0, osd_scrub_end_hour]两部分。因为0和24是重叠的，所以实际上这两个时间段是连续的。

### 预留

```
PG
    |-- scrubber: Scrubber
        |-- reserved: bool  // 是否已预留
        |-- reserve_failed: bool // 预留失败，只要一个Peer预留失败就代表预留失败
        |-- reserved_peers: set<pg_shard_t>  // 预留成功的Peer OSD

OSD
    |-- scrubs_pending: int  // 排队的Scrub任务数目
    |-- scrubs_active: int  // 运行的Scrub任务数目

// Replica处理预留请求
ReplicatedPG::do_request() --> ReplicatedPG::do_sub_op() --> PG::sub_op_scrub_reserve() --> OSDService::inc_scrubs_pending()

// Primary处理预留回复
ReplicatedPG::do_request() --> ReplicatedPG::do_sub_op_reply() --> PG::sub_op_scrub_reserve_reply() --> PG::sched_scrub()
```
ScrubJob满足调度条件后开始执行Scrub前需要向PG的所有OSD节点申请预留，只有预留成功后才允许开始Scrub操作。预留的目的是为了限制OSD进程内同时进行Scrub任务的个数。Ceph将Scrub任务的状态划分为两类：一类是已经在运行，另一类是正在预留阶段还没开始执行的。只有这两类的Scrub任务总数低于osd_max_scrubs配置时才能够预留成功。osd_max_scrubs默认值为1，也就是说OSD进程同一时刻最多只能运行一个Scrub任务。 只有PG的所有OSD节点都预留成功后，Ceph才开始向mClock队列投递PGScrub Op开始真正的Scrub操作。

### 比较

```
PG
    |-- scrubber: Scrubber
        |-- store: std::unique_ptr<Scrub::Store>
        |-- waiting_on: int  // 未完成Scrub的Secondary OSD的数目
        |-- waiting_on_whom: set<pg_shard_t>  // 未完成Scrub的Secondary OSD
        |-- received_maps: map<pg_shard_t, ScrubMap>  // Secondary OSD的ScrubMap
        |-- subset_last_update: eversion_t  // 影响chunk中object的最近的版本号
        |-- active_rep_scrub: OpRequestRef  // (备OSD)等待subset_last_update版本完成
        |-- queue_snap_trim: bool  // Scrub结束后执行SnapTrim，也就是说，Scrub和SnapTrim不能同时执行

PGQueueable::RunVis::operator(PGScrub) --> PG::scrub() --> PG::chunky_scrub()
```
执行Scrub操作的主要逻辑：首先选出一组Object，Object的个数由osd_scrub_chunk_min和osd_scrub_chunk_max两个配置决定；然后向PG的其它OSD节点请求ScrubMap；接收到所有Peer OSD节点的ScrubMap后进行比较。同Recovery一样，此处要考虑一个PGScrub Op和Object的对应关系。


## SnapTrim

| 配置项 | 默认值  | 说明 |
|:--|
| osd_snap_trim_cost | 1MB | |
| osd_snap_trim_priority | 5 | |
| osd_snap_trim_sleep | 0 | 两次PGSnapTrim请求间休眠时间 |
| osd_pg_max_concurrent_snap_trims | 2 | 每个PGSnapTrim对应的Object数目 |

### 客户端删除快照

从客户端来说，快照整体上应该同时包含RBD块快照和CephFS快照两种类型，本节只考虑RBD块的快照。RBD块的快照数据包含两部分内容：一部分是**存储块级别**的快照元数据，保存在header对象的OMAP；另一部分是**Object级别**的快照信息，这部分又由保存在Object属性中的快照元数据和Clone Object两部分内容构成。Ceph删除这两部分内容的方式不同。

```
// RBD客户端向OSD发送删除快照的消息
rbd::Shell::execute() --> rbd::action::snap::execute_remove() --> rbd::action::snap::do_remove_snap() --> librbd::Image::snap_remove2() --> librbd::snap_remove() --> librbd::Operations<librbd::ImageCtx>::snap_remove() --> Operations<I>::snap_remove() --> librbd::Operations<librbd::ImageCtx>::execute_snap_remove() --> librbd::operation::SnapshotRemoveRequest::send() --> cls_client::snapshot_remove() --> ... --> 发送op给rbd_header对象所在的Primary OSD

// OSD删除快照信息
cls_rbd::snapshot_remove() --> cls_cxx_map_remove_key() --> ReplicatedPG::do_osd_ops(CEPH_OSD_OP_OMAPRMKEYS)

// RBD客户端向Monitor发送删除快照的消息
librbd::operation::SnapshotRemoveRequest::send() --> SnapshotRemoveRequest<I>::send_release_snap_id() --> Objecter::delete_selfmanaged_snap() --> 
Objecter::pool_op_submit() --> Objecter::_pool_op_submit() --> MonClient::send_mon_message()

// Monitor删除快照信息
OSDMonitor::prepare_pool_op() --> pg_pool_t::remove_unmanaged_snap() --> pg_pool_t::removed_snaps
```
对第一部分内容，RBD客户端直接向header对象所在的Primary OSD发送CEPH_OSD_OP_OMAPRMKEYS消息，立即删除。对第二部分内容，Ceph采用异步策略：先向Monitor节点发送删除快照的请求，Monitor回复后客户端即可退出，宣告快照已被删除。同时，Monitor修改OSDMap中和快照相关的数据构建OSDMap增量，并在适当的时候将新版OSDMap分发给相关OSD节点，OSD节点接收到新OSDMap后获得待删除快照，从而开始删除Object级别的快照信息。

```
ReplicatedPG(PG)
    |-- snap_trimq: interval_set<snapid_t>  // 待删除的快照列表
    |-- pool: PGPool
        |-- cached_removed_snaps: interval_set<snapid_t>  // 总的快照列表
        |-- newly_removed_snaps: interval_set<snapid_t>  // 一次更新中，新产生的待删除快照列表

// OSD处理MOSDMap消息，扫描PG向Peering队列投递NullEvt事件
OSD::handle_osd_map() --> C_OnMapCommit::finish() --> OSD::_committed_osd_maps() --> OSD::consume_map() --> PG::queue_null() --> PG::queue_peering_event()

// OSD Peer工作线程处理NullEvt事件
OSD::process_peering_events() --> OSD::advance_pg() --> PG::handle_advance_map() --> 
1. PGPool::update()  // 更新PGPool中待删除的快照列表
2. RecoveryState::handle_event(AdvMap) --> RecoveryState::Active::react(AdvMap)--> ReplicatedPG::kick_snap_trim() --> SnapTrimmer::process_event(KickTrim)  // 更新snap_trimq，通知状态机开始删除快照对象
```
OSD分别在PG::snap_trimq和PGPool中保持了待删除快照列表，真正开始删除数据时从PG::snap_trimq中取快照。为什么要在两个地方保存待删除快照？估计考虑到了PG Recovery的不同状态，在PG切换到Active状态时会将PGPool中的待删除列表赋值给snap_trimq。如果更新OSDMap时，PG恰好处于Active状态那么将同时更新PGPool和snap_trimq。上面给出了更新OSDMap时更新待删除快照列表的流程。

### OSD删除快照数据

```
ReplicatedPG(PG)
    |-- snap_trimmer_machine: SnapTrimmer
    |    |-- NotTrimming  // 状态机初始状态
    |    |-- AwaitAsyncWork  // 工作状态
    |    |-- WaitRWLock
    |    |-- WaitScrub  // 等待Scrub结束
    |    |-- WaitRepops  // 等待Replica完成快照对象删除
    |-- snap_trimq: interval_set<snapid_t>  // 待删除的快照列表
    |-- snap_mapper: SnapMapper

// PGSnapTrim入ShardedOpWq队列
AwaitAsyncWork::AwaitAsyncWork() --> OSDService::queue_for_snap_trim()

// PGSnapTrim出ShardedOpWq队列
PGQueueable::RunVis::operator(PGSnapTrim) --> ReplicatedPG::snap_trimmer() --> AwaitAsyncWork::react(DoSnapWork) --> ReplicatedPG::simple_opc_submit() --> ReplicatedPG::issue_repop() --> ReplicatedBackend::submit_transaction() --> ReplicatedBackend::issue_op()
```
同Recovery、Scrub一样，SnapTrim也是通过控制PGSnapTrim来间接控制快照删除的整体速度。一个SnapTrim默认对应删除两个对象的快照，由osd_pg_max_concurrent_snap_trims配置决定。
