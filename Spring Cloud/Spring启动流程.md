### Spring启动流程

---

1. 首先会进行扫描，将所有要加载进容器的对象封装为beanDefintinion，并存入Map中
2. 筛选出所有非懒加载的单例BeanDefintinion，进行创建Bean操作。（多例是访问时才创建）
3. 利用BeanDefintinion创建Bean就是bean的生命周期。
4. 创建Bean之后，Spring会发布一个容器启动的事件（便于程序员自己扩展）



```java
public AnnotationConfigApplicationContext(String... basePackages) {
   // 初始化reader和scanner
   this();
   // 扫描所有需要加载的bean，封装为BeanDefintinon,注册到容器中 
   scan(basePackages);
   // 刷新操作
   refresh();
}

// 初始化reader、sacnner
public AnnotationConfigApplicationContext(DefaultListableBeanFactory beanFactory) {
	super(beanFactory);
	this.reader = new AnnotatedBeanDefinitionReader(this);
	this.scanner = new ClassPathBeanDefinitionScanner(this);
}
```

在AnnoationConfigApplicationContext中，会首先进行reader和scanner的初始化。在初始化reader和scanner时，会将自己当做BeanDefinitionRegistry传进构造参数中去。

两者的区别见下：

[Reader和Scanner的区别]: #AnnotationBeanDefinationReader和ClassPathBeanDefinitionScanner



初始化完Reader和Scanner之后，就会执行扫描或注册，根据传参的不同，执行扫描或注册。

```java
public AnnotationConfigApplicationContext(Class<?>... componentClasses) {
   this();
   // 注册指定的类
   register(componentClasses);
   refresh();
}

public AnnotationConfigApplicationContext(String... basePackages) {
   this();
   // 扫描指定包路径
   scan(basePackages);
   refresh();
}
```

如常用的调用方式 ：

```java
AnnotationConfigApplicationContext context = new AnnotationConfigApplicationContext("com.dzyls.xxx.demo");
```

但无论是扫描或者是注册，都会将要加载的类封装为BeanDefinition，每扫描一个bean都会放入当前容器中的。

```java
protected Set<BeanDefinitionHolder> doScan(String... basePackages) {
   Assert.notEmpty(basePackages, "At least one base package must be specified");
   Set<BeanDefinitionHolder> beanDefinitions = new LinkedHashSet<>();
   for (String basePackage : basePackages) {
      Set<BeanDefinition> candidates = findCandidateComponents(basePackage);
      for (BeanDefinition candidate : candidates) {
         ScopeMetadata scopeMetadata = this.scopeMetadataResolver.resolveScopeMetadata(candidate);
         candidate.setScope(scopeMetadata.getScopeName());
         String beanName = this.beanNameGenerator.generateBeanName(candidate, this.registry);
         if (candidate instanceof AbstractBeanDefinition) {
            postProcessBeanDefinition((AbstractBeanDefinition) candidate, beanName);
         }
         if (candidate instanceof AnnotatedBeanDefinition) {
            AnnotationConfigUtils.processCommonDefinitionAnnotations((AnnotatedBeanDefinition) candidate);
         }
         if (checkCandidate(beanName, candidate)) {
            BeanDefinitionHolder definitionHolder = new BeanDefinitionHolder(candidate, beanName);
            definitionHolder =
                  AnnotationConfigUtils.applyScopedProxyMode(scopeMetadata, definitionHolder, this.registry);
            beanDefinitions.add(definitionHolder);
            // 放入当前的容器中。this.registry就是当前容器，在ClassPathBeanDefinition构造函数中，
             //AnnotationConfigurationApplicationContext将自己当做参数传进来了。
            registerBeanDefinition(definitionHolder, this.registry);
         }
      }
   }
   return beanDefinitions;
}
```

将所有BeanDefinition扫描到容器后，就开始执行刷新了，刷新也是最为复杂的一步。



