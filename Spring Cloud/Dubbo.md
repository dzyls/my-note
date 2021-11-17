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



## 自适应扩展点

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



