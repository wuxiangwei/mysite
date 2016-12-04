title: "Ceph线程池" 
date: 2014-08-04 20:23:29 
categories: [Ceph]
tags: [Ceph]
---

线程池ThreadPool的实现符合*生产者-消费者*模型，这个模型解除生产者消费者间的耦合关系，生产者可以专注处理制造产品的逻辑而不用关心产品的消费，消费者亦然。当然，生产者消费者之间需要一个连接的纽带，那就是产品接口。产品接口是对这两者的约束，生产者生产的产品要符合产品的接口，消费者依据产品接口来消费。

<!--more-->

Thread类是ThreadPool中的消费者，它封装*pthread* API函数，对外提供*Thread::entry()*作为线程的入口函数。Thread只是对消费者的抽象，WorkThread才是ThreadPool的具体消费者，它实现Thread接口函数，设置线程的入口函数为ThreadPool的*worker()*方法。

WorkQueue\_类在ThreadPool中代表产品接口，对外表现为一个队列，队列中存储待处理的数据元素。消费者WorkThread对它的处理主要分成三个步骤：首先调用*\_void\_dequeue()*方法获取队列元素，然后通过*\_void\_process()*方法处理元素，最后使用*\_void\_process\_finish()*方法进行收尾工作。具体调用过程可以参看WorkTread的线程入口函数，也就是*ThreadPool::work()*方法。

```C
template<typename T, typename U=T>
class WorkQueueVal: public WorkQueue_{
    virtual void _enqueue(T) = 0;
    virtual U _dequeue() = 0;    // 向WorkQueueVal加入的元素的类型为T，而读出的元素的类型为U。
};
```

WorkQueueVal<T,U>是WorkQueue的纯虚子类。它包含两个列表，一个为*to_process*用于保存待处理的元素，另一个为*to_finish*用于保存已经处理过的元素。在*\_void_dequeue()*时从其子类中取出元素放入*to_process*列表，以便在*\_void\_process()*时从中取出元素进行处理，并把处理后的元素放入*to_finish*列表。最后，在ThreadWork线程调用*\_void_process_finish()*函数时，从*to_finish*列表中取出元素做最后的处理。此外，需要注意的是WorkQueueVal<T, U>添加元素时使用的类型是T，而读取元素时的类型变成了U，需要子类自己负责转换。或许，这就是WorkQueueVal<T, U>类的设计目的。

