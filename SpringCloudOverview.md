# Spring Cloud体系简介

Spring Cloud是Spring Source推出的一套快速搭建云服务的工具集。在使用Spring Boot作为应用搭建和依赖管理基础的同时，引入了Netflix贡献的服务发现注册、路由网关、断路器等组件，实现了Spring Cloud的完整架构体系。

## 组件介绍

Spring Cloud包含的组件很多，下面简单介绍一些重要的组件，其他组件是对Cloud的有益补充。

#### 配置管理

官方提供的配置管理服务Spring Cloud Config，支持本地、Git以及SVN等几种方式的配置管理方式，不过当然我们更习惯使用ZK或Consul，Spring Cloud项目集中也分别提供了专门封装ZK和Consul操作的项目。

#### 消息总线

官方提供的消息总线Spring Cloud Bus用户在集群中传播状态变化

#### 安全

Spring Cloud Security是官方提供的安全控制工具，不过我没有使用过，猜测是基于Spring Security做的扩展。

其他官方项目还有命令行工具Spring Cloud CLI、Spring Cloud Task等。不过作为一个完整的云平台工具包，其中那些由Netflix、Pivotal贡献的组件也非常重要。

#### 服务发现注册

Eureka是Netflix贡献的服务发现注册组件，可以对标Dubbo基于ZK的服务注册中心。后面我将会单独写一篇文章分析Eureka的原理及源码。

#### 路由网关

Zuul在整个Cloud体系中的作用是服务的路由网关，负责服务的路由、权限控制、服务过滤等。看着是不是和Firefly-Server很像，其实Firefly-Server的架构设计思路和Zuul很像，并且发展方向也是Cloud中的网关角色。

#### 断路器

当服务出现故障的时候，我们需要有一种机制对其进行监控，同时妥善的处理对该服务的调用，避免阻塞式的等待，造成资源占用等问题。Hystrix也是Netflix贡献的组件，可以提供很强的容错能力。后面我也将会专门研究下Hystrix。

Netflix还贡献了负载均衡工具Ribbon、数据流聚合器Turbine（依赖AMQP），而Pivotal贡献了一些大数据分析相关的组件，如Spring Cloud Data Flow等。Spring Cloud正式整合了所有的这些优秀项目，形成了完整的云服务体系。

## 体系结构

Netflix提供了一个Spring Cloud的完整Sample，基于Spring Boot创建的工程实现了开箱即用的依赖管理，组件之间只需要通过配置即可建立联系。

我们可以在Github上下载项目集源码：

[POC of Spring Cloud / Netflix OSS](https://github.com/Oreste-Luci/netflix-oss-example)

这里我引用一下Git上的体系结构图，并基于此图对数据流及组件作用做简单介绍：

![](https://github.com/gulfer/gulfer.github.io/blob/master/pic/netflix-oss-example.png)


    

