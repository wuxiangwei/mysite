[toc]

# Peering过程

## 从OSD启动到加载PG

```
OSD
    |-- pg_map: ceph::unordered_map<spg_t,PG*>
        |-- peering_queue: list<CephPeeringEvtRef>
        |-- missing_loc: MissingLoc
        |-- pg_log: PGLog
        |-- past_intervals: map<epoch_t,pg_interval_t>
        |-- recovery_state: RecoveryState
            |--machine: RecoveryMachine

OSD::init() --> OSD::load_pgs() --> 
1. OSD::_open_lock_pg() --> OSD::_make_pg()
2. PG::read_state() --> PGLog::read_log_and_missing() --> 
2. PG::handle_loaded() --> RecoveryState::handle_event(Load)
```
根据持久化数据实例化PG对象，PG::recovery_state从Initial状态切换到Reset状态。

### Peering工作线程和队列

```
OSD
    |-- osd_tp: ThreadPool  // 处理Peering队列
    |-- osd_op_tp: ShardedThreadPool  // 处理ShardedOp队列
    |-- disk_tp: ThreadPool
    |-- command_tp: ThreadPool
    |-- service: OSDService
    |-- pg_map: ceph::unordered_map<spg_t,PG*>
        |-- osd: OSDService *
            |-- op_wq: ShardedOpWQ
            |-- peering_wq: PeeringWQ(ThreadPool::BatchWorkQueue)

OSD::init() --> OSD::consume_map() --> PG::queue_null() --> PG::queue_peering_event() -->
OSDService::queue_for_peering() --> OSDService::peering_wq
```
启动PG的Peering流程，Peering请求入队列。Peering队列中的元素可以批量处理，也就是说，出队列时一次性可以出多个元素。osd_peering_wq_batch_size限制了批量处理时元素的最大数目。

```
PeeringWQ::_process() --> OSD::process_peering_events() --> 
1. PG::handle_peering_event() --> RecoveryState::handle_event(CephPeeringEvt) --> RecoveryMachine::process_event(NullEvt)
2. OSD::dispatch_context() --> OSD::do_queries()  // 向 peer 节点发送MOSDPGQuery消息

OSD::ms_dispatch() --> OSD::_dispatch() --> OSD::dispatch_op() --> OSD::handle_pg_query() --> 
```
Peering请求出队列。


| 配置项 | 默认值 | 说明 |
|:--|:--|:--|
| osd_peering_wq_batch_size | 20 | Peering队列批量出队列时元素的最大数目 |


OSDServic的op_wq即OSD::op_shardedwq，peering_wq即OSD::peering_wq。

```
PG::read_state() --> PGLog::read_log_and_missing()
```
读取PGLog。

### 一些元数据

meta/snapmapper
meta/osdmap.epoch
meta/pgmeta

```
\_infover  ==> struct_v \_\_u8   // info代码的版本号，8之前之后有差别
\_info  ==> pg_info_t
\_biginfo  ==> past_intervals
\_epoch
\_fastinfo  ==> pg_fast_info_t
```

## Recovery状态机的状态
```
RecoveryMachine
    |-- Initial
    |-- Reset
    |-- Started
        |-- Start
        |-- Primary
        |    |-- Peering
        |    |    |-- GetInfo
        |    |    |-- GetLog
        |    |    |-- GetMissing
        |    |    |-- WaitUpThru
        |    |    |-- Incomplete
        |    |-- WaitActingChange
        |    |-- Active
        |        |-- Activating
        |        |-- Clean
        |        |-- Recovered
        |        |-- Backfilling
        |        |-- WaitRemoteBackfillReserved
        |        |-- WaitLocalBackfillReserved
        |        |-- NotBackfilling
        |        |-- Recovering
        |        |-- WaitRemoteRecoveryReserved
        |        |-- WaitLocalRecoveryReserved
        |-- ReplicaActive
        |    |-- RepNotRecovering
        |    |-- RepRecovering
        |    |-- RepWaitBackfillReserved
        |    |-- RepWaitRecoveryReserved
        |    |-- RepNotRecovering
        |-- Stray
```
状态机各状态间的包含关系。

