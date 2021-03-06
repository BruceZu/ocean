传统Java Web(非Spring Boot)、非Java语言项目接入Spring Cloud方案
http://blog.csdn.net/neosmith/article/details/70049977
实施微服务，我们需要哪些基础框架？
http://blog.csdn.net/neosmith/article/details/52118930
Spring Cloud netflix中文文档
https://www.springcloud.cc/spring-cloud-netflix.html
https://www.springcloud.cc/spring-cloud-dalston.html

# 微服务
Microservice架构模式就是将整个Web应用组织为一系列小的Web服务。这些小的Web服务可以独立地编译及部署，并通过各自暴露的API接口相互通讯。它们彼此相互协作，作为一个整体为用户提供功能，却可以独立地进行扩容

我的理解Microservice是SOA的传承，但一个最本质的区别就在于Smart endpoints and dumb pipes，或者说是真正的分布式的、去中心化的。Smart endpoints and dumb pipes本质就是去ESB，
把所有的“思考”逻辑包括路由、消息解析等放在服务内部（Smart endpoints），去掉一个大一统的ESB，服务间轻（dumb pipes）通信，是比SOA更彻底的拆分

# Netflix的一个请求流程
```
请求A
 |--通过阿里云的负载均衡路由到某台zuul服务器上
 |--Tomcat(8默认使用nio接受，然后从线程池中获取一个线程进行处理)
     |--Zuul(有动态设置filter的接口)
         |----各种配置的Filter(授权，账号验证，合法性验证，请求监控等)
         |----pre,routing,post,error处理
         |--RibbonRoutingFilter进行rout时执行的run方法
             |---构建RibbonCommandContext(serviceId,retryable,request等核心信息)
             |   |----根据配置获取一个LoadBalancerContext(OkHttpLoadBalancingClient,RibbonLoadBalancingHttpClient等)
             |   |--Hystrix
             |     |----根据LoadBalancerContext生成一个HystrixCommand
             |---根据RibbonCommandContext构建RibbonCommand(HystrixExecutable)
                |---根据RibbonCommandFactory抽象工厂类构建RibbonCommand，子类OkHttpRibbonCommandFactory,HttpClientRibbonCommandFactory,RestClientRibbonCommandFactory
                |   |---调用SpringClientFactory.getClient获取LoadBalancerContext的子类OkHttpLoadBalancingClient,RibbonLoadBalancingHttpClient
                |   |---根据serviceId调用SpringClientFactory.getLoadBalancer获取ILoadBalancer
                |   |---生成RibbonCommand(接口),子类有OkHttpRibbonCommand,HttpClientRibbonCommand,RestClientRibbonCommand都继承HystrixCommand
                |
                |---执行RibbonCommand.execute方法进行跳转，实际执行HystrixCommand.execute-->执行toObservable()
                    |---一路执行到executeCommandWithSpecifiedIsolation方法(根据隔离策略执行如下代码THREAD或者SEMAPHORE)
                    |   |---metrics.markCommandStart方法记录开始
                    |   |---调用getUserExecutionObservable-->getExecutionObservable再执行run方法
                    |       |---在HystrixCommand的run中执行LoadBalancerContext.executeWithLoadBalancer【一直阻塞，直到返回结果】
                    |           |---使用构建器LoadBalancerCommand构建请求，执行submit提交
                    |               |---调用LoadBalancerCommand.selectServer根据规则获取目标服务器(LoadBalancerContext.getServerFromLoadBalancer)【服务器列表的获取怎么处理的？】
                    |               |---提交的Rxjava中的Observable中
                    |                   |---调用LoadBalancerContext中的notexx方法进行ServerStats的记录(连接数、失败连接数、总数等)
                    |                   |---调用LoadBalancerContext.execute方法-->调用子类OkHttpLoadBalancingClient,RibbonLoadBalancingHttpClient等execute
                    |                   |     |---构建HttpClient
                    |                   |     |---执行远程请求并返回结果
                    |                   |
                    |                   |---如果失败需要重试，调用Observable.retry执行重试
                    |
                    |---metrics.markCommandDone方法记录开始完成
```

# 6.服务路由
1. 有服务调用延迟，一致性哈希的负载均衡
1. 负载均衡、本地路由优先策略(服务消费者)、路由规则(全局规则，服务提供者规则、服务消 费者规则)
   三则是平等的关系，共同决定最终访问的服务器地址
```
配置同机器/服务器优先、配置同机房有限：
 1.服务器只有region之分，可以将同机器/同机房/同区域的配置成同一个，然后可以优先使用
 2.需要自己实现，在服务器注册的时候通过meta
```

