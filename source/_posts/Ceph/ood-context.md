---
title: Context设计模式
date: 2014-09-12 09:05:45
categories: [Ceph]
tags: [OOD, Ceph]
---

Ceph实现中使用了大量派生于Context抽象类的子类，用法简单却很巧妙，我将它称为Context模式或者上下文模式。上下文模式像极了C的回调函数，而回调函数又像极了Template模式，但Template模式和Context模式似乎有所不同。

<!--more-->

回调函数的使用场景是，函数的实现者不知道何时调用函数，而函数的调用者不知道如何实现函数。回调的实现者实现由调用者提供的函数接口，并将其注册给调用者。调用者在适当的时机调用函数完成实现者指定的功能。

从“你不用调用我，让我调用你”的角度看，Template模式是ODD中的回调函数。Template模式中，父类充当回调中的调用者角色，子类充当实现者角色。一般地，父类定义了程序的主要逻辑功能，而子类只用于填充逻辑中不稳定或者可以多样化的部分，这部分的内容通过多态机制来调用。

Template模式中一个很重要的因素是，**调用者父类和实现者子类共享同一个上下文信息**。但是，如果父类是重量级的并且子类实现回调接口使用的上下文内容同父类的上下文风马牛不相及时，就有必要将二者分离。回头看，C的回调和Template模式之间的区别：**C回调的调用者和实现者是属于两个实体的，而Template模式将实现者和调用者放到了一个实体中。Context模式又从Template模式中将这两者分离开来**。

```C++
class Context {
protected:
  virtual void finish(int r) = 0; /// 派生类实现接口

public:
  virtual void complete(int r) {  /// 调用者调用接口，函数执行结束后销毁对象
    finish(r); /// r用于让子类判断是否执行，r为0时执行，不为0时放弃执行
    delete this;
  }
};
```
从上文的分析可知，Context模式和Template模式的区别是，调用者和实现者是否共享上下文信息。Context类将实现者的上下文信息封装在子类中，提供给调用者*void complete(int r)*方法并且授予其是否执行回调的权利。因为将实现者的上下文信息封装在子类，因此可以不必像C回调那样设计各种不同的回调函数类型，从而统一了所有的回调接口。

```C++
class C_SafeCond : public Context {
  bool *done;     /// true after finish() has been called
public:
  C_SafeCond(bool *d ) : done(d) { /// 主要这里引用的是done的地址
    *done = false;
  }
  void finish(int r) { /// 执行complete后，设置done为true。
    *done = true;
  }
}

/// 从远程OSD中同步读取Object数据
int librados::IoCtxImpl::operate_read(const object_t& oid, ::ObjectOperation *o, bufferlist *pbl, int flags)
{
  bool done;

  Context *onack = new C_SafeCond(&mylock, &cond, &done, &r);

  Objecter::Op *objecter_op = objecter->prepare_read_op(oid, oloc, *o, snap_seq, pbl, flags, onack, &ver);
  objecter->op_submit(objecter_op); /// 发送读object请求，收到对该请求的回复后，执行onack回调

  while (!done){  /// 线程阻塞直到onack被执行
    cond.Wait(mylock);
  }

  return r;
}
```
通常情况下，Context模式适用于异步执行环境。但上面的例子说明，它也可以用于实现同步等待。

