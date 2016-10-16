# AbstractQueuedSynchronizer简介

AbstractQueueSynchronizer(AQS)是Java自1.5版本新增的一个基类，语义是队列同步器，简单来说是对线程建同步机制进行的抽象，是JDK中构建的主要基类。查看其继承关系，可发现JDK concurrent包下许多类都是基于此基类实现线程同步，理解AQS将帮助开发者理解concurrent包及线程同步机制。

## 线程同步

我理解线程同步就是多线程并发下竞争共享资源的使用权时按某些策略进行排队，最终达到共享资源的有序使用。顺带提一句，线程通信和线程同步是两个不同范畴的概念，通信是通过某种方式传递消息；而线程同步是针对并发编程中资源竞争而言。那么怎么做到共享资源的有序使用，这就用到AQS了。

## AQS原理及源码分析

从类名就可以看出来这是个基类，而且有队列，是同步用的，大神给类起名字都有水平。

我们先简单看看AQS的使用。像我们比较常用的ReentrantLock、CountDownLatch等都使用了AQS，但并不是通过直接继承的方式，而是用静态内部类继承AQS，而在类中用聚合的方式来使用同步器的能力。这样做一来符合OOP多聚合、少继承的原则，为实现提供了更大的灵活性，其实就是典型的对象适配器，二来在一定程度上隐藏了基类方法，避免错误重写。

在回到AQS的实现原理上。AQS主要基于两个东西构建同步功能，一是同步状态，一个是同步队列。

先说说同步队列的实现。这里的其实使用的是一个FIFO的双向链表来构建的队列，每个队列元素都是一个Node对象，Node类主要属性及方法如下：

```
static final class Node {

        ......

        // 等待状态
        volatile int waitStatus;

        // 前置节点
        volatile Node prev;
        
        // 后置节点
        volatile Node next;

        // 等待线程
        volatile Thread thread;

        // 等待队列的后置节点
        Node nextWaiter;
        
        // 获取前置节点
        final Node predecessor() throws NullPointerException {
            ......
        }

        ......
    }
```

waitStatus的取值范围，在JDK的doc中有描述。同步队列的基本逻辑是，当线程获取同步状态失败时，AQS会将Thread对象及等待状态等作为参数创建一个Node对象，然后把这个Node挂在同步队列队尾，表示增加了一个等待线程，同时阻塞此线程。这部分逻辑并不复杂，需要注意的是这里都是使用CAS保证队列操作的原子性。这部分逻辑包含在acquire模板方法中。

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
acquire是以排他模式获取同步状态的方法，对应的共享模式方法为acquireShared。此处子类可以选择是否重写tryAcquire方法，不直接使用abstract修饰是由于获取同步状态存在排他与共享两种模式，基类提供的默认实现会抛出UnsupportedOperationException，若子类需要采用某种模式则自行重写。若成功获取同步状态则tryAquire方法返回true。addWaiter入队之后需要再次尝试获取同步状态，即进入acquireQueue方法。

```
final boolean acquireQueued(final Node node, int arg) {
        boolean failed = true;
        try {
            boolean interrupted = false;
            for (;;) {
                final Node p = node.predecessor();
                if (p == head && tryAcquire(arg)) {
                    setHead(node);
                    p.next = null; // help GC
                    failed = false;
                    return interrupted;
                }
                if (shouldParkAfterFailedAcquire(p, node) &&
                    parkAndCheckInterrupt())
                    interrupted = true;
            }
        } finally {
            if (failed)
                cancelAcquire(node);
        }
    }
```
此方法使用自旋来不断尝试获取同步状态，当当前节点的prev节点是head节点时才去尝试获取同步状态，原因在于head节点一定是获取同步状态的节点，后续节点必须检查prev节点是否是head节点，符合FIFO规则，不需要引入额外的线程通信处理。

AQS用一个volatile int类型的变量表示同步状态，当AQS子类调用acquire或release方法时，需传入一个int类型的形参，表示请求获取某种同步状态，这个状态值并没有特定的要求，是根据子类的实际需求来决定的，即获取同步状态的逻辑也是有子类实现，一般来说，这个int型的状态值表示的是待请求的资源数。

上面只简单分析了排他模式下的情况，共享模式与排他模式的区别在于同一时刻有多个线程可以获取同步状态。tryAquireShared方法有一个int型的返回值，表示剩余的可获取资源数，小于0时表示获取失败。此外排他模式还有一种形式，及排他超时模式，如果等待超时则直接返回false。在自己使用AQS实现同步器时，需要首先确定是排他模式还是共享模式，根据不同的模式重写相应的tryAcquireXXX及releaseAcquireXXX等方法。


## AQS使用示例

看看使用AQS的例子，典型的就是ReentrantLock中的NonfairSync，即默认的非公平锁。lock方法：

```
final void lock() {
            if (compareAndSetState(0, 1))
                setExclusiveOwnerThread(Thread.currentThread());
            else
                acquire(1);
        }

protected final boolean tryAcquire(int acquires) {
            return nonfairTryAcquire(acquires);
        }
```

再看核心的获取同步状态的方法：

```
final boolean nonfairTryAcquire(int acquires) {
            final Thread current = Thread.currentThread();
            int c = getState();
            if (c == 0) {
                if (compareAndSetState(0, acquires)) {
                    setExclusiveOwnerThread(current);
                    return true;
                }
            }
            else if (current == getExclusiveOwnerThread()) {
                int nextc = c + acquires;
                if (nextc < 0) // overflow
                    throw new Error("Maximum lock count exceeded");
                setState(nextc);
                return true;
            }
            return false;
        }
```
理解了AQS后再看这部分逻辑就很清楚了，初始状态时使用CAS更新同步状态，并设置当前占有锁的线程，如果此线程再次请求锁，则直接对同步状态做加法，实现重入。

而CountDownLatch是另一种做法，首先是共享模式，然后是可中断：

```
public void await() throws InterruptedException {
        sync.acquireSharedInterruptibly(1);
    }
```
剩下的代码应该就不需要解释了吧，基本上一目了然：

```
// 初始化同步状态值
public CountDownLatch(int count) {
        if (count < 0) throw new IllegalArgumentException("count < 0");
        this.sync = new Sync(count);
    }
......
protected int tryAcquireShared(int acquires) {
            return (getState() == 0) ? 1 : -1;
        }

protected boolean tryReleaseShared(int releases) {
            // Decrement count; signal when transition to zero
            for (;;) {
                int c = getState();
                if (c == 0)
                    return false;
                int nextc = c-1;
                if (compareAndSetState(c, nextc))
                    return nextc == 0;
            }
        }
```
所以AQS基本实现了大部分线程同步的核心抽象逻辑，用户关注同步工具本身的使用，如Lock、CountDownLatch，而AQS则为用户屏蔽了同步本身相关的操作，如同步状态和同步队列的管理等。理解了AQS也就大致理解了JDK中Lock、信号量等同步工具的工作原理。

本文只是对AQS的基本原理做了简单说明，其实其中值得学习的内容还很多，一些细节本人也没有完全理解，还需努力。如果想了解AQS更多详细内容，可参考一下内容（并发编程网还是很牛逼的）：

[AbstractQueuedSynchronizer的介绍和原理分析](http://ifeve.com/introduce-abstractqueuedsynchronizer/)

[深度解析Java8 – AbstractQueuedSynchronizer的实现分析（上）](http://ifeve.com/jdk1-8-abstractqueuedsynchronizer/)


