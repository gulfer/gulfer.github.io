# Spring Cloud体系简介

Spring Cloud是Spring Source推出的一套快速搭建微服务的工具集。在使用Spring Boot作为应用搭建和依赖管理基础的同时，引入了Netflix贡献的服务发现注册、路由网关、断路器等组件，实现了Spring Cloud的完整架构体系。

## 组件介绍

在网上查看Spring Cloud的资料，通常会提到九大功能或组件，分别是分布式配置管理、服务发现注册、路由等。其实除去Spring Cloud官方项目外，有很多其他项目都是对Spring Cloud做的有益补充。

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

Zuul

## 体系结构

Netflix提供了一个Spring Cloud的完整Sample，基于Spring Boot创建的工程实现了开箱即用的依赖管理，组件之间只需要通过配置即可建立联系。

我们可以在Github上下载项目集源码：

[POC of Spring Cloud / Netflix OSS](https://github.com/Oreste-Luci/netflix-oss-example)

这里我引用一下Git上的体系结构图：



