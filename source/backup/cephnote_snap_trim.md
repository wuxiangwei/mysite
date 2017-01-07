[toc]

## 客户端删除快照

以RBD客户端为例。

删除rbd_header对象的OMAP中的快照记录: 
```
rbd::Shell::execute() --> 
rbd::action::snap::execute_remove() --> rbd::action::snap::do_remove_snap() -->
librbd::Image::snap_remove2() --> librbd::snap_remove() --> 
librbd::Operations<librbd::ImageCtx>::snap_remove() --> Operations<I>::snap_remove() -->
librbd::Operations<librbd::ImageCtx>::execute_snap_remove() --> 
librbd::operation::SnapshotRemoveRequest::send() -->
cls_client::snapshot_remove() --> ... --> 发送op给rbd_header对象所在的Primary OSD
cls_rbd::snapshot_remove() --> cls_cxx_map_remove_key() --> 
ReplicatedPG::do_osd_ops(CEPH_OSD_OP_OMAPRMKEYS)
```

删除Pool中的快照id记录: 
```
librbd::operation::SnapshotRemoveRequest::send() -->
SnapshotRemoveRequest<I>::send_release_snap_id() --> Objecter::delete_selfmanaged_snap() --> 
Objecter::pool_op_submit() --> Objecter::_pool_op_submit() --> 
MonClient::send_mon_message() --> ... --> 发送给MON节点
OSDMonitor::prepare_pool_op() --> pg_pool_t::remove_unmanaged_snap() --> pg_pool_t::removed_snaps
```
**RBD删除快照**。Rbd块的rbd_header对象的omap中记录了块的快照，删除快照时将omap中对应的记录删除。可以使用*rados listomapvals*命令查看记录。同时，快照是全局编号的，快照id也记录在pool中，删除快照时Rbd客户端发送消息给MON节点通知删除pool中的快照。

## OSD端删除clone对象 Snap Trim

``` C++
ReplicatedPG::snap_trimmer_machine;  // snap trim状态机
PG::snap_trimq;  // 待trim的快照集合
PG::snap_mapper;  // 
```
关键数据结构。

```
1. PG::mark_clean() --> 
2. PG::activate() --> 
3. PG::scrub_clear_state() -->
4. PG::RecoveryState::Active::react() --> 
5. ReplicatedPG::NotTrimming::react() -->
6. ReplicatedPG::TrimmingObjects::react() -->
7. ReplicatedPG::WaitingOnReplicas::react() -->
8. ReplicatedPG::release_object_locks() -->
PG::queue_snap_trim() --> OSD::queue_for_snap_trim() --> OSDService::op_wq::enqueue()
```
**入队列**。队列的元素为PGSnapTrim实例。

```
ShardedOpWQ::_process() --> PGQueueable::run() --> PGQueueable::RunVis::operator() --> 
ReplicatedPG::snap_trimmer() --> SnapTrimmer::process_event()
```
**出队列**。又碰到熟悉的ShardedOpWQ了，同client op一样，线程池中的线程从队列中取出元素，并处理。不一样的是处理方式，不同的处理方式通过RunVis访问者模式封装。RunVis重载括号运算符，针对不同的队列元素调用不同的函数处理。SnapTrimmer::process_event向状态机投递一个事件。

```
1. ReplicatedPG::TrimmingObjects::react() --> SnapMapper::get_next_object_to_trim()
2. ReplicatedPG::TrimmingObjects::react() --> ReplicatedPG::trim_object()
3. ReplicatedPG::TrimmingObjects::react() --> ReplicatedPG::apply_ctx_stats()
--> ReplicatedPG::simple_opc_submit() --> ReplicatedPG::issue_repop() --> ... -> 发送给Secondary OSD
```
SnapTrimmer是个状态机，NotTrimming是其初始状态，TrimmingObjects为Trim Object的主要状态。状态机进入初始状态时，从PG::snap_trimq中获取要删除的快照编号，并切换到TrimmingObjects状态。TrimmingObjects根据待删除的Snap Id从SnapMapper中获取该快照对应的Object，创建Trim该Object的Transaction并以Repo的方式发送给Secondary OSD。完成Object删除后，在其回调函数中又会向Op队列提交一个Snap Trim的请求，依次循环。每次进入TrimmingObjects状态时，最多可以处理osd_pg_max_concurrent_snap_trims(2)个Object。因此，理论上并不会抢占其它IO。

状态机处于TrimmingObjects状态，并且对给定的Snap ID的所有Object都已经删除，那么状态机将切换到WaitingOnReplicas状态。WaitingOnReplicas状态将已经删除的快照从PG::snap_trimq队列中移除，并将状态机切换到初始状态（即NotTrimming状态）。

## OSD端构建Snapmapper的过程

SnapMapper是snap trim流程的关键，它建立了Object到快照以及快照到Object之间的映射。
```
PG
 |-- SnapMapper
        |-- MapCacher<string, bufferlist>
             |-- OSDriver
                    | -- ObjectStore
```
静态结构。SnapMapper实现snap map逻辑，MapCacher是个cache用以缓存数据，OSDriver是个适配器用以连接MapCacher和底层的ObjectStore并且默认和meta/snapmapper对象绑定，ObjectStore为FileStore。

向Snapmapper更新数据的过程（以Secondary OSD为例）：
```
ReplicatedBackend::handle_message() --> ReplicatedBackend::sub_op_modify() --> 
PG::append_log() --> PG::update_snap_map() --> SnapMapper::add_oid() --> 
1. // 更新Object对应的快照列表，内容为(_OBJ+shard_prefix+hoid, object_snaps)
2. // 内容为(_MAP+snap_id+shard_prefix+hoid,  pair<snapid_t, hobject_t>)
```
上述流程的主要工作是配置Transaction，这些Transaction作为写请求Transaction的一部分，在写数据时完成响应操作。写操作配置Transaction的地方：一个是Primary OSD的ReplicatedPG::make_writeable函数依据客户端快照以及head对象的snapset属性来决定是否克隆新对象，另一个是PG::update_snap_map函数根据PG的日志项来决定是否更新Snapmapper，snapmapper记录保存在omap中，并和meta/snapmapper对象绑定。
