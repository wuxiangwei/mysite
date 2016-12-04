
---
title: "dmClock算法原理分析"
date: 2016-11-10 14:01:26
categories: [mClock]
tags: [mClock]
toc: true
---

**术语**

| 英文 | 中文 |
|:--|:--|
| share/weight | 权重 |
| reservation | 下限 | 
| limit | 上限 |

# 介绍

存储IO调度和CPU、内存调度相比存在以下几个不同点：

* CPU、内存通常是主机内部的资源调度。但存储IO通常是跨主机的，多台主机使用共享的存储资源，主机内IO的变化会影响相邻主机的IO。
* Hypervisor中IO调度层位于客户机IO调度的底层。因此会滋生出许多问题，例如同台主机不同VM间的局部性、同时存在多种不同大小的IO、不同请求的优先级以及突发负载等。


| 算法类别 | 按比例 | 延时 | 下限 | 上限 | 突发负载 |
|:--|:--|:--|:--|:--|:--|
| 第1类算法 | Y | N | N | N | N |
| 第2类算法 | Y | Y | N | N | N |
| 第3类算法 | Y | Y | Y | N | N |
| mClock    | Y | Y | Y | Y | Y |

IO资源调度一般有3类相关的算法。第1类算法按比例分配IO资源，例如SFQ(D)，IO资源主要包括IOPS和带宽资源。第2类算法也是按比例分配IO资源，但IO资源包括延迟。第3类算法还是按比例分配IO资源，支持对给定的客户机提供最低资源保证。

## 一个例子

| VM | IOPS | 延时 | 带宽 | 权重(S) | 上限(L) | 下限(R) |
|:--|:--|:--|:--|:--|:--|:--|
| RD   | 低 | 低 | - | 100 | Inf | 250 |
| OLTP | 高 | 低 | - | 200 | Inf | 250 |
| DM   | 高 | - | - |  300 | 1000 | 0 |

上表给出3个VM，分别跑不同的业务。RD运行远程桌面，对IOPS要求不高但对延迟要求高，当IOPS较低时用户体验会很差因此保证下限为250个IOPS。OLTP要求高IOPS和低延迟，同样地下限为250个IOPS。DM运行数据迁移业务，设置上限为1000 IOPS，避免过度消耗系统带宽影响其它业务。mClock的目标是将VM的IO限制在下限和上限之间。

当系统总IOPS为1200时，按**权重比例**分配IO，那么每个VM分配到的IOPS如下：

| VM | IOPS(按比例) | IOPS(mclock) |
|:--|:--|:--|
| RD | 200 | 250 |
| OLTP | 400 | 380 |
| DM | 600 | 570 |

虽然此时系统拥有能够满足各个VM下限的IOPS（250 + 250 + 0 = 500），但RD只分配到了200IOPS低于其下限值。这种情况下，mclock的做法是：首先分配250 IOPS给RD，然后将剩余的950 IOPS按2:3的比例分配给OLTP和DM。
当系统总IOPS低于500时，无法满足每个VM的下限，因此mclock的做法是将可用的系统IO按照**下限比例**分配给VM，其中DM无法分配到IO。
当系统总IOPS在1500到2000时，直接按**权重比例**分配IO。此时不会出现VM的IO低于下限或者高于上限的情况。
当系统总IOPS高于2000时，直接按**权重比例**分配IO会导致DM的IOPS高于其上限。mclock的做法是：先为DM分配1000个IOPS，然后将剩余的IOPS按1:2的比例分配给RD和OLTP。

总而言之，分配策略会根据总吞吐量和活动VM而动态变化。将VM划分为三类：下限(R)、上限(L)、权重(P)，系统总吞吐量为T，那么IO分配公式如下：

