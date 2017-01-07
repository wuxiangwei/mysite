[TOC]

# 客户消息流程

## 消息从Epoll到达分发队列的过程


Step1: 从Epoll读取IO事件
```
EventCenter::process_events() --> EventDriver::event_wait() --> EpollDriver::event_wait()
```
Step2: 获取fd对应事件的处理函数
```
EventCenter::process_events() --> EventCenter::_get_file_event()
```
Step3: 处理函数将请求放入分发队列
```
EventCenter::process_events() --> FileEvent::read_cb() --> EventCallback::do_request() -->
C_handle_read::do_request() --> AsyncConnection::process() --> DispatchQueue::enqueue()
```
Worker接收到请求后，有两种分发路径：第一种路径是fast dispatch，由Worker线程直接完成对请求的处理，快速高效；另一种路径是，先放到分发队列，然后交由dispatch_thread线程进行处理。Dispatch线程从分发队列取出请求，然后又将请求放入ShardOp队列，交由Shard线程池来处理。客户端请求CEPH_MSG_OSD_OP走的是fast dispatch路径，绕过了dispatch队列，不过最终还是要进入到ShardOp队列。

NetworkStack单例维护一组Worker，Worker线程的入口函数可认为是EventCenter::process_events函数，Worker的数目可以通过修改ms_async_op_threads配置项来完成。

### Connection和Woker的关系

```
PosixNetworkStack(NetworkStack)  // 单例
	|-- threads: vector<std::thread>
	|-- workers: vector<Workers*>
		|-- center: EventCenter
			|-- owner: pthread_t // 指向threads[i]，关联Worker和std::thread
			|-- external_events: deque<EventCallbackRef>  // 外部事件
			|-- driver: EpollDriver  // 内部事件
```
一个进程（例如OSD）会有多个Worker，一个Worker代表一个工作线程。Worker能够处理两类事件：一类是底层Epoll事件，另一类是用户投递给指定Worker的外部事件。NetworkStack单例维护一组Worker，Worker独立于Messenger。


| 配置项                 | 默认值  | 说明               |
| :------------------ | :--- | :--------------- |
| ms_async_op_threads | 3    | Worker的数量        |
| ms_bind_retry_count | 3    | bind失败时，重复尝试的次数  |
| ms_bind_retry_delay | 5    | bind失败后，重试前等待的时间 |
| ms_bind_port_min    | 6800 | bind端口号的起始地址     |
| ms_bind_port_max    | 7300 | bind端口号的结束地址     |


```
AsyncMessenger
	|-- accepting_conns: set<AsyncConnectionRef>  // 接收到的连接集合
	|-- processor: std::vector<Processor>
		|-- worker: Worker
		|-- listen_socket: ServerSocket  // 监听socket
		|-- listen_handler: C_processor_accept  // 接收新连接

Processor::accept() --> AsyncMessenger::add_accept() --> AsyncConnection::accept() -->  ... -> AsyncConnection::_process_connection()
```
服务端建立连接的过程。一个Proccssor负责监听一个端口，同时和一个Worker绑定。Worker对应的线程不仅要处理监听Socket接收断开连接的事件，还要负责属于该监听Socket的所有连接的读、写事件。这是因为，服务端接收新连接时，又将该Worker和新连接绑定了。

```
Objecter
	|-- osd_sessions: map<int,OSDSession*>
		|-- con: ConnectionRef
			|-- worker: Worker
			|-- center: EventCenter

Objecter::get_session() --> AsyncMessenger::get_connection() --> AsyncMessenger::create_connect() -->
AsyncConnection::connect() --> AsyncConnection::_connect() --> ... --> AsyncConnection::_process_connection()
```
客户端建立连接的过程。客户端新建一个Connection时，从已有Woker中选出一个负载最轻的Worker，并与之绑定。后续所有对Connection的读、写操作都由该Worker对应的线程来处理。客户端将与每个相关的OSD建立连接，因此，正常情况连接数比Worker数目多，一个Worker会服务多个连接。这区别于原来的SimpleMessenger，它的每个连接都有独占两个线程，一个用于写，一个用于读。

```
C_handle_read::do_request() --> AsyncConnection::process() --> AsyncConnection::_process_connection()
```
建立连接过程中所有相关的请求都在AsyncConnection::_process_connection函数中进行处理，服务端是，客户端也是。服务端和客户端都会维护建立连接过程中自己的状态，这些状态间的转换通过请求来驱动。这个过程中，将会验证客户端、服务端的权限，例如feature是否吻合。feature包含在Policy中，在实例化Messenger时设置。