```java
@Override
public void refresh() throws BeansException, IllegalStateException {
	synchronized (this.startupShutdownMonitor) {
		StartupStep contextRefresh = this.applicationStartup.start("spring.context.refresh");

		// Prepare this context for refreshing.
        // 准备刷新
		prepareRefresh();

		// Tell the subclass to refresh the internal bean factory.
        // 获取初始化时创建的BeanFactory
		ConfigurableListableBeanFactory beanFactory = obtainFreshBeanFactory();

		// Prepare the bean factory for use in this context.
        // beanFactory的预处理工作，会向容器中添加组件
		prepareBeanFactory(beanFactory);

		try {
			// Allows post-processing of the bean factory in context subclasses.
            // 留给子类实现的方法，可以在beanFactory添加组件后执行一些操作
			postProcessBeanFactory(beanFactory);

			StartupStep beanPostProcess = this.applicationStartup.start("spring.context.beans.post-process");
			// Invoke factory processors registered as beans in the context.
            // beanFactory初始化之后，执行BeanFactory的后处理器
			invokeBeanFactoryPostProcessors(beanFactory);

			// Register bean processors that intercept bean creation.
            // 注册bean的后处理器
			registerBeanPostProcessors(beanFactory);
			beanPostProcess.end();

			// Initialize message source for this context.
            // 国际化相关
			initMessageSource();

			// Initialize event multicaster for this context.
            // 初始化事件分发器
			initApplicationEventMulticaster();

			// Initialize other special beans in specific context subclasses.
            // 留给子类实现的方法
			onRefresh();

			// Check for listener beans and register them.
            // 注册事件监听器
			registerListeners();

			// Instantiate all remaining (non-lazy-init) singletons.
            // 完成BeanFactory的初始化
			finishBeanFactoryInitialization(beanFactory);

			// Last step: publish corresponding event.
            // 发布容器刷新事件
			finishRefresh();
		}

		catch (BeansException ex) {
			// 省略异常情况处理
		}

		finally {
			// Reset common introspection caches in Spring's core, since we
			// might not ever need metadata for singleton beans anymore...
			resetCommonCaches();
			contextRefresh.end();
		}
	}
}
```



### AnnotationBeanDefinationReader和ClassPathBeanDefinitionScanner

---

`AnnotatedBeanDefinitionReader`

```java
public AnnotatedBeanDefinitionReader(BeanDefinitionRegistry registry) {
   this(registry, getOrCreateEnvironment(registry));
}

public AnnotatedBeanDefinitionReader(BeanDefinitionRegistry registry, Environment environment) {
   this.registry = registry;
   this.conditionEvaluator = new ConditionEvaluator(registry, environment, null);
   // 会放一些后置处理器，用于实现一些功能。如ConfigurationClassPostProcessor、AutowiredAnnotationBeanPostProcessor，就是实现@Configuration和@Autowired注解的后置处理器。
   AnnotationConfigUtils.registerAnnotationConfigProcessors(this.registry);
}
```

在初始化`AnnotationBeanDefinitionReader`时，会放一些后置处理器（如ConfigurationClassPostProcessor、AutowiredAnnotationBeanPostProcessor、CommonAnnotationBeanPostProcessor），用来实现一些注解的功能。

 

`ClassPathBeanDefinitionScanner`的初始化则比较简单。

`AnnotationBeanDefinationReader`和`ClassPathBeanDefinitionScanner`两者可以相互替换，从名字上就可以看出区别 ：

- `AnnotationBeanDefinationReader`是读取单个类的。
- `ClassPathBeanDefinitionScanner`是扫描包路径下的。



如果要手动加载某个类或者某个包，可以使用`context.registerBean`或`context.scan`来加载 ：

```java
AnnotationConfigApplicationContext context = new AnnotationConfigApplicationContext("com.dzyls.learn.springframework");
// 加载指定类
context.registerBean(HelloService.class);
HelloService service = context.getBean("helloService", HelloService.class);
service.sayHello();
// 扫描某个包
context.scan("com.xxx.ooo");
```



### BeanDefinition

---

`BeanDefinition`是对要加载的类的封装，会记录一些属性，如 ：

- 是否懒加载

- 是否是单例、是不是多例的

- 初始化方法名称、销毁方法名称（`@Bean(initMethod="init",destoryMethod="destoryMethod")`）

  > @PostConstruct和@PreDestory是JSR-250标准，实现原理基于CommonAnnotationBeanPostProcessor，BeanDefinition类中记录的不是这两个注解标注的方法。而是注解@Bean指定的initMethod和destoryMethod方法。

