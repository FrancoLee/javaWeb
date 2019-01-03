## Servlet3.0

---

`ServletContainerInitializer`接口

 	当`servlet`容器启动的时候会自动扫描所有`jar`包、`src`以及`web`根目录下的`META-INF/services/javax.servlet.ServletContainerInitializer`文件。文件的内容是一个`ServletContainerInitializer`接口的实现类的全限定名。`servlet`容器会实例化这个实现类并且调用`onStartup()`方法。这个方法的第一个参数是一个类的集合（由`@HandlesTypes`注解来指定那些类的子类的集合会被当成参数传入），第二参数是`servletContext`。

​	样例：

```java
//表示第一个参数是继承MyImp接口的所有类的集合
@HandlesTypes(value = {MyImp.class})
public class MyServletContainerInitializer implements ServletContainerInitializer {
    @Override
    public void onStartup(Set<Class<?>> set, ServletContext servletContext) throws ServletException {
        for (Class<?> claz : set) {
            System.out.println(claz.getName());
        }
    }
}

```

​	我们可以在这个方法里手动的给`servletContext`注册一些自己需要的东西，比如`HttpServlet、Filter、Listener`。