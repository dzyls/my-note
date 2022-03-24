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



**总结**：

对于`AnnotatedConfigApplicationContext`，Spring容器启动流程大致可以分为三步：

1. 创建BeanDefinitionReader和BeanDefinitionScaner，在初始化时会放一些后置处理器进去（ConfigurationClassPostProcessor、CommonAnnotationBeanPostProcessor、AutowiredAnnotationBeanPostProcessor等）

2. 进行扫描（scan）或者注册（register），根据传参来决定。将所有要加载的Bean，扫描出来。

3. 执行刷新流程，刷新流程比较复杂，设计到很多后置处理器

   - 获得BeanFactory

   - 调用BeanFactoryPostProcessor的方法

     （Bean工厂的后置处理器和BeanDefinitionRegistyPostProcessor）

   - 注册所有的BeanPostProcessor到beanFactory中（放到beanFactory的一条有序List中，执行时按顺序）

   - 初始化MessageSource（国际化相关）

   - 初始化事件发布器（其实就是吧ApplicationEventMuticaster放到beanFactory里）

   - 注册事件监听器（事件监听机制）

   - 完成bean工厂的初始化，并初始化所有单例、非懒加载的bean（初始化bean前后会执行beanPostProcessor）

   - 完成刷新，发布容器刷新事件



> 以上仅针对于AnnotatedConfigApplicationContext构造参数为class何basePackages 的构造器。
>
> 对于Spring Boot Application，则是没有调用scan。而是通过注解@ComponentScan来实现扫描，ComponentScanAnnotationParser来调用ClassPathBeanDefinitionScaner执行扫描。所以Spring Boot没有在初始化容器时执行扫描。



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



### Spring注入流程

---





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

利用`BeanFactoryProcessor`接口和其子接口`BeanDefinitionRegistyPostProcessor`，我们可以实现对BeanDefinition的任意修改。虽然很少这样使用，但是Spring自己就利用了

在Spring中，使用BeanFactoryPostProcessor实现了一些注解，比如 ：

- ConfigurationClassPostProcessor 实现了对@Configuration注解的扫描，间接的调用`ConfigurationClassParser`完成对`@Component`、`@PropertySource`、`@ComponentScan`、`@Import`、`@ImportSource`扫描。

  > Spring Boot利用`ConfigurationClassPostProcessor`对`@ComponentScan`的扫描，可以实现bean的扫描（`@SpringBootApplication`注解就标注了`@ComponentScan`），不用添加package或者class参数就可扫描。

- MyBatis使用了BeanFactoryPostProcessor接口，定义了一个类`MapperScannerConfigurer`，当执行到BeanFactory后置处理器，`MapperScannerConfigurer`自定义一个`BeanDefinitionScanner`（`ClassPathMapperScanner`），重写了`processBeanDefinitions`，修改了Mapper接口的BeanDefinition属性。（比如将`BeanClass`改为了MapperFactoryBean）





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

Spring容器刷新流程的代码在`AbstractApplicationContext#refresh`方法中，注释已经很完整了。

```java
synchronized (this.startupShutdownMonitor) {

   // Prepare this context for refreshing.
   prepareRefresh();

   // Tell the subclass to refresh the internal bean factory.
   ConfigurableListableBeanFactory beanFactory = obtainFreshBeanFactory();

   // Prepare the bean factory for use in this context.
   prepareBeanFactory(beanFactory);

   try {
      // Allows post-processing of the bean factory in context subclasses.
      postProcessBeanFactory(beanFactory);

      invokeBeanFactoryPostProcessors(beanFactory);

      // Register bean processors that intercept bean creation.
       // 找到所有的beanPostProcessor，排好序，放到BeanFactory的BeanPostProcessorList中。	  		//（其实就是一个CopyOnWriteArrayList）
      registerBeanPostProcessors(beanFactory);
      beanPostProcess.end();

      // Initialize message source for this context.
      // 国际化相关
      initMessageSource();

      // Initialize event multicaster for this context.
       // 事件发布器
      initApplicationEventMulticaster();

      // Initialize other special beans in specific context subclasses.
      onRefresh();

      // Check for listener beans and register them.
       // 注册事件监听器
      registerListeners();

      // Instantiate all remaining (non-lazy-init) singletons.
       // 实例化所有的单例非懒加载的bean
      finishBeanFactoryInitialization(beanFactory);

      // Last step: publish corresponding event.
       // 发布容器刷新事件
      finishRefresh();
   }
```



1. 获取BeanFactory
2. 调用BeanFactoryPostProcessor中的方法
3. 注册BeanPostProcessor到容器中
4. 初始化MessageSource（国际化相关）
5. 初始化事件多播器
6. 注册事件监听器
7. 完成BeanFactory初始化，并初始化单例、非懒加载的bean（在初始化前后，会调用BeanPostProcessor）
8. 完成容器刷新，并发布容器刷新完成事件



### Bean创建流程及生命周期

---

Bean创建是在容器刷新时。

容器在创建时，会创建`BeanDefinitionScanner`和`BeanDefinitionReader`，这两个会将需要加载到容器的bean封装为BeanDefinition对象，放入一个Map中。然后在容器刷新的时候，会根据BeanDefinition来创建Bean。