- 是否是Primary的

等等。

为什么要将类封装为BeanDefinition ？

如果不封装起来，那么要使用到这些属性时，就需要去类上扫描了。不如初始化时直接一次性扫描封装起来，以供后面使用。



### @PostConstruct和@PreDestory

---

@PostConstruct用来标注初始化方法，如init。

@PreDestory用来标注销毁方法。

这两个注解的后置处理器都是在`AnnotationBeanDefinitionReader`在初始化时加载`CommonAnnotationBeanPostProcessor`来实现的，其真正的实现是在其父类`InitDestoryAnnotationBeanPostProcessor`中实现的。

1. `CommonAnnotationBeanPostProcessor`在初始化时，会设置初始化注解和销毁方法注解 ：

   ```java
   public CommonAnnotationBeanPostProcessor() {
   
      // 设置初始化方法注解为 @PostConstruct注解
      setInitAnnotationType(PostConstruct.class);
       //设置销毁方法注解为 @PreDestory注解
      setDestroyAnnotationType(PreDestroy.class);
   
   }
   ```

2. 这两个属性，会在父类`InitDestoryAnnotationBeanPostProcessor`用到。

   `InitDestoryAnnotationBeanPostProcessor`的`buildLifecycleMetadata`中会扫描类中标注了这两个注解的方法 ：

   `buildLifecycleMetadata`会构建一个`LifecycleMeta`对象，**LifecycleMetadata中有两个List，分别用来存储初始化方法和销毁方法，**

   ```java
   private LifecycleMetadata buildLifecycleMetadata(final Class<?> clazz) {
       
      List<LifecycleElement> initMethods = new ArrayList<>();
      List<LifecycleElement> destroyMethods = new ArrayList<>();
      Class<?> targetClass = clazz;
   
      do {
         final List<LifecycleElement> currInitMethods = new ArrayList<>();
         final List<LifecycleElement> currDestroyMethods = new ArrayList<>();
   
         ReflectionUtils.doWithLocalMethods(targetClass, method -> {
            if (this.initAnnotationType != null && method.isAnnotationPresent(this.initAnnotationType)) {
               LifecycleElement element = new LifecycleElement(method);
               currInitMethods.add(element);
            }
            if (this.destroyAnnotationType != null && method.isAnnotationPresent(this.destroyAnnotationType)) {
               currDestroyMethods.add(new LifecycleElement(method));
            }
         });
   
         initMethods.addAll(0, currInitMethods);
         destroyMethods.addAll(currDestroyMethods);
         targetClass = targetClass.getSuperclass();
      }
      while (targetClass != null && targetClass != Object.class);
   
      return (initMethods.isEmpty() && destroyMethods.isEmpty() ? this.emptyLifecycleMetadata :
            new LifecycleMetadata(clazz, initMethods, destroyMethods));
   }
   ```

   Spring可以通过这个`LifecycleMetadata`对象来调用初始化方法和销毁方法 ：

   ```java
   public void invokeInitMethods(Object target, String beanName) throws Throwable {
      Collection<LifecycleElement> checkedInitMethods = this.checkedInitMethods;
      Collection<LifecycleElement> initMethodsToIterate =
            (checkedInitMethods != null ? checkedInitMethods : this.initMethods);
      if (!initMethodsToIterate.isEmpty()) {
         for (LifecycleElement element : initMethodsToIterate) {
            if (logger.isTraceEnabled()) {
               logger.trace("Invoking init method on bean '" + beanName + "': " + element.getMethod());
            }
            element.invoke(target);
         }
      }
   }
   
   public void invokeDestroyMethods(Object target, String beanName) throws Throwable {
      Collection<LifecycleElement> checkedDestroyMethods = this.checkedDestroyMethods;
      Collection<LifecycleElement> destroyMethodsToUse =
            (checkedDestroyMethods != null ? checkedDestroyMethods : this.destroyMethods);
      if (!destroyMethodsToUse.isEmpty()) {
         for (LifecycleElement element : destroyMethodsToUse) {
            if (logger.isTraceEnabled()) {
               logger.trace("Invoking destroy method on bean '" + beanName + "': " + element.getMethod());
            }
            element.invoke(target);
         }
      }
   }
   ```

