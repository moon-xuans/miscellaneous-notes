# SpringCloud

## 1.前言

**cloud升级**

![img](https://raw.githubusercontent.com/moon-xuans/mediaImage/main/2022/202207311918944.png)

## 2.SpringCloud组件

### 2.1.Eureka服务注册与发现

#### 2.1.1.Eureka基础知识

**什么是服务治理**

 在传统的rpc远程调用框架中，管理每个服务与服务之间依赖关系比较复杂，管理比较复杂，所以需要使用服务治理，管理服务于服务之间依赖关系，可以实现服务调用、负载均衡、容错等，实现服务发现与注册。

**什么是服务注册与发现**

Eureka采用了CS的设计架构，Eureka Server 作为服务注册功能的服务器，它是服务注册中心。而系统中的其他微服务，使用 Eureka的客户端连接到 Eureka Server并维持心跳连接。这样系统的维护人员就可以通过 Eureka Server 来监控系统中各个微服务是否正常运行。

在服务注册与发现中，有一个注册中心。当服务器启动的时候，会把当前自己服务器的信息 比如 服务地址通讯地址等以别名方式注册到注册中心上。另一方（消费者|服务提供者），以该别名的方式去注册中心上获取到实际的服务通讯地址，然后再实现本地RPC调用RPC远程调用框架核心设计思想：在于注册中心，因为使用注册中心管理每个服务与服务之间的一个依赖关系(服务治理概念)。在任何rpc远程框架中，都会有一个注册中心(存放服务地址相关信息(接口地址))

**Eureka系统架构和Dubbo的架构对比**

![img](https://raw.githubusercontent.com/moon-xuans/mediaImage/main/2022/202207311922376.png)

**Eureka两组件** 

*Eureka包含两个组件：Eureka Server和Eureka Client*

- Eureka Server提供服务注册服务

各个微服务节点通过配置启动后，会在EurekaServer中进行注册，这样EurekaServer中的服务注册表中将会存储所有可用服务节点的信息，服务节点的信息可以在界面中直观看到。

- EurekaClient通过注册中心进行访问

是一个Java客户端，用于简化Eureka Server的交互，客户端同时也具备一个内置的、使用轮询(round-robin)负载算法的负载均衡器。在应用启动后，将会向Eureka Server发送心跳(默认周期为30秒)。如果Eureka Server在多个心跳周期内没有接收到某个节点的心跳，EurekaServer将会从服务注册表中把这个服务节点移除（默认90秒）

#### 2.1.2.Eureka集群

微服务RPC远程服务调用最核心的是什么？

 高可用，试想你的注册中心只有一个only one， 它出故障了，会导致整个为服务环境不可用，所以搭建Eureka注册中心集群 ，实现负载均衡+故障容错

![img](https://raw.githubusercontent.com/moon-xuans/mediaImage/main/2022/202207311926591.png)

#### 2.1.3.Eureka自我保护

##### 2.1.3.1.故障现象

保护模式主要用于一组客户端和Eureka Server之间存在网络分区场景下的保护。一旦进入保护模式，Eureka Server将会尝试保护其服务注册表中的信息，不再删除服务注册表中的数据，也就是不会注销任何微服务。

如果在Eureka Server的首页看到以下这段提示，则说明Eureka进入了保护模式：

> EMERGENCY! EUREKA MAY BE INCORRECTLY CLAIMING INSTANCES ARE UP WHEN THEY'RE NOT. RENEWALS ARE LESSER THAN THRESHOLD AND HENCE THE INSTANCES ARE NOT BEING EXPIRED JUST TO BE SAFE 

![img](https://raw.githubusercontent.com/moon-xuans/mediaImage/main/2022/202207311930392.png)

##### 2.1.3.2.导致原因

** 为什么会产生Eureka自我保护机制？**

为了防止EurekaClient可以正常运行，但是 与 EurekaServer网络不通情况下，EurekaServer不会立刻将EurekaClient服务剔除

**什么是自我保护模式？**

默认情况下，如果EurekaServer在一定时间内没有接收到某个微服务实例的心跳，EurekaServer将会注销该实例（默认90秒）。但是当网络分区故障发生(延时、卡顿、拥挤)时，微服务与EurekaServer之间无法正常通信，以上行为可能变得非常危险了——因为微服务本身其实是健康的，此时本不应该注销这个微服务。Eureka通过“自我保护模式”来解决这个问题——当EurekaServer节点在短时间内丢失过多客户端时（可能发生了网络分区故障），那么这个节点就会进入自我保护模式。



![img](https://raw.githubusercontent.com/moon-xuans/mediaImage/main/2022/202207311931276.png)

在自我保护模式中，Eureka Server会保护服务注册表中的信息，不再注销任何服务实例。

它的设计哲学就是宁可保留错误的服务注册信息，也不盲目注销任何可能健康的服务实例。一句话讲解：好死不如赖活着

综上，自我保护模式是一种应对网络异常的安全保护措施。它的架构哲学是宁可同时保留所有微服务（健康的微服务和不健康的微服务都会保留）也不盲目注销任何健康的微服务。使用自我保护模式，可以让Eureka集群更加的健壮、稳定。

> 一句话：某时刻某一个微服务不可用了，Eureka不会立即清理，依旧会对该微服务的信息进行保存（属于CAP里面的AP）

### 2.2.ZooKeeper服务注册与发现

#### 2.2.1.注册中心ZooKeeper

ZooKeeper是一个分布式协调工具，可以实现注册中心功能。

### 2.3.Consul服务注册与发现

#### 2.3.1.Consul简介

**是什么**

Consul 是一套开源的分布式服务发现和配置管理系统，由 HashiCorp 公司用 Go 语言开发。

提供了微服务系统中的服务治理、配置中心、控制总线等功能。这些功能中的每一个都可以根据需要单独使用，也可以一起使用以构建全方位的服务网格，总之Consul提供了一种完整的服务网格解决方案。

它具有很多优点。包括： 基于 raft 协议，比较简洁； 支持健康检查, 同时支持 HTTP 和 DNS 协议 支持跨数据中心的 WAN 集群 提供图形界面 跨平台，支持 Linux、Mac、Windows。

**能干嘛**

- 服务发现 ——提供HTTP和DNS两种发现方式
- 健康检测——支持多种方式，HTTP、TCP、Docker、Shell脚本定制化监控
- KV存储——Key、Value的存储方式
- 多数据中心——Consul支持多数据中心
- 可视化Web界面

#### 2.3.2.三个注册中心异同点

![img](https://raw.githubusercontent.com/moon-xuans/mediaImage/main/2022/202207311950274.png)

**AP架构**

当网络分区出现后，为了保证可用性，系统B可以返回旧值，保证系统的可用性。

结论：违背了一致性C的要求，只满足可用性和分区容错，即AP

**CP架构**

当网络分区出现后，为了保证一致性，就必须拒接请求，否则无法保证一致性

结论：违背了可用性A的要求，只满足一致性和分区容错，即CP

### 2.4.Ribbon负载均衡服务调用

#### 2.4.1.概述

**是什么**

Spring Cloud Ribbon是基于Netflix Ribbon实现的一套*客户端*    负载均衡的工具。

简单的说，Ribbon是Netflix发布的开源项目，主要功能是提供客户端的软件负载均衡算法和服务调用。Ribbon客户端组件提供一系列完善的配置项如连接超时，重试等。简单的说，就是在配置文件中列出Load Balancer（简称LB）后面所有的机器，Ribbon会自动的帮助你基于某种规则（如简单轮询，随机连接等）去连接这些机器。我们很容易使用Ribbon实现自定义的负载均衡算法。

**能干嘛**

*LB负载均衡(Load Balance)是什么*

简单的说就是将用户的请求平摊的分配到多个服务上，从而达到系统的HA（高可用）。

常见的负载均衡有软件Nginx，LVS，硬件 F5等。

*Ribbon本地负载均衡客户端 VS Nginx服务端负载均衡区别*

 Nginx是服务器负载均衡，客户端所有请求都会交给nginx，然后由nginx实现转发请求。即负载均衡是由服务端实现的。

Ribbon本地负载均衡，在调用微服务接口时候，会在注册中心上获取注册信息服务列表之后缓存到JVM本地，从而在本地实现RPC远程服务调用技术。

> 集中式LB :即在服务的消费方和提供方之间使用独立的LB设施(可以是硬件，如F5, 也可以是软件，如nginx), 由该设施负责把访问请求通过某种策略转发至服务的提供方；
>
> 进程内LB:将LB逻辑集成到消费方，消费方从服务注册中心获知有哪些地址可用，然后自己再从这些地址中选择出一个合适的服务器。Ribbon就属于进程内LB，它只是一个类库，集成于消费方进程，消费方通过它来获取到服务提供方的地址。

#### 2.4.2.架构说明

![img](https://raw.githubusercontent.com/moon-xuans/mediaImage/main/2022/202207311958792.png)

Ribbon在工作时分成两步

- 第一步先选择 EurekaServer ,它优先选择在同一个区域内负载较少的server.

- 第二步再根据用户指定的策略，在从server取到的服务注册列表中选择一个地址。

其中Ribbon提供了多种策略：比如轮询、随机和根据响应时间加权。

>  总结：Ribbon其实就是一个软负载均衡的客户端组件，他可以和其他所需请求的客户端结合使用，和eureka结合只是其中的一个实例。

#### 2.4.3.Ribbon核心组件IRule

IRule:根据特定算法中从服务列表中选取一个要访问的服务

- RoundRobinRule:轮询
- RandomRule:随机
- RetryRule:先按照RoundRobinRule的策略获取服务，如果获取服务失败则在指定时间内会进行重试，获取可用的服务
- WeightedResponseTimeRule：对RoundRobinRule的扩展，响应速度越快的实例选择权重越大，越容易被选择
- BestAvailableRule:会先过滤掉由于多次访问故障而处于断路器跳闸状态的服务，然后选择一个并发量最小的服务
- AvailabilityFilteringRule:先过滤掉故障实例，再选择并发较小的实例
- ZoneAvoidanceRule:默认规则,复合判断server所在区域的性能和server的可用性选择服务器

### 2.5.OpenFeign服务接口调用

#### 2.5.1.概述

**OpenFeign是什么**

Feign是一个声明式WebService客户端。使用Feign能让编写Web Service客户端更加简单。

它的使用方法是定义一个服务接口然后在上面添加注解。Feign也支持可拔插式的编码器和解码器。Spring Cloud对Feign进行了封装，使其支持了Spring MVC标准注解和HttpMessageConverters。Feign可以与Eureka和Ribbon组合使用以支持负载均衡.

**能干嘛**

Feign旨在使编写Java Http客户端变得更容易。

前面在使用Ribbon+RestTemplate时，利用RestTemplate对http请求的封装处理，形成了一套模版化的调用方法。但是在实际开发中，由于对服务依赖的调用可能不止一处，往往一个接口会被多处调用，所以通常都会针对每个微服务自行封装一些客户端类来包装这些依赖服务的调用。所以，Feign在此基础上做了进一步封装，由他来帮助我们定义和实现依赖服务接口的定义。在Feign的实现下，我们只需创建一个接口并使用注解的方式来配置它(以前是Dao接口上面标注Mapper注解,现在是一个微服务接口上面标注一个Feign注解即可)，即可完成对服务提供方的接口绑定，简化了使用Spring cloud Ribbon时，自动封装服务调用客户端的开发量。

Feign集成了Ribbon

利用Ribbon维护了Payment的服务列表信息，并且通过轮询实现了客户端的负载均衡。而与Ribbon不同的是，通过feign只需要定义服务绑定接口且以声明式的方法，优雅而简单的实现了服务调用

**Feign和OpenFeign两者区别**

Feign:Feign是Spring Cloud组件中的一个轻量级RESTful的HTTP服务客户端。Feign内置了Ribbon，用来做客户端负载均衡，去调用服务注册中心的服务。Feign的使用方式是：使用Feign的注解定义接口，调用这个接口，就可以调用服务注册中心的服务。

OpenFeign:OpenFeign是Spring Cloud 在Feign的基础上支持了SpringMVC的注解，如@RequesMapping等等。OpenFeign的@FeignClient可以解析SpringMVC的@RequestMapping注解下的接口，并通过动态代理的方式产生实现类，实现类中做负载均衡并调用其他服务。

#### 2.5.2.OpenFeign超时控制

 默认Feign客户端只等待一秒钟，但是服务端处理需要超过1秒钟，导致Feign客户端不想等待了，直接返回报错。

为了避免这样的情况，有时候我们需要设置Feign客户端的超时控制。

yml文件中开启配置

#### 2.5.3.OpenFeign日志打印功能

Feign 提供了日志打印功能，我们可以通过配置来调整日志级别，从而了解 Feign 中 Http 请求的细节。

说白了就是对Feign接口的调用情况进行监控和输出。

### 2.6.Hystrix断路器

#### 2.6.1.概述

**分布式系统面临的问题**

复杂分布式体系结构中的应用程序有数十个依赖关系，每个依赖关系在某些时候将不可避免地失败。

![img](https://raw.githubusercontent.com/moon-xuans/mediaImage/main/2022/202207312059845.png)

服务雪崩

多个微服务之间调用的时候，假设微服务A调用微服务B和微服务C，微服务B和微服务C又调用其它的微服务，这就是所谓的“扇出”。如果扇出的链路上某个微服务的调用响应时间过长或者不可用，对微服务A的调用就会占用越来越多的系统资源，进而引起系统崩溃，所谓的“雪崩效应”.

对于高流量的应用来说，单一的后端依赖可能会导致所有服务器上的所有资源都在几秒钟内饱和。比失败更糟糕的是，这些应用程序还可能导致服务之间的延迟增加，备份队列，线程和其他系统资源紧张，导致整个系统发生更多的级联故障。这些都表示需要对故障和延迟进行隔离和管理，以便单个依赖关系的失败，不能取消整个应用程序或系统。

所以，通常当你发现一个模块下的某个实例失败后，这时候这个模块依然还会接收流量，然后这个有问题的模块还调用了其他的模块，这样就会发生级联故障，或者叫雪崩。

**是什么**

Hystrix是一个用于处理分布式系统的延迟和容错的开源库，在分布式系统里，许多依赖不可避免的会调用失败，比如超时、异常等，Hystrix能够保证在一个依赖出问题的情况下，不会导致整体服务失败，避免级联故障，以提高分布式系统的弹性。

“断路器”本身是一种开关装置，当某个服务单元发生故障之后，通过断路器的故障监控（类似熔断保险丝），向调用方返回一个符合预期的、可处理的备选响应（FallBack），而不是长时间的等待或者抛出调用方无法处理的异常，这样就保证了服务调用方的线程不会被长时间、不必要地占用，从而避免了故障在分布式系统中的蔓延，乃至雪崩。

**能干嘛**

- 服务降级
- 服务熔断
- 接近实时的监控

#### 2.6.2.Hystrix重要概念

**服务降级**

服务器忙，请稍后再试，不让客户端等待并立刻返回一个友好提示，fallback。

哪些情况会触发降级：

- 程序运行异常
- 超时
- 服务熔断触发服务降级
- 线程池/信号量打满也会导致服务降级

**服务熔断**

类比保险丝达到最大服务访问后，直接拒绝访问，拉闸限电，然后调用服务降级的方法并返回友好提示

> 就是保险丝，服务的降级->进而熔断->恢复调用链路

**服务限流**

秒杀高并发等操作，严禁一窝蜂的过来拥挤，大家排队，一秒钟N个，有序进行

### 2.7.zuul路由网关

#### 2.7.1.概述简介

**是什么**

Zuul是一种提供动态路由、监视、弹性、安全性等功能的边缘服务。

Zuul是Netflix出品的一个基于JVM路由和服务端的负载均衡器。

API网关为微服务架构中的服务提供了统一的访问入口，客户端通过API网关访问相关服务。API网关的定义类似于设计模式中的门面模式，它相当于整个微服务架构中的门面，所有客户端的访问都通过它来进行路由及过滤。它实现了请求路由、负载均衡、校验过滤、服务容错、服务聚合等功能。

![img](https://raw.githubusercontent.com/moon-xuans/mediaImage/main/2022/202207312121712.png)![img](https://raw.githubusercontent.com/moon-xuans/mediaImage/main/2022/202207312121717.png)

Zuul包含了如下最主要的功能：代理+路由+过滤

**能干嘛**

- 路由
- 过滤
- 负载均衡

网关为入口，由网关与微服务进行交互，所以网关必须要实现负载均衡的功能；

网关会获取微服务注册中心里面的服务连接地址，再配合一些算法选择其中一个服务地址，进行处理业务。

这个属于客户端侧的负载均衡，由调用方去实现负载均衡逻辑。

![img](https://raw.githubusercontent.com/moon-xuans/mediaImage/main/2022/202207312122904.png)

- 灰度发布

在灰度发布开始后，先启动一个新版本应用，但是并不直接将流量切过来，而是测试人员对新版本进行线上测试，启动的这个新版本应用，就是我们的金丝雀。如果没有问题，那么可以将少量的用户流量导入到新版本上，然后再对新版本做运行状态观察，收集各种运行时数据，如果此时对新旧版本做各种数据对比，就是所谓的A/B测试。新版本没什么问题，那么逐步扩大范围、流量，把所有用户都迁移到新版本上面来。

![img](https://raw.githubusercontent.com/moon-xuans/mediaImage/main/2022/202207312123673.png)

#### 2.7.2.过滤器

**功能**

过滤功能负责对请求过程进行额外的处理，是请求校验过滤及服务聚合的基础。

**过滤器的生命周期**

![](https://raw.githubusercontent.com/moon-xuans/mediaImage/main/2022/202207312124669.png)

过滤类型：

- pre：在请求被路由到目标服务前执行，比如权限校验、打印日志等功能；
- routing：在请求被路由到目标服务时执行
- post：在请求被路由到目标服务后执行，比如给目标服务的响应添加头信息，收集统计数据等功能；
- error：请求在其他阶段发生错误时执行。

### 2.8.Gateway新一代网关

#### 2.8.1.概述简介

**是什么**

Gateway是在Spring生态系统之上构建的API网关服务，基于Spring 5，Spring Boot 2和 Project Reactor等技术。

Gateway旨在提供一种简单而有效的方式来对API进行路由，以及提供一些强大的过滤器功能， 例如：熔断、限流、重试等

SpringCloud Gateway 是 Spring Cloud 的一个全新项目，基于 Spring 5.0+Spring Boot 2.0 和 Project Reactor 等技术开发的网关，它旨在为微服务架构提供一种简单有效的统一的 API 路由管理方式。

SpringCloud Gateway 作为 Spring Cloud 生态系统中的网关，目标是替代 Zuul，在Spring Cloud 2.0以上版本中，没有对新版本的Zuul 2.0以上最新高性能版本进行集成，仍然还是使用的Zuul 1.x非Reactor模式的老版本。而为了提升网关的性能，SpringCloud Gateway是基于WebFlux框架实现的，而WebFlux框架底层则使用了高性能的Reactor模式通信框架Netty。

Spring Cloud Gateway的目标提供统一的路由方式且基于 Filter 链的方式提供了网关基本的功能，例如：安全，监控/指标，和限流。

> SpringCloud Gateway 使用的Webflux中的reactor-netty响应式编程组件，底层使用了Netty通讯框架。

**能干嘛**

- 反向代理
- 鉴权
- 流量控制
- 熔断
- 日志监控

**微服务架构中网关在那里**

![img](https://raw.githubusercontent.com/moon-xuans/mediaImage/main/2022/202207312133942.png)

**为什么选择GateWay?**

- SpringCloud GateWay的特性

1. 基于Spring Framework 5, Project Reactor 和 Spring Boot 2.0 进行构建；
2. 动态路由：能够匹配任何请求属性；
3. 可以对路由指定 Predicate（断言）和 Filter（过滤器）；
4. 集成Hystrix的断路器功能；
5. 集成 Spring Cloud 服务发现功能；
6. 易于编写的 Predicate（断言）和 Filter（过滤器）；
7. 请求限流功能；
8. 支持路径重写。

- SpringCloud GateWay与Zuul的区别

1. Zuul 1.x，是一个基于阻塞 I/ O 的 API Gateway
2. Zuul 1.x 基于Servlet 2. 5使用阻塞架构它不支持任何长连接(如 WebSocket) Zuul 的设计模式和Nginx较像，每次 I/ O 操作都是从工作线程中选择一个执行，请求线程被阻塞到工作线程完成，但是差别是Nginx 用C++ 实现，Zuul 用 Java 实现，而 JVM 本身会有第一次加载较慢的情况，使得Zuul 的性能相对较差。
3. Zuul 2.x理念更先进，想基于Netty非阻塞和支持长连接，但SpringCloud目前还没有整合。 Zuul 2.x的性能较 Zuul 1.x 有较大提升。在性能方面，根据官方提供的基准测试， Spring Cloud Gateway 的 RPS（每秒请求数）是Zuul 的 1. 6 倍。
4. Spring Cloud Gateway 建立 在 Spring Framework 5、 Project Reactor 和 Spring Boot 2 之上， 使用非阻塞 API。
5. Spring Cloud Gateway 还 支持 WebSocket， 并且与Spring紧密集成拥有更好的开发体验

#### 2.8.2.三大核心理念

- Route(路由)

路由是构建网关的基本模块，它由ID，目标URI，一系列的断言和过滤器组成，如果断言为true则匹配该路由

- Predicate(断言)

开发人员可以匹配HTTP请求中的所有内容(例如请求头或请求参数)，如果请求与断言相匹配则进行路由

- Filter(过滤)

指的是Spring框架中GatewayFilter的实例，使用过滤器，可以在请求被路由前或者之后对请求进行修改。

#### 2.8.3.Gateway工作流程

客户端向 Spring Cloud Gateway 发出请求。然后在 Gateway Handler Mapping 中找到与请求相匹配的路由，将其发送到 Gateway Web Handler。

Handler 再通过指定的过滤器链来将请求发送到我们实际的服务执行业务逻辑，然后返回。

过滤器之间用虚线分开是因为过滤器可能会在发送代理请求之前（“pre”）或之后（“post”）执行业务逻辑。

Filter在“pre”类型的过滤器可以做参数校验、权限校验、流量监控、日志输出、协议转换等，

在“post”类型的过滤器中可以做响应内容、响应头的修改，日志的输出，流量监控等有着非常重要的作用。

### 2.9.SpringCloud Config分布式配置中心

#### 2.9.1.概述

**分布式系统面临的配置问题**

  微服务意味着要将单体应用中的业务拆分成一个个子服务，每个服务的粒度相对较小，因此系统中会出现大量的服务。由于每个服务都需要必要的配置信息才能运行，所以一套集中式的、动态的配置管理设施是必不可少的。

SpringCloud提供了ConfigServer来解决这个问题，我们每一个微服务自己带着一个application.yml，上百个配置文件的管理。

**是什么**

SpringCloud Config为微服务架构中的微服务提供集中化的外部配置支持，配置服务器为各个不同微服务应用的所有环境提供了一个中心化的外部配置。

**怎么玩**

SpringCloud Config分为服务端和客户端两部分。

服务端也称为分布式配置中心，它是一个独立的微服务应用，用来连接配置服务器并为客户端提供获取配置信息，加密/解密信息等访问接口

客户端则是通过指定的配置中心来管理应用资源，以及与业务相关的配置内容，并在启动的时候从配置中心获取和加载配置信息配置服务器默认采用git来存储配置信息，这样就有助于对环境配置进行版本管理，并且可以通过git客户端工具来方便的管理和访问配置内容。

**能干嘛**

- 集中管理配置文件
- 不同环境不同配置，动态化的配置更新，分环境部署比如dev/test/prod/beta/release
- 运行期间动态调整配置，不再需要在每个服务部署的机器上编写配置文件，服务会向配置中心统一拉取配置自己的信息
- 当配置发生变动时，服务不需要重启即可感知到配置的变化并应用新的配置
- 将配置信息以REST接口的形式暴露

#### 2.9.2.服务端配置

/{label}-{name}-{profiles}.yml

label：分支(branch)

name ：服务名

profiles：环境(dev/test/prod)

#### 2.9.3.Config客户端之动态刷新

**存在问题**

当引入actuator监控之后，修改git上配置之后，需要运维人员发送Post请求刷新才能使客户端生效。

### 2.10.SpringCloud Bus消息总线

#### 2.10.1.概述

Spring Cloud Bus 配合 Spring Cloud Config 使用可以实现配置的动态刷新。

**是什么**

Spring Cloud Bus 配合 Spring Cloud Config 使用可以实现配置的动态刷新。

![img](https://raw.githubusercontent.com/moon-xuans/mediaImage/main/2022/202207312157636.png)

Spring Cloud Bus是用来将分布式系统的节点与轻量级消息系统链接起来的框架，

它整合了Java的事件处理机制和消息中间件的功能。

Spring Clud Bus目前支持RabbitMQ和Kafka。

**能干嘛**

Spring Cloud Bus能管理和传播分布式系统间的消息，就像一个分布式执行器，可用于广播状态更改、事件推送等，也可以当作微服务间的通信通道。

**为何被称为总线**

*什么是总线*

在微服务架构的系统中，通常会使用轻量级的消息代理来构建一个共用的消息主题，并让系统中所有微服务实例都连接上来。由于该主题中产生的消息会被所有实例监听和消费，所以称它为消息总线。在总线上的各个实例，都可以方便地广播一些需要让其他连接在该主题上的实例都知道的消息。

*基本原理*

ConfigClient实例都监听MQ中同一个topic(默认是springCloudBus)。当一个服务刷新数据的时候，它会把这个信息放入到Topic中，这样其它监听同一Topic的服务就能得到通知，然后去更新自身的配置。

#### 2.10.2.SpringCloud Bus动态刷新

利用消息总线触发一个服务端ConfigServer的/bus/refresh端点，而刷新所有客户端的配置

![](https://raw.githubusercontent.com/moon-xuans/mediaImage/main/2022/202208010822565.png)

再次使用的时候，只需要运维工程师发送一次post请求，一次发送，处处生效。从而实现动态刷新全局广播，并且可以实现动态刷新定点通知。

### 2.11.SpringCloud Stream消息驱动

#### 2.11.1消息驱动概述

**是什么**

屏蔽底层消息中间件的差异,降低切换成本，统一消息的编程模型

**设计思想**

在没有绑定器这个概念的情况下，我们的SpringBoot应用要直接与消息中间件进行信息交互的时候，由于各消息中间件构建的初衷不同，它们的实现细节上会有较大的差异性。通过定义绑定器作为中间层，完美地实现了应用程序与消息中间件细节之间的隔离。通过向应用程序暴露统一的Channel通道，使得应用程序不需要再考虑各种不同的消息中间件实现。

通过定义绑定器Binder作为中间层，实现了应用程序与消息中间件细节之间的隔离。

**Binder**

在没有绑定器这个概念的情况下，我们的SpringBoot应用要直接与消息中间件进行信息交互的时候，由于各消息中间件构建的初衷不同，它们的实现细节上会有较大的差异性，通过定义绑定器作为中间层，完美地实现了应用程序与消息中间件细节之间的隔离。Stream对消息中间件的进一步封装，可以做到代码层面对中间件的无感知，甚至于动态的切换中间件(rabbitmq切换为kafka)，使得微服务开发的高度解耦，服务可以关注更多自己的业务流程

![img](https://raw.githubusercontent.com/moon-xuans/mediaImage/main/2022/202208010832830.png)

通过定义绑定器Binder作为中间层，实现了应用程序与消息中间件细节之间的隔离。

Binder可以生成Binding，Binding用来绑定消息容器的生产者和消费者，它有两种类型，INPUT和OUTPUT，INPUT对应于消费者，OUTPUT对应于生产者。

**Spring Cloud Stream标准流程套路**

*Binder*

很方便的连接中间件，屏蔽差异

*Channel*

通道，是队列Queue的一种抽象，在消息通讯系统中就是实现存储和转发的媒介，通过Channel对队列进行配置

*Source和Sink*

简单的可理解为参照对象是Spring Cloud Stream自身，从Stream发布消息就是输出，接受消息就是输入。

### 2.12.SpringCloud Sleuth分布式请求链路跟踪

#### 2.12.1.概述

**为什么会出现这个技术**

在微服务框架中，一个由客户端发起的请求在后端系统中会经过多个不同的的服务节点调用来协同产生最后的请求结果，每一个前端请求都会形成一条复杂的分布式服务调用链路，链路中的任何一环出现高延时或错误都会引起整个请求最后的失败。

**是什么**

Spring Cloud Sleuth提供了一套完整的服务跟踪的解决方案

在分布式系统中提供追踪解决方案并且兼容支持了zipkin

![img](https://raw.githubusercontent.com/moon-xuans/mediaImage/main/2022/202208010843428.png)

#### 2.12.2.术语

**完整的调用链路**

表示一请求链路，一条链路通过Trace Id唯一标识，Span标识发起的请求信息，各span通过parent id 关联起来

![img](https://raw.githubusercontent.com/moon-xuans/mediaImage/main/2022/202208010845330.png)

**怎么看**

一条链路通过Trace Id唯一标识，Span标识发起的请求信息，各span通过parent id 关联起来

![](https://raw.githubusercontent.com/moon-xuans/mediaImage/main/2022/202208010846334.png)



![img](https://raw.githubusercontent.com/moon-xuans/mediaImage/main/2022/202208010846462.png)

**名词解释**

Trace:类似于树结构的Span集合，表示一条调用链路，存在唯一标识

span:表示调用链路来源，通俗的理解span就是一次请求信息

## 3.SpringCloudAlibaba组件

### 3.1.SpringCloud Alibaba Nacos服务注册和配置中心

#### 3.1.1.Nacos简介

**是什么**

一个更易于构建云原生应用的动态服务发现、配置管理和服务管理平台。

Nacos = Eureka+Config +Bus，Nacos就是注册中心 + 配置中心的组合。

**各种注册中心比较**

![img](https://raw.githubusercontent.com/moon-xuans/mediaImage/main/2022/202208010854641.png)

据说 Nacos 在阿里巴巴内部有超过 10 万的实例运行，已经过了类似双十一等各种大型流量的考验

C是所有节点在同一时间看到的数据是一致的；而A的定义是所有的请求都会收到响应。

**何时选择使用何种模式？**

一般来说，如果不需要存储服务级别的信息且服务实例是通过nacos-client注册，并能够保持心跳上报，那么就可以选择AP模式。当前主流的服务如 Spring cloud 和 Dubbo 服务，都适用于AP模式，AP模式为了服务的可能性而减弱了一致性，因此AP模式下只支持注册临时实例。

如果需要在服务级别编辑或者存储配置信息，那么 CP 是必须，K8S服务和DNS服务则适用于CP模式。

CP模式下则支持注册持久化实例，此时则是以 Raft 协议为集群运行模式，该模式下注册实例之前必须先注册服务，如果服务不存在，则会返回错误。

#### 3.1.2.作为配置中心

![img](https://raw.githubusercontent.com/moon-xuans/mediaImage/main/2022/202208010858371.png)

#### 3.1.3.Nacos集群和持久化配置

Nacos默认自带的是嵌入式数据库derby，可以切换到mysql。

修改nacos的集群配置cluster.conf进行集群配置，加上nginx达到负载均衡

![img](https://raw.githubusercontent.com/moon-xuans/mediaImage/main/2022/202208010901870.png)

### 3.2.SpringCloud Alibaba Sentinel实现熔断与限流

#### 3.2.1.Sentinel

**是什么**

轻量级的流量控制、熔断降级Java库。

**能干嘛**

![img](https://raw.githubusercontent.com/moon-xuans/mediaImage/main/2022/202208010912305.png)

**怎么玩**

- 服务雪崩
- 服务降级
- 服务熔断
- 服务限流

#### 3.2.2.初始化

Sentinel直接运行jar包，即可使用。

Sentinel采用的懒加载，请求必须发送一次才可以看到。

#### 3.2.3.流控规则

![img](https://raw.githubusercontent.com/moon-xuans/mediaImage/main/2022/202208010924162.png)

**流控模式**

- 直接(默认)

表示1秒钟内查询1次就是OK，若超过次数1，就直接-快速失败，报默认错误、

- 关联

当关联资源/testB的qps阀值超过1时，就限流/testA的Rest访问地址，当关联资源到阈值后限制配置好的资源名

- 链路

**流控效果**

- 直接->快速失败
- 预热

公式：阈值除以coldFactor(默认值为3),经过预热时长后才会达到阈值

- 排队等待

匀速排队，阈值必须设置为QPS

#### 3.2.4.降级规则

**降级说明**

Sentinel 熔断降级会在调用链路中某个资源出现不稳定状态时（例如调用超时或异常比例升高），对这个资源的调用进行限制，让请求快速失败，避免影响到其它的资源而导致级联错误。

当资源被降级后，在接下来的降级时间窗口之内，对该资源的调用都自动熔断（默认行为是抛出 DegradeException）。

**降级设置**

- RT（平均响应时间，秒级）

平均响应时间超出阈值且在时间窗口内通过的请求>=5，两个条件同时满足后触发降级

窗口期过后关闭断路器

 RT最大4900（更大的需要通过-Dcsp.sentinel.statistic.max.rt=XXXX才能生效）

- 异常比列（秒级）

QPS >= 5 且异常比例（秒级统计）超过阈值时，触发降级；时间窗口结束后，关闭降级

- 异常数（分钟级）

- 异常数（分钟统计）超过阈值时，触发降级；时间窗口结束后，关闭降级

#### 3.2.5.热点key限流

**是什么**

热点即经常访问的数据，很多时候我们希望统计或者限制某个热点数据中访问频次最高的TopN数据，并对其访问进行限流或者其它操作

#### 3.2.6.服务熔断

![img](https://raw.githubusercontent.com/moon-xuans/mediaImage/main/2022/202208010937479.png)

### 3.3.SpringCloud Alibaba Seata处理分布式事务

 #### 3.3.1.分布式事务问题

单体应用被拆分成微服务应用，原来的三个模块被拆分成三个独立的应用，分别使用三个独立的数据源，

业务操作需要调用三个服务来完成。此时每个服务内部的数据一致性由本地事务来保证，但是全局的数据一致性问题没法保证。

![img](https://raw.githubusercontent.com/moon-xuans/mediaImage/main/2022/202208010945911.png)

**一句话**

一次业务操作需要跨多个数据源或需要跨多个系统进行远程调用，就会产生分布式事务问题.

#### 3.3.2.Seata简介

**是什么**

Seata是一款开源的分布式事务解决方案，致力于在微服务架构下提供高性能和简单易用的分布式事务服务。

**能干嘛**

一个典型的分布式事务过程

*分布式事务处理过程的——ID+三组件模型*

全局唯一的事务ID——Transaction ID XID

三组件：

- Transaction Coordinator(TC):事务协调器，维护全局事务的运行状态，负责协调并驱动全局事务的提交或回滚；
- Transaction Manager(TM):控制全局事务的边界，负责开启一个全局事务，并最终发起全局提交或全局回滚的决议；
- Resource Manager(RM):控制分支事务，负责分支注册、状态汇报，并接收事务协调器的指令，驱动分支（本地）事务的提交和回滚

*处理过程*

1. TM 向 TC 申请开启一个全局事务，全局事务创建成功并生成一个全局唯一的 XID；
2. XID 在微服务调用链路的上下文中传播；
3. RM 向 TC 注册分支事务，将其纳入 XID 对应全局事务的管辖；
4. TM 向 TC 发起针对 XID 的全局提交或回滚决议；
5. TC 调度 XID 下管辖的全部分支事务完成提交或回滚请求。

![img](https://raw.githubusercontent.com/moon-xuans/mediaImage/main/2022/202208010949047.png)

**怎么玩**

本地@Transactional + 全局@GlobalTransactional

![img](https://raw.githubusercontent.com/moon-xuans/mediaImage/main/2022/202208010950652.png)

#### 3.3.3.补充

**分布式事务的执行流程**

![img](https://raw.githubusercontent.com/moon-xuans/mediaImage/main/2022/202208010952917.png)

1. TM 开启分布式事务（TM 向 TC 注册全局事务记录）；
2. 按业务场景，编排数据库、服务等事务内资源（RM 向 TC 汇报资源准备状态 ）；
3. TM 结束分布式事务，事务一阶段结束（TM 通知 TC 提交/回滚分布式事务）；
4. TC 汇总事务信息，决定分布式事务是提交还是回滚；
5. TC 通知所有 RM 提交/回滚 资源，事务二阶段结束。