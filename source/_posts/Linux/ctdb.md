---
title: "ctdb源码分析"
date: 2016-10-13 10:48:26
categories: [Linux]
tags: [Linux, ctdb]
toc: true
---

需求：分析源码，了解ctdb运行机制，定位解决问题。

<!--more-->

## 消息机制

ctdb进程入口函数 /ctdb/server/ctdbd.c 文件的main函数。

### 选择1种消息机制

```C
main();
ctdb_start_daemon();
tevent_context_init();
tevent_context_init_byname();
tevent_backends; // 全局变量
```

``` shell
root@bs-dev:~/repo/samba# grep "tevent_register_backend" -R
lib/tevent/tevent_poll.c:   return tevent_register_backend("poll", &poll_event_ops);
lib/tevent/tevent_poll.c:   return tevent_register_backend("poll_mt", &poll_event_mt_ops);
lib/tevent/tevent_standard.c:   return tevent_register_backend("standard", &std_event_ops);
lib/tevent/tevent_epoll.c:  return tevent_register_backend("epoll", &epoll_event_ops);
```
全局变量 tevent_backends维护一堆消息通知的实现，包括poll、standard和epoll等，默认采用standard机制(实际上就是epoll机制)。

```C
static const struct tevent_ops epoll_event_ops = {
     .context_init       = epoll_event_context_init,
     .add_fd         = epoll_event_add_fd,
     .set_fd_close_fn    = tevent_common_fd_set_close_fn,
     .get_fd_flags       = tevent_common_fd_get_flags,
     .set_fd_flags       = epoll_event_set_fd_flags,
     .add_timer      = tevent_common_add_timer_v2,
     .schedule_immediate = tevent_common_schedule_immediate,
     .add_signal     = tevent_common_add_signal,
     .loop_once      = epoll_event_loop_once,
     .loop_wait      = tevent_common_loop_wait,
 };
```
tevent对epoll的封装函数

### 初始化epoll

监听端口，接受新连接，处理请求。

```C
ctdb_start_daemon();
fde = tevent_add_fd(
    ctdb->ev,
    ctdb,
    ctdb->daemon.sd,  // 服务fd
    TEVENT_FD_READ,
    ctdb_accept_client, // 接受客户端新连接的请求
    ctdb
);

ctdb_daemon_read_cb(); // 处理请求数据
daemon_incoming_packet(); // 依据op，分发请求

// 3种op类型的请求处理函数
daemon_request_call_from_client();      // 主要对数据库操作
daemon_request_message_from_client();
daemon_request_control_from_client();

// 分发消息
daemon_request_message_from_client();
ctdb_request_message();
ctdb_dispatch_message();

// 
message_list_db_fetch_parser(); 
ctdb_client_set_message_handler();
ctdb_register_message_handler();
struct ctdb_message_list {
    struct ctdb_message_list *next, *prev;
    struct ctdb_message_list_header *h;
    ctdb_msg_fn_t message_handler;  // 请求处理函数
    void *message_private;
};


// srvid 消息编号，例如
// ctdb/include/ctdb_protocol.h
#define CTDB_SRVID_TAKEOVER_RUN 0xFB00000000000000LL
```

使用 *struct ctdb_client* 结构体维护客户端连接的相关信息。


### 请求格式

``` C
struct ctdb_req_message;
struct ctdb_req_header;
```