3. 找到方法就很简单了，`InitDestoryAnnotationBeanPostDefinition`会在bean初始化之前调用init方法 ：

   ```java
   private final transient Map<Class<?>, LifecycleMetadata> lifecycleMetadataCache = new ConcurrentHashMap<>(256);
   
   @Override
   public Object postProcessBeforeInitialization(Object bean, String beanName) throws BeansException {
      LifecycleMetadata metadata = findLifecycleMetadata(bean.getClass());
      try {
         metadata.invokeInitMethods(bean, beanName);
      }
     	// 省略
      return bean;
   }
   
   
   @Override
   public void postProcessBeforeDestruction(Object bean, String beanName) throws BeansException {
      LifecycleMetadata metadata = findLifecycleMetadata(bean.getClass());
      try {
         metadata.invokeDestroyMethods(bean, beanName);
      }
   	// 省略
   }
   ```

   在销毁前调用Destory方法。

   调用参数是bean，通过bean.getClass找到缓存中缓存的`LifecycleMetadata`。

**总结 ：**

`@PostConstruct`和`@PreDestory`注解的实现原理依赖于`CommonAnnotationBeanPostProcessor`和其父类`InitDestoryAnnotationBeanPostProcessor`，`InitDestoryAnnoationBeanPostProcessor`会扫描被`@PostConstruct`和`@PreDestory`标注的方法，并封装为LifecycleMetadata对象，在Bean初始化之前调用初始化方法，在Bean销毁前调用销毁方法。



### @Configuration注解及相关注解的实现原理

---

在Spring Boot中，我们通常会使用`@Configuration`注解来配置一个配置类，其实现的原理是在创建`AnntationBeanDefinitionPostProcessor`时，向容器中添加的`ConfigurationClassPostProcessor`来实现的。

`ConfigurationClassPostProcessor`实现了`BeanDefinitionRegistyPostProcessor`，就是一个BeanFactory后置处理器。在容器刷新时会调用BeanFactory后置处理方法。

在`ConfigurationClassPostProcessor`会调用一个解析器 `ConfigurationClassParser`用来解析Configuration类。

`ConfigurationClassParser`与很多功能有关，是一个非常关键的一个解析类 ：

- @Componet
- 解析@ComponetScan信息
- @Import
- @ImportResource
- 解析@PropertySource的配置信息
- 判断该类是否要跳过（`@Conditional`）
- `@Bean`注解是

BeanFactory后置处理器最终会走到 `ConfigurationClassParser#doProcessConfigurationClass`，在这个方法中会 ：

#### **@Component**

```java
protected final SourceClass doProcessConfigurationClass(
      ConfigurationClass configClass, SourceClass sourceClass, Predicate<String> filter){

   if (configClass.getMetadata().isAnnotated(Component.class.getName())) {
      // Recursively process any member (nested) classes first
      processMemberClasses(configClass, sourceClass, filter);
   }
	// 省略后续步骤
}
```



#### **@PropertySource**

```java
for (AnnotationAttributes propertySource : AnnotationConfigUtils.attributesForRepeatable(
      sourceClass.getMetadata(), PropertySources.class,
      org.springframework.context.annotation.PropertySource.class)) {
   if (this.environment instanceof ConfigurableEnvironment) {
      // 读取propertySource注解
      processPropertySource(propertySource);
   }
}
```

```java
private void processPropertySource(AnnotationAttributes propertySource) throws IOException {
    // 读取注解中属性的值
   String name = propertySource.getString("name");
   if (!StringUtils.hasLength(name)) {
      name = null;
   }
   String encoding = propertySource.getString("encoding");
   if (!StringUtils.hasLength(encoding)) {
      encoding = null;
   }
   String[] locations = propertySource.getStringArray("value");

   Class<? extends PropertySourceFactory> factoryClass = propertySource.getClass("factory");
   PropertySourceFactory factory = (factoryClass == PropertySourceFactory.class ?
         DEFAULT_PROPERTY_SOURCE_FACTORY : BeanUtils.instantiateClass(factoryClass));

   for (String location : locations) {

         String resolvedLocation = this.environment.resolveRequiredPlaceholders(location);
         Resource resource = this.resourceLoader.getResource(resolvedLocation);
       	// 将创建的PropertySource放到Environment中
         addPropertySource(factory.createPropertySource(name, new EncodedResource(resource, encoding)));
     
      }
   }
}
```

