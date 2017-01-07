[toc]


## OSD比较Op请求的OSDMap版本

```
MOSDOp
    |-- osdmap_epoch: __u32  // 发送端的OSDMap版本号

OSD
    |-- session_waiting_for_map: set<Session*>
        |-- osdmap: OSDMap
        |-- waiting_on_map: list<OpRequestRef>

OSD::ms_fast_dispatch() --> OSD::dispatch_session_waiting() --> OSD::dispatch_op_fast() --> OSD::op_required_epoch()
```
OSD接收到消息，走Fast dispatch路径。检查消息中osd的版本号(osdmap_epoch)，如果消息的OSDMap版本高于OSD的OSDMap版本号，则OSD向Monitor申请获取新OSDMap版本。同时，将消息保存到Session::waiting_on_map变量，将Session注册到OSD::session_waiting_for_map变量。


## OSD更新OSDMap，重新处理等待的Op

```
OSD::ms_dispatch() --> OSD::_dispatch() --> OSD::handle_osd_map() --> 注册 C_OnMapCommit回调

C_OnMapCommit::finish() --> OSD::_committed_osd_maps() --> OSD::consume_map() --> 
OSD::dispatch_sessions_waiting_on_map() --> OSD::dispatch_session_waiting() --> 重新处理等待的Op
```
OSD接收到更新OSDMap的请求，将OSDMap持久化到磁盘。Commit回调重新处理等待新OSDMap的Session，以及Session中包含的Op请求。


## OSD等待新建PG
```
OSD
    |-- session_waiting_for_pg: map<spg_t, set<Session*> >  // 等待PG的Session
        |-- waiting_for_pg: map<spg_t, list<OpRequestRef> >  // 等待PG的Op请求

OSD::handle_op() --> OSD::get_pg_or_queue_for_pg()
```
OSD找到PG，则入ShardOpWQ队列。否则，将Session加入session_waiting_for_pg变量，等待PG创建。