创建bean的流程 ：

1. 第一步肯定是初始化，执行构造方法了。

2. 注入属性了
3. 执行三个Aware接口的方法（BeanNameAware、BeanFacotryAware、BeanClassLoaderAware）
4. 执行Bean后置处理器的`postProcessBeforeInitialization`（如`ApplicationContextAwareProcessor`、`@PostConstruct`注解）
5. Bean的InitalizingBean的afterPorpertiesSet方法
6. 调用自定义的初始化方法（@Bean注解的init-method）
7. BeanPostProcessor的`postProcessAfterInitialization`方法

```java
protected Object initializeBean(String beanName, Object bean, @Nullable RootBeanDefinition mbd) {
   if (System.getSecurityManager() != null) {
      AccessController.doPrivileged((PrivilegedAction<Object>) () -> {
         invokeAwareMethods(beanName, bean);
         return null;
      }, getAccessControlContext());
   }
   else {
       // invoke Aware 接口的方法(Beanname、beanFactory、BeanClassloader)
      invokeAwareMethods(beanName, bean);
   }

   Object wrappedBean = bean;
   if (mbd == null || !mbd.isSynthetic()) {
       // 调用bean后置处理器的前置方法
      wrappedBean = applyBeanPostProcessorsBeforeInitialization(wrappedBean, beanName);
   }

   try {
       // 调用init方法
       // 1.InitialingBean#afterPropertiesSet
       // 2.Bean注解的init-method
      invokeInitMethods(beanName, wrappedBean, mbd);
   }
   catch (Throwable ex) {
      throw new BeanCreationException(
            (mbd != null ? mbd.getResourceDescription() : null),
            beanName, "Invocation of init method failed", ex);
   }
   if (mbd == null || !mbd.isSynthetic()) {
       // 调用Bean后置处理器的后置方法
      wrappedBean = applyBeanPostProcessorsAfterInitialization(wrappedBean, beanName);
   }

   return wrappedBean;
}
```

```java
private void invokeAwareMethods(String beanName, Object bean) {
   if (bean instanceof Aware) {
      if (bean instanceof BeanNameAware) {
         ((BeanNameAware) bean).setBeanName(beanName);
      }
      if (bean instanceof BeanClassLoaderAware) {
         ClassLoader bcl = getBeanClassLoader();
         if (bcl != null) {
            ((BeanClassLoaderAware) bean).setBeanClassLoader(bcl);
         }
      }
      if (bean instanceof BeanFactoryAware) {
         ((BeanFactoryAware) bean).setBeanFactory(AbstractAutowireCapableBeanFactory.this);
      }
   }
}
```



执行销毁步骤的前提是，调用了`context.registerShutdownHook()`注册了关闭钩子。

销毁的步骤就很简单了 ：

1. 执行`@PreDestory`方法
2. 执行DisposableBean接口
3. 配置的destory-method方法

```java
//Spring容器添加ShutdownHook
@Override
public void registerShutdownHook() {
   if (this.shutdownHook == null) {
      // No shutdown hook registered yet.
      this.shutdownHook = new Thread(SHUTDOWN_HOOK_THREAD_NAME) {
         @Override
         public void run() {
            synchronized (startupShutdownMonitor) {
               doClose();
            }
         }
      };
      Runtime.getRuntime().addShutdownHook(this.shutdownHook);
   }
}
```



```java
protected void doClose() {
   // Check whether an actual close attempt is necessary...
   if (this.active.get() && this.closed.compareAndSet(false, true)) {

      if (!NativeDetector.inNativeImage()) {
         LiveBeansView.unregisterApplicationContext(this);
      }

      // Publish shutdown event.发布关闭容器事件
      publishEvent(new ContextClosedEvent(this)); 

      // Stop all Lifecycle beans, to avoid delays during individual destruction.
      if (this.lifecycleProcessor != null) {
          // 调用lifecycleProcessor的onclose方法
            this.lifecycleProcessor.onClose();
      }

      // Destroy all cached singletons in the context's BeanFactory.
       // 销毁bean
      destroyBeans();

      // Close the state of this context itself.
      // 关闭beanFactory，其实就是把beanFactory的Id设置为null
      closeBeanFactory();

      // Let subclasses do some final clean-up if they wish...
       // 调用子类的onClose方法（留给子类扩展）
      onClose();

      // Reset local application listeners to pre-refresh state.
      if (this.earlyApplicationListeners != null) {
         this.applicationListeners.clear();
         this.applicationListeners.addAll(this.earlyApplicationListeners);
      }

      // Switch to inactive.
       // 设置标记位
      this.active.set(false);
   }
}
```

最重要的是`destroyBean`方法。这个方法中，ApplicationContext会调用beanFactory的`destroySingletons`方法。

```java
// AbstractApplicationContext#destroyBeans
protected void destroyBeans() {
   // 拿到BeanFactory，getBeanFactory由子类实现
   getBeanFactory().destroySingletons();
}
```

而`BeanFactory.destroySingletons`最终会层层调用到 ：