`ConfigurationClassParser`对`@PropertySource`的处理就是读取到注解的属性信息，判断文件是否存在，并根据配置封装为`PropertySource`放到`ConfigurableEnvironment`中。

等到`AbstructAutowiredCapableBeanFactory#doCreateBean` 调用`AutowiredAnnotationBeanPostProcessor#postProcessProperties`时，会将value注入到字段中。



#### **@ComponentScan**

然后是ComponentScan注解功能的实现 ：

```java
// 这里的sourceClass是所有被@Configuration标注的类
Set<AnnotationAttributes> componentScans = AnnotationConfigUtils.attributesForRepeatable(
      sourceClass.getMetadata(), ComponentScans.class, ComponentScan.class);
if (!componentScans.isEmpty() &&
    // 通过@Conditional注解来判断是否要跳过扫描
      !this.conditionEvaluator.shouldSkip(sourceClass.getMetadata(), ConfigurationPhase.REGISTER_BEAN)) {
   for (AnnotationAttributes componentScan : componentScans) {
      // The config class is annotated with @ComponentScan -> perform the scan immediately
      Set<BeanDefinitionHolder> scannedBeanDefinitions =
            this.componentScanParser.parse(componentScan, sourceClass.getMetadata().getClassName());
      // Check the set of scanned definitions for any further config classes and parse recursively if needed
      for (BeanDefinitionHolder holder : scannedBeanDefinitions) {
         BeanDefinition bdCand = holder.getBeanDefinition().getOriginatingBeanDefinition();
         if (bdCand == null) {
            bdCand = holder.getBeanDefinition();
         }
         if (ConfigurationClassUtils.checkConfigurationClassCandidate(bdCand, this.metadataReaderFactory)) {
            parse(bdCand.getBeanClassName(), holder.getBeanName());
         }
      }
   }
}
```

`ConfigurationClassParser`**会扫描所有被@Configuration标注的类**。（其实是`ConfigurationClassPostProcessor`扫描所有的`@Configuration`类，然后依次调用`ConfigurationClassParser#processConfigurationClass`方法）。



并将每个`@ComponentScan`的属性扫描封装为一个个AnnotationAttributes对象放到集合，然后判断是否要跳过扫描（`@Conditional`注解）。如果要扫描，则遍历这个`AnnotationAttribute`集合，根据`AnnotationAttribute`属性，再去扫描解析这些`BeanDefinition`放到容器。并且如果有哪个`BeanDefinition`也标注了`@Configuration`注解，也会去解析这个ConfigurationClass。



以此实现，`@ComponentScan`扫描加载类的功能。

总结 ：`ConfigurationClassPostProcessor`会找到所有标注`@Configuration`注解的类，然后判断是否要跳过，如果不跳过则依次扫描这些类，调用`ConfigurationClassParser`去解析这些类。在解析时发现某个类上有`@ComponentScan`时，就会获得注解属性的`value`（即包路径），然后调用`ClassPathBeanDefinitionScaner`去依次扫描这些包路径。扫描完成后，会再次遍历这些类，看有没有`@Configuration`标注的类，如果有，就递归调用再扫描。

> SpringBoot Application的启动注解有@ComponentScan注解和@Configuration注解。



#### **@Import**

接下来就是`@Import`注解了 ：

```java
processImports(configClass, sourceClass, getImports(sourceClass), filter, true);
```

`@Import`注解比较少用，一般是用`@Configuration`和`@Bean`。





#### **@ImportResource**

再之后就是`@ImportResource`。