### 状态机不同OSD间的消息通信

```
PG
	|-- info: pg_info_t
	|-- past_intervals: map<epoch_t,pg_interval_t>
```
静态结构。


#### Query消息

```
PG
	|-- recovery_state: RecoveryState
		|-- machine: RecoveryMachine
		|	|-- state: RecoveryState
		|-- rctx: RecoveryCtx
			|-- query_map: map<int, map<spg_t, pg_query_t>>  // Key为osd
			|-- notify_list: map<int, vector<pair<pg_notify_t, pg_interval_map_t>>>  // Key为osd

1. GetInfo::get_infos() --> context< RecoveryMachine >().send_query()
2. PeeringWQ::_process() --> OSD::process_peering_events() -->
OSD::dispatch_context() --> OSD::do_queries()
```
Primary发送消息分两步，第1步状态机内部状态准备query相关的数据，将这些数据保存到query_map；第2步，由Peering工作线程统一发送Query消息。状态机的状态一般由其它线程来驱动，通过这种策略避免驱动线程由于网络通信而阻塞。Query消息类型为*MOSDPGQuery*。

```
1. OSD::ms_dispatch() --> OSD::_dispatch() --> OSD::dispatch_op() --> OSD::handle_pg_query() -->
PG::queue_query() --> PG::queue_peering_event() --> OSDService::queue_for_peering(MQuery)
2. PeeringWQ::_process() --> OSD::process_peering_events() --> PG::handle_peering_event() -->
RecoveryState::handle_event() --> RecoveryMachine::process_event(MQuery) --> Stray::react() --> RecoveryMachine::send_notify()
```
Secondary接收到消息后分3步回复Primary。首先，Dispatcher线程根据消息构建Peering事件，将事件放到OSD的Peering队列。然后，由Peering工作线程取出Peering事件，进而取出MQuery状态机事件，将事件投递给状态机。如果状态机处于Stray状态，处理MQuery消息并将结果保存到notify_list变量。第3步，由Peering工作线程取出notify_list发送结果给Primary。

#### Notify消息

Notify消息主要由Secondary发送给Priamry。发送Notify消息和发送Query消息一样，也分2步。首先，状态机的状态准备Notify信息并保持到notify_list；然后，Peering工作线程将其发送给其它OSD进程。Notify消息的类型为 *MOSDPGNotify*。

```
OSD::dispatch_op() --> OSD::handle_pg_notify() --> OSD::handle_pg_peering_evt() --> 
PG::queue_peering_event() --> OSDService::queue_for_peering(MNotifyRec)
```
Primary接收Notify消息分两步。首先，Dispatcher线程接收到消息后，构建Peering事件，将事件放入Peering队列；然后，Peering工作线程将状态机事件投递给状态机，驱动状态机进行处理。

### ~~Flush机制~~

~~Flush的目的是将Transaction持久化到FileStore。~~
```
RecoveryCtx
	|-- on_applied: C_Contexts
	|-- on_safe: C_Contexts

1. Active::Active() --> PG::start_flush()

2. FlushState::~FlushState() --> PG::queue_flushed() --> PG::queue_peering_event(FlushedEvt)
3. Started::react(FlushedEvt) --> ReplicatedPG::on_flushed() --> ReplicatedBackend::on_flushed()
```
~~将flush时间post到Peer工作队列，由Peer工作线程响应事件，将FlushedEvt post到状态机，状态机的顶层状态Started和Reset处理FlushedEvt事件。~~


### GetInfo状态

