## extend

---

`BeanFactoryPostProcessors`接口的`postProcessBeanFactory()`在`BeanFactory`标准初始化后，且所有`Bean`信息被加载但还没有`Bean`被实例化时执行。在`refersh()->invokeBeanFactoryPostProcessors(beanFactory)`方法内被找到并执行。具体流程看源码

```java
@FunctionalInterface
public interface BeanFactoryPostProcessor {

	/**
	 * Modify the application context's internal bean factory after its standard
	 * initialization. All bean definitions will have been loaded, but no beans
	 * will have been instantiated yet. This allows for overriding or adding
	 * properties even to eager-initializing beans.
	 * @param beanFactory the bean factory used by the application context
	 * @throws org.springframework.beans.BeansException in case of errors
	 */
	void postProcessBeanFactory(ConfigurableListableBeanFactory beanFactory) throws BeansException;

}
```

`BeanDefinitionRegistryPostProcessor`接口的`postProcessBeanDefinitionRegistry()`方法在所有标准`bean`定义信息将要被加载，但没有`bean`被实例化时执行。在`BeanFactoryPostProcessors`之前执行。可以通过`BeanDefinitionRegistry `中的方法给容器中添加组件。

```java
 */
public interface BeanDefinitionRegistryPostProcessor extends BeanFactoryPostProcessor {

	/**
	 * Modify the application context's internal bean definition registry after its
	 * standard initialization. All regular bean definitions will have been loaded,
	 * but no beans will have been instantiated yet. This allows for adding further
	 * bean definitions before the next post-processing phase kicks in.
	 * @param registry the bean definition registry used by the application context
	 * @throws org.springframework.beans.BeansException in case of errors
	 */
	void postProcessBeanDefinitionRegistry(BeanDefinitionRegistry registry) throws BeansException;

}

```

`ApplicationListener`  `spring`的事件监听器，用于监听`spring`发布的事件，也能自己通过`annotationConfigApplicationContext.publishEvent(ApplicationEvent)`方法来发布事件。

`publishEvent()`方法：

```java
	/**
	 * Publish the given event to all listeners.
	 * <p>Note: Listeners get initialized after the MessageSource, to be able
	 * to access it within listener implementations. Thus, MessageSource
	 * implementations cannot publish events.
	 *
	 * @param event the event to publish (may be an {@link ApplicationEvent}
	 *              or a payload object to be turned into a {@link PayloadApplicationEvent})
	 */
	@Override
	public void publishEvent(Object event) {
		publishEvent(event, null);
	}

	/**
	 * Publish the given event to all listeners.
	 *
	 * @param event     the event to publish (may be an {@link ApplicationEvent}
	 *                  or a payload object to be turned into a {@link PayloadApplicationEvent})
	 * @param eventType the resolved event type, if known
	 * @since 4.2
	 */
	protected void publishEvent(Object event, @Nullable ResolvableType eventType) {
		Assert.notNull(event, "Event must not be null");

		// Decorate event as an ApplicationEvent if necessary
		ApplicationEvent applicationEvent;
		if (event instanceof ApplicationEvent) {
			applicationEvent = (ApplicationEvent) event;
		} else {
			applicationEvent = new PayloadApplicationEvent<>(this, event);
			if (eventType == null) {
				eventType = ((PayloadApplicationEvent) applicationEvent).getResolvableType();
			}
		}

		// Multicast right now if possible - or lazily once the multicaster is initialized
		if (this.earlyApplicationEvents != null) {
			this.earlyApplicationEvents.add(applicationEvent);
		} else {
			getApplicationEventMulticaster().multicastEvent(applicationEvent, eventType);
		}

		// Publish event via parent context as well...
		if (this.parent != null) {
			if (this.parent instanceof AbstractApplicationContext) {
				((AbstractApplicationContext) this.parent).publishEvent(event, eventType);
			} else {
				this.parent.publishEvent(event);
			}
		}
	}
```



`getApplicationEventMulticaster().multicastEvent(applicationEvent, eventType)`，通过`EventMulticaster`事件派发器的`multicastEvent()`方法来派发事件。

`SimpleApplicationEventMulticaster.multicastEvent()`：

```java

	@Override
	public void multicastEvent(ApplicationEvent event) {
		multicastEvent(event, resolveDefaultEventType(event));
	}

	@Override
	public void multicastEvent(final ApplicationEvent event, @Nullable ResolvableType eventType) {
		ResolvableType type = (eventType != null ? eventType : resolveDefaultEventType(event));
		for (final ApplicationListener<?> listener : getApplicationListeners(event, type)) {
			Executor executor = getTaskExecutor();
			if (executor != null) {
				executor.execute(() -> invokeListener(listener, event));
			}
			else {
				invokeListener(listener, event);
			}
		}
	}
```