```java
AnnotationAttributes importResource =
      AnnotationConfigUtils.attributesFor(sourceClass.getMetadata(), ImportResource.class);
if (importResource != null) {
   String[] resources = importResource.getStringArray("locations");
   Class<? extends BeanDefinitionReader> readerClass = importResource.getClass("reader");
   for (String resource : resources) {
      String resolvedResource = this.environment.resolveRequiredPlaceholders(resource);
      configClass.addImportedResource(resolvedResource, readerClass);
   }
}
```

这个就很简单了，就是读取`@ImportSource`的`location`属性，然后依次加载到`ImportResource`里去。



#### **@Bean**

获取`@Configuration`类中被`@Bean`标注的方法，放到`ConfigurationClass`的`BeanMethod`属性中。

```java
Set<MethodMetadata> beanMethods = retrieveBeanMethodMetadata(sourceClass);
for (MethodMetadata methodMetadata : beanMethods) {
   configClass.addBeanMethod(new BeanMethod(methodMetadata, configClass));
}
```

然后会扫描父接口的default方法，如果父接口default方法也标注了`@bean`注解，也会被加载到容器。

```java
// Process default methods on interfaces
processInterfaces(configClass, sourceClass);
```



这些Bean方法，最后会在`BeanDefinitionRegistyPostPrcocesser`的`postProcessBeanDefinitionRegisty`中（`ConfigurationClassPostProcessor#postProcessBeanDefinitionRegisty`）封装为BeanDefinition，放到容器中。



最后是父类的方法：

```java
// Process superclass, if any
if (sourceClass.getMetadata().hasSuperClass()) {
   String superclass = sourceClass.getMetadata().getSuperClassName();
   if (superclass != null && !superclass.startsWith("java") &&
         !this.knownSuperclasses.containsKey(superclass)) {
      this.knownSuperclasses.put(superclass, configClass);
      // Superclass found, return its annotation metadata and recurse
      return sourceClass.getSuperClass();
   }
}

// No superclass -> processing is complete
return null;
```

由于是递归的，因此只要父类不为空，那么就会一直扫。直到父类到的全限定名以java开头。



### BeanFactoryPostProcessor和BeanPostProcessor

---

`BeanFactoryPostProcessor`是BeanFactory的后置处理器，在容器刷新时执行。

容器刷新是在类`AbstractApplicationContext#refresh`方法中。

`org.springframework.context.support.AbstractApplicationContext#refresh`

```java
@Override
public void refresh() throws BeansException, IllegalStateException {
   synchronized (this.startupShutdownMonitor) {
		
         //省略部分代码
       	// 留给子类实现
         postProcessBeanFactory(beanFactory);

         // Invoke factory processors registered as beans in the context.
         invokeBeanFactoryPostProcessors(beanFactory);
       	// 注册BeanPostProcessor，在Bean创建前后调用指定方法
       	registerBeanPostProcessors(beanFactory);
```

`BeanFactoryPostProcessor`只有一个方法 ：`postProcessBeanFactory`，

常用子接口有 `BeanDefinitionRegistyPostProcessor`，而`ConfigurationClassPostProcessor`就是实现了这个子接口。

因此当容器刷新时，会调用`BeanFacotryPostProcessor`，也就是会调用`ConfigurationClassPostProcessor#postProcessBeanFactory`。



### 后置处理器

---

后置处理器实现了很多功能，如：

#### BeanPostProcessor

- `InitDestoryAnnotationBeanPostProcessor`实现了`PostConstruct`和`PreDestory`注解
- `ApplicationContextAwareProcessor`实现Aware接口调用，如`ApplicationContextAware`接口的实现。



`BeanPostProcessor`的接口的方法，在bean创建前后调用。

详见`org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory#doCreateBean`。



#### BeanFactoryPostProcessor



`BeanFactoryPostProcessor`接口的方法，在容器刷新时调用。





### Aware接口的实现

---

我们常用的`Aware`接口有 ：

- `ApplicationContextAware`接口，用于获取ApplicationContext的
- `BeanNameAware`接口，用户获取`BeanName`的
- `BeanFactoryAware`接口，用于获取BeanFactory



#### ApplicationContextAware

ApplicationContextAware接口的功能实现是依赖于`ApplicaitonContextAwareProcessor`。

