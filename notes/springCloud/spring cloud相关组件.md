### spring cloud相关组件

> **Eureka**：
>
> - Eureka server 的高可用实际上就是将自己作为服务向其他服务注册中心注册自己，这样就可以形成一组互相注册的服务注册中心，。若有两个或两个以上的Eureka Server（服务注册中心）时，他们之间是互相注册的，当服务提供者发送注册请求到一个服务注册中心时，它会将该请求转发到集群中相连的其他注册中心，从而实现注册中心间的服务同步，这样服务提供者的服务信息可以通过任意一台服务中心获取到。
> - eureka可分为三个角色：服务发现者、服务注册者、注册发现中心。
> - 服务注册中心：Eureka提供的服务端，提供服务注册与发现的功能
> - ​	失效剔除：对于那些非正常下线的服务实例（内存溢出、网络故障导致的），服务注册中心不能收到“服务下线”的请求，为了将这些无法提供服务的实例从服务列表中剔除，Eureka Server在启动的时候会创建一个定时任务，默认每隔一段时间（默认60s）将当前清单中超时（默认90s）没有续约的服务剔除出去。
> - ​	自我保护：Eureka Server 在运行期间，会统计心跳失败的比例在15分钟之内是否低于85%，如果出现低于的情况（生产环境由于网络不稳定会导致），Eureka Server会降当前的实例注册信息保护起来，让这些实例不过期，尽可能保护这些注册信息，但是在这保护期间内实例出现问题，那么客户端就很容易拿到实际上已经不存在的服务实例，会出现调用失败的情况，所以客户端必须有容错机制，比如可以使用请求重试、断路器等机制。在本地进行开发时可以使用 eureka.server.enable-self-preseervation=false参数来关闭保护机制，以确保注册中心可以将不可用的实例剔除。
>
> **服务提供者**
>
> ​	服务注册：服务提供者在启动的时候会通过发送Rest请求的方式将自己注册到 Eureka Server ，同时带上自身服务的一些元数据，Eureka Server 接收到这个Rest请求后，将元数据存储在一个双层结构Map中，第一层的key是服务名，第二层key是具体服务的实例名。
>
> ​	服务同步：若有两个或两个以上的Eureka Server时，他们之间是互相注册的，当服务提供者发送注册请求到一个服务注册中心时，它会将该请求转发到集群中相连的其他注册中心，从而实现注册中心间的服务同步，这样服务提供者的服务信息可以通过任意一台服务中心获取到。
>
> ​	服务续约：在注册完服务之后，服务提供者会维护一个心跳来持续告诉Eureka Server：“我还活着”，以防止Eureka Server的“剔除任务”将该服务实例从服务列表中排除出去。
>
> - eureka.instance.lease-renewal-in-seconds=30(续约任务的调用间隔时间，默认30秒，也就是每隔30秒向服务端发送一次心跳，证明自己依然存活)
> - eureka.instance.lease-expiration-duration-in-seconds=90(服务失效时间，默认90秒，也就是告诉服务端，如果90秒之内没有给你发送心跳就证明我“死”了，将我剔除)
>
> **服务消费者**
>
> ​	获取服务：当启动服务消费者的时候，它会发送一个Rest请求给注册中心，获取上面注册的服务清单，Eureka Server会维护一份只读的服务清单来返回给客户端，并且每三十秒更新一次
>
> ​	服务调用：在服务消费者获取到服务清单后，通过服务名可以获得具体提供服务的实例名和该实例的元信息，采用Ribbon实现负载均衡
>
> ​	服务下线：当服务实例进行正常的关闭操作时，它会触发一个服务下线的Rest请求给Eureka Server，告诉服务注册中心“我要下线了”。服务端接收到请求之后，将该服务状态设置为下线，并把下线时间传播出去。
>
> **Ribbon和Feign的区别：** 
>
> - 启动类使用的注解不同，Ribbon使用的是@RibbonClient，Feign使用的是@EnableFeignClients
> - 服务的指定位置不同，Ribbon是在@RibbonClient注解上声明，Feign则是在定义抽象方法的接口中使用@FeignClient声明
> - 调用方式不同，Ribbon需要自己构建http请求，模拟http请求然后使用RestTemplate发送给其他服务，步骤比较繁琐。Feign则是在Ribbon的基础上进行了一次改进，采用接口调用的方式，将需要调用的其他服务的方法定义成抽象方法即可，不需要自己构建http请求，不过要注意的是抽象方法的注解、方法签名要和提供方的完全一致。
>
> **熔断机制：**
>
> ​	当一个服务出现故障时，请求失败次数超过设定的阀值（默认50）之后，该服务就会开启熔断器，之后该服务就不进行任何业务逻辑操作，执行快速失败，直接返回请求失败的信息。其他依赖于该服务的服务就不会因为得不到响应而造成线程阻塞，这是除了该服务和依赖于该服务的部分功能不可用外，其他功能正常。
>
> ​	熔断器还有一个自我修复机制，当一个服务熔断后，经过一段时间（5s）半打开熔断器。半打开的熔断器会检查一部分请求（只能有一个请求）是否正常，其他请求执行快速失败，检查的请求如果响应成功，则可判断该服务正常了，就可关闭该服务的熔断器，反之则继续打开熔断器。这种自我熔断机制和自我修复机制可以使程序更加健壮、也可以为开发和运维减少很多不必要的工作。



