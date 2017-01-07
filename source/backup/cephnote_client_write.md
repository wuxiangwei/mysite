[TOC]

## 客户端写流程

### 发送请求

```
ImageCtx
    |-- aio_work_queue: AioImageRequestWQ(PointerWQ)
			|-- m_items: std::list<T *>
            |-- m_pool: ThreadPool
                    |-- work_queues: vector<WorkQueue_*>  // 工作队列组
					|-- _threads: set<WorkThread*>  // 工作线程组
						|-- last_work_queue: int  // 工作队列组的下标

rbd_aio_write() --> AioImageRequestWQ::aio_write() --> AioImageRequestWQ::queue() --> 
ThreadPool::PointerWQ::queue() --> ThreadPool::PointerWQ::m_items
```
**请求入AioImageRequestWQ队列**。线程池m\_pool是个单例，也就说，同个进程内所有存储块的IO都共享一组线程，默认1线程。线程池中线程的入口函数为ThreadPool::worker，一个线程池可以包含多个工作队列，每个工作队列都必须实现WorkQueue_::void_process接口，接口实现了给定工作队列中元素的具体处理流程。因此，对AioImageRequestWQ队列来说，每个写请求的都在AioImageRequestWQ::process函数中进行处理。

一个线程池，多个工作线程，多个工作队列。每个工作线程对应多个工作队列，那么线程如何选择一个工作队列就成了个问题。工作线程取元素可以以轮为单位，每轮都要访问下每个工作队列，每次访问工作队列最多只取其一个队列元素，如果队列为空则跳过。

每个Image块，一个队列，所有队列被同一个线程池处理。

```
AioImageRequestWQ::process() --> AioImageRequest::send() -->
```
出队列

```
librbd::RBD::AioCompletion
    |-- pc: void* // 具体类型为 librbd::AioCompletion

// Rbd request
librbd::AioImageRequest(librbd::AbstractAioImageWrite(AioImageRequest))
    |-- m_aio_comp: AioCompletion  // RBD 请求完成的回调
        |-- start_time: utime_t  // 入AioImageRequestWQ队列的时间
        |-- pending_count: uint32_t // Rbd请求对应ObjectExtent的数目，请求拆分成多个Object
        |-- complete_cb: callback_t  // 调用者需要完成的事情

// object request
AioObjectWrite(AbstractAioObjectWrite(AioObjectRequest(AioObjectRequestHandle)))
    |-- m_completion: Context  // 在AioObjectRequest类定义，实例化为C_AioRequest类
        |-- m_completion: AioCompletion // rbd 请求的completion

AioImageRequest<I>::send() -->
AbstractAioImageWrite<I>::send_request() --> AbstractAioImageWrite<I>::send_object_requests() -->
AioObjectWrite::send() --> AbstractAioObjectWrite::send() --> AbstractAioObjectWrite::send_pre() -->
AioObjectWrite::send_write() --> AbstractAioObjectWrite::send_write_op() --> 

// Op
Objecter::Op
    |-- oncommit: Context // C_aio_Safe实例
    |    |-- c: librados::AioCompletionImpl
    |-- onack: Context  // C_aio_Ack实例
        |-- c: librados::AioCompletionImpl


librados::AioCompletion
    |-- pc: librados::AioCompletionImpl
        |-- callback_complete: rados_callback_t
        |-- callback_complete_arg: void*  // 为AbstractAioObjectWrite对象
        |-- callback_safe: rados_callback_t
        |-- callback_safe_arg: void*
        |-- io: IoCtxImpl*
            |-- client: RadosClient
                |-- finisher: Finisher

C_AioSafe
    |-- c: AioCompletionImpl

librados::IoCtx::aio_operate(snap_seq, snaps) --> librados::IoCtxImpl::aio_operate(snapc) -->
```
**Rbd请求、Object请求和Op**。一个Rbd请求可能处理多个Object，send_request将Rbd请求转换为多个Object请求。一个Object请求完成时调用C_AioRequest::finish，从而调用AioCompletion::complete_request，complete_request递减pending_count计数，如果计数为0代表所有Object请求均已完成，调用AioCompletion::complete处理上层业务的回调。


```
C_aio_Safe::finish() --> ... -->
C_AioSafe::finish() --> rados_callback(librados::AioCompletionImpl, AbstractAioObjectWrite) --> 
AbstractAioObjectWrite::complete() --> AioObjectRequest<I>::complete() --> 
C_AioRequest::complete() --> C_AioRequest::finish() --> AioCompletion::complete_request() --> 
callback_t() // 回调上层业务
```
**接收响应后的回调过程**。C_aio_Safe::finish创建C_AioSafe对象并将其放入RadosClient的finisher，交由后者执行回调。