`ApplicationContextAwareProcessor`其实是一个Bean后置处理器，在Bean初始化前执行`ApplicationContextAware`接口的`setApplicationAware`方法。

```java
@Override
@Nullable
public Object postProcessBeforeInitialization(Object bean, String beanName) throws BeansException {
   if (!(bean instanceof EnvironmentAware || bean instanceof EmbeddedValueResolverAware ||
         bean instanceof ResourceLoaderAware || bean instanceof ApplicationEventPublisherAware ||
         bean instanceof MessageSourceAware || bean instanceof ApplicationContextAware ||
         bean instanceof ApplicationStartupAware)) {
      return bean;
   }

   AccessControlContext acc = null;

   if (System.getSecurityManager() != null) {
      acc = this.applicationContext.getBeanFactory().getAccessControlContext();
   }

   if (acc != null) {
      AccessController.doPrivileged((PrivilegedAction<Object>) () -> {
         invokeAwareInterfaces(bean);
         return null;
      }, acc);
   }
   else {
      invokeAwareInterfaces(bean);
   }

   return bean;
}

private void invokeAwareInterfaces(Object bean) {
   if (bean instanceof EnvironmentAware) {
      ((EnvironmentAware) bean).setEnvironment(this.applicationContext.getEnvironment());
   }
   if (bean instanceof EmbeddedValueResolverAware) {
      ((EmbeddedValueResolverAware) bean).setEmbeddedValueResolver(this.embeddedValueResolver);
   }
   if (bean instanceof ResourceLoaderAware) {
      ((ResourceLoaderAware) bean).setResourceLoader(this.applicationContext);
   }
   if (bean instanceof ApplicationEventPublisherAware) {
      ((ApplicationEventPublisherAware) bean).setApplicationEventPublisher(this.applicationContext);
   }
   if (bean instanceof MessageSourceAware) {
      ((MessageSourceAware) bean).setMessageSource(this.applicationContext);
   }
   if (bean instanceof ApplicationStartupAware) {
      ((ApplicationStartupAware) bean).setApplicationStartup(this.applicationContext.getApplicationStartup());
   }
   if (bean instanceof ApplicationContextAware) {
       // 调用了ApplicationContextAware的方法
      ((ApplicationContextAware) bean).setApplicationContext(this.applicationContext);
   }
}
```

除了`ApplicationContextAware`接口外，还调用了许多其他`Aware`接口的方法。



#### BeanFactoryAware



#### BeanNameAware







### Spring容器刷新流程

---









### BeanFactory的子接口和子类

---





### Spring Boot启动流程

---

```java
public ConfigurableApplicationContext run(String... args) {
   StopWatch stopWatch = new StopWatch();
   stopWatch.start();
   DefaultBootstrapContext bootstrapContext = createBootstrapContext();
   ConfigurableApplicationContext context = null;
   configureHeadlessProperty();
   SpringApplicationRunListeners listeners = getRunListeners(args);
   listeners.starting(bootstrapContext, this.mainApplicationClass);
   try {
      ApplicationArguments applicationArguments = new DefaultApplicationArguments(args);
      ConfigurableEnvironment environment = prepareEnvironment(listeners, bootstrapContext, applicationArguments);
      configureIgnoreBeanInfo(environment);
      Banner printedBanner = printBanner(environment);
      context = createApplicationContext();
      context.setApplicationStartup(this.applicationStartup);
      prepareContext(bootstrapContext, context, environment, listeners, applicationArguments, printedBanner);
      refreshContext(context);
      afterRefresh(context, applicationArguments);
      stopWatch.stop();
      if (this.logStartupInfo) {
         new StartupInfoLogger(this.mainApplicationClass).logStarted(getApplicationLog(), stopWatch);
      }
      listeners.started(context);
      callRunners(context, applicationArguments);
   }
   catch (Throwable ex) {
      handleRunFailure(context, ex, listeners);
      throw new IllegalStateException(ex);
   }

   try {
      listeners.running(context);
   }
   catch (Throwable ex) {
      handleRunFailure(context, ex, null);
      throw new IllegalStateException(ex);
   }
   return context;
}
```