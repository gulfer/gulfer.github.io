# Paxos与分布式

在介绍Paxos之前，我们需要先搞清楚什么是分布式系统，分布式系统能解决什么样的问题，又会带来哪些问题，分布式系统是怎么解决这些问题的。在此仅以个人肤浅的见识抛砖引玉，大家共同探讨下。

## 分布式系统

Wiki上给出的分布式系统的解释是：“A distributed system is a model in which components located on networked computers communicate and coordinate their actions by passing messages. The components interact with each other in order to achieve a common goal. Three significant characteristics of distributed systems are: concurrency of components, lack of a global clock, and independent failure of components.” 

从上面的定义中可以发现分布式系统名字的由来：相对集中式系统而言，分布式系统中的各个节点（组件）在空间上是任意分布的，可以不同进程、不同机房、甚至不同地域，节点之间通过消息传递的方式进行通信和协调。

为什么要有分布式系统，其实从分布式系统的分类就可以看出来：分布式计算、分布式存储。。。当集中式的应用所依赖的计算资源、存储资源不再满足需求，自然而然需要额外的资源投入。《失控》九律的第一条就是分布式，KK认为分布式系统应是充分利用“集体智慧”的，也就是集体资源。

定义的后半段提到了分布式系统设计上的三个要点：

* 节点（组件）并发性

这里的并发性，应该不是指系统面临的并发量，而是指系统中各个节点并发的访问共享资源时，如何保证操作的效率

* 全局时钟缺失

分布式系统中的各个节点没有统一的全局时钟，这就带来如何保证某个操作序列在所有节点上都保持一个统一的执行顺序问题

* 组件失效

分布式系统中的节点可能出现任何形式的失效，可能是单个节点的失效也可能是多个节点失效，在系统设计时需要考虑到这些失效情况

分布式系统的确可以用于解决系统资源的问题，但同时也带来了新的问题。诚如上述三个要点所提到的，如何保证所有组件之间数据一致？如何在某些组件失效的情况下保证整个系统的可用性，这就引出了著名的CAP理论和BASE理论。关于这两个名词的解释本文不做赘述。

## 分布式一致性

在大致了解了分布式系统能够解决什么问题，又会带来什么问题之后，我们把关注点聚焦到一个最重要的问题上：在分布式系统中，我们对某个节点进行的操作，如何准确的同步到其他节点，也就是节点间的一致性问题，也就是分布式一致性问题。Paxos协议由此应运而生。

Paxos作者Leslie Lamport在论文中是这样描述一致性算法要求的：

Assume a collection of processes that can propose values. A consensus al- gorithm ensures that a single one among the proposed values is chosen. If no value is proposed, then no value should be chosen. If a value has been chosen, then processes should be able to learn the chosen value. The safety requirements for consensus are:• Only a value that has been proposed may be chosen,• Only a single value is chosen, and• A process never learns that a value has been chosen unless it actually has been.We won’t try to specify precise liveness requirements. However, the goal is to ensure that some proposed value is eventually chosen and, if a value has been chosen, then a process can eventually learn the value.

翻译过来就是：1、在对分布式进程赋值时，所赋的值必须是某个进程（节点）所提出（propose）的；2、只有一个值被接收（accept）并选择（chosen）；3、某个进程（节点）不会得知（learn）值的信息，除非值已被选择。之所以标注其中的某些单词，是因为Paxos将使用这些关键字作为其协议中重要的角色，后面会谈到。

在了解Paxos协议之前，我们自己会如何考虑分布式一致性方案呢？

我们在方案的执行过程中归纳出三类角色：当系统中某个节点收到操作指令时，需要作为提交者（Proposer）向其他节点提交提案（赋值），其他节点的角色就是接收者（Acceptor）。当然我们不可能保证所有接受者都能同时收到提案，因为有可能某个接受者进程宕掉了或因为网络原因丢包了，所以为了让这个提案生效，很自然会想到让大多数接受者批准这个提案就行了，最后是提案执行者（Learner）使提案生效了。

