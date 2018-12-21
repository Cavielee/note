### ApplicationContextAware

该接口可用于获取 Spring 上下文



通过实现接口方法 `setApplicationContext(ApplicationContext ctx)` ，Spring 会自动为其注入 Spring 上下文（ApplicationContext）

```java
private ApplicationContext ctx;

@Override
public void setApplicationContext(ApplicationContext ctx) {
    this.ctx = ctx;
}
```



扩展：可用于获取 Spring 中的某些Bean

例如 `ctx.getBeansWithAnnotation(xxx.class)` 该方法可以获取 Spring 工厂中被 xxx 注解修饰的 Bean。