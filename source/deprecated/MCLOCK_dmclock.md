---
title: "dmClock源码分析之DMC模块"
date: 2016-11-17 15:01:26
categories: [mClock]
tags: [mClock]
toc: true
---

# Tag计算

Tag值的计算依赖于两类参数，一类是用户配置的上限(limit)、下限(reservation)以及权重(weight)，另一类是动态统计出来的delta和rho。根据dmClock算法的描述，delta和rho这两个参数由客户端来统计，并将其夹带到请求中发送给服务端。

<!--more-->

## 客户端

从客户端发送请求到请求到达服务端的内部队列的函数调用过程，后文记为**过程A**：
```
SimulatedClient::run_req() --> SimulatedClient::submit_f() --> SimulatedServer::post() --> 
PushPriorityQueue::add_request() --> PushPriorityQueue::schedule_request() --> 
PushPriorityQueue::submit_request() --> PushPriorityQueue::submit_top_request() --> 
PriorityQueueBase::pop_process_request() --> PushPriorityQueue::handle_f() --> SimulatedServer::inner_post()
```

从服务端发送响应到客户端接收到响应的函数调用过程，后文记为**过程B**：
```
SimulatedServer::run() --> SimulatedServer::client_resp_f() --> 
SimulatedClient::receive_response() --> SimulatedClient::run_resp() --> ServiceTracker::track_resp()
```

根据dmClock算法，发送给给定服务端的请求中携带的delta的定义为：
> 从最近一次发送给该服务端请求的时刻到当前时刻这段时间内，客户发送给其它服务端的请求的数目。

为计算delta，对任意一台服务端发送请求时记录下此时全局已发送的请求数目，计算delta时只要将当前的全局已发送请求数目减去最近一次发送时的全局已发送请求数目即可。算法实现中使用响应来近似请求，具体做法如下：

对给定的服务端，使用t1代表最近一次客户向其发送请求的时刻，t2代表当前时刻，即本次向其发送请求的时刻。算法实现中用到的关键数据结构：

``` C++
// 一个服务端的响应统计情况
struct ServerInfo {
    Counter delta_prev_req;  // t1时刻的全局响应数目
    Counter rho_prev_req;  // t1时刻的全局reservation响应数目
    uint32_t my_delta;  // >t1时间段内，属于本服务端的响应数目
    uint32_t my_rho;  // >t1的时间段内，属于本服务端的reservation响应数目
};

template<typename S>
class ServiceTracker {
    Counter delta_counter;  // 所有服务端的响应数目
    Counter rho_counter;  // 所有服务端的Reservation响应数目
    std::map<S, ServerInfo> server_map;  // 每个服务端独立的响应统计情况
};
```

**过程B**每接收到一个响应，递增全局的delta_counter以及特定服务端的my_delta。对rho也一样，详细过程参考ServiceTracker::track_resp函数。
**过程A**在准备请求阶段计算delta，公式如下：

``` C++
delta = 1 + delta_counter - it->second.delta_prev_req - it->second.my_delta
```

其中，it代表ServiceTracker::server_map的迭代器。计算delta时之所以减去my_delta，是因为在t1~t2这个时间段内除了能够接收到其它服务端的响应外还会接收到来自自己服务端的响应，因此要减去自己服务端的响应个数。

## 服务端

从客户端发送请求到请求到达服务端的优先级队列的函数调用过程：

```
SimulatedClient::run_req() --> SimulatedClient::submit_f() --> SimulatedServer::post() --> 
PushPriorityQueue::add_request() --> PushPriorityQueue::do_add_request()
```
在这过程中，请求被添加上了一个伪Tag。之所以假是因为Tag中除当前时间外其它的内容都为0。具体参看PushPriorityQueue::do_add_request函数。

从请求开始调度到请求送到服务端的内部队列的函数调用过程：
```
PushPriorityQueue::schedule_request() --> PushPriorityQueue::submit_request() -->
PushPriorityQueue::submit_top_request() --> PriorityQueueBase::pop_process_request() --> 
PushPriorityQueue::handle_f() --> SimulatedServer::inner_post()
```
PriorityQueueBase::pop_process_request开始计算请求的tag值，具体计算过程在RequestTag::tag_calc函数完成，计算过程和算法描述基本一致。注意，对第一个请求，RequestTag计算Tag时，依赖的前个请求为默认的prev_tag(0.0, 0.0, 0.0, TimeZero)，这样第一个请求的各个Tag值将为请求到达的时间。


### 数据结构

#### DMC客户

