[toc]


| 线程池 | 线程数目 | 用途 |
|:--|
| osd_tp | osd_op_threads | Peering队列 |
| osd_op_tp | | ShardOp队列 |
| disk_tp | osd_disk_threads | remove_wq队列，删除PG |
| command_tp | 1 | command_wq队列，执行命令 |


```
MCommand(Message)
    |-- fsid: uuid_t  // 集群FSID
    |-- cmd: std::vector<string>  // 命令行

// OSD处理Commad请求
OSD::_dispatch() --> OSD::handle_command() --> 入command_wq队列 --> OSD::do_command()

// Client发送Command请求
rados_osd_command() --> RadosClient::osd_command() --> Objecter::osd_command() -->
Objecter::submit_command() --> Objecter::_send_command()
```
OSD支持的命令行记录在osd_commands全局变量。
命令的路由路径。命令消息先进Dispatch队列，Dispatch线程从Dispatch队列取出请求放入到 command_wq队列，command_tp线程池消费command_wq队列中的请求。OSD::do_command处理命令。


```
1. OSD::add_newly_split_pg() --> 
2. OSD::consume_map() -->
3. OSD::_dispatch() --> OSD::dispatch_op() --> OSD::handle_pg_remove() --> OSD::_remove_pg()
```