通过` getApplicationListeners(event, type)`获取监听当前事件的监听器数组，遍历。

如果当前监听器的`executor!=null`,那么交由`executor.executor()`来决定以什么方式来执行（是否异步）。若`executor==null`以同步的方式执行`invokeListener()->doInvokeListener()->onApplicationEvent()`.

`getApplicationListeners(event, type)`：获取监听器数组

```Java
protected Collection<ApplicationListener<?>> getApplicationListeners(
			ApplicationEvent event, ResolvableType eventType) {

		Object source = event.getSource();
		Class<?> sourceType = (source != null ? source.getClass() : null);
		ListenerCacheKey cacheKey = new ListenerCacheKey(eventType, sourceType);

		// Quick check for existing entry on ConcurrentHashMap...
		ListenerRetriever retriever = this.retrieverCache.get(cacheKey);
		if (retriever != null) {
			return retriever.getApplicationListeners();
		}

		if (this.beanClassLoader == null ||
				(ClassUtils.isCacheSafe(event.getClass(), this.beanClassLoader) &&
						(sourceType == null || ClassUtils.isCacheSafe(sourceType, this.beanClassLoader)))) {
			// Fully synchronized building and caching of a ListenerRetriever
			synchronized (this.retrievalMutex) {
				retriever = this.retrieverCache.get(cacheKey);
				if (retriever != null) {
					return retriever.getApplicationListeners();
				}
				retriever = new ListenerRetriever(true);
				Collection<ApplicationListener<?>> listeners =
						retrieveApplicationListeners(eventType, sourceType, retriever);
				this.retrieverCache.put(cacheKey, retriever);
				return listeners;
			}
		}
		else {
			// No ListenerRetriever caching -> no synchronization necessary
			return retrieveApplicationListeners(eventType, sourceType, null);
		}
	}
```

`getApplicationEventMulticaster()`获事件派发器

```java
/**
	 * Return the internal ApplicationEventMulticaster used by the context.
	 *
	 * @return the internal ApplicationEventMulticaster (never {@code null})
	 * @throws IllegalStateException if the context has not been initialized yet
	 */
	ApplicationEventMulticaster getApplicationEventMulticaster() throws IllegalStateException {
		if (this.applicationEventMulticaster == null) {
			throw new IllegalStateException("ApplicationEventMulticaster not initialized - " +
					"call 'refresh' before multicasting events via the context: " + this);
		}
		return this.applicationEventMulticaster;
	}
```

`this.applicationEventMulticaster`的值在`refersh()->initApplicationEventMulticaster()`设置。

`initApplicationEventMulticaster()`：

```java
	/**
	 * Initialize the ApplicationEventMulticaster.
	 * Uses SimpleApplicationEventMulticaster if none defined in the context.
	 *
	 * @see org.springframework.context.event.SimpleApplicationEventMulticaster
	 */
	protected void initApplicationEventMulticaster() {
		ConfigurableListableBeanFactory beanFactory = getBeanFactory();
		if (beanFactory.containsLocalBean(APPLICATION_EVENT_MULTICASTER_BEAN_NAME)) {
			this.applicationEventMulticaster =
					beanFactory.getBean(APPLICATION_EVENT_MULTICASTER_BEAN_NAME, ApplicationEventMulticaster.class);
			if (logger.isTraceEnabled()) {
				logger.trace("Using ApplicationEventMulticaster [" + this.applicationEventMulticaster + "]");
			}
		} else {
			this.applicationEventMulticaster = new SimpleApplicationEventMulticaster(beanFactory);
			beanFactory.registerSingleton(APPLICATION_EVENT_MULTICASTER_BEAN_NAME, this.applicationEventMulticaster);
			if (logger.isTraceEnabled()) {
				logger.trace("No '" + APPLICATION_EVENT_MULTICASTER_BEAN_NAME + "' bean, using " +
						"[" + this.applicationEventMulticaster.getClass().getSimpleName() + "]");
			}
		}
	}
```

如果有容器理里有`PPLICATION_EVENT_MULTICASTER_BEAN_NAME = "applicationEventMulticaster"，ApplicationEventMulticaster.class`则直接从容器中获取，如果没有`new SimpleApplicationEventMulticaster(beanFactory)`.最后将`ApplicationEventMulticaster`放入`this.applicationEventMulticaster`字段，并且把`ApplicationEventMulticaster`注册进容器。

在容器刷新完成后会调用`finishRefresh()->publishEvent(new ContextRefreshedEvent(this))`发布`ContextRefreshedEvent`事件。

在调用容器的`close()`方法时，会调用`doClose()->publishEvent(new ContextClosedEvent(this))`发布`ContextClosedEvent`事件

`registerListeners()`注册监听器，并将早期收集（`refersh`调用后，到`registerListeners()`）的事件发布

