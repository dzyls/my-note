## Dubbo

一个高性能的RPC框架。

支持多种协议，有负载均衡、监控的功能。



### RPC和HTTP

---

RPC（Remote Procedure Call）远程过程调用，是一种调用方式，与之相对的是本地调用。

RPC与HTTP不是一个层级的东西，HTTP是一个应用层的协议，而RPC是一个分布式系统之间的通信，RPC可以使用HTTP协议来传输，也可以自定义协议来传输。

HTTP协议比较冗余，RPC框架出于效率考虑，一般会采用自定义格式。不过也有使用HTTP协议的，毕竟内网不需要太考虑带宽。



## SPI 机制

SPI （service provider interface）：JDK内置的服务提供发现机制，主要是来做服务的扩展实现。

SPI机制在很多场景都有运用，比如数据库连接，Java中有个接口 `java.sql.Driver`，这个接口并没有实现，而是供不同的数据库厂商来实现。然后JDK通过扫描classpath路径下的对应的文件。

简单来说，就是通过扫描配置文件去做服务的扩展实现，方便使用者扩展。



## Dubbo的SPI

Dubbo并没有使用Java的SPI机制，主要是因为Java的SPI机制会将所有的配置文件遍历并将类实例化，假设有个类实例化比较慢比较吃资源，实例化后又不用就造成了资源浪费。Java的SPI无法按需加载。



Dubbo的SPI机制是通过`@Spi(key)`注解+配置文件来实现的，通过注解中的key可以快速找到扩展类。这样如果没有使用的key就不会加载，实现了按需加载。



## 扩展点

Dubbo SPI使用的是类`ExtensionLoader`，通过`ExtensionLoader`，可以实现加载指定的类。

Dubbo的SPI有两个规则：

1. 去指定目录下扫描加载指定的类，包括

   - META-INF/dubbo
   - META-INF/dubbo/internal
   - META-INF/services

   这三个目录

2. 文件的内容和Java SPI的不一样，是以key-value的形式。key是字符串，value是对应的实现的扩展类



**源码解析**

```java
public class SpiTest {

    public static void main(String[] args) {
        ExtensionLoader<HelloService> loader = ExtensionLoader.getExtensionLoader(HelloService.class);
        HelloService helloService = loader.getExtension("helloService");
        helloService.sayHello();
    }

}
```

如上是一个通过ExtensionLoader获取扩展类，并调用的代码示例。

第一步，获取ExtensionLoader：

```java
public static <T> ExtensionLoader<T> getExtensionLoader(Class<T> type) {
    if (type == null) {
        throw new IllegalArgumentException("Extension type == null");
    }
    if (!type.isInterface()) {
        throw new IllegalArgumentException("Extension type (" + type + ") is not an interface!");
    }
    if (!withExtensionAnnotation(type)) {
        throw new IllegalArgumentException("Extension type (" + type +
                ") is not an extension, because it is NOT annotated with @" + SPI.class.getSimpleName() + "!");
    }
	// 如果要获取的类的ExtensionLoader不存在，则创建一个，并放入缓存中
    ExtensionLoader<T> loader = (ExtensionLoader<T>) EXTENSION_LOADERS.get(type);
    if (loader == null) {
        EXTENSION_LOADERS.putIfAbsent(type, new ExtensionLoader<T>(type));
        loader = (ExtensionLoader<T>) EXTENSION_LOADERS.get(type);
    }
    return loader;
}
```

第二步，通过ExtensionLoader获取扩展类：

```java
public T getExtension(String name) {
    if (StringUtils.isEmpty(name)) {
        throw new IllegalArgumentException("Extension name == null");
    }
    if ("true".equals(name)) {
        return getDefaultExtension();
    }
    final Holder<Object> holder = getOrCreateHolder(name);
    Object instance = holder.get();
    // 双重检查机制
    if (instance == null) {
        synchronized (holder) {
            instance = holder.get();
            if (instance == null) {
                // 如果缓存中不存在，则创建一个扩展类
                instance = createExtension(name);
                holder.set(instance);
            }
        }
    }
    return (T) instance;
}
```

重点在createExtension方法里：

