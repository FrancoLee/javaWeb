## IOC

---

 `@Configuration` 注解，表示这个类是个配置类

`@ComponentScan`  配置包扫描策略，`excludeFilters`和`includeFilters`通过配置规则排除和包含那些类，如果使用`includeFilters`实现只包含时要先禁用`useDefaultFilters`。规则使用`@Filter`来配置。

`@Filter` 过滤器,可以指定过滤规则。可以通过`ANNOTATION,ASSIGNABLE_TYPE,ASPECTJ,REGEX,CUSTOM`这5中方式来定义规则。`CUSTOM`是自己定义过滤器，实现类要继承`TypeFilter`接口。

`@Condition` 按条件来决定是否加载一个`bean`,参数为一个继承了`Condition`接口的实现类。

`@Import` 快速的导入一个组件，可以使用`@ImportSeletor`来定义那些组件要被`IOC`容器注册。`@ImportSeletor	`的参数是一个继承了`ImportSelector`接口的是实现类，这个类要实现`selectImports`方法，这个方法返回包含类的全类名的数组，注意范围值不会为`null`，否则会报空指针异常。或者使用`@ImportBeanDefinitionRegistrar`,参数为继承了`ImportBeanDefinitionRegistrar`接口的实现类,通过实现方法中传入的参数手工的注册组键。

```java
@Import({MyImportBeanDefinitionRegistrar.class,MyImportBeanDefinitionRegistrar.class})
```

`FactoryBean`    `spring`提供`FactoryBean`接口，继承此接口的实例被祖册成主键后会通过实例的` getObject()` ` 来获取实例 getObjectType() isSingleton()`，用于设置获取实例的类型和是否单例。

```java
 @Bean
    public MyFactoryBean myFactoryBean(){
        return new MyFactoryBean();
    }


public class MyFactoryBean implements FactoryBean<JedisCache> {
    @Override
    public JedisCache getObject() throws Exception {
        return new JedisCache();
    }

    @Override
    public Class<?> getObjectType() {
        return JedisCache.class;
    }

    @Override
    public boolean isSingleton() {
        return false;
    }
}
//getBean("MyFactoryBean")获取的是getObject()的返回值
//getBean("&MyFactoryBean")获取的是工厂本身
```

`@Bean`注册一个`bean`,可用`initMethod`指定初始化方法，`destroyMethod`指定销毁方法。或则通过实现`InitializingBean` ` DisposableBean`接口中的方法来初始化和销毁。

`JSR250`提供了两个注解也能实现初始化和销毁方法。`@PostConstruct`和`@preDestroy`。

`BeanPostProcessor`（后置处理器）接口提供了两个方法，`postProcessBeforeInitialization` 表示在任何实例化方法前调用此方法，`postProcessAfterInitialization`表示在任何实例化方法后调用此方法。

`doCreateBean`方法代码片段：

```java
Object exposedObject = bean;
		try {
			populateBean(beanName, mbd, instanceWrapper);//给bean赋初值
			exposedObject = initializeBean(beanName, exposedObject, mbd);//调用初始化方法
		}
		catch (Throwable ex) {
			if (ex instanceof BeanCreationException && beanName.equals(((BeanCreationException) ex).getBeanName())) {
				throw (BeanCreationException) ex;
			}
			else {
				throw new BeanCreationException(
						mbd.getResourceDescription(), beanName, "Initialization of bean failed", ex);
			}
		}
```

`initializeBean` 初始化一个`bean`

```java
protected Object initializeBean(final String beanName, final Object bean, @Nullable RootBeanDefinition mbd) {
		if (System.getSecurityManager() != null) {
			AccessController.doPrivileged((PrivilegedAction<Object>) () -> {
				invokeAwareMethods(beanName, bean);
				return null;
			}, getAccessControlContext());
		}
		else {
			invokeAwareMethods(beanName, bean);
		}

		Object wrappedBean = bean;
		if (mbd == null || !mbd.isSynthetic()) {
			wrappedBean = applyBeanPostProcessorsBeforeInitialization(wrappedBean, beanName);//调用后置处理器中的postProcessBeforeInitialization方法
		}

		try {
			invokeInitMethods(beanName, wrappedBean, mbd);//执行初始化方法
		}
		catch (Throwable ex) {
			throw new BeanCreationException(
					(mbd != null ? mbd.getResourceDescription() : null),
					beanName, "Invocation of init method failed", ex);
		}
		if (mbd == null || !mbd.isSynthetic()) {
			wrappedBean = applyBeanPostProcessorsAfterInitialization(wrappedBean, beanName);//调用后置处理器中的postProcessAfterInitialization方法
		}

		return wrappedBean;
	}
```

`applyBeanPostProcessorsBeforeInitialization`源码

```java
	public Object applyBeanPostProcessorsBeforeInitialization(Object existingBean, String beanName)
			throws BeansException {
		//获取bean的所有postProcessBeforeInitialization方法并执行，当执行后返回类型为null时直接返回上次调用结果。
		Object result = existingBean;
		for (BeanPostProcessor processor : getBeanPostProcessors()) {
			Object current = processor.postProcessBeforeInitialization(result, beanName);

            if (current == null) {
				return result;
			}
			result = current;
		}
		return result;
	}
```

`spring`中可以通过给`bean`继承`EnvironmentAware,ApplicationEventPublisherAware,ApplicationContextAware`等接口来获取`spring`底层的一些组件。`spring`通过`ApplicationContextAwareProcessor`类的`postProcessBeforeInitialization`来实现。源码如下：

```java

@Override
	@Nullable
	public Object postProcessBeforeInitialization(final Object bean, String beanName) throws BeansException {
		AccessControlContext acc = null;

		if (System.getSecurityManager() != null &&
				(bean instanceof EnvironmentAware || bean instanceof EmbeddedValueResolverAware ||
						bean instanceof ResourceLoaderAware || bean instanceof ApplicationEventPublisherAware ||
						bean instanceof MessageSourceAware || bean instanceof ApplicationContextAware)) {
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
```



`springIOC`大致流程：（加载配置文件后调用`refresh()`来刷新容器中的组件）

`refresh()->finishBeanFactoryInitialization(beanFactory)->beanFactory.preInstantiateSingletons()->getBean()->doGetBean()->createBean()->doCreateBean()->(createBeanInstance(),populateBean(),initializeBean())`

[`springIOC`源码详解](https://blog.csdn.net/u013510838/article/details/75126299)




