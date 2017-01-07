
## Tgt

### 初始化ISCSI驱动

```
iscsi_transport_list  // 全局变量
    |-- iscsi_tcp: iscsi_transport  // 全局变量
        |-- iscsi_tcp_init
iscsi_transport_init() --> iscsi_transport_register() // 在main函数前执行

tgt_drivers: tgt_driver  // 全局变量
    |-- iscsi: tgt_driver
        |-- iscsi_init: .init
iscsi_driver_constructor() --> register_driver(&iscsi)  // 在main函数前执行

main() --> lld_init() --> lld_init_one() --> 
iscsi_init() --> iscsi_tcp_init() --> iscsi_tcp_bind_loopback() --> tgt_event_add(accept_connection)
```
初始化iscsi驱动。*iscsi_tcp_bind_loopback*监听端口，有新连接时调用*accept_connection*函数，连接接收到新请求后调用*iscsi_tcp_event_handler*处理函数。


```
iscsi_tcp_conn_list: list_head  // 全局变量
    |-- iscsi_tcp_connection
        |-- fd: int  // 连接的fd
        |-- iscsi_conn: iscsi_connection  // iscsi连接
        |-- rx_size: int  // 
        |-- rx_buffer: unsigned char* // 读buffer
        |-- stats: iscsi_stats  // 统计信息

scsi_cmd
    |-- dev: scsi_lu
    |    |-- dev_id: uint64_t  // 从请求中解析
    |    |-- cmd_perform  // tgt_device_create时配置target_cmd_perform
    |-- out_sdb: scsi_data_buffer
        |-- buffer: uint64_t
        |-- length: uint32_t

// 从接收数据到入队列
iscsi_tcp_event_handler() --> iscsi_rx_handler() --> iscsi_task_rx_done() --> iscsi_data_out_rx_done() -->
iscsi_scsi_cmd_execute() --> iscsi_target_cmd_queue() --> target_cmd_queue() --> 
target_cmd_perform() --> scsi_cmd_perform() --> device_type_template::ops::cmd_perform()
```
读取用户请求，将请求入BS队列。


### 初始化BS模块

```
bst_list: list_head  // 全局变量
    |-- rbd_bst: backingstore_template // 全局变量
        |-- bs_thread_cmd_submit: bs_cmd_submit

main() --> bs_init() --> bs_init_signalfd() --> register_bs_module() 
```
Tgt启动，初始化bs模块。


```
g_poollist
    |-- poollist
        |-- bs_thd: bs_thread_info  // 入口函数 procaioresp
        |    |-- nr_worker_threads: int  // 线程池大小
        |    |-- worker_thread: pthread_t*  // 线程池
        |-- aio_thd: aio_thread_info  // 入口函数bs_rbd_thread_worker_fn

bs_thread_info
    |-- pending_list: list_head
    |-- request_fn: request_func_t  // 请求处理函数bs_rbd_request


bs_rbd_init() --> bs_rbd_thread_open()

main() --> event_loop() --> 
mmc_rw() --> bs_thread_cmd_submit() --> bs_thread_info::pending_list
```
Tgt启动，加载配置，创建线程。**每个Pool对应2组线程**，一组线程用于处理请求，将请求根据不同的存储块分发到不同的Image队列；另一组线程用于等待响应，请求完成后将数据发送给用户。每个线程组的线程个数为CPU个数的两倍。