```java
private T createExtension(String name) {
    // getExtensionClasses会先从缓存中获取，如果没找到就会去文件里扫描啦
    Class<?> clazz = getExtensionClasses().get(name);
    if (clazz == null) {
        throw findException(name);
    }
    try {
        T instance = (T) EXTENSION_INSTANCES.get(clazz);
        if (instance == null) {
            // 不存在则反射创建
            EXTENSION_INSTANCES.putIfAbsent(clazz, clazz.newInstance());
            instance = (T) EXTENSION_INSTANCES.get(clazz);
        }
        // 依赖注入setter方法
        injectExtension(instance);
        // 包装类
        Set<Class<?>> wrapperClasses = cachedWrapperClasses;
        if (CollectionUtils.isNotEmpty(wrapperClasses)) {
            for (Class<?> wrapperClass : wrapperClasses) {
                instance = injectExtension((T) wrapperClass.getConstructor(type).newInstance(instance));
            }
        }
        return instance;
    } catch (Throwable t) {
        throw new IllegalStateException("Extension instance (name: " + name + ", class: " +
                type + ") couldn't be instantiated: " + t.getMessage(), t);
    }
}
```

getExtensionClasses方法还是比较清晰明了的：

```java
private Map<String, Class<?>> getExtensionClasses() {
    Map<String, Class<?>> classes = cachedClasses.get();
    if (classes == null) {
        synchronized (cachedClasses) {
            classes = cachedClasses.get();
            if (classes == null) {
                // 缓存中没找到就去文件扫描了
                classes = loadExtensionClasses();
                cachedClasses.set(classes);
            }
        }
    }
    return classes;
}

// synchronized in getExtensionClasses
private Map<String, Class<?>> loadExtensionClasses() {
    cacheDefaultExtensionName();

    Map<String, Class<?>> extensionClasses = new HashMap<>();
    // 扫描三个路径
    loadDirectory(extensionClasses, DUBBO_INTERNAL_DIRECTORY, type.getName());
    loadDirectory(extensionClasses, DUBBO_INTERNAL_DIRECTORY, type.getName().replace("org.apache", "com.alibaba"));
    loadDirectory(extensionClasses, DUBBO_DIRECTORY, type.getName());
    loadDirectory(extensionClasses, DUBBO_DIRECTORY, type.getName().replace("org.apache", "com.alibaba"));
    loadDirectory(extensionClasses, SERVICES_DIRECTORY, type.getName());
    loadDirectory(extensionClasses, SERVICES_DIRECTORY, type.getName().replace("org.apache", "com.alibaba"));
    return extensionClasses;
}
```

上述的代码用语言描述就是：

1. 先找实现类，如果缓存里没有，就扫描文件然后反射创建实例，放入缓存
2. 然后执行setter方法注入
3. 如果有包装类就要包装一次【包装类必须要有一个构造函数可以注入被包装的类】



## 自适应扩展点

自适应扩展点是使用@Adaptive注解，`@Adaptive`注解可以标注在方法上，也可以标注在类上。

标注在方法上，会针对这个方法生成一个只包含这个方法的代理类。

标注在类上，则只是标注这个类是自适应扩展的。

示例：

```java
ExtensionLoader<HelloService> loader = ExtensionLoader.getExtensionLoader(HelloService.class);
HelloService helloService = loader.getAdaptiveExtension();
helloService.sayHello();
```

**getAdaptiveExtension**

```java
public T getAdaptiveExtension() {
    // 从缓存中取，如果缓存不存在则创建。
    Object instance = cachedAdaptiveInstance.get();
    if (instance == null) {
        if (createAdaptiveInstanceError == null) {
            synchronized (cachedAdaptiveInstance) {
                instance = cachedAdaptiveInstance.get();
                if (instance == null) {
                    try {
                        instance = createAdaptiveExtension();
                        cachedAdaptiveInstance.set(instance);
                    } catch (Throwable t) {
                        createAdaptiveInstanceError = t;
                        throw new IllegalStateException("Failed to create adaptive instance: " + t.toString(), t);
                    }
                }
            }
        } else {
            throw new IllegalStateException("Failed to create adaptive instance: " + createAdaptiveInstanceError.toString(), createAdaptiveInstanceError);
        }
    }

    return (T) instance;
}
```

从缓存中取，如果缓存不存在则通过`createAdaptiveExtension`创建。



**createAdaptiveExtension**

```java
private T createAdaptiveExtension() {
    try {
        return injectExtension((T) getAdaptiveExtensionClass().newInstance());
    } catch (Exception e) {
        throw new IllegalStateException("Can't create adaptive extension " + type + ", cause: " + e.getMessage(), e);
    }
}

private Class<?> getAdaptiveExtensionClass() {
    // 这一步会判断类是否被@Adaptive标注，如果被标注，则使用cachedAdaptiveClass存储
    getExtensionClasses();
    if (cachedAdaptiveClass != null) {
        return cachedAdaptiveClass;
    }
    return cachedAdaptiveClass = createAdaptiveExtensionClass();
}
```