当多个Proposer同时发起对某个值的变更提案时怎么办？我们会想到对每个提案设置编号，这样才能通过编号大小来确定哪个提案应该被执行。这里有一点需要注意，对于某个值到底应该设置，我们关注的是尽快取得一致，而非是某个Proposer的提案一定要被执行。这样，Acceptor或者说大多数的Acceptor（Majority）只要认可更大的那个提案编号，就能够获得最终的一致性。这是我个人对Paxos最简单的理解，Paxos的过程是一个两阶段提交（2PC）的过程，两阶段提交的第一阶段提交是准备提案（Prepare），询问所有Acceptor是否能够接收提案，还是有编号更大的提案，如果有编号更大的提案，则继续执行准备提案，直到大多数Acceptor都反馈可以执行提案，再发起第二阶段提交即执行提案。而Learner角色相对简单，用于记录本次执行的提案。在重新准备提案的过程中，如果已经达到多数Acceptor同意执行某个提案，则该提案将被最终执行。

这就是Paxos协议的大致过程，总的来说，就是在某项操作相关的所有提案中，选择大多数节点反馈同意的编号最大的那一个的过程。

## Zookeeper与Paxos

需要说明的是，大多数的分布式应用虽然基于Paxos协议来保证一致性，但出于这些应用自身的实际情况，多少都对Paxos做了精简或变形。Zookeeper使用的一致性算法ZAB，就是Paxos的一种变形算法。

Zookeeper在设计上设置了Leader这样一个特殊的角色，这个角色属于分布式环境下的某一个节点，并且规定：只有Leader能发起提案，且先到Leader节点的提案将先执行，这样就保证了操作的执行顺序。而由于Leader可能存在宕机或网络中断的情况，因此需要对Leader进行选举，选举过程也用到了ZAB算法。相对的，其他节点将被设置为Follower角色。需要注意的是选举出来的Leader，必须包含全部已经执行提案的日志。

Zookeeper的基本用法这里不做详细介绍了，只说明他是如何应用ZAB算法的。我们先对号入座，了解下Zookeeper各个组件对应Paxos的哪些角色：

Zookeeper集群（伪集群）中的各个Server原本应该对应Paxos中的Proposer，但上面提到Zookeeper要求所有提案都需要由Leader发起，这样其实真正的Proposer就只有一个了，就是Leader，而各个Follower实际上只承担Acceptor和Learner的角色。提案就是我们对Zookeeper发送的各种操作指令，这些指令都有一个唯一的编号，就是zxid，我们可以在Zookeeper CLP中查看每个节点的zxid。当各个Server发现无法连接到Leader时，会发起选举，这时各Server是无法接受请求的。

ZAB选举算法对每个节点设定的三种状态，分别是：Leading，也就是成为Leader的状态；Following，成为Follower的状态；Looking，尚未选举出Leader时的状态。状态切换如下图：

![State](https://github.com/gulfer/gulfer.github.io/blob/master/pic/election.png)

ZAB选举执行实际上分为三个阶段：首先是进行选举，篇幅限制不详细描述选举过程，只需要了解大致过程是根据64位zxid进行选择，较大者胜出，并更新朝代编号（Epoch，zxid的前32位），有兴趣的可以查看Zookeeper源码中的FastLeaderElection类；选举出Leader后，进入Recovery阶段，由Leader通知所有Follower已经改朝换代，并在Follower间同步数据；最后Leader会发起广播（Broadcast），选举完成。

这就是ZAB对Paxos所做的变形，虽然提案的提出机制有所差异，但最终的选择机制是相同的，仍然是Paxos的思路。

## 小结

其实Zookeeper并不是一种Paxos协议的典型实现，但Zookeeper及其ZAB算法的核心思想都来源于Paxos协议。Paxos协议处理的是分布式系统中的数据一致性问题，一个复杂的分布式系统可能出现很多其他问题。比如脑裂，分布式系统中某个节点宕机或假死的情况，需要有一个协调者对这种情况进行监测，Zookeeper就提供了很好的基于心跳的监测机制。事实上现在业内的共识是，分布式系统的确需要一种产品来实现各个分布式节点之间的协调配合，不管是用Zookeeper还是其他产品，而这种产品或多或少都会精简或扩展Paxos原始算法。

参考书目：

[《Leslie Lamport：Paxos made Simple》](http://lamport.azurewebsites.net/pubs/paxos-simple.pdf)原版论文，建议同志们不要看翻译版，我看过两个版本的翻译，都有不同程度的歧义。

[《PAXOS到ZOOKEEPER分布式一致性原理与实践》](http://download.csdn.net/detail/zhangyy1975/9404283)亲测可下，不过Paxos算法描述那部分也是直接翻译上面的论文

