title: Ceph的Paxos算法实现
date: 2014-10-08 20:07:09
categories: [Ceph]
tags: [Ceph, Paxos]
toc: true
---

Ceph使用Paxos算法来解决Monitor间数据的一致性问题。本文介绍的内容位于Leader选举之后，被推选为Leader的Montior节点从其它节点中收集状态信息，统一Quoruma成员的状态，然后进入正常工作的流程。

<!--more-->

##Recovery阶段

在Leader选举成功后，Leader和Peon都进入Recovery阶段。该阶段的目的是为了保证新Quorum的所有成员状态一致，这些状态包括：最后一个批准(Committed)的提案，最后一个没批准的提案，最后一个接受(Acceppted)的提案。每个节点的这些状态都持久化到磁盘。对旧Quorum的所有成员来说，最后一个通过的提案应该都是相同的，但对不属于旧Quorum的成员来说，它的最后一个通过的提案是落后的。

![](http://images.cnitblog.com/blog/571795/201410/081934320306301.jpg)

同步已批准提案的方法是，Leader先向新Quorum的所有Peon节点发送OP_COLLECT消息，并在消息中携带Leader自己的第1个和最后1个批准的提案的值的版本号。Peon收到OP_COLLECT消息后，将自己的第1个和最后1个批准的提案的值的版本号返回给Leader，并且如果Peon的最后1个批准的版本号大于Leader最后一个批准的版本号时，将所有在大于Leader最后一个版本号的提案值发送给Leader。Leader将根据这些信息填补自己错过的提案。这样，当Leader接收到所有Peon对OP_COLLECT消息的回应后，也就将自己更新到了最新的状态。这时Leader又反过来将最新状态同步到其它节点。


为获取新Quorum所有成员中的最大提案号，Leader在发送OP_COLLECT消息时，提出它知道的最大的提案号，并将该提案号附加在OP_COLLECT消息中。如果Peon已接受的最大提案号大于Leader提出的提案号，则拒绝接受Leader提出的提案号并将自己已接受的最大提案号通过OP_LAST消息发送给Leader。Leader收到OP_LAST消息后，发现自己的提案号不是最大时，就在Peon接受的最大提案号的基础上提出更大的提案号，重新进入Recovery阶段。这样，最终可以获取到最大的提案号。

总而言之，Recovery阶段的目的是让新Quorum中所有节点处于一致状态。实现这一目的的方法分成两步：首先，在Leader节点收集所有节点的状态，通过比较得到最新状态；然后，Leader将最新状态同步给其它节点。有两个比较重要的状态，最后一次批准的提案和已接受的最大提案号。

**注意** 区分提案号(proposal number)、提案值(value)、提案值的版本号(value version)这三个概念。提案号由Leader提出，为避免不同Leader提出的提案号不冲突，同个Leader提出的提案号是不连续的。提案的值的版本号是连续的。

代码注释
```
函数
void Paxos::leader_init()
void Paxos::peon_init()
void Paxos::collect(version_t oldpn)
void Paxos::handle_collect(MMonPaxos *collect)
void Paxos::handle_last(MMonPaxos *last)
void Paxos::handle_commit(MMonPaxos *commit)
Paxos属性
uncommitted_v、uncommitted_pn、uncommitted_value
last_committed、accepted_pn
配置
OPTION(mon_lease, OPT_FLOAT, 5)     // Lease租期
```

## Lease阶段

Paxos算法分成两个阶段，第一个阶段为Prepare阶段。在这阶段中，(a)Proposer选择它知道的**最大**提案号n，并向所有Acceptor发送Prepare消息。(b)Acceptor承诺不再接受编号小于n的提案，(c)并返回它**接受的编号小于n的提案中编号最大的提案**给Proposer。这个过程中，如果Proposer选择的不是最大的提案号，那么Acceptor将拒绝Proposer的提案，而Proposer遭到拒绝后会提出编号更大的提案。这样循环反复，Proposer最终可以提出编号**最大**的提案。另外，Acceptor返回**接受的编号小于n的提案中编号最大的提案**给Proposer的目的是为让Proposer决定新提出的提案的值。对Ceph而言，由于Leader可以控制提案的进度，运行一次Paxos算法只有一个提案在审批，每次算法Leader都能够由自己决定提案的值，所以Peon不必返回**接受的编号小于n的提案中编号最大的提案**。

Ceph中Paxos算法的实现，省略了Prepare阶段，并且Leader选举成功后每次执行算法使用同一个提案号。在Prepare阶段要完成(a)、(b)和(c)三件事，前两件事在Recovery阶段完成，Leader和Peon的**已接受的最大提案号**保持相同。最后一件事情，由于Leader的存在不需要做。 

Paxos算法的第二阶段为Accept阶段。在这个阶段中，(d)Proposer根据在Prepare阶段中学习到的知识提出提案。(e)Acceptor根据接受到的提案的提案号决定拒绝还是接受。最后，(f)Proposer根据反馈情况决定提案是否得到批准。对Ceph来说，每次算法只有一个提案所以可以直接决定提案的值，因此不必关心(d)。对(e)和(f)的实现和标准Paxos算法保持一致。

![](http://7vzu17.com1.z0.glb.clouddn.com/2014/10/08/ceph_paxos_02.jpg)

Ceph的Paxos存在如下几个状态：                 
1) Recovery状态：Leader选举结束后进入该状态。该状态的目的是同步Quorum成员间的状态；            
2) Active状态：即空闲状态，没有执行Paxos算法审批提案；              
3) Updating状态：正在执行Paxos算法审批提案；          
4) Updating Previous状态：正在执行Paxos算法审批旧提案，旧提案即Leader选举之前旧Leader提出但尚未批准的提案。



代码注释
```
函数
void Paxos::begin(bufferlist& v)
void handle_begin(MMonPaxos *begin)
void handle_accept(MMonPaxos *accept)
void commit()
void handle_commit(MMonPaxos *commit)
Paxos属性
proposals //提案列表
accepted  // 接受(accept)新提案的节点，包括Leader和Peon
new_value
```