## 消息从分发队列到各个处理实体的过程
```
AsyncMessenger
	|-- dispatch_queue: DispatchQueue

// 按优先级分发
DispatchQueue::entry() --> Messenger::ms_deliver_dispatch() --> 
Dispatcher::ms_dispatch() --> OSD::ms_dispatch() --> OSD::_dispatch()

// 快速分发
AsyncConnection::process() --> DispatchQueue::fast_dispatch() -->  Messenger::ms_fast_dispatch() -->
Dispatcher::ms_fast_dispatch() -->  OSD:ms_fast_dispatch() --> OSD::dispatch_session_waiting --> OSD::dispatch_op_fast()
```

DispatchQueue实例的dispatch_thread线程从队列中取消息，并通过Messenger将消息分发给所有的订阅者，OSD实例是其中一位订阅者。
Fast dispatch的情况，请求先放入OSD::Session的waiting_on_map队列。然后调用dispatch_op_fast处理请求，如果OSD的OSDMap epoch不低于请求的OSDMap，则处理请求并将请求出队列。否则，请求**不能**出队列，同时向Monitor获取请求OSDMap epoch对应的OSDMap。

### OSDMap版本问题

```
OSD
    |-- session_waiting_for_map: set<Session*>  // 等待OSDMap的Session
        |-- waiting_on_map: list<OpRequestRef>

Connection
    |-- priv: RefCountedObject  // OSD::Session

// OSD接受新连接时，设置新连接的priv为OSD::Session实例
AsyncConnection::_process_connection() --> AsyncConnection::handle_connect_msg() --> 
Messenger::ms_deliver_handle_fast_accept() --> OSD::ms_handle_fast_accept() --> Connection::set_priv()

// OSD接收到新OSDMap，持久化到数据库后处理waiting_on_map的请求
OSD::ms_dispatch() --> OSD::_dispatch() --> OSD::handle_osd_map --> new C_OnMapCommit --> 
C_OnMapCommit::finish() --> OSD::_committed_osd_maps() --> OSD::consume_map() --> 
OSD::dispatch_sessions_waiting_on_map() --> OSD::dispatch_session_waiting()
```

OSD接收到新OSDMap后重新将请求出waiting_on_map队列。

处理客户请求时，有两个OSDMap的版本号：一个是客户持有OSDMap的版本号，夹带在MOSDOp消息；另一个是OSD持有的OSDMap版本号。处理请求时要求OSD的版本号不低于客户的版本号，如果OSD的版本号低于客户就向Monitor获取新版本，得到新版本后再处理请求。处理请求的OSDMap版本以OSD持有的版本号为准。

```
OpRequest
    |-- send_map_update: bool  // 如果客户OSDMap版本号低于OSD的OSDMap版本号，为True
    |-- sent_epoch: epoch_t  // 客户消息携带的OSDMap版本号
```
Op入ShardOp队列时，修改上面两个字段。
如果客户的OSDMap比较旧，在Op出队列时向客户发送消息以更新OSDMap，这就是所谓的sharemap。

``` C++
void OSD::handle_op(OpRequestRef& op, OSDMapRef& osdmap) {
  if (!osdmap->get_primary_shard(_pgid, &pgid)) {
    // missing pool or acting set empty -- drop
    return;
}
```
如果给定的Pool已经不存在，则丢弃消息。
如果给定的pgid找不到acting，则丢弃消息。
如果给定的PG还没创建，加入Session::waiting_for_pg，后续处理同Session::waiting_on_map。


## 消息从OSD实体到达Op队列(mClock队列)的过程
```
OSD::ms_fast_dispatch() --> OSD::dispatch_session_waiting() -->
OSD::dispatch_op_fast() --> OSD::handle_op() --> OSD::enqueue_op() -->
PG::queue_op() --> ShardedThreadPool::ShardedWQ::queue() --> ShardedOpWQ::_enqueue() -->
ShardData --> OpQueue::enqueue() --> mClockClientQueue::enqueue() --> 
mClockQueue::enqueue() --> PullPriorityQueue::add_request() --> PriorityQueueBase::do_add_request()
```

### Shard问题

| 配置项                          | 默认值  | 说明             |
| :--------------------------- | :--- | :------------- |
| osd_op_num_shards            | 5    | 每个OSD的Shard数量  |
| osd_op_num_threads_per_shard | 2    | 每个Shard对应的线程数量 |
| osd_op_threads | 2 | Peering的线程数，用于处理peering_wq队列 |

