# RateLimter解析

服务限流、流量整形都是当下挺热门的话题，什么令牌桶、漏桶算法，再加上git上挺多重复的轮子。其实限流的基本算法并没有那么复杂，关键在于具体场景的使用权衡。本文将对Guava提供的限流工具RateLimiter进行源码解析，这也是当前应用较为广泛的限流工具。其实现了基本的令牌桶和漏桶功能，之所以说是基本的功能，是因为RateLimiter只实现了单桶单速，没有双桶或双速。

网上令牌桶算法的介绍很多，可以参考Wiki上的介绍：[Token bucket](https://en.wikipedia.org/wiki/Token_bucket)，这里推荐一篇比较言简意赅的文章：[揭秘令牌桶](http://www.cnblogs.com/foonsun/p/5687978.html)

在解析RateLimiter之前，有必要聊一聊目前常见的限流思路。我们在项目中实现过一个粗糙的流量控制单元，这个功能是基于计数器实现的。大致原理是为每个服务创建一个计数器，每次接收到服务请求时计数，当计数值达到预定的阈值，则抛出异常并中断请求，同时使用单独的清理线程定时对计数器进行重置。而阈值可以是请求数、成功数或失败数，比如Hystrix。这种方式的好处在于容易实现，缺点在于中断方式有些粗暴，无法应对那些正常的突发流量暴涨。另一种就是上面提到的漏桶算法，简单说就是将不确定速率的请求转换为确定速率的请求，并在确定速率下对请求进行处理，可以借这种场景进行理解：设置一个队列，我们不用考虑请求入队的速率，只用固定的速率出队处理，而队列溢出则丢弃，大概是这意思。漏桶算法也可以有效实现限流，并且实现也很简单，但是与计数器相同，仍然无法处理突发情况。再有就是令牌桶算法，其算法核心思想是可变速率，这个速率的上限是通过桶中令牌数量确定的，令牌桶允许请求数量突发性的暴涨，但暴涨将预支后续的吞吐量。漏桶和令牌桶都是基于QPS的流控方式，而计数器是基于请求阈值的。

## 使用方法

RateLimiter的使用方法比较简单。先通过一个确定的QPS数值创建RateLimiter实例，每次执行操作前获取一次令牌。

```
RateLimiter r = RateLimiter.create(1d);
ExecutorService service = Executors.newFixedThreadPool(10);
for (int i=0;i<100;i++) {
	r.acquire();
	service.submit(new Task(i));
}
```

上面的代码使用create方法创建了QPS为1的限流器实例（即每秒生成一个令牌），在执行具体任务时需调用acquire方法获取令牌。acquire方法是一个阻塞方法，在未获取到令牌时无法执行后续操作，也可以使用tryAcquire实现非阻塞，立即返回true或false。

RateLimiter包含两个重载的acquire方法，上例中的方法是无参数的，默认获取一个令牌，另一个带参数的，参数获取令牌的个数。这个方法在具体使用时也非常有用，比如在根据处理字节数进行流控时，可表示每次允许处理的字节数。

## 内部实现

RateLimiter本身是个抽象类，查看继承关系可以看到其子类SmoothRateLimiter，这个类的子类又有SmoothBursty和SmoothWarmingUp。create方法分为两类，一类是不设置预热期的，另一类设置了预热期，前者在具体创建RateLimiter实例时使用的是SmoothBursty，后者是SmoothWarmingUp（平滑预热）。预热功能我们将在下一小节进行分析，本节主要关注SmoothBursty（平滑爆发）。

令牌桶的关键点实际上是两块，一是生成令牌，二是使用令牌，我们将分别解析这RateLimiter是如何执行这两个操作。

回到create方法，当我们设置QPS之后，create方法调用其重载方法：

```
public static RateLimiter create(double permitsPerSecond) {
    return create(SleepingStopwatch.createFromSystemTimer(), permitsPerSecond);
  }
  
static RateLimiter create(SleepingStopwatch stopwatch, double permitsPerSecond) {
    RateLimiter rateLimiter = new SmoothBursty(stopwatch, 1.0 /* maxBurstSeconds */);
    rateLimiter.setRate(permitsPerSecond);
    return rateLimiter;
  }

```
SmoothBursty的构造方法包含两个参数，其中SleepingStopwatch参数是一个睡眠闹钟，其内部实现使用了Guava的工具类Uninterruptibles，简单来说就是对sleep操作进行了封装，使当前操作在一定时间内保持阻塞。另一个参数是最大爆发时间，固定为1秒，表示令牌的最大保存时间。创建SmoothBursty实例后，需要对其设置QPS，这是通过抽象方法doSetRate实现的。我们在RateLimiter的子类SmoothRateLimiter中看到doSetRate的实现。

```
final void doSetRate(double permitsPerSecond, long nowMicros) {
    resync(nowMicros);
    double stableIntervalMicros = SECONDS.toMicros(1L) / permitsPerSecond;
    this.stableIntervalMicros = stableIntervalMicros;
    doSetRate(permitsPerSecond, stableIntervalMicros);
  }
```

在这里需要插播一下SmoothRateLimiter中的几个共享变量：

```
double storedPermits			当前存储的令牌（直译为许可）
double maxPermits				存储令牌的最大数量
double stableIntervalMicros		生成令牌的间隔时间（微秒）
double nextFreeTicketMicros		上一个令牌的生成时间（微秒）
```
SoomthRateLimiter在进行QPS设置时需对上述变量进行计算并赋值，需要注意的是这里的间隔时间、生成时间均为相对时间，均是SleepingStopwatch启动后的相对时间：

```
final void doSetRate(double permitsPerSecond, long nowMicros) {
    resync(nowMicros);
    double stableIntervalMicros = SECONDS.toMicros(1L) / permitsPerSecond;
    this.stableIntervalMicros = stableIntervalMicros;
    doSetRate(permitsPerSecond, stableIntervalMicros);
  }
  
...

private void resync(long nowMicros) {
    // if nextFreeTicket is in the past, resync to now
    if (nowMicros > nextFreeTicketMicros) {
      storedPermits = min(maxPermits,
          storedPermits + (nowMicros - nextFreeTicketMicros) / stableIntervalMicros);
      nextFreeTicketMicros = nowMicros;
    }
  }

```
在设置QPS前，需要调用resync方法对当前微秒值进行令牌添加，这里的入参即为SleepingStopwatch当前已阻塞的微秒数。resync方法的逻辑是：如果当前已阻塞时间大于上一次获取令牌的时间，将已存储的令牌增加（当前阻塞时间 - 距上一次令牌生成的时间）/令牌生成间隔，比如如果阻塞时间已达到1000微秒，上一次令牌生成时的阻塞时间为500微秒，令牌生成间隔为100微秒（QPS=10），则新生成5个令牌。最后再将上次令牌生成时间更新为当前阻塞时间。resync不仅在setRate时用到，在调用aquire方法获取令牌时也会用到，也就是说令牌的生成其实是在变更QPS设定和获取令牌时进行的，这样的好处在于不需要单独维护线程去生成令牌。

补充完令牌后，再设置生成令牌的间隔时间，公式就是1秒/QPS，再转成微秒数。而当前存储的令牌数和最大存储令牌数的计算方式则继续通过抽象方法由子类实现。SmoothBursty中的计算方式为，最大存储令牌数为爆发时间（上面提到的默认1秒）*QPS，当前存储的令牌数在初始化时为0.0。

看完令牌的生成过程，我们再看一下使用令牌的过程，也就是aquire方法。

```
public double acquire() {
    return acquire(1);
  }
  
public double acquire(int permits) {
    long microsToWait = reserve(permits);
    stopwatch.sleepMicrosUninterruptibly(microsToWait);
    return 1.0 * microsToWait / SECONDS.toMicros(1L);
  }
```
acquire方法中，先调用reserve方法预订所需的令牌个数，返回的mirosToWait是获取这些令牌所需等待的时间，然后使用SleepingStopwatch阻塞这么长的时间，最后返回阻塞的时长。

```
final long reserve(int permits) {
    checkPermits(permits);
    synchronized (mutex()) {
      return reserveAndGetWaitLength(permits, stopwatch.readMicros());
    }
  }
```
在执行reserve方法时，需对真正获取令牌的方法reserveAndGetWaitLength进行同步。

```
final long reserveAndGetWaitLength(int permits, long nowMicros) {
    long momentAvailable = reserveEarliestAvailable(permits, nowMicros);
    return max(momentAvailable - nowMicros, 0);
  }

final long reserveEarliestAvailable(int requiredPermits, long nowMicros) {
    resync(nowMicros);
    long returnValue = nextFreeTicketMicros;
    double storedPermitsToSpend = min(requiredPermits, this.storedPermits);
    double freshPermits = requiredPermits - storedPermitsToSpend;

    long waitMicros = storedPermitsToWaitTime(this.storedPermits, storedPermitsToSpend)
        + (long) (freshPermits * stableIntervalMicros);

    this.nextFreeTicketMicros = nextFreeTicketMicros + waitMicros;
    this.storedPermits -= storedPermitsToSpend;
    return returnValue;
  }
```
这里我们可以看到最终获取令牌的方法是SmoothRateLimiter类中的reserveEarliestAvailable方法。此方法一上来仍然调用resync方法生成令牌，也就是说setRate和获取令牌时会生成令牌；第二步是取到所需令牌和当前存储令牌的较小值和差值；第三步是计算等待生成令牌所需的时间；最后更新上一次令牌的等待时间以及已存储的令牌数。解释一下其中计算等待时间的算法：storedPermitsToWaitTime方法在Smooth
Bursty类中直接返回为0，因此只计算令牌差值和每个令牌生成间隔的积就可知一共需等待的时长。在SmoothRateLimiter的另一个子类SmoothWarmingUp中，storedPermitsToWaitTime方法实现相对复杂，后面再说。

至此aquire方法完成，**总的来说RateLimiter就是通过令牌数量计算阻塞时间，从而达到流量控制**。

## 预热

SmoothWarmingUp提供了预热机制，在调用RateLimiter的create方法时允许设置一个预热时间，在预热时间

## Hystrix的熔断与流控

提到流控我们往往会想到Hystrix，不过以我的理解，Hystrix和流控的关注点是不同层面的，解决的问题也有所差异。Hystrix主要是做熔断的，在服务化体系中用来解决雪崩效应，是用比较直接的方式进行资源的隔离，从而保护整体系统。而流控虽然也是保护系统资源，但是使用的是比较柔和的方式，不会中断某个服务，而是通过阻塞进行缓冲。而且Hystrix实现熔断的方式是基于计数器，并不是基于某些桶算法。

不过Hystrix除了熔断机制本身，还有很多值得学习的设计思路，比如RxJava的使用、命令模式的使用等等，后续将再写一篇作业进行分析。

## 小结

总结一下，本文通过对RateLimiter代码进行简单的解析，介绍了令牌桶的基本实现原理，同时比较了流控与Hystrix熔断的差别。RateLimiter适用于处理处理存在突发流量暴涨的限流场景，但由于并不是企业级产品，在分布式环境下，还是需要基于分布式架构设计的工具，比如redis来实现流控。说点题外话，这次去学习RateLimiter，还是在项目中遇到了类似的问题，想借鉴其设计思路。我想说的是，读代码还是要带着问题去读，要参照某个场景去思考，另外就是先理解原理再读代码，还是先读代码再从中提取理论，我倾向前者。