```java
private final Map<String, Object> disposableBeans = new LinkedHashMap<>();

public void destroySingletons() {

   synchronized (this.singletonObjects) {
      this.singletonsCurrentlyInDestruction = true;
   }

   String[] disposableBeanNames;
   synchronized (this.disposableBeans) {
      // 拿到所有要销毁的bean（其实不是Bean，而是DisposableBeanAdapter，         DisposableBeanAdapter用来包装DisposableBean）
      disposableBeanNames = StringUtils.toStringArray(this.disposableBeans.keySet());
   }
   for (int i = disposableBeanNames.length - 1; i >= 0; i--) {
      // destorySingleton最终会根据DisposableBeanAdapter的destroy方法，然后再调用Bean的destroy方法
      destroySingleton(disposableBeanNames[i]);
   }

   this.containedBeanMap.clear();
   this.dependentBeanMap.clear();
   this.dependenciesForBeanMap.clear();

   clearSingletonCache();
}
```



这里有一个重要的一点：`disposableBean`这个Map中，保存的**并不是（实现DisposableBean接口的）Bean**实例，而是`DiposableBeanAdapter`适配器（适配器模式），`DisposableBeanAdapter`会包装`Bean`，然后会间接调用`Bean#destroy`方法，在调用bean的destroy前后，会做一些处理。

`DisposableBeanAdapter#destroy`

```java
public void destroy() {
   if (!CollectionUtils.isEmpty(this.beanPostProcessors)) {
      // 会调用到CommonAnnotationBeanPostProcessor（@PreDestroy）
      // 
      for (DestructionAwareBeanPostProcessor processor : this.beanPostProcessors) {
         processor.postProcessBeforeDestruction(this.bean, this.beanName);
      }
   }

   if (this.invokeDisposableBean) {
       // 调用Bean的destroy方法
       ((DisposableBean) this.bean).destroy();
   }

   if (this.invokeAutoCloseable) {
       // 调用Bean的close方法（AutoCloseable接口）
       ((AutoCloseable) this.bean).close();
   }
   else if (this.destroyMethod != null) {
      // 自定义的销毁方法（destory-method）
      invokeCustomDestroyMethod(this.destroyMethod);
   }
   else if (this.destroyMethodName != null) {
      Method methodToInvoke = determineDestroyMethod(this.destroyMethodName);
      if (methodToInvoke != null) {
         invokeCustomDestroyMethod(ClassUtils.getInterfaceMethodIfPossible(methodToInvoke));
      }
   }
}
```

从源码上看，销毁方法的执行顺序是 ：

1. `@PreDestroy`方法
2. `Disposablebean#destroy`方法
3. `AutoCloseable#close`方法
4. 注解或配置文件指定的`destroy-method`方法



还有一个疑问，`DisposableBeanAdapter`是何时放在`Map`容器的呢？

经过Debug，发现是 ：在创建Bean后，返回bean之前，会注册`DisposableBeanAdapter`。

`AbstractAutowireCapableBeanFactory#doCreateBean`

```java
try {
   registerDisposableBeanIfNecessary(beanName, bean, mbd);
}
catch (BeanDefinitionValidationException ex) {
   throw new BeanCreationException(
         mbd.getResourceDescription(), beanName, "Invalid destruction signature", ex);
}

return exposedObject;
```



然后 ：

```java
protected void registerDisposableBeanIfNecessary(String beanName, Object bean, RootBeanDefinition mbd) {
   AccessControlContext acc = (System.getSecurityManager() != null ? getAccessControlContext() : null);
   if (!mbd.isPrototype() && requiresDestruction(bean, mbd)) {
      if (mbd.isSingleton()) {
         // Register a DisposableBean implementation that performs all destruction
         // work for the given bean: DestructionAwareBeanPostProcessors,
         // DisposableBean interface, custom destroy method.
         registerDisposableBean(beanName, new DisposableBeanAdapter(
               bean, beanName, mbd, getBeanPostProcessorCache().destructionAware, acc));
      }
      else {
         // A bean with a custom scope...
         Scope scope = this.scopes.get(mbd.getScope());
         if (scope == null) {
            throw new IllegalStateException("No Scope registered for scope name '" + mbd.getScope() + "'");
         }
         scope.registerDestructionCallback(beanName, new DisposableBeanAdapter(
               bean, beanName, mbd, getBeanPostProcessorCache().destructionAware, acc));
      }
   }
}
```

如果是 单例，并且需要在关闭的时候调用方法（实现了`DisposableBean`接口、或者使用了`@PreDestory`，或者指定了`destory-method`，或者实现了`AutoCloseable`接口），都会封装为`DisposableBeanAdapter`对象，在容器关闭时，触发关闭钩子，调用关闭方法。



再谈`CommonAnnotationBeanPostProcessor`：

`CommonAnnotationBeanPostProcessor`实现了JSR-250，即`@PostConstract`和`@PreDestroy`两个注解。

`CommonAnnotationBeanPostPrcessor`继承了`InitDestroyAnnotationBeanPostProcessor`。

`InitDestoryAnnotationBeanPostProcessor`的父接口是`DestructionAwareBeanPostPrcessor`接口，