Shard问题。每个OSD默认有5个Shard，每个Shard对应一个mClock队列。从PG的角度，OSD将PG划分为5组，给定PG的请求入给定的队列。属于同个PG的请求入同个队列，保证顺序性；不同Shard的PG的请求入不同的队列，保证并发性。从dmClock客户端的角度看，一个OSD代表一个Server，但实际上一个OSD对应5个dmClock队列，这在constraint-based阶段会有问题。一种解决方法是，dmClock客户端将(OSD ID, Shard Index)视作一个Server。但这样就要求客户端知道每个OSD的Shard数量，而每个OSD的Shard数量可能是不同的，这比较不好处理。另一种解决方法是，在Shad 队列前面放一个dmClock队列。

### 请求大小

请求的优先级在mClockQueue::enqueue函数中被**忽略**。

```
OpRequest
	|-- request: Message
		|-- data: bufferlist
```
以OpRequest作为实例化PGQueueable的入参时，PGQueueable::cost被设置为请求的大小。

### DmClock Client

```
Message --> OpRequest::request --> PGQueueable::owner --> PriorityQueueBase::client_map
```
**dmClock的客户**。每个消息中携带消息的源头，这个源头最终保存到优先级队列。不过，优先级队列保存的client，并不只有IP地址和用户名字，还有请求类型(客户IO，Sub IO、快照删除IO、恢复IO以及Scrub IO)。后文将优先级队列中保持的客户称为**内部客户**。也就说，一个外部客户在优先级队列中可能对应多个内部客户。


```
Message::header::priority --> OpRequest --> PGQueueable::priority --> 忽略
Message::data::length() --> OpRequest --> PGQueueable::cost --> RequestTag::reservation
```
**消息的priority和cost**。消息的优先级暂时似乎没用到，初始化时为0，同时在mClockQueue中被**忽略**。代码中的cost指消息体的大小，cost最终被加到请求的reservation tag。也就说，cost越大延迟也会越大。

## 消息出Op队列到被处理的流程
消费Op队列的线程池的线程的入口函数：
```
WorkThreadSharded::entry() --> ShardedThreadPool::shardedthreadpool_worker() -->
BaseShardedWQ::_process() --> ShardedWQ --> ShardedOpWQ::_process() --> 
```
WorkThreadSharded是 ShardedThreadPool线程池的工作线程。

消息出Op队列：
```
mClockClientQueue(OpQueue)
	|-- queue: mClockQueue<Request, InnerClient>  // Request为<PG, PGQueueable>对
		|-- queue: PullPriorityQueue

ShardedOpWQ::_process() --> 
OpQueue::dequeue() --> mClockClientQueue::dequeue() --> mClockQueue::dequeue() --> PullPriorityQueue::pull_request()
```

处理消息：
```
ShardedOpWQ::_process() --> 
PGQueueable::run() --> PGQueueable::RunVis::operator() --> 
OSD::dequeue_op() --> PG::do_request() --> 
ReplicatedPG::do_request() --> ReplicatedPG::do_op() -->
```
ShardedOpWQ::_process函数处理队列元素时，将调用PG::lock_suspend_timeout函数对PG加锁。因此，同个PG的请求是顺序执行的。

## 处理消息的流程

```
PG
	|-- info: pg_info_t
	|	|-- last_complete: eversion_t
	|	|-- last_update: eversion_t
	|-- pg_log: PGLog
		|-- log: IndexedLog(pg_log_t)
			|-- log: mempool::osd::list<pg_log_entry_t>  // the actual log
			|-- head: eversion_t  // newest entry
			|-- tail: eversion_t
```

### 给Replica发消息

```

ReplicatedPG::do_op() -->  ReplicatedPG::execute_ctx() --> ReplicatedPG::issue_repop() --> ReplicatedBackend::submit_transaction() -->
ReplicatedBackend::issue_op() --> ReplicatedPG::send_message_osd_cluster()
```
发送消息给Secondary OSD。

```
ReplicatedPG::do_request() -->  ReplicatedBackend::handle_message() --> ReplicatedBackend::sub_op_modify_reply() --> 
```
接收来自Secondary OSD的响应。
发送响应给用户。

Primary处理Secondary响应的处理函数，参考ReplicatedBackend::sub_op_modify_reply函数。所有Peer都完成请求后的回调函数为C_OSD_RepopCommit和C_OSD_RepopApplied。