```
AsyncConnection
	|-- async_msgr: AsyncMessenger
		|-- my_inst： entity_inst_t
			|-- name: entity_name_t

Message
	|-- connection: ConnectionRef
	|	|-- peer_addr: entity_addr_t  // 连接建立时设置peer地址
	|-- header: ceph_msg_header
		|-- src: ceph_entity_name  // 源的名字，即AsyncMessenger::my_inst的名字

1. IoCtxImpl::queue_aio_write() --> IoCtxImpl::aio_write_list
2. Objecter::op_submit() --> Objecter::_op_submit_with_budget() --> Objecter::_op_submit() -->
Objecter::_send_op() --> AsyncConnection::send_message() --> AsyncConnection::out_q
```
**发送消息**。AbstractAioImageWrite::send_request 将写请求转换为若干写Object的请求。
AsyncConnection::send_message为Message设置ceph_entity_name和connection，Message的源entity_inst_t由两部分构成，一部分来自ceph_entity_name，另一部分来自connection。


```
Objecter::get_session() --> AsyncMessenger::get_connection() --> AsyncMessenger::create_connect() -->
AsyncConnection::connect() --> AsyncConnection::_connect()
```
客户端和OSD节点建立连接的过程。AsyncConnection有个简单的状态机，在AsyncConnection::_process_connection函数中完成建立过程。

```
C_handle_read::do_request() --> AsyncConnection::process() --> AsyncConnection::_process_connection()
```
客户端处理请求的流程。


### 接收响应

```
entity_name_t
    |-- _num: int64_t  // osd.0 中的0

Objecter
    |-- osd_sessions: map<int,OSDSession*>  // 维护osd编号和Session间的对应关系
         |-- ops: map<ceph_tid_t,Op*> ops  // 维护tid和Op间的对应关系
            |-- onack, oncommit: Context  // 具体类参见上文
                |-- c: AioCompletionImpl
                    |-- callback_complete, callback_safe: rados_callback_t
                    |-- io: IoCtxImpl
                        |-- client: RadosClient
                        |-- aio_write_waiters: map<ceph_tid_t, std::list<AioCompletionImpl*> >

Objecter::ms_fast_dispatch() --> Objecter::ms_dispatch() --> Objecter::handle_osd_op_reply() --> 
IoCtxImpl::C_aio_Safe::finish() --> C_AioSafe::finish() --> AioObjectRequest<I>::complete() --> 
AioObjectRequest::m_completion::complete()
```
**接收响应，开始回调**，通知上层应用请求完成的消息。响应消息包含osd和tid，handle_osd_op_reply根据osd_num找到Session，继而根据tid找到Op，根据Op回调onack和oncommit函数。


**问题 1**: client.xxx 中的xxx是怎么生成的？
```
RadosClient
    |-- messenger: Messenger
            |-- my_inst: entity_inst_t
                    |-- name: entity_name_t
                            |-- _num: int64_t
```
静态结构。首先，client.xxx中的“xxx”对应于entity_name_t实例的_num属性。如上所示，一个client.xxx对应于一个Messenger实例，Messenger实例由RadosClient对象维护。在RadosClient::connect中初始化client.xxx，xxx的值从Monitor中获取。


**问题 2**：应用程序需要维护哪些实例？

```
RadosClient(实例由应用程序维护)
    |-- objecter: Objecter
    |    |--osd_sessions: map<int,OSDSession*>
    |        |--con: Connection(AsyncConnection)
    |-- messenger: AsyncMessenger
        |-- stack: NetworkStack (单例)
            |-- workers: vector<Worker*>

AsyncConnection
    |-- worker: Worker
    |-- center: EventCenter
```
单例NetworkStack实例维护一组Worker，新建一个AsyncConnection时选择一个负载最小的Work给AsyncConnection。

**问题 3** AsyncConnection发送消息的流程？

```
AsyncConnection::send_message() --> 
EventCenter::dispatch_event_external() --> EventCenter::external_events -->
```
send_message函数将消息放入AsyncConnection::out_q队列，同时向EventCenter投递一个写事件，写事件的回调函数为C_handle_write实例。

```
EventCenter::process_events() --> C_handle_write::do_request() -->
AsyncConnection::handle_write() --> AsyncConnection::write_message() --> 
AsyncConnection::_try_send() --> ConnectedSocket::send() // 通过socket发送数据
```
EventCenter::process_events函数可以认为是NetworkStack中worker线程的入口函数，worker数目由ms_async_op_threads配置项决定。

```
RadosClient（实例由应用程序维护）
	|-- objecter: Objecter
		|-- osd_sessions: map<int, OSDSession*>
			|-- con: AsyncConection
				|--out_q: map<int, list<pair<bufferlist, Message*> > >
```
消息的优先级。发送请求时，请求先入AsyncConnection::out_q队列，out_q队列是个简单的优先级队列，所有相同优先级的请求被分到一个list内， 不同优先级的请求分到不同的list。请求出out_q队列，out_q队列出请求的方式比较简单粗暴，先处理所有最高优先级的请求，再处理所有次高优先级的请求，以此类推。因此，越低优先级的请求就越可能会被饿死。详情参考AsyncConnection::_get_next_outgoing函数。