``` C++
// C: 客户ID
// R: 请求类型
// B: B叉堆，堆的分叉数
template<typename C, typename R, uint B>
class PriorityQueueBase {
     std::map<C,ClientRecRef> client_map;  // 每个客户的记录
};

class ClientRec {
    uint32_t cur_rho;
    uint32_t cur_delta;
    ClientInfo info;  // 包含limit、reservation、proportion参数及其倒数

    RequestTag  prev_tag;  // 前一个请求的Tag值
    std::deque<ClientReq> requests;  // 请求队列
};
```
服务端中每个客户即ClientRec类的一个实例，客户的请求保存到ClientRec::requests队列，该队列是FCFS队列。也就说，同个客户端的请求的相对顺序保持不变。dmClock改变的是不同客户端间的处理顺序。


#### 请求堆

``` C++
// I: 元素类型
// T: 元素类型 
// C: 用于比较两个T类型的参数大小的函数类
// K: 堆的分叉，K=2代表2叉堆
// heap_info: T的成员变量
template<typenmae I, typename T, IndIntruHeapData T::*heap_info, typename C, uint K=2>
class IndIntruHeap {
protected:
    std::vector<I> data;  // 存储元素的容器
    index_t count;  // 元素个数
    C comparator;  // 重建堆时，比较两元素的优先顺序
public:
    void push(const I& item);  // 加入新元素
    void adjust(T& item);

protected:
    // 从下至上重建堆
    void shift_up(index_t i);
    // 由上到下重建堆
    // 如果i为最后1个元素，不调整
    // 
    typename std::enable_if<(K>2)&&EnableBool,void>::type sift_down(index_t i);
    // 给定元素被修改后，调整其在堆中的位置
    // 如果i为根节点，则向下调整
    // 如果i大于其parent节点，则向上调整
    // 如果i小于其parent节点，则向下调整
    void sift(index_t i);
};
```
参考[浅谈算法和数据结构: 五 优先级队列与堆排序](http://www.cnblogs.com/yangecnu/archive/2014/03/02/3577568.html) 获取更多关于堆的内容。


``` C++
void IndIntruHeap::sift_up(index_t i) {
    // 入参i代表元素在IndIntruHeap::data中的位置（或者说，数组下标）
    while (i > 0) {
        index_t pi = parent(i);  // i的父节点位置
        if (!comparator(*data[i], *data[pi])) {
            // data[i] 优先级低于 data[pi]，保持相对结构不变
            break;
        }

        // data[i] 优先级低于 data[pi]，将data[i]上移
        // 因为pi的左节点的优先级低于pi，而pi的优先级低于i
        // 所以pi的左节点的优先级也低于i，i可以作为其父节点，不必再进行比较
        std::swap(data[i], data[pi]);
        // 修改数据的位置信息
        intru_data_of(data[i]) = i;
        intru_data_of(data[pi]) = pi;
        // data[i] 继续与其新的父节点进行比较
        i = pi;
   }
} 

// 此处的std::enable_if可以理解为返回值函数重载，实际上是模板偏特化
// 代码中实现了连个shift_down函数，用以区分K==2和K>2两种情况，这两种情况分别调用两个不同的函数
template<bool EnableBool=true>
typename std::enable_if<(K>2)&&EnableBool,void>::type IndIntruHeap::sift_down(index_t i) {
    // count为元素数目，i大于它为非法
    if (i >= count) return;
    while (true) {
        // li为i最左边的子节点
        index_t li = lhs(i);

        if (li < count) {
            // ri为i最右边的子节点
            index_t ri = std::min(rhs(i), count - 1);

            // 查找优先级最高的子节点
            index_t min_i = li;
            for (index_t k = li + 1; k <= ri; ++k) {
                if (comparator(*data[k], *data[min_i])) {
                    min_i = k;
                }
            }

            if (comparator(*data[min_i], *data[i])) {
                // i优先级低于子节点min_i，交换data[i]和data[min_i]的位置
                // 因为min_i已经是i所有子节点中优先级最高的，因此它可以作为其余子节点的父节点
                std::swap(data[i], data[min_i]);
                intru_data_of(data[i]) = i;
                intru_data_of(data[min_i]) = min_i;
                // i继续和新的子节点进行比较
                i = min_i;
            } else {
                // i节点比所有的子节点优先级都高，退出
                break;
            }
        } else {
            // i节点已是叶子节点
            break;
        }
    }
} // sift_down 
```

### 优先级比较

``` C++
// 请求的Tag，包含预留(reservation)、上限(limit)和权重(proportion)。
class RequestTag {
    double reservation;  // 预留
    double proportion;  // 权重
    double limit;  // 上限
    bool ready;  // 请求是否允许在Weight-based阶段被处理，当Limit低于当前时间（请求调度）
    Time arrival;  // 接收到请求的时间
};
```
请求调度的Weight-based阶段，先从Limit堆中筛选出能够在这个阶段被处理的请求，设置请求的ready字段为True。筛选的条件是请求的L Tag低于当前时间。筛选结束后再从Ready堆中选择请求。详情参考 PriorityQueueBase::do_next_request函数。

``` C++
// tag_field代表RequestTag类的一个成员变量
template<double RequestTag::*tag_field, ReadyOption ready_opt, bool use_prop_delta>
struct ClientCompare {
    bool operator()(const ClientRec& n1, const ClientRec& n2) const {
        // n1没有请求，n2有请求，返回false
        // n1没有请求，n2也没有请求，返回false，保持堆结构不变
        // n1有请求，n2没有请求，返回true
        // n1有请求，n1也有请求
    }
};
```
比较客户n1和客户n2的Next请求的优先级。根据mClock算法，Tag值越小优先级应该越高。该函数的本意是，判断“n1的Next请求的优先级高于n2？”。下文使用tag1和tag2分别表示n1和n2的Next请求的Tag值。

对**reservation堆**，依据“谁小谁优先级高”的原则，若tag1小于tag2，则返回True；
对**ready堆**，如果t1、t2两者的ready状态相同（都为true或者都为false），那么遵循“谁小谁优先级高”的原则；如果t1、t2的两者的ready状态不同（一个为true一个为false），那么依据“**谁ready谁优先级高**”的原则；
对**limit堆**，如果t1、t2两者的ready状态相同，那么遵循“谁小谁优先级高”的原则。如果两者的ready状态不同，按照“**谁没ready谁优先级高**”的原则，对已经Ready的请求将在请求调度中直接处理掉。

### Tag校准

根据mClock算法，当有**新VM**启动时所有**老VM**请求的P Tag都要向左偏移一个位置（即P Tag减去偏移量），以避免新VM请求偏低影响公平性。偏移量为所有老VM的剩余请求中最小的P Tag减去当前时间（新VM第一个请求的到达时间）。反过来思考，将所有老VM的剩余请求在P Tag时间轴上向左偏移和将新VM的请求向右偏移的结果是相同的，一样能够消除不公平性。但这在代码实现上就简单了，只要在新VM中记录偏移量，并在Tag比较时加上偏移量即可，不必对所有老VM的剩余请求大动干戈。

``` C++
template<typename C, typename R, uint B>
class PriorityQueueBase {
    c::IndIntruHeap<ClientRecRef, ClientRec, &ClientRec::reserv_heap_data,
                    ClientCompare<&RequestTag::reservation, ReadyOption::ignore, false>,
                    B> resv_heap;
    c::IndIntruHeap<ClientRecRef, ClientRec, &ClientRec::lim_heap_data,
                    ClientCompare<&RequestTag::limit, ReadyOption::lowers, false>,
                    B> limit_heap;
    c::IndIntruHeap<ClientRecRef, ClientRec, &ClientRec::ready_heap_data,
                    ClientCompare<&RequestTag::proportion, ReadyOption::raises, true>,
                    B> ready_heap;
};
```
算法实现中分两步完成Tag校准：

1. 接收请求时，检查Client是否空闲，若空闲则计算偏移量，并将偏移量保存到ClientRec::prop_delta变量。参考PriorityQueueBase::do_add_request函数的*if (client.idle)*分支；
2. 比较P Tag时， 为请求的P Tag加上Client的prop_delta后再比较。参考ClientCompare的operator()函数。

另外，注意只有P Tag需要校准。ClientCompare模板的最后一个参数代表是否使用prop_delta，只有ready_heap变量为True。

### 判断客户为空闲

客户是否为空闲，在Tag校准中非常重要，只有发现空闲的客户时才需要进行校准。以下是算法实现中，判断客户空闲的一些手段：
- 服务端第一次接收到来自该客户的请求，就认为该客户为空闲，参考 PriorityQueueBase::do_add_request函数
- 定时检查，将超过idle_age(10分钟)时间内没有新增请求的客户标记为idle, 参考 PriorityQueueBase::do_clean函数

对定时检查策略，除了将客户标记为idle状态外还会清理长久没有发送请求的客户。默认情况下，定时检查的时间间隔为6分钟，超过10分钟没新请求的客户将被标记为idle状态，超过15分钟没有新请求的客户将被从服务端清理。

### 请求调度

请求调度，从堆结构中获取待处理请求的函数调用过程：

```
PushPriorityQueue::schedule_request() --> PushPriorityQueue::next_request --> PriorityQueueBase::do_next_request()
```
DmClock算法的请求调度流程在PriorityQueueBase::do_next_request函数中实现。

**递减剩余请求的R Tag**
根据DmClock算法，在Weight-based阶段每处理一个请求时，为避免影响R tag的排序，需要将该客户的剩余请求的R tag值递减 1/ri。这个过程在函数PushPriorityQueue::submit_request()以及PriorityQueueBase::reduce_reservation_tags函数中实现。