`DestructionAwareBeanPostProcessor`父接口是`BeanPostProcessor`。

`BeanPostProcessor`中的两个方法会在实例化后调用（`@PostConstruct`的原理）。

而`DestuctionAwareBeanPostProcessor`则是在容器销毁前会调用其方法`postProcessBeforeDestruction`。（`@PreDestory`原理）

### BeanFactory和ApplicationContext

---

`BeanFactory`和`ApplicationContext`分别是两个接口，`ApplicationContext`是`BeanFactory`的子接口的子接口。

`BeanFactory`接口非常简单，只有几个方法，就是一个容器。

```
public interface BeanFactory {

   String FACTORY_BEAN_PREFIX = "&";
	
   Object getBean(String name) throws BeansException;

   <T> T getBean(String name, Class<T> requiredType) throws BeansException;

   Object getBean(String name, Object... args) throws BeansException;

   <T> T getBean(Class<T> requiredType) throws BeansException;

   <T> T getBean(Class<T> requiredType, Object... args) throws BeansException;

   <T> ObjectProvider<T> getBeanProvider(Class<T> requiredType);

   <T> ObjectProvider<T> getBeanProvider(ResolvableType requiredType);

   boolean containsBean(String name);

   boolean isSingleton(String name) throws NoSuchBeanDefinitionException;

   boolean isPrototype(String name) throws NoSuchBeanDefinitionException;

   boolean isTypeMatch(String name, ResolvableType typeToMatch) throws NoSuchBeanDefinitionException;

   boolean isTypeMatch(String name, Class<?> typeToMatch) throws NoSuchBeanDefinitionException;

   Class<?> getType(String name) throws NoSuchBeanDefinitionException;

   Class<?> getType(String name, boolean allowFactoryBeanInit) throws NoSuchBeanDefinitionException;

   String[] getAliases(String name);

}
```

通过`BeanFactory`，我们可以根据类型、名称，获得bean；判断bean是单例还是多例；获取Bean的别名等等操作。



而ApplicationContext不仅实现了BeanFactory接口，还提供了一些高级功能 ：

- 事件监听机制
- 国际化
- 支持`@PostConstract`和`@PreDestroy`注解



最主要的区别是 ：beanFactory是**懒加载**的，而ApplicationContext默认情况下是**容器刷新**时就创建实例。这个是因为早期的电脑内存小性能差，懒加载能避免创建没有使用的类。

BeanFactory使用 ：

```java
public static void main(String[] args) throws Exception {
    DefaultListableBeanFactory beanFactory = new DefaultListableBeanFactory();
    ClassPathResource resource = new ClassPathResource("bean.xml");
    XmlBeanDefinitionReader reader = new XmlBeanDefinitionReader(beanFactory);
    reader.loadBeanDefinitions(resource);
    UserComponent component = beanFactory.getBean(UserComponent.class);
    component.destroy();
}
```

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xsi:schemaLocation="http://www.springframework.org/schema/beans
       http://www.springframework.org/schema/beans/spring-beans.xsd">
    <bean id="userComponent" class="com.example.spring.inject.demo.component.UserComponent"/>