整个流程很清晰，如果缓存中有，则直接返回；如果没有则通过反射创建一个，放入缓存。

```java
private Class<?> createAdaptiveExtensionClass() {
    String code = new AdaptiveClassCodeGenerator(type, cachedDefaultName).generate();
    ClassLoader classLoader = findClassLoader();
    // 生成代码并编译
    org.apache.dubbo.common.compiler.Compiler compiler = ExtensionLoader.getExtensionLoader(org.apache.dubbo.common.compiler.Compiler.class).getAdaptiveExtension();
    return compiler.compile(code, classLoader);
}
```

对`@Adaptive`标注的方法，会动态生成代码并编译加载。

如Protocol的export方法会生成 :

```java
    public com.alibaba.dubbo.rpc.Exporter export(com.alibaba.dubbo.rpc.Invoker arg0) {
        if (arg0 == null) throw new IllegalArgumentException("com.alibaba.dubbo.rpc.Invoker argument == null");
        if (arg0.getUrl() == null)
            throw new IllegalArgumentException("com.alibaba.dubbo.rpc.Invoker argument getUrl() == null");
        // 通过Invoker的url属性获取对应的协议
        com.alibaba.dubbo.common.URL url = arg0.getUrl();
        String extName = (url.getProtocol() == null ? "dubbo" : url.getProtocol());
        if (extName == null)
            throw new IllegalStateException("Fail to get extension(com.alibaba.dubbo.rpc.Protocol) name from url(" + url.toString() + ") use keys([protocol])");
        // 通过Invoker的url参数动态的获取协议
        com.alibaba.dubbo.rpc.Protocol extension = (com.alibaba.dubbo.rpc.Protocol) ExtensionLoader.getExtensionLoader(com.alibaba.dubbo.rpc.Protocol.class).getExtension(extName);
        return extension.export(arg0);
    }
```

如上，生成的代码，会动态的从Invoker中获取Url参数，动态的获取协议，执行暴露服务的操作。



示例：

```java
@SPI
public interface IProtocol {

    @Adaptive({"protocol"})
    void importUrl(URL url);

}
```

```java
public class MyProtocol implements IProtocol {

    @Override
    public void importUrl(URL param) {
        System.out.println(this.getClass().getSimpleName());
    }

}
```

```java
IProtocol protocol = ExtensionLoader.getExtensionLoader(IProtocol.class).getAdaptiveExtension();
URL url = new URL("my","127.0.0.1",8080);
protocol.importUrl(url);
```





## 服务暴露过程

Dubbo使用了URL作为配置总线，因此在Dubbo里经常可以见到URL的配置和解析。

Dubbo的服务暴露过程，其实就三大步 ：

- 检测配置，有些配置为空会默认创建，并组装成URL
- 暴露服务，包括本地暴露和远程暴露
- 注册至服务注册中心



```java
public class ServiceBean<T>  extends ServiceConfig<T> implements ApplicationListener<ContextRefreshedEvent> {
    @Override
    public void onApplicationEvent(ContextRefreshedEvent event) {
        if (!isExported() && !isUnexported()) {
            export();
        }
    }
    
    // 最终执行的是父类的export方法
    @Override
    public void export() {
        super.export();
        // Publish ServiceBeanExportedEvent
        publishExportEvent();
    }
}
```

ServiceBean实现了`ApplicationListener<ContextRefreshedEvent>` ，当Spring触发容器刷新时间时，会触发ServiceBean执行export暴露。

ServiceBean是一个具体的Service，会有具体的service的信息，无论是使用注解标注还是xml配置的。



最终还是执行的是父类serviceConfig的export方法 ：

