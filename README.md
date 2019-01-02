分析，在做秒杀系统的设计之初，一直在思考如何去设计这个秒杀系统，使之在现有的技术基础和认知范围内，能够做到最好；同时也能充分的利用公司现有的中间件来完成系统的实现。

我们都知道，正常去实现一个WEB端的秒杀系统，前端的处理和后端的处理一样重要；前端一般会做CDN，后端一般会做分布式部署，限流，性能优化等等一系列的操作，并完成一些网络的优化，比如IDC多线路（电信、联通、移动）的接入，带宽的升级等等。而由于目前系统前端是基于微信小程序，所以关于前端部分的优化就尽可能都是在代码中完成，CDN这一步就可以免了；

## 关于秒杀的更多思考，在原有的秒杀架构的基础上新增了新的实现方案
#### 原有方案：
    通过分布式锁的方式控制最终库存不超卖，并控制最终能够进入到下单环节的订单，入到队列中慢慢去消费下单
#### 新增方案“
    请求进来之后，通过活动开始判断和重复秒杀判断之后，即进入到消息队列，然后在消息的消费端去做库存判断等操作，通过消息队列达到削峰的操作
    
  其实，我觉得两种方案都是可以的，只是具体用在什么样的场景；原有方案更适合流量相对较小的平台，而且整个流程也会更加简单；而新增方案则是许多超大型平台采用的方案，通过消息队列达到削峰的目的；而这两种方案都加了真实能进入的请求限制，通过redis的原子自增来记录请求数，当请求量达到库存的n倍时，后面再进入的请求，则直接返回活动太火爆的提示；

1、架构介绍
后端项目是基于SpringCloud+SpringBoot搭建的微服务框架架构

前端在微信小程序商城上

### 核心支撑组件
- 服务网关 Zuul
- 服务注册发现 Eureka+Ribbon
- 认证授权中心 Spring Security OAuth2、JWTToken
- 服务框架 Spring MVC/Boot
- 服务容错 Hystrix
- 分布式锁 Redis
- 服务调用 Feign
- 消息队列 Kafka
- 文件服务 私有云盘
- 富文本组件 UEditor
- 定时任务 xxl-job
- 配置中心 apollo

2、关于秒杀的场景特点分析
#### 秒杀系统的场景特点
- 秒杀时大量用户会在同一时间同时进行抢购，网站瞬时访问流量激增；
- 秒杀一般是访问请求量远远大于库存数量，只有少部分用户能够秒杀成功；
- 秒杀业务流程比较简单，一般就是下订单操作；

 

#### 秒杀架构设计理念
- 限流：鉴于只有少部分用户能够秒杀成功，所以要限制大部分流量，只允许少部分流量进入服务后端（暂未处理）；
- 削峰：对于秒杀系统瞬时的大量用户涌入，所以在抢购开始会有很高的瞬时峰值。实现削峰的常用方法有利用缓存或者消息中间件等技术；
- 异步处理：对于高并发系统，采用异步处理模式可以极大地提高系统并发量，异步处理就是削峰的一种实现方式；
- 内存缓存：秒杀系统最大的瓶颈最终都可能会是数据库的读写，主要体现在的磁盘的I/O，性能会很低，如果能把大部分的业务逻辑都搬到缓存来处理，效率会有极大的提升；
- 可拓展：如果需要支持更多的用户或者更大的并发，将系统设计为弹性可拓展的，如果流量来了，拓展机器就好；

 

#### 秒杀设计思路
- 由于前端是属于小程序端，所以不存在前端部分的访问压力，所以前端的访问压力就无从谈起；
- 1、秒杀相关的活动页面相关的接口，所有查询能加缓存的，全部添加redis的缓存；
- 2、活动相关真实库存、锁定库存、限购、下单处理状态等全放redis；
- 3、当有请求进来时，首先通过redis原子自增的方式记录当前请求数，当请求超过一定量，比如说库存的10倍之后，后面进入的请求则直接返回活动太火爆的响应；而能进入抢购的请求，则首先进入活动ID为粒度的分布式锁，第一步进行用户购买的重复性校验，满足条件进入下一步，否则返回已下单的提示；
- 4、第二步，判断当前可锁定的库存是否大于购买的数量，满足条件进入下一步，否则返回已售罄的提示；
- 5、第三步，锁定当前请求的购买库存，从锁定库存中减除，并将下单的请求放入kafka消息队列；
- 6、第四步，在redis中标记一个polling的key（用于轮询的请求接口判断用户是否下订单成功），在kafka消费端消费完成创建订单之后需要删除该key，并且维护一个活动id+用户id的key，防止重复购买；
- 7、第五步，消息队列消费，创建订单，创建订单成功则扣减redis中的真实库存，并且删除polling的key。如果下单过程出现异常，则删除限购的key，返还锁定库存，提示用户下单失败；
- 8、第六步，提供一个轮询接口，给前端在完成抢购动作后，检查最终下订单操作是否成功，主要判断依据是redis中的polling的key的状态；
- 9、整个流程会将所有到后端的请求拦截的在redis的缓存层面，除了最终能下订单的库存限制订单会与数据库存在交互外，基本上无其他的交互，将数据库I/O压力降到了最低；

 

#### 关于限流

SpringCloud zuul的层面有很好的限流策略，可以防止同一用户的恶意请求行为

 1 zuul:
 2     ratelimit:
 3         key-prefix: your-prefix  #对应用来标识请求的key的前缀
 4         enabled: true
 5         repository: REDIS  #对应存储类型（用来存储统计信息）
 6         behind-proxy: true  #代理之后
 7         default-policy: #可选 - 针对所有的路由配置的策略，除非特别配置了policies
 8              limit: 10 #可选 - 每个刷新时间窗口对应的请求数量限制
 9              quota: 1000 #可选-  每个刷新时间窗口对应的请求时间限制（秒）
10               refresh-interval: 60 # 刷新时间窗口的时间，默认值 (秒)
11                type: #可选 限流方式
12                     - user
13                     - origin
14                     - url
15           policies:
16                 myServiceId: #特定的路由
17                       limit: 10 #可选- 每个刷新时间窗口对应的请求数量限制
18                       quota: 1000 #可选-  每个刷新时间窗口对应的请求时间限制（秒）
19                       refresh-interval: 60 # 刷新时间窗口的时间，默认值 (秒)
20                       type: #可选 限流方式
21                           - user
22                           - origin
23                           - url
 

#### 关于负载与分流

当一个活动的访问量级特别大的时候，可能从域名分发进来的nginx就算是做了高可用，但实际上最终还是单机在线，始终敌不过超大流量的压力时，我们可以考虑域名的多IP映射。也就是说同一个域名下面映射多个外网的IP，再映射到DMZ的多组高可用的nginx服务上，nginx再配置可用的应用服务集群来减缓压力；

这里也顺带介绍redis可以采用redis cluster的分布式实现方案，同时springcloud hystrix 也能有服务容错的效果；

而关于nginx、springboot的tomcat、zuul等一系列参数优化操作对于性能的访问提升也是至关重要；

补充说明一点，即使前端是基于小程序实现，但是活动相关的图片资源都放在自己的云盘服务上，所以活动前活动相关的图片资源上传CDN也是至关重要，否则哪怕是你IDC有1G的流量带宽，也会分分钟被吃完；
