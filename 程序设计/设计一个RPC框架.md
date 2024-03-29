### 如何设计一个RPC框架

---

如何设计一个RPC框架？

首先要考虑到，一个RPC框架使用的场景。

既然使用到RPC框架了，那就需要考虑高可用性，以及故障转移的机制。

**高可用**

从高可用的角度来讲，肯定是要做成集群。

一台服务端宕机，另外的服务端依然可以顺利接管服务。

这也要求了，服务调用之间，是无状态的。上一次的调用A服务端，下一次调用B服务端是不影响的。

**注册中心**

既然是集群，那肯定不能通过手工配置调用地址。那就需要一个调用地址。

可以选用zookeeper集群来做注册中心，也可以选用nacos或者consul，也可以自己搞一个Namesrv。



**负载均衡策略**

客户端从注册中心获取服务端的地址，肯定不能一直调用一个服务端，这样有可能造成某个服务端压力巨大，而其他服务端很空闲。

这就需要一个负载均衡策略，最简单的有两种 ：

1. 随机
2. 轮询

也可以写的复杂点，最短响应时间的优先，或者可配置权重的策略。



**故障转移**

故障转移比较简单，如果客户端调用某个服务端失败，是再次尝试调用？还是直接返回给上层？

这个要根据实际情况来看，有些接口是做了幂等性的，重复调用是不影响的，这种就可以根据负载均衡策略再选一个服务端来调用。

有些接口可能没有做幂等性，这种就需要具体情况具体分析了。

可以做成注解的形式，如果有注解，则重复调用。



**调用方式**

调用方式，是采用HTTP调用，还是Netty？

两种方式都可以，如果团队对Netty比较熟悉，可以用Netty；如果不熟悉，那用HTTP比较好了。



客户端通过动态代理来调用接口，服务端监听某个端口，收到请求时进行解析，然后调用对应的方法，返回对应的结果。



**序列化协议**

在传输协议上，可以选择一些效率高和带宽占用少的压缩协议。比如Kyro或者protoBuf。



**接口的顺序性考虑**

有些方法调用可能有顺序性的问题，比如先在插入新数据，再删除旧数据，但如果删除数据在前，可能会导致新旧两条数据都被删掉了，这种情况就很麻烦。

首先可以考虑使用一致性哈希，确保同一个ID的数据分布在同一个集群。但是又会有一个问题，比如哈希算法导致分布到某台机器的请求特别多怎么办？

最好的办法是从业务的角度来做，插入和删除做成一个方法，这样就不会有什么顺序性的问题了。