```java
public class ServiceConfig<T> extends AbstractServiceConfig {
    
    public synchronized void export() {
        // 进行检查
        checkAndUpdateSubConfigs();

        if (!shouldExport()) {
            return;
        }
		
        // 如果不是延迟加载
        if (shouldDelay()) {
            DELAY_EXPORT_EXECUTOR.schedule(this::doExport, getDelay(), TimeUnit.MILLISECONDS);
        } else {
            doExport();
        }
    }
    
    protected synchronized void doExport() {
        if (unexported) {
            throw new IllegalStateException("The service " + interfaceClass.getName() + " has already unexported!");
        }
        if (exported) {
            return;
        }
        exported = true;

        if (StringUtils.isEmpty(path)) {
            path = interfaceName;
        }
        // 最终执行的是doExportUrls
        doExportUrls();
    }
    
    private void doExportUrls() {
        // 获取所有的注册中心，注册中心可以有多个
        List<URL> registryURLs = loadRegistries(true);
        for (ProtocolConfig protocolConfig : protocols) {
            // 遍历多种协议，每种协议都向这些注册中心注册
            doExportUrlsFor1Protocol(protocolConfig, registryURLs);
        }
    }
}
```

`loadRegistries`方法最终拼接的是如下的url ：

`registry://localhost:9090/org.apache.dubbo.registry.RegistryService?application=dubboTest&dubbo=2.0.2&pid=13236&qos.enable=false&registry=spring-cloud&release=2.7.3&timestamp=1637221991818`



最终是doExportUrlsForProtocol方法 ：

```java
doExportUrlsForProtocol (){
    // 省略一大段拼接URL的代码
	// export service
    URL url = new URL(name, host, port, getContextPath(protocolConfig).map(p -> p + "/" + path).orElse(path), map);
    
    // 获取配置的scope，scope有两种local或remote，默认remote
    String scope = url.getParameter(SCOPE_KEY);
    
    // don't export when none is configured
    if (!SCOPE_NONE.equalsIgnoreCase(scope)) {
    // export to local if the config is not remote (export to remote only when config is remote)
	      if (!SCOPE_REMOTE.equalsIgnoreCase(scope)) {
              // 暴露到本地
                exportLocal(url);
           }
        
    
}
```

```java
private void doExportUrlsFor1Protocol(ProtocolConfig protocolConfig, List<URL> registryURLs) {
   	// 省略一大段拼接URL的步骤，下面是重点了
    URL url = new URL(name, host, port, getContextPath(protocolConfig).map(p -> p + "/" + path).orElse(path), map);

    String scope = url.getParameter(SCOPE_KEY);
    // don't export when none is configured
    if (!SCOPE_NONE.equalsIgnoreCase(scope)) {

        // export to local if the config is not remote (export to remote only when config is remote)
        if (!SCOPE_REMOTE.equalsIgnoreCase(scope)) {
            exportLocal(url);
        }
        // export to remote if the config is not local (export to local only when config is local)
        if (!SCOPE_LOCAL.equalsIgnoreCase(scope)) {
			// 遍历向每个注册中心暴露服务
            if (CollectionUtils.isNotEmpty(registryURLs)) {
                for (URL registryURL : registryURLs) {
                    //if protocol is only injvm ,not register
                    if (LOCAL_PROTOCOL.equalsIgnoreCase(url.getProtocol())) {
                        continue;
                    }
                    url = url.addParameterIfAbsent(DYNAMIC_KEY, registryURL.getParameter(DYNAMIC_KEY));
                    // 加载监控的地址
                    URL monitorUrl = loadMonitor(registryURL);
                    if (monitorUrl != null) {
                        url = url.addParameterAndEncoded(MONITOR_KEY, monitorUrl.toFullString());
                    }


                    // For providers, this is used to enable custom proxy to generate invoker
                    String proxy = url.getParameter(PROXY_KEY);
                    if (StringUtils.isNotEmpty(proxy)) {
                        registryURL = registryURL.addParameter(PROXY_KEY, proxy);
                    }
					// 生成invoker
                    Invoker<?> invoker = PROXY_FACTORY.getInvoker(ref, (Class) interfaceClass, registryURL.addParameterAndEncoded(EXPORT_KEY, url.toFullString()));
                    // 再包装一层
                    DelegateProviderMetaDataInvoker wrapperInvoker = new DelegateProviderMetaDataInvoker(invoker, this);
					// 生成Exporter，向
                    Exporter<?> exporter = protocol.export(wrapperInvoker);
                    exporters.add(exporter);
                }
            } else {
                // 如果没有注册中心，则直接暴露了
                Invoker<?> invoker = PROXY_FACTORY.getInvoker(ref, (Class) interfaceClass, url);
                DelegateProviderMetaDataInvoker wrapperInvoker = new DelegateProviderMetaDataInvoker(invoker, this);

                Exporter<?> exporter = protocol.export(wrapperInvoker);
                exporters.add(exporter);
            }
            /**
             * @since 2.7.0
             * ServiceData Store
             */
            MetadataReportService metadataReportService = null;
            if ((metadataReportService = getMetadataReportService()) != null) {
                metadataReportService.publishProvider(url);
            }
        }
    }
    this.urls.add(url);
}
```

