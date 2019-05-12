## 名词解释

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

基于JDK8的动态代理实现。

```java
public static Object newProxyInstance(ClassLoader loader,
                                          Class<?>[] interfaces,
                                          InvocationHandler h)
        throws IllegalArgumentException
    {
        //...
				//克隆接口的字节码
        final Class<?>[] intfs = interfaces.clone();
        //...
				//从缓存中获取或生成指定的代理类
        Class<?> cl = getProxyClass0(loader, intfs);
        try {
            //获取构造函数
            final Constructor<?> cons = cl.getConstructor(constructorParams);
            final InvocationHandler ih = h;
            //根据构造函数构造出代理类
            return cons.newInstance(new Object[]{h});
        } 
        //...
    }
    
private static Class<?> getProxyClass0(ClassLoader loader,
                                           Class<?>... interfaces) {
  			//...接口的数量不能超过65535
        if (interfaces.length > 65535) {
            throw new IllegalArgumentException("interface limit exceeded");
        }

  			// WeakCache<ClassLoader, Class<?>[], Class<?>> proxyClassCache=new WeakCache<>(new KeyFactory(), new ProxyClassFactory());
  			//如果指定的类加载器已经生成代理实现类，那么直接从缓存获取副本，否则生成新的代理实现类。
        return proxyClassCache.get(loader, interfaces);
    }

//proxyClassCache的get方法
public V get(K key, P parameter) {

  			//...key为classloader，parameter为接口的Class数组
  			//删除过时的entry
        expungeStaleEntries();
				//构造CacheKey key为null时，cacheKey为object对象，否则为虚引用对象
        Object cacheKey = CacheKey.valueOf(key, refQueue);

        //根据cacheKey加载二级缓存
        ConcurrentMap<Object, Supplier<V>> valuesMap = map.get(cacheKey);
        if (valuesMap == null) {
          //如果不存在，构造二级缓存
            ConcurrentMap<Object, Supplier<V>> oldValuesMap
                = map.putIfAbsent(cacheKey,
                                  valuesMap = new ConcurrentHashMap<>());
            if (oldValuesMap != null) {
               //如果出于并发情况，返回了缓存map，将原缓存map赋值给valuesMap
                valuesMap = oldValuesMap;
            }
        }

        //构造二级缓存key，subKey
        Object subKey = Objects.requireNonNull(subKeyFactory.apply(key, parameter));
  			//获取生成代理类的代理类工厂
        Supplier<V> supplier = valuesMap.get(subKey);
        Factory factory = null;

        while (true) {
          //循环获取生成代理类的代理类工厂
            if (supplier != null) {
                // 如果代理类工厂不为空，通过get方法获取代理类。该supplier为WeakCache的内部类Factory
                V value = supplier.get();
                if (value != null) {
                    return value;
                }
            }
            
            if (factory == null) {
              //代理工厂类为null，创建代理工厂类
                factory = new Factory(key, parameter, subKey, valuesMap);
            }

            if (supplier == null) {
                supplier = valuesMap.putIfAbsent(subKey, factory);
                if (supplier == null) {
                    // successfully installed Factory
                    supplier = factory;
                }
                // else retry with winning supplier
            } else {
                if (valuesMap.replace(subKey, supplier, factory)) {
                    // successfully replaced
                    // cleared CacheEntry / unsuccessful Factory
                    // with our Factory
                    supplier = factory;
                } else {
                    // retry with current supplier
                    supplier = valuesMap.get(subKey);
                }
            }
        }
    }

//Factory的get方法
public synchronized V get() { // serialize access
            // re-check
            Supplier<V> supplier = valuesMap.get(subKey);
            if (supplier != this) {
              //如果在并发等待的时候有变化，返回null，继续执行外层的循环。
                return null;
            }
            //创建新的代理类
            V value = null;
            try {
              //通过ProxyClassFactory的apply方法生成代理类
                value = Objects.requireNonNull(valueFactory.apply(key, parameter));
            } finally {
                if (value == null) { // remove us on failure
                    valuesMap.remove(subKey, this);
                }
            }
            //用CacheValue包装value值(代理类)
            CacheValue<V> cacheValue = new CacheValue<>(value);

            //将cacheValue放入reverseMap
            reverseMap.put(cacheValue, Boolean.TRUE);
            return value;
        }

//ProxyClassFactory类的apply方法
public Class<?> apply(ClassLoader loader, Class<?>[] interfaces) {

            Map<Class<?>, Boolean> interfaceSet = new IdentityHashMap<>(interfaces.length);
            //校验class是否正确，校验class是否是interface，校验class是否重复
  					//...
						//代理类的包名
            String proxyPkg = null;     // package to define proxy class in
  					//代理类的访问修饰符
            int accessFlags = Modifier.PUBLIC | Modifier.FINAL;
						//记录非public修饰的被代理类接口，用来作为代理类的包名，同时校验所有非public修饰的被代理类接口必须处于同一包名下
            for (Class<?> intf : interfaces) {
                int flags = intf.getModifiers();
                if (!Modifier.isPublic(flags)) {
                    accessFlags = Modifier.FINAL;
                    String name = intf.getName();
                    int n = name.lastIndexOf('.');
                    String pkg = ((n == -1) ? "" : name.substring(0, n + 1));
                    if (proxyPkg == null) {
                        proxyPkg = pkg;
                    } else if (!pkg.equals(proxyPkg)) {
                        throw new IllegalArgumentException(
                            "non-public interfaces from different packages");
                    }
                }
            }

            if (proxyPkg == null) {
                // 如果没有非public的接口类，包名使用com.sun.proxy package
                proxyPkg = ReflectUtil.PROXY_PACKAGE + ".";
            }

            /*
             * Choose a name for the proxy class to generate.
             */
            long num = nextUniqueNumber.getAndIncrement();
  					//构造代理类名称，使用包名+代理类前缀+自增值作为代理类名称
            String proxyName = proxyPkg + proxyClassNamePrefix + num;

            //生成代理类的字节码文件
            byte[] proxyClassFile = ProxyGenerator.generateProxyClass(
                proxyName, interfaces, accessFlags);
            try {
              	//通过native的方法生成代理类
                return defineClass0(loader, proxyName,
                                    proxyClassFile, 0, proxyClassFile.length);
            } 
 					//...
        }
```

## 总结