```
PG
	|-- info: pg_info_t
	|-- peering_info: map<pg_shard_t, pg_info_t>
	|-- past_intervals: map<epoch_t,pg_interval_t>
        |-- first: epoch_t
        |-- last: epoch_t
        |-- up: vector<int32_t>
        |-- acting: vector<int32_t>
```
GetInfo状态是Peering状态的**初始**子状态，Peering状态又是Primary状态的初始子状态。OSD启动后，如果PG为Priamry，则进入Primary状态，进而进入到GetInfo状态。GetInfo状态的目的是获取Peer的pg_info_t，pg_info_t保存在PG::peering_info变量。因此，进入GetInfo状态的第一件事情就是检查peering_info中是否包含给定Peer的pg_info_t，如果**不存在**，则发送Query消息获取。


```
Peering
    |-- PriorSet
        |-- probe: set<pg_shard_t>  // current+prior，GetInfo状态要去探测pg_info_t的OSDs
        |-- pg_down: bool  // 找不到足够数量的OSD，并且历史OSD状态为down，无法获取pg_info_t
        |-- blocked_by: map<int, epoch_t>  // 找不到足够数量的OSD的情况下，状态为down的历史OSD节点
        |-- down: set<int>
        |-- pcontdec: IsPGRecoverablePredicate  // RPCRecPred类，检查依据现有的OSD是否能够恢复数据

GetInfo::GetInfo() --> PG::build_prior()
```
**构建PriorSet，以确定获取pg_info_t的目标OSD节点**。目标OSD来自两个地方，第一个地方是**当前**PG的acting和up集合内的OSD节点，另一个地方是PG**历史**上的acting OSD节点。历史上的OSD节点从PG::past_intervals中获取，past_interval由一组连续的OSDMap构成，PG在这些OSDMap中的acting节点相同，没变过。对给定的interval，如果interval的结束版本last低于info.history.last_epoch_started， 直接忽略。此处解释下**last_epoch_started**，它是上次成功Peering的版本（参考官网）。给定interval中的OSD大致可以分为几类：第一类，在当前的OSDMap中依然up，加入PriorSet::probe；第二类，在当前OSDMap中已经不存在，加入PriorSet::down；第三类，在当前的OSDMap中仍然存在，不过已经down，加入PriorSet::down。


```
PG
    |-- info: pg_info_t
    |    |-- history: pg_history_t
    |    |-- last_backfill: hobject_t  // 可用于判断PG incomplete状态
    |-- dirty_info: bool

GetInfo
	|-- peer_info_requested: set<pg_shard_t>  // 正在请求pg_info_t的osd

GetInfo::react(MNotifyRec) --> PG::proc_replica_info()
```
**获取目标OSD节点的pg_info_t**。目标OSD节点在构建PriorSet时确定。GetInfo以Query的方式发送获取pg_info_t::INFO请求，目标OSD以Notify方式回复pg_info_t。Primary接收到Peer的pg_info_t后，将其与PG::info::history进行合并，合并操作基本都是取高版本替换。如果合并后，PG.info.history.last_epoch_started更高了，那就要重新构建PriorSet，过滤掉较老的interval生成的OSD节点。

对给定的interval，检查其acting OSD，保证至少有一个OSD能够在interval内提供完整的Objects。也就说，对给定的interval，要能够从它acting中至少找出一个OSD，满足以下条件：
- OSD是up的；
- OSD是Completed的，否则它只有一份日志拷贝，没有Object数据。

满足条件，则进入GetLog状态，否则设置PG状态为Down，继续等待。。。


### GetLog状态

```
PG
	|-- actingbackfill: set<pg_shard_t>

GetLog
	|-- auth_log_shard: pg_shard_t  // 权威PGLog的OSD
	|-- msg: MOSDPGLog

GetLog::GetLog() --> PG::choose_acting() --> PG::find_best_info()
```
**选择权威OSD(auth_log_shard)，初始化actingbackfill**。如果auth_log_shard就是Priamry自己，进入下状态（GetMissing状态）。如果不是自己:
1. 如果Primary的pg_info_t::last_update低于auth_log的tail，进入InCompleate状态；
2. 否则，以Query方式向auth_log_shard请求pg_log_t。请求的起始版本是，高于权威OSD pg_log的tail版本中最小的last_update版本。(pg_log的tail版本是最旧的版本，head版本是最新的版本)