先看一下，exportLocal ：

```java
private void exportLocal(URL url) {
    // 转换为injvm的本地协议了
    URL local = URLBuilder.from(url)
            .setProtocol(LOCAL_PROTOCOL)
            .setHost(LOCALHOST_VALUE)
            .setPort(0)
            .build();
    // 本地暴露
    Exporter<?> exporter = protocol.export(
            PROXY_FACTORY.getInvoker(ref, (Class) interfaceClass, local));
    exporters.add(exporter);
    logger.info("Export dubbo service " + interfaceClass.getName() + " to local registry url : " + local);
}
```

injvm协议类似于下 ：

`injvm://127.0.0.1/com.dzyls.api.HelloService?anyhost=true&application=dubboTest&bean.name=ServiceBean:com.dzyls.api.HelloService&bind.ip=192.168.99.1&bind.port=20880&deprecated=false&dubbo=2.0.2&dynamic=true&generic=false&interface=com.dzyls.api.HelloService&methods=hello&pid=13236&qos.enable=false&register=true&release=2.7.3&side=provider&timestamp=1637223528156`

再之后是 ：

```java
Exporter<?> exporter = protocol.export(
        PROXY_FACTORY.getInvoker(ref, (Class) interfaceClass, local));
```
`PROXY_FACTORY.getInvoker(ref, (Class) interfaceClass, local)`会用javassist创建invoker，使用javassist是因为比JDK快。

**Invoker**

```java
public interface Invoker<T> extends Node {

    Class<T> getInterface();

    Result invoke(Invocation invocation) throws RpcException;

}
```

invoker接口就定义了两个方法 ：getInterface和invoke。

使用invoker，是为了屏蔽调用的细节，统一执行的步骤。这样调用者，使用invoker发起调用，既可以发起本地，也可以发送远程调用。



**protocol.export()**

protocol.export方法是一个自适应扩展点的一个应用，Protocol接口 ：

```java
@SPI("dubbo")
public interface Protocol {


    int getDefaultPort();

    @Adaptive
    <T> Exporter<T> export(Invoker<T> invoker) throws RpcException;

    @Adaptive
    <T> Invoker<T> refer(Class<T> type, URL url) throws RpcException;

    void destroy();

}
```

export和refer方法是被`@Adaptive`注解标注的，因此会生成代理类，代理类会根据Invoker中的URL参数获得对应的协议，然后Dubbo的SPI机制选择对应的实现类执行真正的export方法。



**为什么要暴露在本地**

在本地暴露了，避免了网络传输，直接调用本地的就行。



本地暴露完之后，就是搞远程暴露了 :

```java
Invoker<?> invoker = PROXY_FACTORY.getInvoker(ref, (Class) interfaceClass, registryURL.addParameterAndEncoded(EXPORT_KEY, url.toFullString()));

DelegateProviderMetaDataInvoker wrapperInvoker = new DelegateProviderMetaDataInvoker(invoker, this);

Exporter<?> exporter = protocol.export(wrapperInvoker);
```

第一步是获取Invoker。

registyURL 格式如下 ：

`registry://localhost:9090/org.apache.dubbo.registry.RegistryService?application=dubboTest&dubbo=2.0.2&pid=19824&qos.enable=false&registry=spring-cloud&release=2.7.3&timestamp=1637233051363`

协议名是registry，Dubbo会自适应调用`RegistryProtocol`，去向注册中心注册服务，并开启本地端口暴露服务。



**RegistryProtocol#export()**