### Replica处理消息
```
ReplicatedPG
	|-- last_complete_ondisk: eversion_t  // Op已落盘，C_OSD_RepModifyCommit回调

ReplicatedPG::do_request() --> 
PGBackend::handle_message() --> ReplicatedBackend::handle_message() --> 
ReplicatedBackend::sub_op_modify() --> ReplicatedPG::queue_transactions() --> 
ObjectStore::queue_transactions() --> FileStore::queue_transaction()
```
Secondary OSD分别为Op的commit和applied阶段添加C_OSD_RepModifyCommit和C_OSD_RepModifyApply回调函数，这两个回调函数通知Priamry OSD请求的处理情况。回调函数向主OSD发送的Reply的优先级为CEPH_MSG_PRIO_HIGH。

### Replica回调


### Primary处理消息
```
ReplicatedPG::do_op() -->  ReplicatedPG::execute_ctx() --> ReplicatedPG::issue_repop() -->
ECBackend::submit_transaction() --> ReplicatedBackend::submit_transaction() -->
ReplicatedPG::queue_transactions() --> ObjectStore::queue_transactions() -->  
FileStore::queue_transaction() --> JournalingObjectStore::_op_journal_transactions(C_JournaledAhead) --> 
FileJournal::submit_entry() --> FileJournal::writeq 入日志项队列
```
submit_entry将Op放入writeq队列，使用writeq_cond通知write_thread线程开始写日志。

#### 快照Transaction
```
OpContext
	|-- at_version: eversion_t  // pg 当前的版本号
	|-- log: vector<pg_log_entry_t>  // pg log

ReplicatedPG::execute_ctx() --> ReplicatedPG::prepare_transaction() --> ReplicatedPG::make_writeable 准备Transaction。
```
ReplicatedPG::execute_ctx函数，根据Pool的快照模式，设置OpContext::snapc。如果Pool级别快照，则直接从Pool中获取。否则，从消息体中获取。SnapContex主要包括当前快照id以及快照集合。

ReplicatedPG::execute_ctx使用get_next_version函数初始化OpContext::at_version，后面添加pg log entry时，使用at_version作为entry的版本号。at_version 由OSDMap版本号和PG递增的版本号组合而成。

**问题** snap属性是记录到attr中的，跟OMAP是什么关系？ 参考snap trim文档。

#### PGLog

```
PG
	|-- info: pg_info_t
	|-- pg_log: PGLog

ReplicatedBackend::submit_transaction() --> ReplicatedPG::log_operation() -->  PG::append_log() --> PG::write_if_dirty() -->
1. PG::prepare_write_info() --> PG::_prepare_write_info()
2. PGLog::write_log_and_missing() --> PGLog::_write_log_and_missing()
```
**append_log将pg log entry添加到pg_log**。每添加一条entry，同时更新info.last_update为entry的版本，** 注意**如果info.last_update和info.last_complete相同，同时更新info.last_complete。

_prepare_write_info将PG.info记录到Transaction。_write_log_and_missing将PG.pg_log记录到Transaction。最后，写Journal时将info和pg log写到日志盘，写数据时将info和pg log写到数据盘。


#### FileStore落盘

先写日志再写数据：
```
OpSequencer
	|-- jq: list<uint64_t>
	|-- q: list<Op*>  // 写日志的回调函数中将Op入队列

FileStore(JournalingObjectStore)
	|-- apply_finishers: vector<Finisher*>  // 写数据完成时调用
	|-- ondisk_finishers: vector<Finisher*>  // 写Journal完成时调用
	|-- op_tp: ThreadPool  // 默认2线程
	|-- op_wq: OpWQ
	|-- journal: Journal*  // 具体类FileJournal
		|-- writeq： list<write_item>
		|-- write_thread: Writer  // 写Journal线程
		|-- finisher: Finisher*  // 处理回调线程
			|-- finisher_thread: FinisherThread

FileJournal::write_thread_entry() --> 
1. FileJournal::prepare_multi_write()  // 多条日志项可以合并成一个写操作
2. FileJournal::do_write() --> FileJournal::queue_completions_thru() --> 
Journal::finisher::queue()

C_JournaledAhead::finish() --> FileStore::_journaled_ahead() --> FileStore::queue_op()
```
**写Journal、Journal回调、入op_wq队列**。通过Journal::finisher队列将写Journal和调用Commit回调这两个任务放到不同的线程进行处理，write_thread线程负责写Journal，finisher负责执行回调。回调函数一方面将op入OpSequencer::q队列，将osr入FileStore::op_wq队列，同个PG的Op入相同的OpSequencer队列，从而保证顺序性。另一方面执行PG层的Commit回调。

