# Spring Cloud体系简介

Spring Cloud是Spring Source推出的一套快速搭建云服务的工具集。在使用Spring Boot作为应用搭建和依赖管理基础的同时，引入了Netflix贡献的服务发现注册、路由网关、断路器等组件，实现了Spring Cloud的完整架构体系。所以Spring Boot也是公认的目前最流行的微服务框架。

## 组件介绍

Spring Cloud很大程度上是基于Spring Boot的，Spring Boot提供了一种构建应用、组织依赖的规范，而Spring Cloud中的组件就是依照这种规范存在并相互联系的。简单的说，Eureka提供服务发现注册功能；通过Ribbon或Feign实现服务均衡负载；使用Spring Cloud Config作为配置管理；使用Hystrix作为断路器（或者叫熔断器）保证服务可用性。

Netflix提供了一个Spring Cloud的完整Sample，基于Spring Boot创建的工程实现了开箱即用的依赖管理，组件之间只需要通过配置即可建立联系。我们可以在Github上下载项目集源码：

[POC of Spring Cloud / Netflix OSS](https://github.com/Oreste-Luci/netflix-oss-example)

#### 配置管理

官方提供的配置管理服务Spring Cloud Config，支持本地、Git以及SVN等几种方式的配置管理方式，不过当然我们更习惯使用ZK或Consul，Spring Cloud项目集中也分别提供了专门封装ZK和Consul操作的项目。Spring Cloud Config Server是一个Spring Boot应用，因为内嵌了WEB Server（默认是Tomcat），因此可以直接通过命令启动，默认端口8888：

```
mvnw spring-boot:run

```
或直接执行jar

```
java -jar zuul.jar
```
其他组件均可通过这两种方式执行。

需要注意的是，本文中的例子均是基于刚才提到的Netflix OSS Example，这个项目集中的组件大部分都依赖这套Config Server，包括服务注册时获取Eureka的地址等，都需要通过Config Server获取相应的配置。

[配置Repo](https://github.com/Oreste-Luci/netflix-oss-example-config-repo)

#### 服务发现

Eureka是Netflix贡献的服务发现注册组件，可以对标Dubbo基于ZK的服务注册中心。Eureka的服务也是基于REST的，Spring Cloud通过Spring Boot对Eureka做了集成。使用Spring Boot启动Eureka的方法非常简单，需要配置@EnableEurekaServer注解：

```
@SpringBootApplication
@EnableEurekaServer
public class EurekaApplication {
    public static void main(String[] args) {
        SpringApplication.run(EurekaApplication.class, args);
    }
}
```
并添加一些服务端的配置，Eureka服务端口指定为8761：

```
server:
  port: 8761

eureka:
  client:
    registerWithEureka: false
    fetchRegistry: false
    server:
      waitTimeInMsWhenSyncEmpty: 0
      
spring:
  application:
    name: eureka-service
```
而客户端在发布服务时需要对应的配置@EnableDiscoveryClient：

```
@EnableAutoConfiguration
@SpringBootApplication
@EnableDiscoveryClient
@EnableCircuitBreaker
@EnableFeignClients(value = "service.a.*")
@ComponentScan("service.a.*")
public class ApplicationA {
    public static void main(String[] args) {
        SpringApplication.run(ApplicationA.class, args);
    }
}
```
启动时会将服务自动注册到Eureka，可以在Eureka自带的dashboard中查看服务状态等信息：

![](https://github.com/gulfer/gulfer.github.io/blob/master/pic/ScreenShot_eureka.png)

后面我将会单独写一篇文章分析Eureka的原理及源码。

#### 路由网关

Zuul在整个Cloud体系中的作用是服务的路由网关，负责服务的路由、权限控制、服务过滤等。Firefly-Server的架构设计思路和Zuul很像，发展方向也是Cloud中的网关角色。只不过Zuul本质上是个Servlet，附加功能通过Filter提供，但并不是Java Web应用中的Filter。

Zuul架构:
![](https://github.com/gulfer/gulfer.github.io/blob/master/pic/zuul.png)

路由网关在确保内部微服务无状态的基础上，对外统一暴露RESTful API，集成Ribbon做服务负载均衡，更重要的是，作为统一入口，网关可以实现了权限验证、安全校验等功能。

使用时仅需要添加Zuul和Eureka依赖，在主应用类中配置@EnableZuulProxy注解，并在配置文件中指定应用名及端口即可。Zuul作为网关，可以配置服务的入口路径以及重定向路径、配置反向代理及负载均衡策略、配置URL重写等，俨然nginx。

#### 负载均衡

Netflix贡献的Ribbon在Cloud中负责对服务进行软负载均衡。上节提到的Zuul配置，可用以下方式配置Ribbon的负载均衡策略：

```
# 最大重试次数（不包括首次请求）
api.ribbon.MaxAutoRetries=1
# 下一服务最大重试次数（不包括首次请求）
api.ribbon.MaxAutoRetriesNextServer=1
# 是否重试
api.ribbon.OkToRetryOnAllOperations=true
# 节点列表刷新间隔
api.ribbon.ServerListRefreshInterval=1000
# 连接超时
api.ribbon.ConnectTimeout=2000
# 读超时
api.ribbon.ReadTimeout=10000
# 节点
api.ribbon.listOfServers=192.168.1.100:8001,192.168.1.101:8002
```
Ribbon可以和Eureka或Consul等服务发现组件结合使用，通过Eureka获取服务列表并选择要调用的服务。Ribbon提供了多种负载均衡策略，源码我没有读过，对其机制并不太了解，还需研究。

Ribbon架构
![](https://github.com/gulfer/gulfer.github.io/blob/master/pic/ribbon.png)

图中的Edge Service可以理解为集成了Ribbon的Zuul之类的网关服务。

#### 断路器

当服务出现故障的时候，我们需要有一种机制对其进行监控，同时妥善的处理对该服务的调用，避免阻塞式的等待，造成资源占用等问题。尤其是调用关系相对复杂的微服务系统内部，某些高并发的服务调用失败时如果没有隔离措施，整个系统都有可能被拖垮。Hystrix也是Netflix贡献的组件，可增强微服务系统的容错能力，并且自带了dashboard。

Spring Cloud的源码中，Zuul就是通过继承HystrixCommand抽象来实现自己的RibbonCommand，包装了服务调用的具体实现，同时利用Hystrix完成服务的监控。后面也将会专门研究下Hystrix。

Netflix还贡献了数据流聚合器Turbine（使用AMQP），而Pivotal贡献了一些大数据分析相关的组件，如Spring Cloud Data Flow等。Spring Cloud正式整合了所有的这些优秀项目，形成了完整的云服务体系。

## 体系结构

这里我引用一下Git上的体系结构图，并基于此图对数据流及组件作用做简单介绍：

![](https://github.com/gulfer/gulfer.github.io/blob/master/pic/netflix-oss-example.png)


    