</beans>
```



我们常用的ApplicationContext有：

- `GenericApplicationContext`及其子类`AnnotationConfigApplicationContext`

常用的`BeanFactory`有：

- `DefaultListableBeanFactory`



### 向容器中添加bean的方式

---

添加Bean到容器中的方式常见的有 ：

- xml配置【老项目才用到】
- @Component注解及其衍生注解（@Service等）
- @Configuration和@Bean

这是常用的，那不常用的呢？

- `@Import(XXX.class)` 这个注解也能加载Bean

- 实现`ImportSelector`接口，搭配`@Import`注解和spring.factories文件（Spring Boot的SPI原理，后续会介绍）

- 代码手动向容器中注入（BeanFactory的后置处理器）

  ```java
  @Override
  public void postProcessBeanFactory(ConfigurableListableBeanFactory beanFactory) throws BeansException {
      DefaultListableBeanFactory factory = (DefaultListableBeanFactory) beanFactory;
      RootBeanDefinition definition = new RootBeanDefinition(PicComponent.class);
      factory.registerBeanDefinition("picComponent", definition);
  }
  ```

> 网上有些资料，说使用BeanFactoryAware也可以，本地实测在普通的Bean不行（Spring Boot 2.5.9）。
>
> 此处应与容器刷新流程相关。
>
> 1. 拿到BeanFactory
> 2. BeanFactoryPostProcessor接口
> 3. 注册BeanPostProcessor到容器中
> 4. 初始化国际化
> 5. 初始化事件发布器
> 6. 初始化事件监听器
> 7. 初始化所有的单例非懒加载Bean
> 8. 发布容器刷新事件
>
> Bean创建的流程有：
>
> 1. 初始化bean
> 2. 填充属性
> 3. 执行三个Aware接口
> 4. 执行Bean的后置处理器（@PostConstruct、ApplicationContextAware接口）
> 5. 执行InitilzingBean接口的afterPropertiesSet
> 6. 执行init-method

在普通的bean是不可以的，常用的方法是使用BeanFactoryPostProcessor的后置处理方法来手动加载Bean，比如MyBatis就是使用这个接口，将Mapper接口放入了容器，然后我们就能像普通的Bean一样注入，并调用SQL语句。



### FactoryBean的实现原理

---

使用FactoryBean可以让程序员去定制实例化Bean的逻辑，甚至可以用来生成代理对象返回给容器进行加载，由此可以做一些事情，比如，Bean初始化的逻辑的修改。

MyBatis是就是利用FactoryBean接口，将Mapper接口的JDK代理对象注入到Spring容器的。

在创建Bean时，会调用`FactoryBean`的`getObject`方法。

```java
// Create bean instance.
if (mbd.isSingleton()) {
   // 会创建FactoryBean放入到容器
   sharedInstance = getSingleton(beanName, () -> {
      try {
         return createBean(beanName, mbd, args);
      }
      catch (BeansException ex) {
         destroySingleton(beanName);
         throw ex;
      }
   });
   // 调用FactoryBean的getObject方法，如果不是FactoryBean就会跳过
   beanInstance = getObjectForBeanInstance(sharedInstance, name, beanName, mbd);
}
```



### BeanName

---

在扫描Bean时，Bean的名称默认是使用驼峰命名法。我们也可以使用注解或者配置文件的方式，来指定beanName。

默认情况下，BeanName的生成，是基于BeanNameGenerator这个接口。我们可以重写这个接口的子类来实现自定义BeanNameGenerator。这里的场景是，微内核架构的插件，Spring在加载所需的插件时，每个插件可能会有同名的Bean，这样就会造成覆盖。因此可以自定义BeanNameGenerator，来给BeanName拼接后缀来区别。



如果想统一对某个包下的Bean进行特殊命名，那么要使用`@ComponentScan`指定`BeanNameGenerator`。

如果不想使用注解的方式，也可以手动调用applicationContext来注册beanDefinition。

```java
applicationContext.registBeanDefinition(beanName,beanDefinition);
```



### 事件监听机制

---

Spring容器发布事件和接收事件很简单：

```java
@Autowired
private ApplicationContext applicationContext;

public void event(){
    applicationContext.pubilshEvent(new CustomEvent(this));
}
```

```java
@Component
public class CustomEventListener implements ApplicationListener<CustomEvent> {
    @Override
    public void onApplicationEvent(CustomEvent event) {
        System.out.println("receiver custom event...");
    }
}
```



容器刷新时，会初始化事件发布器和事件监听器。

`AbstractApplicationContext#refresh`

```java
// Initialize event multicaster for this context.
initApplicationEventMulticaster();

// Initialize other special beans in specific context subclasses.
onRefresh();

// Check for listener beans and register them.
registerListeners();
```

registerListener，会将监听器注册到事件发布器中的一个`LinkedHashSet`。

```java
public void addApplicationListener(ApplicationListener<?> listener) {
   synchronized (this.defaultRetriever) {
      Object singletonTarget = AopProxyUtils.getSingletonTarget(listener);
      if (singletonTarget instanceof ApplicationListener) {
         this.defaultRetriever.applicationListeners.remove(singletonTarget);
      }
      this.defaultRetriever.applicationListeners.add(listener);
      this.retrieverCache.clear();
   }
}
```

事件发布器发布事件时，就是根据事件类型将事件监听器找出来，然后调用即可。

```java
@Override
public void multicastEvent(final ApplicationEvent event, @Nullable ResolvableType eventType) {
   ResolvableType type = (eventType != null ? eventType : resolveDefaultEventType(event));
   Executor executor = getTaskExecutor();
   for (ApplicationListener<?> listener : getApplicationListeners(event, type)) {
      if (executor != null) {
         executor.execute(() -> invokeListener(listener, event));
      }
      else {
         invokeListener(listener, event);
      }
   }
}
```

默认是同步执行的（executor默认情况下是为空的）。

如果要设置为异步，可以对这个executor进行赋值。



### MyBatis的Mapper接口如何加载到Spring容器

---

在上面的内容记录了 ：

- 代码加载Bean的的实践方式有 ：使用BeanFactoryPostProcessor后置处理器去手动加载
- FactoryBean可以定制Bean的实例化过程，甚至可以手动生成代理对象放进去，这一点想象空间就非常大，以做很多的事情。

MyBatis搭配Spring的自动加载Mapper接口就是基于这两点原理。

回忆一下，不使用Spring的MyBatis的使用流程 ：

1. SqlSessionFactoryBuilder根据配置文件生成SqlSessionFactory
2. SqlSessionFactory可以创建SqlSession
3. SqlSession使用`getMapper(Class)`获取Mapper对象然后调用方法或者`sqlSession.selectList`直接拿到结果。如果是增删改，会根据结果还是异常去决定是否回滚。

从源码可得知`sqlSession#getMapper（）`其实是通过MapperRegistry拿到`MapperProxyFactory`，然后`MapperProxyFactory`可以创造`MapperProxy`，这就是一个JDK代理对象。调用`Mapper`接口的各种方法，最终会调用到`MapperProxy#invoke`方法，`invoke`方法最终还是根据增删改查来调用`sqlSession`去执行增删改查。