```java
	/**
	 * Add beans that implement ApplicationListener as listeners.
	 * Doesn't affect other listeners, which can be added without being beans.
	 */
	protected void registerListeners() {
		// Register statically specified listeners first.
		for (ApplicationListener<?> listener : getApplicationListeners()) {
			getApplicationEventMulticaster().addApplicationListener(listener);
		}

		// Do not initialize FactoryBeans here: We need to leave all regular beans
		// uninitialized to let post-processors apply to them!
		String[] listenerBeanNames = getBeanNamesForType(ApplicationListener.class, true, false);
		for (String listenerBeanName : listenerBeanNames) {
			getApplicationEventMulticaster().addApplicationListenerBean(listenerBeanName);
		}

		// Publish early application events now that we finally have a multicaster...
		Set<ApplicationEvent> earlyEventsToProcess = this.earlyApplicationEvents;
		this.earlyApplicationEvents = null;
		if (earlyEventsToProcess != null) {
			for (ApplicationEvent earlyEvent : earlyEventsToProcess) {
				getApplicationEventMulticaster().multicastEvent(earlyEvent);
			}
		}
	}
```



`@EventListener`标识在方法上，使得一个方法可以监听事件。`classes`属性可以设置要监听的类型。

```java
 *
 * @author Stephane Nicoll
 * @since 4.2
 * @see EventListenerMethodProcessor
 */
@Target({ElementType.METHOD, ElementType.ANNOTATION_TYPE})
@Retention(RetentionPolicy.RUNTIME)
@Documented
public @interface EventListener {

	/**
	 * Alias for {@link #classes}.
	 */
	@AliasFor("classes")
	Class<?>[] value() default {};

	/**
	 * The event classes that this listener handles.
	 * <p>If this attribute is specified with a single value, the
	 * annotated method may optionally accept a single parameter.
	 * However, if this attribute is specified with multiple values,
	 * the annotated method must <em>not</em> declare any parameters.
	 */
	@AliasFor("value")
	Class<?>[] classes() default {};

	/**
	 * Spring Expression Language (SpEL) attribute used for making the
	 * event handling conditional.
	 * <p>Default is {@code ""}, meaning the event is always handled.
	 * <p>The SpEL expression evaluates against a dedicated context that
	 * provides the following meta-data:
	 * <ul>
	 * <li>{@code #root.event}, {@code #root.args} for
	 * references to the {@link ApplicationEvent} and method arguments
	 * respectively.</li>
	 * <li>Method arguments can be accessed by index. For instance the
	 * first argument can be accessed via {@code #root.args[0]}, {@code #p0}
	 * or {@code #a0}. Arguments can also be accessed by name if that
	 * information is available.</li>
	 * </ul>
	 */
	String condition() default "";

}

```

在源码中说到可以看看`EventListenerMethodProcessor`的实现，也就是说这个注解是用`EventListenerMethodProcessor`来解析的。

`EventListenerMethodProcessor`继承了`SmartInitializingSingleton`并重写了`afterSingletonsInstantiated`

`SmartInitializingSingleton`的`afterSingletonsInstantiated`在所有的单实例`bean`初始化后被调用。

```java
public interface SmartInitializingSingleton {

	/**
	 * Invoked right at the end of the singleton pre-instantiation phase,
	 * with a guarantee that all regular singleton beans have been created
	 * already. {@link ListableBeanFactory#getBeansOfType} calls within
	 * this method won't trigger accidental side effects during bootstrap.
	 * <p><b>NOTE:</b> This callback won't be triggered for singleton beans
	 * lazily initialized on demand after {@link BeanFactory} bootstrap,
	 * and not for any other bean scope either. Carefully use it for beans
	 * with the intended bootstrap semantics only.
	 */
	void afterSingletonsInstantiated();

}
```

也就是对应与`refersh()->finishBeanFactoryInitialization(beanFactory)->beanFactory.preInstantiateSingletons()`方法中，创建完所有的单实例`bean`(非懒加载的)后会执行下面代码

```java
	// Trigger post-initialization callback for all applicable beans...
		for (String beanName : beanNames) {
			Object singletonInstance = getSingleton(beanName);
			if (singletonInstance instanceof SmartInitializingSingleton) {
				final SmartInitializingSingleton smartSingleton = (SmartInitializingSingleton) singletonInstance;
				if (System.getSecurityManager() != null) {
					AccessController.doPrivileged((PrivilegedAction<Object>) () -> {
						smartSingleton.afterSingletonsInstantiated();
						return null;
					}, getAccessControlContext());
				}
				else {
					smartSingleton.afterSingletonsInstantiated();
				}
			}
		}
```

也就是如果`bean`继承了`SmartInitializingSingleton`接口的话，调用`bean`的`afterSingletonsInstantiated()`方法。

