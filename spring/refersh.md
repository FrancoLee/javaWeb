## refersh

---

`refersh()`容器刷新方法。会按顺序调用一下几个方法来完成容器的刷新操作。

一、[`prepareRefresh()`](#1)

二、`prepareBeanFactory(beanFactory)`

三、`postProcessBeanFactory(beanFactory)`

四、`invokeBeanFactoryPostProcessors(beanFactory)`

五、`registerBeanPostProcessors(beanFactory)`

六、`initMessageSource()`

七、`initMessageSource()`

八、`onRefresh()`

九、`registerListeners()`

十、`finishBeanFactoryInitialization(beanFactory)`

十一、`finishRefresh()`























<span id="1">`prepareRefresh()`</span>



























