MyBatis的原理就是以上了，但MyBatis和Spring搭配就有以下疑问了 ：

- Mapper接口没有@Component注解，怎么会被扫描到的？
- Mapper接口没有实现，怎么加载到容器中的？（接口不能实现就不能加载到容器咯）



原理之前已经介绍过了，下面开始源码分析 ：

MyBatis有个配置类`MapperScannerConfigurer`实现了`BeanDefinitionRegistryPostProcessor`接口 ：

```java
public class MapperScannerConfigurer
    implements BeanDefinitionRegistryPostProcessor, InitializingBean, ApplicationContextAware, BeanNameAware {
```

它实现了`postProcessBeanDefinitionRegistry`方法，并在这个方法中创建了一个`ClassPathMapperScanner`（ClassPathBeanDefinitionScanner的子类），并调用了这个`ClassPathMapperScanner`去配置的`basePackages`扫描。

```java
public void postProcessBeanDefinitionRegistry(BeanDefinitionRegistry registry) {
  if (this.processPropertyPlaceHolders) {
    processPropertyPlaceHolders();
  }

  ClassPathMapperScanner scanner = new ClassPathMapperScanner(registry);
  // 省略配置scanner的代码
  scanner.registerFilters();
  scanner.scan(
      StringUtils.tokenizeToStringArray(this.basePackage, ConfigurableApplicationContext.CONFIG_LOCATION_DELIMITERS));
}
```

`ClassPathMapperScanner#doScan`会首先调用父类的`ClassPathBeanDefinitionScanner#scan`方法。



**MapperScan注解实现原理**

MapperScan是MyBatis自定义的注解，用于指定Mapper接口的类或包的位置，然后Spring就会将指定的类或包加载到容器中。

首先看MapperScan注解 ：

```java
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.TYPE)
@Documented
@Import(MapperScannerRegistrar.class)
@Repeatable(MapperScans.class)
public @interface MapperScan {
 	// 省略属性值   
}
```

重点中`@Import`导入的这个类。



`MapperScannerRegistrar`是`ImportBeanDefinitionRegitrar`的子类 :

```java
public class MapperScannerRegistrar implements ImportBeanDefinitionRegistrar{
  // 读取MapperScan注解中的值，并放入BeanDefinitionMap中
  @Override
  public void registerBeanDefinitions(AnnotationMetadata importingClassMetadata, BeanDefinitionRegistry registry) {
    AnnotationAttributes mapperScanAttrs = AnnotationAttributes
        .fromMap(importingClassMetadata.getAnnotationAttributes(MapperScan.class.getName()));
    if (mapperScanAttrs != null) {
      registerBeanDefinitions(importingClassMetadata, mapperScanAttrs, registry,
          generateBaseBeanName(importingClassMetadata, 0));
    }
  }
  
  // 解析MapperScan注解的值，并放入property中。后续会在MapperScanConfigurer中取出并去扫描指定的Package
  void registerBeanDefinitions(AnnotationMetadata annoMeta, AnnotationAttributes annoAttrs,
      BeanDefinitionRegistry registry, String beanName) {

    BeanDefinitionBuilder builder = BeanDefinitionBuilder.genericBeanDefinition(MapperScannerConfigurer.class);
    builder.addPropertyValue("processPropertyPlaceHolders", true);

    Class<? extends Annotation> annotationClass = annoAttrs.getClass("annotationClass");
    if (!Annotation.class.equals(annotationClass)) {
      builder.addPropertyValue("annotationClass", annotationClass);
    }

    Class<?> markerInterface = annoAttrs.getClass("markerInterface");
    if (!Class.class.equals(markerInterface)) {
      builder.addPropertyValue("markerInterface", markerInterface);
    }

    Class<? extends BeanNameGenerator> generatorClass = annoAttrs.getClass("nameGenerator");
    if (!BeanNameGenerator.class.equals(generatorClass)) {
      builder.addPropertyValue("nameGenerator", BeanUtils.instantiateClass(generatorClass));
    }

    Class<? extends MapperFactoryBean> mapperFactoryBeanClass = annoAttrs.getClass("factoryBean");
    if (!MapperFactoryBean.class.equals(mapperFactoryBeanClass)) {
      builder.addPropertyValue("mapperFactoryBeanClass", mapperFactoryBeanClass);
    }

    String sqlSessionTemplateRef = annoAttrs.getString("sqlSessionTemplateRef");
    if (StringUtils.hasText(sqlSessionTemplateRef)) {
      builder.addPropertyValue("sqlSessionTemplateBeanName", annoAttrs.getString("sqlSessionTemplateRef"));
    }

    String sqlSessionFactoryRef = annoAttrs.getString("sqlSessionFactoryRef");
    if (StringUtils.hasText(sqlSessionFactoryRef)) {
      builder.addPropertyValue("sqlSessionFactoryBeanName", annoAttrs.getString("sqlSessionFactoryRef"));
    }

    List<String> basePackages = new ArrayList<>();
    basePackages.addAll(
        Arrays.stream(annoAttrs.getStringArray("value")).filter(StringUtils::hasText).collect(Collectors.toList()));

    basePackages.addAll(Arrays.stream(annoAttrs.getStringArray("basePackages")).filter(StringUtils::hasText)
        .collect(Collectors.toList()));

    basePackages.addAll(Arrays.stream(annoAttrs.getClassArray("basePackageClasses")).map(ClassUtils::getPackageName)
        .collect(Collectors.toList()));

    if (basePackages.isEmpty()) {
      basePackages.add(getDefaultBasePackage(annoMeta));
    }

    String lazyInitialization = annoAttrs.getString("lazyInitialization");
    if (StringUtils.hasText(lazyInitialization)) {
      builder.addPropertyValue("lazyInitialization", lazyInitialization);
    }

    String defaultScope = annoAttrs.getString("defaultScope");
    if (!AbstractBeanDefinition.SCOPE_DEFAULT.equals(defaultScope)) {
      builder.addPropertyValue("defaultScope", defaultScope);
    }

    builder.addPropertyValue("basePackage", StringUtils.collectionToCommaDelimitedString(basePackages));

    registry.registerBeanDefinition(beanName, builder.getBeanDefinition());

  }

}
```

