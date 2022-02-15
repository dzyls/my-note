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
```

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



### @Autowired的实现

---

`@Autowired`注解是根据类型注入的注解，也是比较常用的注解。

其实现的原理在`AutowiredAnnoationBeanPostProcesser`，从名字上看，这个类是一个`BeanPostProcesser`。

```java
public class AutowiredAnnotationBeanPostProcessor implements SmartInstantiationAwareBeanPostProcessor,
      MergedBeanDefinitionPostProcessor, PriorityOrdered, BeanFactoryAware {
          
      }
```

其中AutowiredAnnoationBeanPostProcesser实现了 MergedBeanDefinitionPostProcesser中的PostPrcessMergedBeanDefinition方法 ：

```java
@Override
public void postProcessMergedBeanDefinition(RootBeanDefinition beanDefinition, Class<?> beanType, String beanName) {
   InjectionMetadata metadata = findAutowiringMetadata(beanName, beanType, null);
   metadata.checkConfigMembers(beanDefinition);
}
```

通过这个方法，来扫描类中的方法和属性，并封装为InjectionMetadata。

```java
private InjectionMetadata findAutowiringMetadata(String beanName, Class<?> clazz, @Nullable PropertyValues pvs) {

   String cacheKey = (StringUtils.hasLength(beanName) ? beanName : clazz.getName());

   InjectionMetadata metadata = this.injectionMetadataCache.get(cacheKey);
   if (InjectionMetadata.needsRefresh(metadata, clazz)) {
      synchronized (this.injectionMetadataCache) {
         metadata = this.injectionMetadataCache.get(cacheKey);
         if (InjectionMetadata.needsRefresh(metadata, clazz)) {
            if (metadata != null) {
               metadata.clear(pvs);
            }
             // 关注重点
            metadata = buildAutowiringMetadata(clazz);
             // 重点在上面
            this.injectionMetadataCache.put(cacheKey, metadata);
         }
      }
   }
   return metadata;
}

private InjectionMetadata buildAutowiringMetadata(Class<?> clazz) {
   if (!AnnotationUtils.isCandidateClass(clazz, this.autowiredAnnotationTypes)) {
      return InjectionMetadata.EMPTY;
   }

   List<InjectionMetadata.InjectedElement> elements = new ArrayList<>();
   Class<?> targetClass = clazz;

   do {
      final List<InjectionMetadata.InjectedElement> currElements = new ArrayList<>();
	// 扫描所有的属性
      ReflectionUtils.doWithLocalFields(targetClass, field -> {
         MergedAnnotation<?> ann = findAutowiredAnnotation(field);
         if (ann != null) {
            if (Modifier.isStatic(field.getModifiers())) {
               if (logger.isInfoEnabled()) {
                  logger.info("Autowired annotation is not supported on static fields: " + field);
               }
               return;
            }
            boolean required = determineRequiredStatus(ann);
            currElements.add(new AutowiredFieldElement(field, required));
         }
      });
	// 扫描所有的方法
      ReflectionUtils.doWithLocalMethods(targetClass, method -> {
         Method bridgedMethod = BridgeMethodResolver.findBridgedMethod(method);
         if (!BridgeMethodResolver.isVisibilityBridgeMethodPair(method, bridgedMethod)) {
            return;
         }
         MergedAnnotation<?> ann = findAutowiredAnnotation(bridgedMethod);
         if (ann != null && method.equals(ClassUtils.getMostSpecificMethod(method, clazz))) {
            if (Modifier.isStatic(method.getModifiers())) {
               if (logger.isInfoEnabled()) {
                  logger.info("Autowired annotation is not supported on static methods: " + method);
               }
               return;
            }
            if (method.getParameterCount() == 0) {
               if (logger.isInfoEnabled()) {
                  logger.info("Autowired annotation should only be used on methods with parameters: " +
                        method);
               }
            }
            boolean required = determineRequiredStatus(ann);
            PropertyDescriptor pd = BeanUtils.findPropertyForMethod(bridgedMethod, clazz);
            currElements.add(new AutowiredMethodElement(method, required, pd));
         }
      });

      elements.addAll(0, currElements);
      // 从当前类，一直扫描到父类，直到Object（父类的属性也会被注入）
      targetClass = targetClass.getSuperclass();
   }
   while (targetClass != null && targetClass != Object.class);

   return InjectionMetadata.forElements(elements, clazz);
}
```



那代码是如何运行到AutowiredAnnoationBeanPostProcesser的呢？

可以猜测一下，什么时候会对类需要注入的属性进行扫描呢？

答案是在扫描到要加载的Bean之后，才会去扫描这些Bean中需要注入的属性。那就是在容器启动时，即容器的刷新方法（refresh)。

refresh方法在`AbstractApplicationContext`抽象父类中（模板方法模式），在容器刷新时，会执行一个方法叫 ：`registerBeanPostProcessors`。

```java
@Override
public void refresh() throws BeansException, IllegalStateException {
   synchronized (this.startupShutdownMonitor) {
		// 省略部分重点
         registerBeanPostProcessors(beanFactory);
         beanPostProcess.end();
```

这个方法会执行所有的BeanPostProcessor。