![](http://ohn764ue3.bkt.clouddn.com/DmClock/paper/Eq.png)

mclock通过一种创新的tagging分配策略来达成上述公式。
上述公式存在的问题是，Tp可能小于0，也就是说系统总吞吐量要低于所有VM的总下限值。当出现这种情况时，mclock按**下限比例**来分配IO资源。这在某些情况下可能无法满足用户需求，例如有些VM没有分配下限值或者下限值为0，那么该VM可能永远无法分配到IO。不过这可以通过为每个VM都设置下限来解决。此外，针对这种情况也可考虑通过为下限设置优先级来解决（本文不再涉及）。


# mClock算法

适用于单主机的情况。
mClock是种基于tag的算法，基于tag的算法的**基本思路**: 为每个请求赋予1个tag，scheduler按照tag从小到大的顺序依次处理请求。

| VM | 1th | 2nd | 3rd | 4th | 5th | 
|:--|:--|:--|:--|:--|:--|
| A | **2** | **4** | **6** | 8 | 10 | 
| B | **3** | **6** | 9 | 12 | 15 | 
| C | **6** | 12 | 18| 24 | 30 | 

举个例子，假设A、B、C三台VM的权重分别为1/2,1/3和1/6。
scheduler为每个请求设置tag的方法：在上个请求权重的基础上递增1/wi。假设主机分别接收到了5个来自各台VM的请求，每个请求的tag如上表所示。假设主机在给定的时间段内只能处理其中的6个请求，那么按照tag的顺序，被处理的请求为：A的前3个、B的前2个、C的前1个请求。A、B、C被处理的请求数目恰好是它们权重的比例3:2:1。

这种tag赋值方法隐含着一个前提条件，那就是每个VM都是同时启动同时处于活动状态的。假如B一开始处于非活动状态，直到A接收到第5个请求时B才启动，这时B的连续若干个请求的tag值都会小于A，这将导致IO分配不符合预先配置的比例。为解决这个问题需要引入一个全局虚拟时钟（global virtual time），每个请求的tag尽量保持与gvt相同。

mClock提出两个主要的概念：多时钟(multiple real-time clocks)和动态时钟选择(dynamic clock selection)。多时钟是指mClock分别为上限(L)、下限(R)和权重(P)提供独立的tag，每个VM的请求都包含3个不同的tag值。动态始终选择指scheduler动态选择1个tag来调度IO资源。

mClock由三大组件组成：Tag赋值、Tag微调、请求调度。


## Tag赋值 Tag Assigment

![](http://ohn764ue3.bkt.clouddn.com/DmClock/paper/Symbols.png)

Tag赋值的目的是为每个VM的请求赋予L、R和P三个维度的tag值。三个维度的形式都相同，此处以下限为例。

![](http://ohn764ue3.bkt.clouddn.com/DmClock/paper/Eq_3.png)

上式有两个关键特性。**首先**，在请求拥挤的时间点上能够将积压的请求以1/ri的间隔分开。**其次**，VM空闲一段时间后重新活动时，该VM的第一个请求将被赋值为当前时间。从而可以避免tag值过小，其它VM分配不到IO，从而导致权重比例失调的问题。同样地，对于上限L tag，如果Tag值大于当前时间，说明此时VM已经达到了上限值，应该限制分配IO资源。对权重P tag，亦如是。

## Tag校准 Tag Adjustment

解决新VM和老VM请求调度的公平性问题。问题的原因是，老VM运行一段时间后的它的请求的tag值偏离当前时间过大，而新VM的请求又从当前时间开始赋值。从而导致新VM请求的P值低于老VM请求的P值，新VM的请求被优先处理。

![](http://ohn764ue3.bkt.clouddn.com/DmClock/paper/adjust_01.png)

如上图所示，T1时刻启动VM A，它的第一个请求的tag被设置为当前时间。随后，如果请求较多那么A的请求将偏离当前时间较远。假设在T2时刻时，A还剩余颜色为蓝色的请求没有被处理，而此时VM B被激活。B的请求将从T2开始递增，而A请求相当于从T3开始递增。从T2时刻开始，A的IO分配将受到B的影响，并且B的请求的起点比A低，所以B请求将会被优先处理，这将影响按比例分配的原则。

![](http://ohn764ue3.bkt.clouddn.com/DmClock/paper/adjust_02.png)

解决方法是以新VM加入的时间点为准，将老VM中未处理的请求的tag值调整到当前时间点。如此，所有VM都位于相同的起跑线。

## 请求调度 Request Scheduling

![](http://ohn764ue3.bkt.clouddn.com/DmClock/paper/alg.png)

mClock调度时分两个阶段，这两个阶段交替执行。

- **constraint-based**阶段，保证VM的IO不低于下限；
- **weight-based**阶段，按P值比例分配吞吐量，同时不分配L值达到上限的VM.

Scheduler首先要满足每个VM的下限，因此优先处理所有下限值小于当前时间的请求。当然, 这些请求的处理顺序依旧按照R tag从小到大进行，直到所有R tag小于当前时间的请求全部处理结束为止。这个阶段就是constraint-based阶段。
constraint-based阶段结束后，接着进入weight-based阶段。与constraint-based阶段不同的是，该阶段不是按R tag而是按P tag从小到大的顺序处理请求。并且只处理L tag值低于当前时间的请求，因为L tag大于当前时间的请求其对应的VM已经达到上限，因此暂不处理。

![](http://ohn764ue3.bkt.clouddn.com/DmClock/paper/req_sched_01.png)

weight-based阶段对请求的处理会对剩余请求的R tag的顺序构成影响。如上图所示，所有请求的R tag的顺序是BABABA。进入weight-based阶段后，VM B的第1个请求被处理掉了，因此 R tag的顺序变成了ABABA。如果在weight-based阶段VM A和VM B的比例相差较大，那么对R tag顺序的影响也越大。这种影响将导致VM B在constraint-based阶段无法分配到足够多的IO而无法达到下限要求。

![](http://ohn764ue3.bkt.clouddn.com/DmClock/paper/req_sched_02.png)

解决方法，每处理一个VM的请求时就将该VM的剩余请求的R tag值递减1/ri，如上图所示。

## 特定于存储的问题

mClock算法能够应用于多种资源的调度，例如CPU、内存和存储。在应用于存储资源调度时，有诸多特定于存储相关的问题。

### 突发请求 Burst Handling

目标：对突发请求赋予一定的优先处理的权利。
什么是突发请求？连续的两个请求之间的时间差较大。因此，判断是否为突发请求，只要在接收到请求时比较上个请求和当前时间之间的差值即可。对突发的请求，在计算P tag时采用如下公式即可达到优先处理的目的。

![](http://ohn764ue3.bkt.clouddn.com/DmClock/paper/Eq_04.png)

其中，sigma代表优先的力度，其值越大被优先处理的概率越大。注意由于只调整了P tag的赋值情况，因而不会对R tag构成影响。

![](http://ohn764ue3.bkt.clouddn.com/DmClock/paper/burst_01.png)

举个例子，如上图所示。

### 请求类型 Request Type

无差别对待读请求和写请求。不区分读写请求的原因是，如果调整了读写请求的顺序将出现一致性问题。

### IO大小 IO Size

不同大小的IO的延迟不同，一般来说，大小越大的IO延迟也越大。因此可以设置一个参考IO，对其它大小的IO设置tag时，在参考IO的基础上增大或减少一定的比例。这个比例可以根据公式计算出来。

### 请求局部性 Request Location

IO请求具有一定的空间局部性，为提高整体的处理性能，mClock允许批量地处理一组来自同个VM的请求。只要这些请求的总大小接近逻辑块(logical block number space)大小，例如4MB。同时，设定一次批量处理中请求的数目，例如最多处理8个。

上述的优化方法会影响IO的时延。举个例子，本来按照排序是第N个进行处理的请求，由于批量处理前面N个请求（属于其它VM）都各自夹带了8个相邻的请求，从而实际上将自己排到了第8N的位置。

# dmClock算法

不同于单主机的情况，给定VM的请求全部经由该主机进行处理，多主机的情况中给定VM的请求将分散给不同的主机进行处理。因此，在Tag赋值阶段，处理请求的主机并不知道VM的前一个请求的Tag值，只知道本主机处理的来自该VM的前个请求的Tag值。为解决这个问题，VM向目标主机发送请求时需要携带相关信息。目标主机根据这些信息来分配Tag的大小。

![](http://ohn764ue3.bkt.clouddn.com/DmClock/paper/Eq_5.png)

对给定的VM，delta代表处理当前请求的存储主机从接收到前个请求到当前请求这段时间内，VM发送给其它主机的请求数目。同样地，rou代表对给定的VM，处理当前请求的目标主机从接收到前个请求到当前请求这段时间内，VM发送给其它主机并且在constraint-based阶段被处理的请求的个数。

![](http://ohn764ue3.bkt.clouddn.com/DmClock/paper/dmclock_01.png)

举个例子，如上图所示，对Server3而言，只在T1、T2两个时间点接收到请求。在T2时刻时，delta值为Server1和Server2在T1、T2时间段中接收的请求数目，为8。rou为在T1、T2时间段内Server1和Server2中在constraint-based阶段处理掉的请求个数，为3。