| 配置项 | 默认值 | 说明 |
|:--|:--|:--|
| osd_client_op_priority | 63 | 客户端消息的优先级 |


```
IoCtxImpl(实例由应用程序维护)
    |-- poolid: int64_t
    |-- objecter: Objecter
```

```
ImageCtx (实例由应用程序维护)
```

### RadosClient

#### 延迟请求

```
RadosClient
    |-- monclient: MonClient
    |    |-- hunting: bool
    |    |-- timer: SafeTimer  // 定时器入口MonClient::tick
    |    |-- sub_new: map<string,ceph_mon_subscribe_item> // string代表what，可以为"osdmap"
    |    |-- sub_sent: map<string,ceph_mon_subscribe_item>
    |        |-- start: __le64  // epoch
    |        |-- flag:  __u8 // 标识，例如CEPH_SUBSCRIBE_ONETIME
    |-- objecter: Objecter
        |-- initialized: atomic_t // 标识Objecter是否已初始化
        |-- timer: ceph::timer<ceph::mono_clock>  // 定时器入口Objecter::tick
        |-- osdmap: OSDMap
        |-- osd_sessions: map<int,OSDSession*>
            |-- ops: map<ceph_tid_t,Op*>
                |-- target: op_target_t  // 包含Op对应的PG
                    |-- pgid: pg_t

// Objecter定时检查请求
Objecter::tick() --> Objecter::_maybe_request_map() --> MonClient::renew_subs()

// Objecter处理新OSDMap消息的流程
Objecter::ms_dispatch() --> Objecter::handle_osd_map() --> Objecter::_scan_requests()
```

MonClient的Hunting状态是向Monitor节点发送MAuth请求，接收到响应后结束Hunting状态。

Objecter每5秒(*objecter_tick_interval*)检查一次Session队列中的Pending请求，如果存在已经延迟的请求，就向Monitor请求新OSDMap。请求是否延迟的判断标准是请求停留在队列的时间超过10秒（*objecter_timeout*）。Objecter接收到新OSDMap后检查Session队列中的请求，如果请求对应的PG或者PG对应的OSD发生变化，则重发请求。


| 配置项 | 默认值  | 说明 |
|:--|
| mon_client_hunt_interval | 3 | |
| mon_client_ping_interval | 10 | |
| mon_client_hunt_interval_backoff | 2 | |
| mon_client_hunt_interval_max_multiple | 10 |
| objecter_tick_interval | 5 |定时检查Pending请求的时间|
| objecter_timeout | 10 |请求超时时间|


##  OSD端

```
MOSDOp
    |-- osdmap_epoch: __u32  // 请求的OSDMap epoch
```


``` C++
struct object_locator_t {
    int64_t pool;  // pool id
    string key;  //  key? RADOS对象名字
    string nspace;  // namespace
    int64_t hash;  // hash?
};
```


``` C++
class ReplicatedPG {
    // object_contexts是个cache
    SharedLRU<hobject_t, ObjectContext, hobject_t::ComparatorWithDefault> object_contexts;
    // snapset_contexts是个cache，对应于Rados对象的snapset属性
    map<hobject_t, SnapSetContext*, hobject_t::BitwiseComparator> snapset_contexts;
};
```

hobject_t
ObjectContext

对象的snapset属性
ReplicatedPG::get_snapset_context

L6600
ReplicatedPG::make_writeable


合法性检查

ReplicatedPG::do_request
1. 检查PG是否为active状态。若否，入waiting_for_active队列；
1. 检查PG是否为replay状态。若是，入waiting_for_active队列；
2. 检查Pool是否cache tier，若是，并且消息无CEPH_FEATURE_OSD_CACHEPOOL标识，不处理；

ReplicatedPG::do_op
1. 检查RADSO对象名字长度、命名空间长度、对象大小是否超标；
2. 检查消息源地址是否在OSDMap的黑名单；
3. 检查OSD是否已满，若已满，则丢弃消息；
4. 不能写快照；
5. 给定的对象是否正在Scrub；
6. 检查是否原Object是否已丢失，若是，入wait_for_unreadable_object队列；
7. 检查Object是否处于degrade或backfill状态，若是，入wait_for_degraded_object队列；
8. 检查Object的快照是否在处理，若是，则入wait_for_degraded_object队列；
9. 检查Object是否snap promotion，若是，则入wait_for_blocked_object队列；
10. 检查缓存是否已满，若是，则入block_write_on_full_cache队列；



配置项

osd_max_object_name_len (2048)  // RADOS对象名字的字符长度
osd_max_write_size (2的90次幂)  // RADOS对象的最大size限制


``` C++
namespace dmc {

enum class qos_client_type {fg_pool, fg_rbd}
struct qos_client_t {
    qos_client_type type;
    uint32_t num;
};

};

class MOSDOp {
    dmc::ReqParams qos_param;
    dmc::qos_client_t qos_client;
};
```