MapperScanRegistrar读取MapperScan注解的中的属性值，并记录起来。

后续在`MapperScanConfigurer`会读取这些属性值，然后去扫描指定的包或类。

`MapperScanConfigurer#postProcessBeanDefinitionRegistry`

```java
@Override
public void postProcessBeanDefinitionRegistry(BeanDefinitionRegistry registry) {
  if (this.processPropertyPlaceHolders) {
    processPropertyPlaceHolders();
  }

  // 设置属性值
  ClassPathMapperScanner scanner = new ClassPathMapperScanner(registry);
  scanner.setAddToConfig(this.addToConfig);
  scanner.setAnnotationClass(this.annotationClass);
  scanner.setMarkerInterface(this.markerInterface);
  scanner.setSqlSessionFactory(this.sqlSessionFactory);
  scanner.setSqlSessionTemplate(this.sqlSessionTemplate);
  scanner.setSqlSessionFactoryBeanName(this.sqlSessionFactoryBeanName);
  scanner.setSqlSessionTemplateBeanName(this.sqlSessionTemplateBeanName);
  scanner.setResourceLoader(this.applicationContext);
  scanner.setBeanNameGenerator(this.nameGenerator);
  scanner.setMapperFactoryBeanClass(this.mapperFactoryBeanClass);
  if (StringUtils.hasText(lazyInitialization)) {
    scanner.setLazyInitialization(Boolean.valueOf(lazyInitialization));
  }
  if (StringUtils.hasText(defaultScope)) {
    scanner.setDefaultScope(defaultScope);
  }
  scanner.registerFilters();
  // 扫描指定的package
  scanner.scan(
      StringUtils.tokenizeToStringArray(this.basePackage, ConfigurableApplicationContext.CONFIG_LOCATION_DELIMITERS));
}
```

在实际开发中，我们也可以使用这种方法，来自定义注解，去扫描指定的包。



### Spring循环依赖

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

SpringBoot其实是对Spring的简化，在没有SpringBoot时，启动一个Spring程序要写配置文件，要手动`new ConfigurableApplicationContext`，然后还要手动注册销毁钩子`context.registShutdownHook`。写代码倒是其次，主要是写配置文件太麻烦了。并且还要自己一个一个往`pom.xml`写依赖。这也是一种麻烦，是可避免的。



针对以上痛点，Spring Boot做了以下的优化 ：

1. 减少了xml配置文件
2. SpringBoot启动及其简单，使用注解+main方法就可以启动（底层还是调用了ConfigurableApplocationContext）。
3. 更方便的依赖管理（虽然有时候会引入不需要的jar包）



### Spring Boot自动装配原理

---

假设你要实现插件式、可插拔的功能需求，要怎么来实现？

Java中有一个特性，可以满足这个 ：SPI。Dubbo就是使用改良版的SPI来做插件式开发。

Spring则是提供了一种类SPI机制，更为的灵活。



在SpringBoot程序中，我们要标注`@SpringBootApplication`注解，否则Spring Boot就不会去扫描包。

理所当然，这个`@SpringBootApplication`注解被`@ComponentScan`注解标注。

```java
@SpringBootConfiguration
@EnableAutoConfiguration
@ComponentScan(excludeFilters = { @Filter(type = FilterType.CUSTOM, classes = TypeExcludeFilter.class),
      @Filter(type = FilterType.CUSTOM, classes = AutoConfigurationExcludeFilter.class) })
public @interface SpringBootApplication {//省略}
```

`@ComponentScan`我们认识，但是`@EnableAutoConfiguration`是什么情况？

```java
@AutoConfigurationPackage
@Import(AutoConfigurationImportSelector.class)
public @interface EnableAutoConfiguration {}
```

可以看到这个`@EnableAutoConfiguration`使用了`@Import`注解标注了，之前有说过，`@Import`注解可以手动向容器添加bean。

说到`@Import`注解，之前提过，处理`@Import`注解的步骤在`ConfigurationClassParser#processImports`方法中。

