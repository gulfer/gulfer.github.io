# AbstractQueuedSynchronizer简介

AbstractQueueSynchronizer(AQS)是Java自1.5版本新增的一个基类，字面意思是队列同步器，简单来说是对线程建同步机制进行的抽象，是JDK中构建的主要基类。查看其继承关系，了解到JDK concurrent包下许多类都是基于此基类实现线程同步，理解AQS将帮助开发者理解concurrent包及线程同步机制。

## 线程同步

我理解线程同步就是多线程并发下竞争共享资源的使用权时按某些策略进行排队，最终达到共享资源的有序使用。顺带提一句，线程通信和线程同步是两个不同范畴的概念，通信是通过某种方式传递消息，对Java来说比如wait、notify、共享变量等；而线程同步是针对并发编程中资源竞争而言。那么怎么做到共享资源的有序使用，常见的做法就是加锁，这就用到AQS了。

## AQS原理及源码分析

从类名就可以看出来这是个基类，而且有队列，是同步用的，大神给类起名字都有水平。

我们先简单看看AQS的使用。像我们比较常用的ReentrantLock、CountDownLatch等都使用了AQS，但并不是通过直接继承的方式，而是用静态内部类继承AQS，而在类中用聚合的方式来使用同步器的能力。这样做符合OOP多聚合、少继承的原则，为实现提供了更大的灵活性，其实就是典型的对象适配器。

在回到AQS的实现原理上。AQS主要基于两个东西构建同步功能，一是同步状态，一个是同步队列。

AQS用一个volatile int类型的变量表示同步状态，当AQS子类调用acquire或release方法时，需传入一个int类型的形参，表示请求获取某种同步状态：

```
public final void acquire(int arg) {
        if (!tryAcquire(arg) &&
            acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
            selfInterrupt();
    }
......
protected boolean tryAcquire(int arg) {
        throw new UnsupportedOperationException();
    }
```
acquire是以排他模式获取同步状态的方法，对应的共享模式方法为acquireShared。此处子类可以选择是否重写tryAcquire方法，不直接使用abstract修饰是由于获取同步状态存在排他与共享两种模式，基类提供的默认实现会抛出UnsupportedOperationException，若子类需要采用某种模式则自行重写。可以看到未能获取状态且

可以看到队列

## AQS使用示例

下面是一个使用AQS的例子