```
OpWQ::_process() --> FileStore::_do_op() --> FileStore::_do_transactions() --> FileStore::_do_transaction()

OpWQ::_process_finish() --> FileStore::_finish_op() --> FileStore::apply_finishers
```
**出op_wq队列，写数据，入apply_finishers队列**。FileStore::op_tp线程池完成出队列，写数据流程，写完数据后将回调交给FileStore::apply_finishes队列处理。


| 配置项                         | 代码实体                  | 说明                                |
| :-------------------------- | :-------------------- | :-------------------------------- |
| filestore_omap_backend_path | FileStore::object_map | 默认路径为current/omap，将DB运行在SSD上可提高性能 |


### Primary回调

```
ReplicatedPG
	|-- repop_queue: xlist<RepGather*>
	|	|-- on_committed: list<std::function<void()>>
	|	|-- on_applied: list<std::function<void()>>
	|	|-- applies_with_commit: const bool  // commit和applied合二为一
	|	|-- pg_local_last_complete: eversion_t  // Primary的PG::info::last_complete
	|-- peer_last_complete_ondisk: eversion_t  // 接收到Secondary回复后更新
    |-- last_complete_ondisk: eversion_t  // 本地commit后的版本号
    |-- min_last_complete_ondisk: eversion_t  // 全部Peer中的最小版本号
    |-- last_update_ondisk: eversion_t
    |-- waiting_for_ack: map<eversion_t, list<pair<OpRequestRef,version_t>>>
    |-- waiting_for_ondisk: map<eversion_t, list<pair<OpRequestRef,version_t>>>
	|-- pgbackend: ReplicatedBackend
		|-- in_progress_ops: map<ceph_tid_t, InProgressOp>
			|-- on_commit: Context  // C_OSD_RepopCommit，所有Replica完成commit时调用
			|-- on_applied: Context  // C_OSD_RepopApplied，所有Replica完成applied时调用
            |-- waiting_for_commit: set<pg_shard_t>  // 等待commit的OSD
            |-- waiting_for_applied: set<pg_shard_t>  // 等待applied的OSD
```

Primary有3种回调：本地、InProgressOp和RepGather回调。**本地**回调是Priamry节点完成commit或applied事件时的回调，主要用于删除InProgressOp中自己的注册信息。**InProgressOp**回调在全部Replica都完成commit或applied时调用，主要用于触发RepGather回调。**RepGather**回调主要向客户端返回消息，并且释放Op锁。


```
|-- in_progress_ops: map<ceph_tid_t, InProgressOp>
    |-- waiting_for_commit: set<pg_shard_t>  // 等待commit的OSD
    |-- waiting_for_applied: set<pg_shard_t>  // 等待applied的OSD

ReplicatedBackend::submit_transaction() ==>
1. C_OSD_OnOpApplied --> ReplicatedBackend::op_applied()
2. C_OSD_OnOpCommit --> ReplicatedBackend::op_commit()
```
Priamry**本地**applied后，调用回调C_OSD_OnOpApplied，将自己从InProgressOp::waiting_for_applied删除，并检查是否所有Replica都完成applied，若全部完成则调用InProgressOp的on_applied回调。最后，检查InProgressOp是否完成，若完成则从in_progress_ops字典中删除。Primary**本地**commit后，调用回调。做的事情同C_OSD_OnOpApplied相同，只是将自己从InProgressOp::waiting_for_commit中删除。


```

|-- in_progress_ops: map<ceph_tid_t, InProgressOp>
	|-- on_commit: Context  // C_OSD_RepopCommit，所有Replica完成commit时调用
	|-- on_applied: Context  // C_OSD_RepopApplied，所有Replica完成applied时调用

ReplicatedPG::issue_repop() ==>
1. C_OSD_RepopCommit --> ReplicatedPG::repop_all_committed() --> ReplicatedPG::eval_repop()
2. C_OSD_RepopApplied --> ReplicatedPG::repop_all_applied() --> ReplicatedPG::eval_repop()
```
**全部**Replica完成commit时，用RepGather::pg_local_last_complete更新Primary的PG::last_complete_ondisk，RepGather::pg_local_last_complete为PG::info::last_complete的值。全部Replica完成applied的回调过程同commit过程。eval_repop函数调用RepGather的回调。
全部Replica的回调和本地回调不同，本地回调的回调函数直接保存到op，由FileStore直接调用。全部Replica的回调保存到in_progress_ops，相当于全局变量。