```java
@Override
public <T> Exporter<T> export(final Invoker<T> originInvoker) throws RpcException {
    URL registryUrl = getRegistryUrl(originInvoker);
    // url to export locally
    // 从invoker获取provider的URL，此处获取的是dobbo://
    URL providerUrl = getProviderUrl(originInvoker);

    // 省略部分代码
    //export invoker
    // 开启本地服务
    final ExporterChangeableWrapper<T> exporter = doLocalExport(originInvoker, providerUrl);
    

    // url to registry
    // 去注册中心去注册
    final Registry registry = getRegistry(originInvoker);
    final URL registeredProviderUrl = getRegisteredProviderUrl(providerUrl, registryUrl);
    ProviderInvokerWrapper<T> providerInvokerWrapper = ProviderConsumerRegTable.registerProvider(originInvoker,
            registryUrl, registeredProviderUrl);
    //to judge if we need to delay publish
    boolean register = registeredProviderUrl.getParameter("register", true);
    if (register) {
        register(registryUrl, registeredProviderUrl);
        providerInvokerWrapper.setReg(true);
    }

    // Deprecated! Subscribe to override rules in 2.6.x or before.
    registry.subscribe(overrideSubscribeUrl, overrideSubscribeListener);

    exporter.setRegisterUrl(registeredProviderUrl);
    exporter.setSubscribeUrl(overrideSubscribeUrl);
    //Ensure that a new exporter instance is returned every time export
    return new DestroyableExporter<>(exporter);
}
```

`RegistyProtocol`的export先是获取到providerUrl，接着调用`doLocalExport`启动本地服务。

```java
private <T> ExporterChangeableWrapper<T> doLocalExport(final Invoker<T> originInvoker, URL providerUrl) {
    String key = getCacheKey(originInvoker);

    return (ExporterChangeableWrapper<T>) bounds.computeIfAbsent(key, s -> {
        Invoker<?> invokerDelegate = new InvokerDelegate<>(originInvoker, providerUrl);
        // 再次调用export
        return new ExporterChangeableWrapper<>((Exporter<T>) protocol.export(invokerDelegate), originInvoker);
    });
}
```

在doLocalExport中依旧调用了export方法，去开启本地的端口。默认情况下，会根据URL自适应到DubboProtocol#export。

**DubboProtocol#export**

```java
@Override
public <T> Exporter<T> export(Invoker<T> invoker) throws RpcException {
    URL url = invoker.getUrl();

    // export service.
    String key = serviceKey(url);
    // 用exporter包装了一下invoker
    DubboExporter<T> exporter = new DubboExporter<T>(invoker, key, exporterMap);
    // key => com.dzyls.api.HelloService:20880
    // 此处放入了exporterMap，通过exporterMap的key可以获取到exporter和invoker
    exporterMap.put(key, exporter);

    //省略部分代码
	
    // 打开端口，默认是Netty
    openServer(url);
    optimizeSerialization(url);

    return exporter;
}
```

用DubboExporter包装了一下invoker，并把exporter放入了DubboProtocol的exporterMap，key是接口全限定名 + 接口，value是exporter。

**openServer**

```
private void openServer(URL url) {
    // find server.
    String key = url.getAddress();
    //client can export a service which's only for server to invoke
    boolean isServer = url.getParameter(IS_SERVER_KEY, true);
    if (isServer) {
        ExchangeServer server = serverMap.get(key);
        // 经典DCL检查
        if (server == null) {
            synchronized (this) {
                server = serverMap.get(key);
                if (server == null) {
                    serverMap.put(key, createServer(url));
                }
            }
        } else {
            // server supports reset, use together with override
            server.reset(url);
        }
    }
}

private ExchangeServer createServer(URL url) {
    
    // 省略拼接参数的代码

    ExchangeServer server;
    try {
    	// 根据URL创建server
        server = Exchangers.bind(url, requestHandler);
    } catch (RemotingException e) {
        throw new RpcException("Fail to start server(url: " + url + ") " + e.getMessage(), e);
    }

	// 省略部分代码

    return server;
}
```

通过URL创建server。



**Filter**

Dubbo的spi机制会将wrapper结尾的类缓存起来，当加载具体某个类会包装实现类。

Dubbo有个`ProtocolFilterWrapper`也实现了Protocol接口，在它的export方法中，会将filter一一添加。

```java
@Override
public <T> Exporter<T> export(Invoker<T> invoker) throws RpcException {
    // 如果是registry协议，则调用registryProtocol的export
    if (REGISTRY_PROTOCOL.equals(invoker.getUrl().getProtocol())) {
        return protocol.export(invoker);
    }
    // 否则添加filter
    return protocol.export(buildInvokerChain(invoker, SERVICE_FILTER_KEY, CommonConstants.PROVIDER));
}
```