选择best info，对best有些偏好：
- 更新的last_update；
- 更长的log；
- 最好是Priamry自己。

```
PG
    |-- backfill_targets: set<pg_shard_t>
    |-- actingbackfill: set<pg_shard_t>

GetLog::GetLog() --> PG::choose_acting()
```

**从Up、Acting、Peer中，选择Actingbackfill、Backfill_targets、Want**。Acting的数目是Pool副本数目，优先从Up中选择，再从Acting中选择，最后从Peer中选择，选择过程中一旦满足数目要求就结束。选择标准主要看两方面，一方面是Incomplete，另一方面是last_update。对Up中的OSD，如果Complete并且last_update不低于权威OSD，则加入Acting_backfill，否则同时加入Backfill和Acting_backfill。对Acting中的OSD，只有Complete并且last_update不低于权威OSD时才加入Acting_backfill，否则忽略。对Peer中的OSD，选择方法同Acting。对所有加入Acting_backfill或者backfill的OSD，同时加到Want。


```
MOSDPGLog
	|-- log: pg_log_t // pg_log

Stray::react() --> PG::fulfill_log()
```
**权威OSD回复pg_log_t**，消息类型为*MOSDPGLog*。如果Primary请求的起始(since)版本低于pg_log的tail版本，则返回全部的pg_log，正常不会出现这种情况；否则，返回从请求的起始版本开始的所有日志。


```
PG
	|-- pg_log: PGLog

GetLog
	|-- msg: MOSDPGLog  // 权威OSD回复的pg_log_t消息

GetLog::react(MLogRec) --> GetLog::react(GotLog) --> PG::proc_master_log() --> PG::merge_log()
```
**构建master log**。


### GetMissing状态

```
PG
	|-- blocked_by: set<int>
	|-- peer_missing: map<pg_shard_t, pg_missing_t> // pg的missing列表

GetMissing
	|-- peer_missing_requested: set<pg_shard_t>  // 请求pg_log_t的OSD节点

GetMissing::GetMissing() -->...--> PG::handle_pg_query() -->  PG::fulfill_log()
```

**构建peer_missing**。遍历actingbackfill，检查OSD的pg_info_t。
- Peer的pg和master log没有交集(Peer的last_update低于master log的tail)，加入peer_missing。
- Peer的pg backfill ?? ，加入peer_missing；？？全量恢复？
- Peer的pg info和master log相同 (Peer的last_update等于last_complete，并且和master log的last_update相等)，加入peering_missing；
- Peer的pg info和master log有交集 (Peer的last_update大于master log的tail)，但不完全相同，向其请求pg log。请求log的起始版本为Peer的last_epoch_started，如果Peer的last_epoch_started高于tail，则从last_epoch_started开始；否则请求全部pg log。

```
PG
    |-- peer_info: map<pg_shard_t, pg_info_t>
    |-- might_have_unfound: set<pg_shard_t>
    |-- peer_missing: map<pg_shard_t, pg_missing_t>
    |-- pg_log: PGLog
        |-- log: IndexedLog(pg_log_t)
            |-- log: mempool::osd::list<pg_log_entry_t>

GetMissing::react(MLogRec) --> PG::proc_replica_log() --> PGLog::proc_replica_log() -->
PGLog::_merge_divergent_entries() --> PGLog::_merge_object_divergent_entries()
```
请求Peer日志的方式和GetLog状态相同，也是使用Query消息，对方直接发送log给自己。Primary接收到Peer OSD的pg log后，计算pg_missing（**详细计算过程待分析**），并将结果保存到peer_missing。peer_mising_requested中所有OSD的pg log都接收到后，投递Activate事件，将状态机转移到Active状态。这次状态转移是由GetMissing的父状态Peering状态设定的。



