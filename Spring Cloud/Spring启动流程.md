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