# 7.容错策略
针对各个服务调用采取的失败后的处理策略；hystrix是针对调用失败后的处理结果？
```
Hystrix的fallback(如果配置了)会在断路circuit被打开(根据每次请求)、请求池满了(线程池、信号灯)、请求失败、请求超时的情况下调用
```
## 7.1.1. 通讯链路故障
1. 服务提供方突然宕机
```
1.调用方会尝试请求，并且记录结果，最终会打开断路；请求失败后会轮询尝试其他服务器
2.注册中心定时清理，将宕机的机器踢出，并且等待其他注册端定时更新
```
1. 网络发生闪断
```
调用方会尝试请求，并且记录结果，最终会打开断路；请求失败后会轮询尝试其他服务器
```
## 7.1.2 服务端超时
1. 服务端I/O线程没有即使从网络中读取到请求消息
1. 服务端业务处理缓慢，或长时间阻塞，如查询数据库，由于没有索引时间比较长等
1. 服务端发生长时间的Full GC，导致业务线程暂停运行，无法及时回应请求
## 7.1.3 服务调用失败
1. 服务端动态控流，返回的控流移除
1. 访问权限验证失败，返回权限相关异常
1. 服务端消息队列积压率超过最大阀值，返回系统租塞异常
## 7.2.1 失败自动切换(Failover)
服务调用失败自动切换策略指的是当发生RPC调用异常时，重新选路，查找下一个可用的服务提供者

客户端配置容错策略

场景：

1. 读操作，因为通常它是幂等的
1. 幂等性服务，保证调用1次和N次的效果是一样
## 7.2.2 失败通知(Failback)
失败了直接通知给消费者，由消费者捕获异常进行后续处理
## 7.2.2 失败缓存(Failcache)
失败后自动恢复
## 7.2.2 失败通知(Failback)
获取到服务调用异常后，直接忽略异常，记录异常日志，针对非核心服务在高峰期才去的限流等
# 8.服务调用
采用异步的方式调用，通过Future实现？

1. spring cloud的调用是采用异步还是同步的？每个节点(提供者、消费者等)上有hold的线程吗？他们是什么状态？
2. 消费方如果在同一个接口中要调用多个服务A（50ms），B(60ms) 那他们最终是多长？dubbo是最长的那个60ms
```
1.支持同步和异步调用，除hystrix上配置了线程池隔离，否则都是在请求的线程中完成
2.可以通过异步的方式调用A、B然后再等待结果，跟dubbo一样
```

# 9. 服务注册中心
1. 安全加固，通过username/password注册？
2. 调用通过固定或者注册中心动态生成的token方式调用
```
1.服务注册支持user/password
2.没有此功能
```

# 10 服务发布和引用
1. 怎么实现服务的灰度发布？
1. 怎么实现接口的热升级，是否需要暂停服务？
```
1.升级或者灰度发布可以将服务器改为out of service状态，然后再更新，可以灰度发布
```

# 14. 流量控制
1. 能否控制某个服务的流量，能否控制某个接口的流量？
## 14.2.1 动态控流因子
1. 应用进程所再主机VM的CPU使用率
 - 通过linux的top等命令查看什么进程使用CPU高
 - 通过jstack查看VM中线程快照情况，通过分析线程看看哪里有问题
1. 应用进程所再主机VM的内存使用率
 - 通过linux的top等命令查看什么进程使用CPU高
 - 通过jmap查看内存转储快照(heapdump文件)情况，通过分析快照看看哪里有问题
```
可以通过设置信号灯和线程数来控制流量
```

# 15. 服务降级
通过服务调用者的本地Mock（存根）实现

1. spring cloud是否支持服务降级？
```
无法指定某个接口服务进行降级，但是可以控制某台机器的流量
```
# 16. 服务优先调度
与动态控流不同，控流最终会拒绝消息，导致部分请求失败。优先级调度是再资源紧张时，优先执行高优先级的服务，在保障高优先级服务能够
被合理调度的同时，也兼顾处理部分优先级低的消息，他们之间的关系存在一定的比例关系


1. spring cloud是否支持？

# 18. 分布式消息追踪
1. 调用链路的记录和分析
1. 客户端埋点还是服务器端？
1. trace id的生成

# 19.可靠性设计
## 19.1 服务状态检测
1. 基于服务注册中心状态检测
1. 链路有效性状态检测机制
1. 服务健康度检测
1. 服务故障隔离
 - 进程(线程池)级故障隔离
 - VM级故障隔离
 - 物理机故障隔离
 - 机房故障隔离
