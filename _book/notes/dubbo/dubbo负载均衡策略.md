###  dubbo 负载均衡策略和集群容错策略都有哪些？动态代理策略呢？ 

##### dubbo 负载均衡策略

###### random loadbalance

​	默认情况下，dubbo 是 random load balance ，即**随机**调用实现负载均衡，可以对 provider 不同实例**设置不同的权重**，会按照权重来负载均衡，权重越大分配流量越高，一般就用这个默认的就可以了。

###### roundrobin loadbalance

​	这个的话默认就是均匀地将流量打到各个机器上去，但是如果各个机器的性能不一样，容易导致性能差的机器负载过高。所以此时需要调整权重，让性能差的机器承载权重小一些，流量少一些。

举个栗子。

​	跟运维同学申请机器，有的时候，我们运气好，正好公司资源比较充足，刚刚有一批热气腾腾、刚刚做好的虚拟机新鲜出炉，配置都比较高：8 核 + 16G 机器，申请到 2 台。过了一段时间，我们感觉 2 台机器有点不太够，我就去找运维同学说，“哥儿们，你能不能再给我一台机器”，但是这时只剩下一台 4 核 + 8G 的机器。我要还是得要。

​	这个时候，可以给两台 8 核 16G 的机器设置权重 4，给剩余 1 台 4 核 8G 的机器设置权重 2。

###### leastactive loadbalance

​	这个就是自动感知一下，如果某个机器性能越差，那么接收的请求越少，越不活跃，此时就会给**不活跃的性能差的机器更少的请求**。

###### consistanthash loadbalance

​	一致性 Hash 算法，相同参数的请求一定分发到一个 provider 上去，provider 挂掉的时候，会基于虚拟节点均匀分配剩余的流量，抖动不会太大。**如果你需要的不是随机负载均衡**，是要一类请求都到一个节点，那就走这个一致性 Hash 策略。

### dubbo 集群容错策略

###### failover cluster 模式

​	失败自动切换，自动重试其他机器，**默认**就是这个，常见于读操作。（失败重试其它机器）

​	可以通过以下几种方式配置重试次数：

```xml
<dubbo:service retries="2" />Copy to clipboardErrorCopied
```

​	或者

```xml
<dubbo:reference retries="2" />Copy to clipboardErrorCopied
```

​	或者

```xml
<dubbo:reference>
    <dubbo:method name="findFoo" retries="2" />
</dubbo:reference>Copy to clipboardErrorCopied
```

###### failfast cluster 模式

​	一次调用失败就立即失败，常见于非幂等性的写操作，比如新增一条记录（调用失败就立即失败）

#### [failsafe cluster 模式](https://doocs.github.io/advanced-java/#/./docs/distributed-system/dubbo-load-balancing?id=failsafe-cluster-模式)

​	出现异常时忽略掉，常用于不重要的接口调用，比如记录日志。

​	配置示例如下：

```xml
<dubbo:service cluster="failsafe" />Copy to clipboardErrorCopied
```

​	或者

```xml
<dubbo:reference cluster="failsafe" />Copy to clipboardErrorCopied
```

#### failback cluster 模式

​	失败了后台自动记录请求，然后定时重发，比较适合于写消息队列这种。

#### [forking cluster 模式](https://doocs.github.io/advanced-java/#/./docs/distributed-system/dubbo-load-balancing?id=forking-cluster-模式)

​	**并行调用**[多个]() provider，只要一个成功就立即返回。常用于实时性要求比较高的读操作，但是会浪费更多的服务资源，可通过 `forks="2"` 来设置最大并行数。

###### broadcacst cluster

​	逐个调用所有的 provider。任何一个 provider 出错则报错（从`2.1.0` 版本开始支持）。通常用于通知所有提供者更新缓存或日志等本地资源信息。

###### dubbo动态代理策略

​	默认使用 javassist 动态字节码生成，创建代理类。但是可以通过 spi 扩展机制配置自己的动态代理策略。