```java
private static <T> Invoker<T> buildInvokerChain(final Invoker<T> invoker, String key, String group) {
    Invoker<T> last = invoker;
    List<Filter> filters = ExtensionLoader.getExtensionLoader(Filter.class).getActivateExtension(invoker.getUrl(), key, group);

    if (!filters.isEmpty()) {
        for (int i = filters.size() - 1; i >= 0; i--) {
            final Filter filter = filters.get(i);
            final Invoker<T> next = last;
            last = new Invoker<T>() {

                @Override
                public Class<T> getInterface() {
                    return invoker.getInterface();
                }

                @Override
                public URL getUrl() {
                    return invoker.getUrl();
                }

                @Override
                public boolean isAvailable() {
                    return invoker.isAvailable();
                }

                @Override
                public Result invoke(Invocation invocation) throws RpcException {
                    Result asyncResult;
                    try {
                        asyncResult = filter.invoke(next, invocation);
                    } catch (Exception e) {
                        // onError callback
                        if (filter instanceof ListenableFilter) {
                            Filter.Listener listener = ((ListenableFilter) filter).listener();
                            if (listener != null) {
                                listener.onError(e, invoker, invocation);
                            }
                        }
                        throw e;
                    }
                    return asyncResult;
                }

                @Override
                public void destroy() {
                    invoker.destroy();
                }

                @Override
                public String toString() {
                    return invoker.toString();
                }
            };
        }
    }

    return new CallbackRegistrationInvoker<>(last, filters);
}
```



**总结**

Dubbo的服务暴露开始于Spring容器刷新的时候，ServiceBean调用export执行父类ServiceConfig的export方法，会先获取注册中心的地址，之后是本地暴露【exportLocal】，调用proxyFactory.getInvoker使用javassit生成Invoker，然后Dubbo根据injvm调用对应的Protocol的export方法。在之后是远程暴露，远程暴露会先调用RegistryProtocol的export，继而执行doLocalExport，会调用DubboProtocal启动本地的Server，并把invoker用exporter包装了一下，放入了map中，key是接口的全限定名+端口，value是exporter。然后RegistryProtocol的export方法会向注册中心去注册。

时间 ：Spring容器刷新时。

入口 ：serviceBean的export方法，父类的serviceConfig的export方法

第一步 ：先执行本地暴露，serviceConfig调用exportLocal，走的是InjvmProtocol的export方法，会调用javassit生成invoker

第二步 ：再执行远程暴露，serviceConfig调用RegistryProtocol的export，export会去先调用doLocalExport方法，这一步会调用DubboProtocol的export方法，开启NettyServer

第三步 ：RegistryProtocol接着将服务记录到ProviderConsumerRegTable，然后会去注册中心去注册服务。



## 服务引用过程

服务引用过程大致就是，调用者去注册中心获取提供者的列表，然后封装成一个个invoker代理类，通过这个代理类去和提供者交互。

服务引用的入口是 ：`RenferenceBean#afterPropertiesSet()`

`RenferenceBean`类继承了ReferenceConfig，实现了 `FactoryBean`，并重写了`FactoryBean#getObject()`，则`getObject`方法里调用了父类的`RenerfenceConfig#get()`。

`Dubbo`有两种加载方式 ：懒汉式和饿汉式。

默认为懒汉式，懒汉式是用到才会触发加载，这里是用了`FactoryBean#getObject`去加载的。

```java
@Override
public Object getObject() {
    // 懒汉式，用到时才调用RenerfenceConfig#get()加载
    return get();
}

@Override
@SuppressWarnings({"unchecked"})
public void afterPropertiesSet() throws Exception {
    //省略一大块代码
    
    // 饿汉式，直接去调用getObject加载    
    if (shouldInit()) {
        getObject();
    }
}
```

> 复习一下Spring的BeanFactory和FactoryBean ：
>
> BeanFactory是IOC容器，FactoryBean是Bean，是归BeanFactory管的。
>
> FactoryBean接口可以用来对Bean进行封装，当真正要获取这个Bean时，会调用FactoryBean#getObject()，可以在这个方法中进行一些封装操作。



父类的`RenferenceConfig#get()`会调用`init`操作。

上源码 ：

```java
private void init() {
    if (initialized) {
        return;
    }
    checkStubAndLocal(interfaceClass);
    checkMock(interfaceClass);
    // 省略拼接map的操作

    // 重点来了 ： 创建代理
    ref = createProxy(map);

    String serviceKey = URL.buildKey(interfaceName, group, version);
    ApplicationModel.initConsumerModel(serviceKey, buildConsumerModel(serviceKey, attributes));
    initialized = true;
}
```