### [微服务技术栈]

##### 微服务开发

​	作用：快速开发服务。

- Spring

- Spring MVC

- Spring Boot

  [官网](https://spring.io/)，Spring 目前是 JavaWeb 开发人员必不可少的一个框架，SpringBoot 简化了 Spring 开发的配置目前也是业内主流开发框架。



##### 微服务注册发现

​	作用：发现服务，注册服务，集中管理服务

###### Eureka

- Eureka Server : 提供服务注册服务,各个节点启动后，会在 Eureka Server 中进行注册。
- Eureka Client : 简化与 Eureka Server 的交互操作
- Spring Cloud Netflix : [GitHub](https://github.com/spring-cloud/spring-cloud-netflix)，[文档](https://cloud.spring.io/spring-cloud-netflix/reference/html/)

###### Zookeeper

> ZooKeeper is a centralized service for maintaining configuration information, naming, providing distributed synchronization, and providing group services.

Zookeeper 是一个集中的服务,用于维护配置信息、命名、提供分布式同步和提供组服务。

###### Zookeeper 和 Eureka 区别

​	Zookeeper 保证 CP，Eureka 保证 AP：

- C：数据一致性；

- A：服务可用性；

- P：服务对网络分区故障的容错性，这三个特性在任何分布式系统中不能同时满足，最多同时满足两个。

  

##### 微服务配置管理

​	作用：统一管理一个或多个服务的配置信息,集中管理。

###### Disconf

​	Distributed Configuration Management Platform(分布式配置管理平台) ,它是专注于各种分布式系统配置管理 的通用组件/通用平台, 提供统一的配置管理服务,是一套完整的基于 zookeeper 的分布式配置统一解决方案。

###### SpringCloudConfig

###### Apollo

​	Apollo（阿波罗）是携程框架部门研发的分布式配置中心，能够集中化管理应用不同环境、不同集群的配置，配置修改后能够实时推送到应用端，并且具备规范的权限、流程治理等特性，用于微服务配置管理场景。



##### 权限认证

​	作用：根据系统设置的安全规则或者安全策略,用户可以访问而且只能访问自己被授权的资源，不多不少。

###### 	Spring Security

###### 	apache Shiro



##### 批处理

​	作用: 批量处理同类型数据或事物

###### 	Spring Batch

###### 	Quartz



##### 微服务调用 (协议)

###### Rest

​	通过 HTTP/HTTPS 发送 Rest 请求进行数据交互

​	Remote Procedure Call

​	它是一种通过网络从远程计算机程序上请求服务，而不需要了解底层网络技术的协议。RPC 不依赖于具体的网络传输协议，tcp、udp 等都可以。

###### gRPC

​	A high-performance, open-source universal RPC framework 所谓 RPC(remote procedure call 远程过程调用) 框架实际是提供了一套机制，使得应用程序之间可以进行通信，而且也遵从 server/client 模型。使用的时候客户端调用 server 端提供的接口就像是调用本地的函数一样。

###### RMI

​	Remote Method Invocation

纯 Java 调用



##### 服务接口调用

> 作用：多个服务之间的通讯

###### Feign(HTTP)

​	Spring Cloud Netflix 的微服务都是以 HTTP 接口的形式暴露的，所以可以用 Apache 的 HttpClient 或 Spring 的 RestTemplate 去调用，而 Feign 是一个使用起来更加方便的 HTTP 客戶端，使用起来就像是调用自身工程的方法，而感觉不到是调用远程方法。



##### 服务熔断

> 作用: 当请求到达一定阈值时不让请求继续.

###### Hystrix

###### Sentinel

- A lightweight powerful flow control component enabling reliability and monitoring for microservices. (轻量级的流量控制、熔断降级 Java 库)

  


#### 服务的负载均衡

> 作用：降低服务压力,增加吞吐量

###### Ribbon

​	Spring Cloud Ribbon 是一个基于 HTTP 和 TCP 的客户端负载均衡工具,它基于 Netflix Ribbon 实现

###### Nginx

​	Nginx (engine x) 是一个高性能的 HTTP 和反向代理 web 服务器,同时也提供了 IMAP/POP3/SMTP 服务

###### Nginx 与 Ribbon 区别

​	Nginx 属于服务端负载均衡,Ribbon 属于客户端负载均衡.Nginx 作用与 Tomcat,Ribbon 作用与各个服务之间的调用 (RPC)



###### 消息队列

> 作用: 解耦业务,异步化处理数据

###### Kafka

###### RabbitMQ

###### RocketMQ

###### activeMQ



##### 日志采集 (elk)

> 作用:收集各服务日志提供日志分析、用户画像等

###### Elasticsearch

###### Logstash

###### Kibana



##### API 网关

> 作用:外部请求通过 API 网关进行拦截处理,再转发到真正的服务

#### Zuul



##### 服务监控

> 作用:以可视化或非可视化的形式展示出各个服务的运行情况 (CPU、内存、访问量等)

###### Zabbix

###### Nagios

###### Metrics



##### 服务链路追踪

> 作用:明确服务之间的调用关系

###### Zipkin

###### Brave



##### 数据存储

> 作用: 存储数据

##### 关系型数据库

###### MySql

###### Oracle

###### PostgreSql



##### 非关系型数据库

###### Mongodb

###### Elasticsearch



##### 缓存

> 作用: 存储数据

###### redis



##### 分库分表

> 作用: 数据库分库分表方案.

##### shardingsphere

##### Mycat



##### 服务部署

> 作用: 将项目快速部署、上线、持续集成.

###### Docker

###### Jenkins

###### Kubernetes(K8s)

###### Mesos



##### [微服务治理策略]

###### 服务的注册和发现

> 解决问题: 集中管理服务

> 解决方法: eureka 、zookeeper

###### 负载均衡

> 解决问题: 降低服务器硬件压力

> 解决方法: nginx 、 Ribbon

###### 通讯

> 解决问题: 各个服务之间的沟通桥梁

> 解决方法 :
>
> - 同步消息
>   1. rest
>   2. rpc
> - 异步消息
>   1. MQ

###### 配置管理

> 解决问题: 随着服务的增加配置也在增加,如何管理各个服务的配置

> 解决方法: nacos 、 spring cloud config 、 Apollo

###### 容错和服务降级

> 解决问题: 在微服务当中，一个请求经常会涉及到调用几个服务，如果其中某个服务不可以，没有做服务容错的话，极有可能会造成一连串的服务不可用，这就是雪崩效应.

> 解决方法: hystrix

###### 服务依赖关系

> 解决问题: 多个服务之间来回依赖,启动关系的不明确

> 解决方法:

> 1. 应用分层: 数据层,业务层 数据层不需要依赖业务层,业务层依赖数据,规定上下依赖关系避免循环圈

###### 服务文档

> 解决问题: 降低沟通成本

> 解决方法: swagger 、 java doc

###### 服务安全问题

> 解决问题: 敏感数据的安全性

> 解决方法: oauth 、 shiro 、 spring security

###### 流量控制

> 解决问题: 避免一个服务上的流量过大拖垮整个服务体系

> 解决方法: Hystrix

###### 自动化测试

> 解决问题: 提前预知异常,确定服务是否可用

> 解决方法: junit

###### 服务上线,下线的流程

> 解决问题: 避免服务随意的上线下线

> 解决方法: 新服务上线需要经过管理人员审核.服务下线需要告知各个调用方进行修改,直到没有调用该服务才可以进行下线.

###### 兼容性

> 解决问题: 服务开发持续进行如何做到兼容

> 解决方法: 通过版本号的形式进行管理,修改完成进行回归测试

###### 服务编排

> 解决问题: 解决服务依赖问题的一种方式

> 解决方法: docker & k8s

###### 资源调度

> 解决问题: 每个服务的资源占用量不同,如何分配

> 解决方法: JVM 隔离、classload 隔离 ; 硬件隔离

###### 容量规划

> 解决问题: 随着时间增长,调用逐步增加,什么时候追加机器

> 解决方法: 统计每日调用量和响应时间, 根据机器情况设置阈值,超过阈值就可以追加机器