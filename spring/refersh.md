## refersh

---

`refersh()`容器刷新方法。会按顺序调用一下几个方法来完成容器的刷新操作。

1.  [`prepareRefresh()`](#1)
2. `obtainFreshBeanFactory()`
3. `prepareBeanFactory(beanFactory)`
4. `postProcessBeanFactory(beanFactory)`
5. `invokeBeanFactoryPostProcessors(beanFactory)`
6. `registerBeanPostProcessors(beanFactory)`
7. `initMessageSource()` 
8. `initMessageSource()` 
9. `onRefresh()` 
10. `registerListeners()` 
11. `finishBeanFactoryInitialization(beanFactory)` 
12. `finishRefresh()`

### <span id="1">`prepareRefresh()`</span>: 

​	刷新前的预处理

```java
/**
	 * Prepare this context for refreshing, setting its startup date and
	 * active flag as well as performing any initialization of property sources.
	 */
	protected void prepareRefresh() {
		this.startupDate = System.currentTimeMillis();
		this.closed.set(false);
		this.active.set(true);

		if (logger.isDebugEnabled()) {
			if (logger.isTraceEnabled()) {
				logger.trace("Refreshing " + this);
			} else {
				logger.debug("Refreshing " + getDisplayName());
			}
		}

		// Initialize any placeholder property sources in the context environment
		initPropertySources();

		// Validate that all properties marked as required are resolvable
		// see ConfigurablePropertyResolver#setRequiredProperties
		getEnvironment().validateRequiredProperties();

		// Allow for the collection of early ApplicationEvents,
		// to be published once the multicaster is available...
		this.earlyApplicationEvents = new LinkedHashSet<>();
	}
```

+ `initPropertySources()`初始化属性设置，里面是空方法，为了是让子类继承后可以复写这个方法去实现一些个性化的属性设置。

+ `	getEnvironment().validateRequiredProperties()`属性的校验

+ `this.earlyApplicationEvents = new LinkedHashSet<>()`用于保存容器早期发生的事件，使得在派发器实例化后可以派发这些事件。

###  `obtainFreshBeanFactory()`:

​	获取`BeanFactory`

```java
/**
	 * Tell the subclass to refresh the internal bean factory.
	 *
	 * @return the fresh BeanFactory instance
	 * @see #refreshBeanFactory()
	 * @see #getBeanFactory()
	 */
	protected ConfigurableListableBeanFactory obtainFreshBeanFactory() {
		refreshBeanFactory();
		return getBeanFactory();
	}

/**
	 * Do nothing: We hold a single internal BeanFactory and rely on callers
	 * to register beans through our public methods (or the BeanFactory's).
	 * @see #registerBeanDefinition
	 */
	@Override
	protected final void refreshBeanFactory() throws IllegalStateException {
		if (!this.refreshed.compareAndSet(false, true)) {
			throw new IllegalStateException(
					"GenericApplicationContext does not support multiple refresh attempts: just call 'refresh' once");
		}
		this.beanFactory.setSerializationId(getId());
	}
```

+ 通过`refreshBeanFactory()`给`beanFactoiry`设置一个序列话的`id`
+ `beanFactory`的类型为`this.beanFactory = new DefaultListableBeanFactory()`
+ ` getBeanFactory()`返回`GenericApplicationContext.beanFactory() ` `DefaultListableBeanFactory`类型

### `prepareBeanFactory(beanFactory)`:

​	做一些`beanFactory`使用之前的准备工作。	

```java
/**
	 * Configure the factory's standard context characteristics,
	 * such as the context's ClassLoader and post-processors.
	 *
	 * @param beanFactory the BeanFactory to configure
	 */
	protected void prepareBeanFactory(ConfigurableListableBeanFactory beanFactory) {
		// Tell the internal bean factory to use the context's class loader etc.
		beanFactory.setBeanClassLoader(getClassLoader());
		beanFactory.setBeanExpressionResolver(new StandardBeanExpressionResolver(beanFactory.getBeanClassLoader()));
		beanFactory.addPropertyEditorRegistrar(new ResourceEditorRegistrar(this, getEnvironment()));

		// Configure the bean factory with context callbacks.
		beanFactory.addBeanPostProcessor(new ApplicationContextAwareProcessor(this));
		beanFactory.ignoreDependencyInterface(EnvironmentAware.class);
		beanFactory.ignoreDependencyInterface(EmbeddedValueResolverAware.class);
		beanFactory.ignoreDependencyInterface(ResourceLoaderAware.class);
		beanFactory.ignoreDependencyInterface(ApplicationEventPublisherAware.class);
		beanFactory.ignoreDependencyInterface(MessageSourceAware.class);
		beanFactory.ignoreDependencyInterface(ApplicationContextAware.class);

		// BeanFactory interface not registered as resolvable type in a plain factory.
		// MessageSource registered (and found for autowiring) as a bean.
		beanFactory.registerResolvableDependency(BeanFactory.class, beanFactory);
		beanFactory.registerResolvableDependency(ResourceLoader.class, this);
		beanFactory.registerResolvableDependency(ApplicationEventPublisher.class, this);
		beanFactory.registerResolvableDependency(ApplicationContext.class, this);

		// Register early post-processor for detecting inner beans as ApplicationListeners.
		beanFactory.addBeanPostProcessor(new ApplicationListenerDetector(this));

		// Detect a LoadTimeWeaver and prepare for weaving, if found.
		if (beanFactory.containsBean(LOAD_TIME_WEAVER_BEAN_NAME)) {
			beanFactory.addBeanPostProcessor(new LoadTimeWeaverAwareProcessor(beanFactory));
			// Set a temporary ClassLoader for type matching.
			beanFactory.setTempClassLoader(new ContextTypeMatchClassLoader(beanFactory.getBeanClassLoader()));
		}

		// Register default environment beans.
		if (!beanFactory.containsLocalBean(ENVIRONMENT_BEAN_NAME)) {
			beanFactory.registerSingleton(ENVIRONMENT_BEAN_NAME, getEnvironment());
		}
		if (!beanFactory.containsLocalBean(SYSTEM_PROPERTIES_BEAN_NAME)) {
			beanFactory.registerSingleton(SYSTEM_PROPERTIES_BEAN_NAME, getEnvironment().getSystemProperties());
		}
		if (!beanFactory.containsLocalBean(SYSTEM_ENVIRONMENT_BEAN_NAME)) {
			beanFactory.registerSingleton(SYSTEM_ENVIRONMENT_BEAN_NAME, getEnvironment().getSystemEnvironment());
		}
	}
```

+ 设置`beanFactory`的类加载器，表达式解析器等...
+ 设置忽略一些接口的实现类的自动装配。
+ 设置可以解析的自动装配，使得能在任何组件中自动注入`BeanFactory，ResourceLoader，(ApplicationEventPublisher，ApplicationContext`
+ 添加后置处理器`ApplicationListenerDetector的实现后处理器`
+ 注册`Environment，SystemProperties，SystemEnvironment`

```java
    // 在Bean初始化后检查是否实现了ApplicationListener接口,
    // 是则加入当前的applicationContext的applicationListeners列表
    beanFactory.addBeanPostProcessor(new ApplicationListenerDetector(this));
    // 检查容器中是否包含名称为loadTimeWeaver的bean，实际上是增加Aspectj的支持
    // AspectJ采用编译期织入、类加载期织入两种方式进行切面的织入
    // 类加载期织入简称为LTW（Load Time Weaving）,通过特殊的类加载器来代理JVM默认的类加载器实现
    if (beanFactory.containsBean(LOAD_TIME_WEAVER_BEAN_NAME)) {
        // 添加BEAN后置处理器：LoadTimeWeaverAwareProcessor
        // 在BEAN初始化之前检查BEAN是否实现了LoadTimeWeaverAware接口，
        // 如果是，则进行加载时织入，即静态代理。
        beanFactory.addBeanPostProcessor(new LoadTimeWeaverAwareProcessor(beanFactory));
        // Set a temporary ClassLoader for type matching.
        beanFactory.setTempClassLoader(new ContextTypeMatchClassLoader(beanFactory.getBeanClassLoader()));
    }
作者：wangjie2016
链接：https://www.jianshu.com/p/5836d3d6dc72

```

### `postProcessBeanFactory(beanFactory)`

​	空方法，为了让子类能在`BeanFactory`创建并准备完成后能调用一些自己实现的处理工作。



```java
	/**
	 * Modify the application context's internal bean factory after its standard
	 * initialization. All bean definitions will have been loaded, but no beans
	 * will have been instantiated yet. This allows for registering special
	 * BeanPostProcessors etc in certain ApplicationContext implementations.
	 *
	 * @param beanFactory the bean factory used by the application context
	 */
	protected void postProcessBeanFactory(ConfigurableListableBeanFactory beanFactory) {
	}
```

### `invokeBeanFactoryPostProcessors(beanFactory)`

​	执行`BeanFactoryPostProcessors`接口，大致流程就是获得所有实现了`BeanDefinitionRegistryPostProcessor，BeanFactoryPostProcessor`接口的类，先按优先级执行`BeanDefinitionRegistryPostProcessor`接口中的方法，接着按优先级执行`BeanFactoryPostProcessor`接口中的方法。代码实在长，就不贴了

### `registerBeanPostProcessors(beanFactory)`

 	添加`BeanPostProcessorChecker`,然后注册所有实现了`BeanPostProcessor`接口中方法的实现类，并按`PriorityOrdered、Ordered`接口来设置优先级。`MergedBeanDefinitionPostProcessor`类型的放在`internalPostProcessors`数组中，其他的放在`orderedPostProcessors，nonOrderedPostProcessors`数组中。`internalPostProcessors`是最后被注册的。最后还会添加一个`ApplicationListenerDetector`类型的`BeanPostProcessor`，用于处理监听器。

```java
public static void registerBeanPostProcessors(
			ConfigurableListableBeanFactory beanFactory, AbstractApplicationContext applicationContext) {

		String[] postProcessorNames = beanFactory.getBeanNamesForType(BeanPostProcessor.class, true, false);

		// Register BeanPostProcessorChecker that logs an info message when
		// a bean is created during BeanPostProcessor instantiation, i.e. when
		// a bean is not eligible for getting processed by all BeanPostProcessors.
		int beanProcessorTargetCount = beanFactory.getBeanPostProcessorCount() + 1 + postProcessorNames.length;
		beanFactory.addBeanPostProcessor(new BeanPostProcessorChecker(beanFactory, beanProcessorTargetCount));

		// Separate between BeanPostProcessors that implement PriorityOrdered,
		// Ordered, and the rest.
		List<BeanPostProcessor> priorityOrderedPostProcessors = new ArrayList<>();
		List<BeanPostProcessor> internalPostProcessors = new ArrayList<>();
		List<String> orderedPostProcessorNames = new ArrayList<>();
		List<String> nonOrderedPostProcessorNames = new ArrayList<>();
		for (String ppName : postProcessorNames) {
			if (beanFactory.isTypeMatch(ppName, PriorityOrdered.class)) {
				BeanPostProcessor pp = beanFactory.getBean(ppName, BeanPostProcessor.class);
				priorityOrderedPostProcessors.add(pp);
				if (pp instanceof MergedBeanDefinitionPostProcessor) {
					internalPostProcessors.add(pp);
				}
			}
			else if (beanFactory.isTypeMatch(ppName, Ordered.class)) {
				orderedPostProcessorNames.add(ppName);
			}
			else {
				nonOrderedPostProcessorNames.add(ppName);
			}
		}

		// First, register the BeanPostProcessors that implement PriorityOrdered.
		sortPostProcessors(priorityOrderedPostProcessors, beanFactory);
		registerBeanPostProcessors(beanFactory, priorityOrderedPostProcessors);

		// Next, register the BeanPostProcessors that implement Ordered.
		List<BeanPostProcessor> orderedPostProcessors = new ArrayList<>();
		for (String ppName : orderedPostProcessorNames) {
			BeanPostProcessor pp = beanFactory.getBean(ppName, BeanPostProcessor.class);
			orderedPostProcessors.add(pp);
			if (pp instanceof MergedBeanDefinitionPostProcessor) {
				internalPostProcessors.add(pp);
			}
		}
		sortPostProcessors(orderedPostProcessors, beanFactory);
		registerBeanPostProcessors(beanFactory, orderedPostProcessors);

		// Now, register all regular BeanPostProcessors.
		List<BeanPostProcessor> nonOrderedPostProcessors = new ArrayList<>();
		for (String ppName : nonOrderedPostProcessorNames) {
			BeanPostProcessor pp = beanFactory.getBean(ppName, BeanPostProcessor.class);
			nonOrderedPostProcessors.add(pp);
			if (pp instanceof MergedBeanDefinitionPostProcessor) {
				internalPostProcessors.add(pp);
			}
		}
		registerBeanPostProcessors(beanFactory, nonOrderedPostProcessors);

		// Finally, re-register all internal BeanPostProcessors.
		sortPostProcessors(internalPostProcessors, beanFactory);
		registerBeanPostProcessors(beanFactory, internalPostProcessors);

		// Re-register post-processor for detecting inner beans as ApplicationListeners,
		// moving it to the end of the processor chain (for picking up proxies etc).
		beanFactory.addBeanPostProcessor(new ApplicationListenerDetector(applicationContext));
	}
```

### `initMessageSource()`

​		初始化`MessageSource`组件（用于国际化功能，消息绑定，消息解析）。

```java
protected void initMessageSource() {
		ConfigurableListableBeanFactory beanFactory = getBeanFactory();
		if (beanFactory.containsLocalBean(MESSAGE_SOURCE_BEAN_NAME)) {
			this.messageSource = beanFactory.getBean(MESSAGE_SOURCE_BEAN_NAME, MessageSource.class);
			// Make MessageSource aware of parent MessageSource.
			if (this.parent != null && this.messageSource instanceof HierarchicalMessageSource) {
				HierarchicalMessageSource hms = (HierarchicalMessageSource) this.messageSource;
				if (hms.getParentMessageSource() == null) {
					// Only set parent context as parent MessageSource if no parent MessageSource
					// registered already.
					hms.setParentMessageSource(getInternalParentMessageSource());
				}
			}
			if (logger.isTraceEnabled()) {
				logger.trace("Using MessageSource [" + this.messageSource + "]");
			}
		} else {
			// Use empty MessageSource to be able to accept getMessage calls.
			DelegatingMessageSource dms = new DelegatingMessageSource();
			dms.setParentMessageSource(getInternalParentMessageSource());
			this.messageSource = dms;
			beanFactory.registerSingleton(MESSAGE_SOURCE_BEAN_NAME, this.messageSource);
			if (logger.isTraceEnabled()) {
				logger.trace("No '" + MESSAGE_SOURCE_BEAN_NAME + "' bean, using [" + this.messageSource + "]");
			}
		}
	}

```

查看容器中是否有`id`为`messageSource`，类型为`MessageSource`的组件。如果没有`new DelegatingMessageSource()`,并把`messageSource`注册到容器中。

### `initApplicationEventMulticaster()`

​	初始化事件派发器。具体看`extend.md`中的内容

### `onRefresh()`

​	空方法，留给子类重写。

## `registerListeners()`

​	注册监听器，并将早期产生的事件发布出去。具体可以看`extend.md`的内容

### `finishBeanFactoryInitialization(beanFactory)`

​	注册所有的剩下的非懒加载的单实例`bean`，具体可以看`IOC.md`的内容

### `finishRefresh()`

​	完成容器刷新的最最后处理。

```java
protected void finishRefresh() {
		// Clear context-level resource caches (such as ASM metadata from scanning).
		clearResourceCaches();

		// Initialize lifecycle processor for this context.
		initLifecycleProcessor();

		// Propagate refresh to lifecycle processor first.
		getLifecycleProcessor().onRefresh();

		// Publish the final event.
		publishEvent(new ContextRefreshedEvent(this));

		// Participate in LiveBeansView MBean, if active.
		LiveBeansView.registerApplicationContext(this);
	}
```

+ 清除`Resource`缓存
+ 实例化生命周期处理器`LifecycleProcessor`
+ 调用生命周期处理器的`onRefresh`方法
+ 发布容器刷新完成事件
+ 检查`spring.liveBeansView.mbeanDomain`是否存在，有则创建一个`MBeanServer`

`initLifecycleProcessor()`

```java
/**
	 * Initialize the LifecycleProcessor.
	 * Uses DefaultLifecycleProcessor if none defined in the context.
	 *
	 * @see org.springframework.context.support.DefaultLifecycleProcessor
	 */
	protected void initLifecycleProcessor() {
		ConfigurableListableBeanFactory beanFactory = getBeanFactory();
		if (beanFactory.containsLocalBean(LIFECYCLE_PROCESSOR_BEAN_NAME)) {
			this.lifecycleProcessor =
					beanFactory.getBean(LIFECYCLE_PROCESSOR_BEAN_NAME, LifecycleProcessor.class);
			if (logger.isTraceEnabled()) {
				logger.trace("Using LifecycleProcessor [" + this.lifecycleProcessor + "]");
			}
		} else {
			DefaultLifecycleProcessor defaultProcessor = new DefaultLifecycleProcessor();
			defaultProcessor.setBeanFactory(beanFactory);
			this.lifecycleProcessor = defaultProcessor;
			beanFactory.registerSingleton(LIFECYCLE_PROCESSOR_BEAN_NAME, this.lifecycleProcessor);
			if (logger.isTraceEnabled()) {
				logger.trace("No '" + LIFECYCLE_PROCESSOR_BEAN_NAME + "' bean, using " +
						"[" + this.lifecycleProcessor.getClass().getSimpleName() + "]");
			}
		}
	}
```

查看容器中是否有`LifecycleProcessor`类型，名为`lifecycleProcessor`的生命周期处理器。如果没有就自己穿件一个默认的生命周期处理器`DefaultLifecycleProcessor`，并注册进容器。



























sysy
