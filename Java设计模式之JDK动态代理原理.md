## 前言

### 名词解释

- 静态代理：编译期就已确定代理对象。即编码出代理类。

- 动态代理：运行时动态生成代理对象。可对被代理类做出统一的处理，如日志打印，统计调用次数等。

- JDK动态代理：即JDK中自带的动态代理生成方式。JDK动态代理的实现依赖于被代理类必须实现自接口。

- cglib动态代理：cglib工具包实现的动态代理生成方式，通过字节码来实现动态代理，不需要被代理类必须实现接口。

  

## 动态代理核心源码实现

```java
public Object getProxy() {
    //jdk 动态代理的使用方式
    return Proxy.newProxyInstance(
      this.getClass().getClassLoader(),
      target.getClass().getInterfaces(),
      this//InvocationHandler接口的自定义实现类
  );
}
```

使用JDK动态代理，首先要自定义InvocationHandler接口的实现类，书写代理类的控制逻辑。

示例：

```java
public class JDKDynamicProxyHandler implements InvocationHandler {

  private Object target;

  public JDKDynamicProxyHandler(Class clazz) {
    try {
      this.target = clazz.getDeclaredConstructor().newInstance();
    } catch (InstantiationException | IllegalAccessException | InvocationTargetException | NoSuchMethodException e) {
      e.printStackTrace();
    }
  }

  @Override
  public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {

    preAction();
    Object result = method.invoke(target, args);
    postAction();
    return result;
  }

  public Object getProxy() {
    return Proxy.newProxyInstance(
        this.getClass().getClassLoader(),
        target.getClass().getInterfaces(),
        this
    );
  }

  private void preAction() {
    System.out.println("JDKDynamicProxyHandler.preAction()");
  }

  private void postAction() {
    System.out.println("JDKDynamicProxyHandler.postAction()");
  }
}
```

具体在使用时，只需要通过以下来获取代理类

```java
Object proxy = Proxy.newProxyInstance(
    this.getClass().getClassLoader(),
    target.getClass().getInterfaces(),
    invocationHandler);
```

这段代码的核心逻辑在Proxy的newProxyInstance中。

```java
public static Object newProxyInstance(ClassLoader loader,
                                      Class<?>[] interfaces,
                                      InvocationHandler h) {
    Objects.requireNonNull(h);

    final Class<?> caller = System.getSecurityManager() == null
                                ? null
                                : Reflection.getCallerClass();

    /*
     * Look up or generate the designated proxy class and its constructor.
     */
    Constructor<?> cons = getProxyConstructor(caller, loader, interfaces);

    return newProxyInstance(caller, cons, h);
}
```