```
|-- RepGather
	|-- on_committed: list<std::function<void()>>
	|-- on_applied: list<std::function<void()>>

ReplicatedPG::execute_ctx() ==>  注册RepGather回调
```
RepGather::on_applied回调发送CEPH_OSD_FLAG_ACK消息给用户。
RepGather::on_committed回调发送CEPH_OSD_FLAG_ONDISK消息给用户。


# dmClock Op队列分析

``` C++
// T: 请求 std::pair<PGRef, PGQueueable>
// K: 客户 std::pair<entity_inst_t,osd_op_type_t>，osd_op_type_t为op类型
template <typename T, typename K>
class mClockQueue : public OpQueue <T, K> {
    dmc::PullPriorityQueue<K,T> queue; // K为client id，T为请求类型
};
```

## 配置Tag参数

``` C++
template<typename C, typename R, uint B>
class PriorityQueueBase {
    ClientInfoFunc client_info_f;  // 创建客户信息的工厂函数
};
```

```
mClockOpClassQueue::mclock_op_tags_t::mclock_op_tags_t --> 
mClockClientQueue::op_class_client_info_f --> 
mClockQueue --> PullPriorityQueue --> PriorityQueueBase::client_info_f
```
mClockClientQueue队列的客户信息的工厂函数为mclock_op_tags_t，它不区分客户，只区分IO类型。也就说，相同IO类型的内部客户具有相同的Tag配置。它的主要目标是区分IO类型，为不同类型的IO分配不同的优先级。mClockClassQueue队列亦如是。

**重要** 目前实现只支持不同IO类型的优先级，不支持不同客户端的优先级。

## 区分IO类型

``` C++
mClockClientQueue::enqueue() --> mClockClientQueue::get_inner_client() --> 
mClockClientQueue::get_osd_op_type() --> pg_queueable_visitor_t::operator()
```
pg_queueable_visitor_t::operator()根据PGQueueable::qvariant中不同的IO来定义新的IO类型**osd_op_type_t**。对RequestOp来说，分为两类：一类是client op，另一类是scrub op。这两者通过消息类型来区分。另外，Snap Trim删除Secondary OSD上的副本时，通过Reop实现，因此会和client op**混淆**。


```
OSDService::queue_for_recovery() --> OSDService::_maybe_queue_recovery() --> 
OSDService::_queue_for_recovery() --> OSDService::op_wq::queue()
```
**Recovery IO**入队列过程。OSDService::_queue_for_recovery函数准备队列元素PGQueueable的qvariant为PGRecovery的实例。


```
OSDService::queue_for_scrub() --> OSDService::op_wq::queue()
```
**Scrub IO**入队列过程。

# dmClock配置项

在config_opts.h文件中搜索关键mclock。

# 文件列表

配置选项：config_opts.h

# Ceph开发环境

## vstart

``` shell
cd build
make vstart
```
编译

``` shell
OSD=3 MON=3 ../src/vstart.sh -d -n -x
```
部署集群

``` shell
bin/ceph -s
```
敲ceph命令

``` shell
../src/stop.sh
rm -rf dev out
```
销毁集群

## Gdb


| 命令                                    | 说明                                |
| :------------------------------------ | :-------------------------------- |
| attach pid                            | attach到指定的进程号pid上                 |
| n                                     | 单步运行                              |
| bt                                    | 查看调用栈                             |
| b func                                | 设置断点，断点位置为func                    |
| i b                                   | 查看断点                              |
| d 1 2 3                               | 删除编号为1、2和3的断点。d**不带参数**代表删除所有断点   |
| p var                                 | 查看变量var                           |
| display var                           | 查看变量var，以后每次运行到断点时就会自动显示该变量       |
| undisplay var                         | 取消display                         |
| set args                              | 设置应用程序的输入参数，例如rbd命令的参数            |
| start                                 | 启动应用程序                            |
| s                                     | 进入函数                              |
| finish                                | 完成函数，并退出到上层函数                     |
| set scheduler-locking off（默认）/on/step | 选项on代表只有当前被调试进程。解决多线程调试，s不进某函数的问题 |



**Gdb调试超时问题**

线程池ShardedThreadPool的线程每处理一个请求时都会设置超时时间和自杀时间。调试时，如果这两个时间设置地比较短就容易超时导致进程退出。解决这个问题，可以在ceph.conf文件中修改osd_op_thread_timeout和suicide_timeout参数，延长超时和自杀时间。（对vstart的情况，直接修改vstart.sh文件）