```java
private T createProxy(Map<String, String> map) {
    // 先判断是不是本地服务
    if (shouldJvmRefer(map)) {
        // 如果是本地服务就去暴露服务的Protocol的exporterMap中取出invoker了
        URL url = new URL(LOCAL_PROTOCOL, LOCALHOST_VALUE, 0, interfaceClass.getName()).addParameters(map);
        invoker = REF_PROTOCOL.refer(interfaceClass, url);
        
    } else {
        urls.clear(); // reference retry init will add url to urls, lead to OOM
        // 配置了url，要么是点对点，要么是注册中心
        if (url != null && url.length() > 0) { // user specified URL, could be peer-to-peer address, or register center's address.
            String[] us = SEMICOLON_SPLIT_PATTERN.split(url);
            if (us != null && us.length > 0) {
                for (String u : us) {
                   	// 删除循环处理放入容器的代码
                }
            }
        } else { // assemble URL from register center's configuration
            // if protocols not injvm checkRegistry
            // 配置了注册中心
            if (!LOCAL_PROTOCOL.equalsIgnoreCase(getProtocol())){
                checkRegistry();
                // 加载注册中心
                List<URL> us = loadRegistries(false);
                if (CollectionUtils.isNotEmpty(us)) {
                    for (URL u : us) {
                        // 拼接URL，并放入urls
                    }
                }
                if (urls.isEmpty()) {
                    throw new IllegalStateException();
                }
            }
        }

        if (urls.size() == 1) {
            // 如果只有一个URL，直接获取invoker了
            invoker = REF_PROTOCOL.refer(interfaceClass, urls.get(0));
        } else {
            List<Invoker<?>> invokers = new ArrayList<Invoker<?>>();
            URL registryURL = null;
            // 多个url 转换为 invoker
            for (URL url : urls) {
                invokers.add(REF_PROTOCOL.refer(interfaceClass, url));
                if (REGISTRY_PROTOCOL.equals(url.getProtocol())) {
                    registryURL = url; // use last registry url
                }
            }
            if (registryURL != null) { // registry url is available
                // use RegistryAwareCluster only when register's CLUSTER is available
                URL u = registryURL.addParameter(CLUSTER_KEY, RegistryAwareCluster.NAME);
                // The invoker wrap relation would be: RegistryAwareClusterInvoker(StaticDirectory) -> FailoverClusterInvoker(RegistryDirectory, will execute route) -> Invoker
                // 多个invoker合并，只暴露出一个invoker
                invoker = CLUSTER.join(new StaticDirectory(u, invokers));
            } else { // not a registry url, must be direct invoke.
                invoker = CLUSTER.join(new StaticDirectory(invokers));
            }
        }
    }

    // 省略部分代码
    // create service proxy
    // 通过invoker获取代理
    return (T) PROXY_FACTORY.getProxy(invoker);
}
```



**总结**

服务的引用过程就是获得代理对象的过程。起始于ReferenceBean的afterPropertiesSet方法中，在这个方法中会调用父类的ReferenceConfig的init方法执行初始化，在初始化方法里，会创建代理。创建代理时，会先判断是否是本地服务，如果是本地服务那么就去缓存里面去找Invoker；如果是远程服务，那么就会从注册中心中获取服务提供者的信息，然后封装为Invoker对象，如果有多个Invoker会合并成一个。然后通过获得的Invoker对象创建代理。

在从注册中心获得服务提供者的地址后，DubboProtocol#refer在创建Invoker对象时，会使用NettyClient进行通信。



大致的流程是，先检查配置，构建一个URL，通过URL自适应扩展调用protocol.refer得到相应的invoker。如果有多个invoker，就合并为一个invoker以供调用。然后再通过invoker获取代理，这个代理就是被调用的对象。





## 服务调用过程

**服务引用猜想**

服务引用的过程，大致是从注册中心获取服务的URL，生成本地的invoker代理对象，当调用具体的方法时，使用从本地的服务列表中通过负载均衡算法选出一个服务提供者进行调用，使用Netty将参数传给provider。



**约定协议**

RPC框架生产者和消费者交互要有一个固定的应用层协议，常见自定义的应用层协议通常有三种玩法：

- 固定长度 ：会有一些浪费，长度定的太短不够用，太长又浪费
- 特殊字符隔断 ：比较自由
- header + body ：头部固定长度，并且头部会写明body的长度，body长度不固定，这样就很方便了

dubbo就是属于header + body