```java
for (SourceClass candidate : importCandidates) {
	if (candidate.isAssignable(ImportSelector.class)) {
	// Candidate class is an ImportSelector -> delegate to it to determine imports
	Class<?> candidateClass = candidate.loadClass();
	ImportSelector selector = ParserStrategyUtils.instantiateClass(candidateClass, ImportSelector.class,
	this.environment, this.resourceLoader, this.registry);
	Predicate<String> selectorFilter = selector.getExclusionFilter();
	if (selectorFilter != null) {
		exclusionFilter = exclusionFilter.or(selectorFilter);
	}
	if (selector instanceof DeferredImportSelector) {
        // 如果selector是DeferredImportSelector子类，则通过handle，
        // handle方法不会调用ImportSelector#selectImport
		this.deferredImportSelectorHandler.handle(configClass, (DeferredImportSelector) selector);
	}else {
        // 如果不是DeferredImportSelector子类，则调用selectImport方法
		String[] importClassNames = selector.selectImports(currentSourceClass.getMetadata());
		Collection<SourceClass> importSourceClasses = asSourceClasses(importClassNames, exclusionFilter);
		processImports(configClass, currentSourceClass, importSourceClasses, exclusionFilter, false);
			}
	}else if(candidate.isAssignable(ImportBeanDefinitionRegistrar.class)) {
		// Candidate class is an ImportBeanDefinitionRegistrar ->
		// delegate to it to register additional bean definitions
		Class<?> candidateClass = candidate.loadClass();
		ImportBeanDefinitionRegistrar registrar =
								ParserStrategyUtils.instantiateClass(candidateClass, ImportBeanDefinitionRegistrar.class,
		this.environment, this.resourceLoader, this.registry);
	configClass.addImportBeanDefinitionRegistrar(registrar,currentSourceClass.getMetadata());
	}else {
		// Candidate class not an ImportSelector or ImportBeanDefinitionRegistrar ->
		// process it as an @Configuration class
        // 处理Configuration注解
		this.importStack.registerImport(currentSourceClass.getMetadata(), candidate.getMetadata().getClassName());
		processConfigurationClass(candidate.asConfigClass(configClass), exclusionFilter);
			}
	}
```

在`processImports`方法中，会处理`ImportSelector`、`ImportBeanDefinitionRegistrar`、`@Configuration`注解的类。

而`AutoConfigurationImportSelector`类是`DeferredImportSelector`的子类，因此不会调用`selectImports`方法（在老版本的Spring Boot是会调用这个方法的），而是会调用`getAutoConfigurationEntry`这个方法。

```java
protected AutoConfigurationEntry getAutoConfigurationEntry(AnnotationMetadata annotationMetadata) {
   if (!isEnabled(annotationMetadata)) {
      return EMPTY_ENTRY;
   }
   AnnotationAttributes attributes = getAttributes(annotationMetadata);
   // 调用SpringFactoryLoader拿到spring.factories中的配置类
   List<String> configurations = getCandidateConfigurations(annotationMetadata, attributes);
   // 去重
   configurations = removeDuplicates(configurations);
   // 去除注解中配置要清除的
   Set<String> exclusions = getExclusions(annotationMetadata, attributes);
   checkExcludedClasses(configurations, exclusions);
   configurations.removeAll(exclusions);
   configurations = getConfigurationClassFilter().filter(configurations);
   // 发布事件
   fireAutoConfigurationImportEvents(configurations, exclusions);
   return new AutoConfigurationEntry(configurations, exclusions);
}
```

```java
protected List<String> getCandidateConfigurations(AnnotationMetadata metadata, AnnotationAttributes attributes) {
   // 调用springFactoriesLoader拿到所有的配置类
   List<String> configurations = 		SpringFactoriesLoader.loadFactoryNames(getSpringFactoriesLoaderFactoryClass(),
         getBeanClassLoader());
   return configurations;
}
```



如果需要spring boot加载自定义的starter，只需要在`spring.factories`文件中添加自定义的`ImportSelector`实现类或`@Configuration`标注的类即可。如 ：

```properties
org.springframework.boot.autoconfigure.EnableAutoConfiguration=com.exmple.component.importer.CustomizeImportSelector
```

```java
public class CustomizeImportSelector implements ImportSelector {
    @Override
    public String[] selectImports(AnnotationMetadata importingClassMetadata) {
        return new String[]{"com.exmple.component.configuration.AutoImportConfiguration"};
    }
}
```

这样就可以使用到Spring Boot的SPI机制。



**总结 ：**Spring Boot是对Spring的简化，使用了`@ComponentScan`注解扫描Main类的包，因此不需要我们手动写`@ComponentScan`或者配置；使用了`@EnableAutoConfiguration`注解，这个注解使用了`@Import`注解标注，并导入了`AutoConfigurationImportSelector`类，这个类会调用SpringFactoriesLoader从spring.factories拿到所有所需要的加载配置类。



### Spring使用的设计模式

---

- 单例
- 工厂
- 模板
- 适配器
- 代理模式
- 观察者模式
- 策略模式
- 装饰器
