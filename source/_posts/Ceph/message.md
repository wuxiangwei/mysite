---
title: "Ceph消息处理"
date: 2014-10-09 21:10:31
categories: [Ceph]
toc: true
---


总体上，Ceph的消息处理框架是发布者订阅者的设计结构。Messenger担当发布者的角色，Dispatcher担当订阅者的角色。Messenger将接收到的消息通知给已注册的Dispatcher，由Dispatcher完成具体的消息处理。

<!--more-->

在服务端，SimpleMessenger通过Accepter实例监听端口，接收来自客户端的连接。Accepter接受客户端的连接后，为该连接创建一个Pipe实例。Pipe实例负责具体消息的接收和发送，一个Pipe实例包含一个读线程和一个写线程。读线程读取到消息后，有三种分发消息的方法：

1. **快速分发**，直接在Pipe的读线程中处理掉消息。可快速分发的消息在Dispatcher的*ms_can_fast_dispatch*中注册。
2. **正常分发**，将消息放入DispatchQueue，由单独的线程按照消息的优先级从高到低进行分发处理。需要注意的是，属于同个SimpleMessenger实例的Pipe间使用同个DispatchQueue。
3. **延迟分发**，为消息随机设置延迟时间，定时时间到时由单独的线程走快速分发或正常分发的流程分发消息。

Pipe的写线程将消息放入out_q队列，按照消息的优先级从高到低发送消息。另外，消息(Message)中携带了seq序列号，Pipe使用in_seq和out_seq记录它接收到和发送出去的消息的序列号。发送消息时，Pipe用out_seq设置消息的序列号；接收消息时，通过比较消息的序列号和in_seq来确定消息是否为旧消息，如果为旧消息则丢弃，否则使用消息的序列号更新in_seq。

