# Paxos与Zookeeper

什么是分布式
什么是一致性
为什么要有Paxos
Paxos是怎么做的
Zookeeper是怎么做的

在介绍Paxos和Zookeeper之前，我们需要先搞清楚什么是分布式系统，分布式系统能解决什么样的问题，又带来哪些问题，分布式系统是怎么解决这些问题的。事实上我完全不是这方面的专家，对理论、技术的了解也并不深入，在此只想以个人肤浅的见识抛砖引玉，大家共同探讨下。

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

在看Paxos协议之前，我们如果自己设计分布式一致性方案会如何设计呢？

我们在方案的执行过程中归纳出三类角色：当系统中某个节点收到操作指令时，需要作为提交者（Proposer）向其他节点提交提案（赋值），其他节点的角色就是接收者（Acceptor）。当然我们不可能保证所有接受者都能同时收到提案，因为有可能某个接受者进程宕掉了或因为网络原因丢包了，所以为了让这个提案生效，很自然会想到让大多数接受者批准这个提案就行了，最后是提案执行者（Learner）使提案生效了。

当多个Proposer同时发起对某个值的变更提案时怎么办？我们会想到对每个提案设置编号，这样才能通过编号大小来确定哪个提案应该被执行。这里有一点需要注意，对于某个值到底应该设置，我们关注的是尽快取得一致，而非是某个Proposer的提案一定要被执行。这样，Acceptor或者说大多数的Acceptor（Majority）只要认可更大的那个提案编号，就能够获得最终的一致性。这是我个人对Paxos最简单的理解，Paxos的过程是一个两阶段提交（2PC）的过程，两阶段提交的第一阶段提交是准备提案，询问所有Acceptor是否能够接收提案，还是有编号更大的提案，如果有编号更大的提案，则继续执行准备提案，直到大多数Acceptor都反馈可以执行提案，再发起第二阶段提交即执行提案。而Learner角色相对简单，用于记录本次执行的提案。

## Zookeeper与Paxos

介绍了下原理，我们还是回归代码吧。需要说明的是，大多数的分布式应用虽然基于Paxos协议来保证一致性，但出于这些应用自身的实际情况，多少都对Paxos做了精简或变形。Zookeeper使用的一致性算法ZAB，就是Paxos的一种变形算法。

Zookeeper在设计上设置了Leader这样一个特殊的角色，这个角色属于分布式环境下的某一个节点，并且规定：先到Leader节点的操作将先执行，这样就保证了操作的执行顺序。而由于Leader可能存在宕机或网络中断的情况，因此需要对Leader进行选举，选举过程就用到了ZAB算法。

## Hystrix的熔断与流控



## 小结





