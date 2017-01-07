
---                                                                                                                                      
title: "dmClock算法测试"                                                                   
date: 2016-11-18 15:01:26                                                                           
categories: [mClock]                                                                                
tags: [mClock]                                                                                      
toc: true                                                                                           
---  

### 实验配置

本实验，采用3个客户端，每个客户端的配置如下：

| 客户端 | 下限 | 权重 | 上限 |
|:--|:--|:--|:--|
| client.0 | 30 | 1 | Inf |
| client.1 |  0 | 1 | 60  |
| client.2 |  0 | 2 | 100 |

另外，在服务端请求处理能力充裕的条件下，每个客户端均可以达到200 IOPS的能力。不过本实验限制服务端的处理能力为 100 IOPS，也就说客户端最多只能提供100 IOPS。

### 结果分析

![](DMClock_test.png)

3台客户端的启动时间不同，如上图所示，client.0最先启动，client.1在T1时刻启动，client.2在T2时刻启动。下面分析每个时间段内各个客户端的IOPS分配情况：

0~T1时间段，只有client.0处于运行状态。虽然client.0自身有200IOPS的能力，但由于服务端只能提供100IOPS，因此client.0只能享受100IOPS的服务。

T1时刻，client.1开始启动，从而导致client.0的IOPS开始下降。下降过程中短暂的波动后，client.0和client.1的IOPS达到相同的50 IOPS，恰好平分服务端的100 IOPS。client.0和client.1之所以恰好平分的原因是，两者的**权重**均为1，并且client.0没有低于其下限，client.1没有高于其上限。

T2时刻，client.2开始启动，从而导致client.0和client.1的IOPS开始下降。值得注意的是，虽然client.0和client.1都开始下降，但两者最终稳定的IOPS是不同的，client.0稳定在30 IOPS，client.1稳定在23 IOPS，而client.2的IOPS却一直保持在较高的 46 IOPS。首先，服务端只能提供100 IOPS，三个客户端的权重比例为1:1:2，按照权重比例分配IO的结果应该是25，25，50。由于25 IOPS已低于client.0的下限值30IOPS。因此，为保证client.0的下限，dmClock优先满足client.0的下限。所以才有client.0最终稳定在30 IOPS的结果。其次，对剩余的70个IOPS，dmClock将其按照client.1和client.2的权重比例分配。

T3时刻，client.0开始关机，从而导致client.1和client.2的IOPS开始上升。最终，client.1的IOPS稳定在33 IOPS，client.2稳定在66 IOPS。此时，这两者恰好按照权重比例划分服务端总的IOPS。

T4时刻，client.2开始关机，从而导致client.1的IOPS开始上升。正常来说，此时只剩client.1一个客户应该独占服务端的100 IOPS。但实验结果显示client.1只占用了60个IOPS，这是因为client.1的上限值设置为了60 IOPS。

综上分析，dmClock确实已经实现了保证下限、限制上限以及按比例分配IO的能力。
