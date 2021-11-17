## 限流的作用和意义

限制一个时间范围内的访问数量，来保护系统，一旦达到阈值则采取拒绝策略，如降级、排队、拒绝。

通过这种方式，损失一部分用户的可用性，确保大部分用户的可用性。



**限流的例子**

在实际的开发中，使用限流的有很多：

- Nginx限流模块
- 设置数据库连接池、线程池大小来限制总的并发数
- Guava的RateLimiter
- 信号量限流



## 限流算法

常见的限流算法有：

- 固定窗口限流
- 滑动窗口限流
- 漏桶限流
- 令牌桶限流

下面一一介绍每种算法的优缺点。

**固定窗口限流**

固定窗口限流，就是一个计数器，定时去重置这个计数器。

优点就是简单，缺点是会有临界的问题，比如在59秒和在1:01秒都有100个请求，这样就会有两百个请求。



**滑动窗口限流**

滑动窗口就是对固定窗口进行更细粒度的细分，比如一分钟限流120个请求，那就分为60个窗口，每秒钟限流两个。每次只需要计算当前滑动窗口的总数是否大于阈值即可。



**漏桶限流**

漏桶算法是，维护一个容器，这个容器会以恒定的速度漏水，每次有新请求就判断当前容器是否可以存放就可以了。



漏桶算法是无法应对突发流量的，是用来平滑突发流量。

消息中间件就是使用了这种思想，当生产的速率大于消费速率，那么消息将会堆积在中间件中，消息的处理能力取决于消费者。



**令牌桶限流**

令牌桶算法是，维护一个容器，定时速率去向这个容器中存放令牌，如果有新请求，就判断请求的数量是否大于令牌桶中的令牌数量即可。



## 熔断和降级

在微服务系统中，调用链路可能会非常的长，如果某个节点响应缓慢或不可用，可能会导致请求阻塞，继而引发服务雪崩。

熔断通常是通过指标来判断是否需要熔断和降级：

- 如异常比例超过一定数量，触发熔断
- 异常数量超过阈值，触发熔断
- 平均响应时间较慢，也可以触发熔断



## Sentinel简介

Sentinel是一个轻量级的流量控制组件，使用滑动窗口算法，有限流、降级等功能，是替换Hystrix的框架。

**特性**

- Sentinel有丰富的应用场景，比如秒杀、削峰填谷【排队】、集群流量控制
- Dashboard 实时监控，实时看到应用单台的请求数
- 开源支持，不仅是Spring Cloud，还支持dubbo、gRPC
- SPI支持，可以自定义数据源，如，可以从数据库来获取限流，自己实现
- 调用链路



**Sentinel限流示例**

```java
public class MainApp {

    public static String HELLO_RESOURCE = "HelloResource";
	
    // 初始化限流规则
    public static void initRules(){
        List<FlowRule> rules = new ArrayList<>();
        FlowRule flowRule = new FlowRule();
        flowRule.setResource(HELLO_RESOURCE);
        flowRule.setGrade(RuleConstant.FLOW_GRADE_QPS);
        flowRule.setCount(5);
        rules.add(flowRule);
        FlowRuleManager.loadRules(rules);
    }

    public static void main(String[] args) throws InterruptedException {
        initRules();
        while (true){
            sayHello();
            TimeUnit.MILLISECONDS.sleep(100L);
        }
    }

    public static void sayHello(){
        try (Entry entry = SphU.entry(HELLO_RESOURCE)){
            System.out.println("Hello");
        } catch (BlockException e) {
            System.err.println("Block");
        }
    }

}
```



如上，使用是很简单的，初始化限流规则后，通过`SphU.entry`即可获取元素，如果成功获取，则执行try代码块的内容，如果没有则抛出异常。



***还有一个获取限流元素的方法***，如`SphO.entry`

```java
if (SphO.entry(HELLO_RESOURCE)) {
    try {
        System.out.println("hello");
    } finally {
        SphO.exit();
    }
} else {
    System.out.println("block");
}
```

通过`SphO`返回的是boolean值，需要特别注意的是，`SphO.exit`需要放在finally代码块中确保执行。

从执行效率上，try-catch效率是不如if-else快，但是通过`SphO`获取需要用finaly来确保释放一定执行，看起来代码很臃肿。



***@SentinelResource***注解也可以标注方法，用来标识这个方法是限流资源。



## Sentinel的FlowRule

```java
public static void initRules(){
    List<FlowRule> rules = new ArrayList<>();
    FlowRule flowRule = new FlowRule();
    flowRule.setResource(HELLO_RESOURCE);
    flowRule.setGrade(RuleConstant.FLOW_GRADE_QPS);
    flowRule.setLimitApp("default");
    flowRule.setStrategy(RuleConstant.STRATEGY_CHAIN);
    flowRule.setControlBehavior(RuleConstant.CONTROL_BEHAVIOR_DEFAULT);
    flowRule.setClusterMode(false);
    flowRule.setCount(5);
    rules.add(flowRule);
    FlowRuleManager.loadRules(rules);
}
```

上述代码是初始化一个FlowRule设置的一些属性，下面针对这些属性一一说明。

**Resource**

Resource是资源名称，也可通过`flowRule.setRefResource()`来设置一个引用类型的资源。



**Grade**

Grade是设定流量控制的类型，Sentinel中有两种流量控制类型：

- 基于QPS ：针对每秒的请求数，当QPS达到阈值，则触发限流策略
- 基于线程数 ：基于线程数，可以确保请求这个资源的数量不会超过阈值，导致其他请求没有线程处理。

```java
public static final int FLOW_GRADE_THREAD = 0;
public static final int FLOW_GRADE_QPS = 1;
```



**LimitApp**

是否根据调用源进行限流，默认是default，不对源进行限流。

- default ：不区分调用者，所有都会限流
- app_name ：会对指定的app_name进行限流
- other ：除了对app_name进行限流



**ControlBehavior**

达到阈值会怎么做 ：

- 直接拒绝 ：默认，超出阈值直接抛出一个异常
- 预热 ：突发流量不会直接拉到最大值，而是逐步提升阈值
- 匀速排队 ：让请求均匀的通过
- 匀速排队 + 预热  

```java
public static final int CONTROL_BEHAVIOR_DEFAULT = 0;
public static final int CONTROL_BEHAVIOR_WARM_UP = 1;
public static final int CONTROL_BEHAVIOR_RATE_LIMITER = 2;
public static final int CONTROL_BEHAVIOR_WARM_UP_RATE_LIMITER = 3;
```



## Sentinel降级

