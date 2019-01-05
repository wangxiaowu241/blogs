上一篇简单介绍了springcloud声明式服务调用Feign的使用，接下来分析下Feign的源码，具体实现及为什么如此实现。

## 启动时Feign的处理

启动类上使用了@EnableFeignClients注解，我们来看下这个注解在哪里使用了，使用idea只要在EnableFeignClients类上按住command同时点击类名就可以查看到这个类在哪里使用了，发现除了启动类，只在FeignClientsRegistrar类中引用了EnableFeignClients。

debug可以发现，当应用启动时会首先调用FeignClientsRegistrar的registerBeanDefinitions()方法。

```java
@Override
public void registerBeanDefinitions(AnnotationMetadata metadata,
      BeanDefinitionRegistry registry) {
    //注册默认配置信息
   registerDefaultConfiguration(metadata, registry);
    //注册每个声明为Feign Client的类
   registerFeignClients(metadata, registry);
}
```

主要看下registerFeignClients()方法。

```java
public void registerFeignClients(AnnotationMetadata metadata,
      BeanDefinitionRegistry registry) {
    //获取扫描classpath下component组件的扫描器
   ClassPathScanningCandidateComponentProvider scanner = getScanner();
   scanner.setResourceLoader(this.resourceLoader);

   Set<String> basePackages;
	//获取启动类上配置的@EnableFeignClients注解的属性
   Map<String, Object> attrs = metadata
         .getAnnotationAttributes(EnableFeignClients.class.getName());
   AnnotationTypeFilter annotationTypeFilter = new AnnotationTypeFilter(
         FeignClient.class);
    //从刚才获取的@EnableFeignClients注解的属性中获取clients属性配置的值
   final Class<?>[] clients = attrs == null ? null
         : (Class<?>[]) attrs.get("clients");
   if (clients == null || clients.length == 0) {
       //如果clients没配置
       //扫描器增加要扫描的过滤器(扫描被@FeignClient注解修饰的类)
      scanner.addIncludeFilter(annotationTypeFilter);
       //获取配置的扫描包的路径，如果没配置，默认为启动类的包路径
      basePackages = getBasePackages(metadata);
   }
   else {
      final Set<String> clientClasses = new HashSet<>();
      basePackages = new HashSet<>();
      for (Class<?> clazz : clients) {
          //如果启动类配置了clients属性的值，将配置的client所在的包名加到扫描器扫描的包中
         basePackages.add(ClassUtils.getPackageName(clazz));
         clientClasses.add(clazz.getCanonicalName());
      }
      AbstractClassTestingTypeFilter filter = new AbstractClassTestingTypeFilter() {
         @Override
         protected boolean match(ClassMetadata metadata) {
            String cleaned = metadata.getClassName().replaceAll("\\$", ".");
            return clientClasses.contains(cleaned);
         }
      };
      scanner.addIncludeFilter(
            new AllTypeFilter(Arrays.asList(filter, annotationTypeFilter)));
   }

    //遍历包名，扫描@FeignClient注解修饰的类（怎么扫描到？前面加了扫描@FeignClient注解的IncludeFilter)
   for (String basePackage : basePackages) {
      Set<BeanDefinition> candidateComponents = scanner
            .findCandidateComponents(basePackage);
       //遍历扫描出来的@FeignClient注解修饰的类
      for (BeanDefinition candidateComponent : candidateComponents) {
         if (candidateComponent instanceof AnnotatedBeanDefinition) {
            // verify annotated class is an interface
             //校验@FeignClient注解修饰的类是否是interface
            AnnotatedBeanDefinition beanDefinition = (AnnotatedBeanDefinition) candidateComponent;
            AnnotationMetadata annotationMetadata = beanDefinition.getMetadata();
             //断言，@FeignClient注解修饰的类必须是interface
            Assert.isTrue(annotationMetadata.isInterface(),
                  "@FeignClient can only be specified on an interface");
             
             //先获取@FeignClient注解的属性值
            Map<String, Object> attributes = annotationMetadata
                  .getAnnotationAttributes(
                        FeignClient.class.getCanonicalName());
			//获得@FeignClient配置的client 的名称(name或value或serviceId)
            String name = getClientName(attributes);
            //注册feign client的配置信息
             registerClientConfiguration(registry, name,
                  attributes.get("configuration"));
			//注册feign client
            registerFeignClient(registry, annotationMetadata, attributes);
         }
      }
   }
}
```

```java
//将feign client交由spring管理，声明为spring的bean
private void registerFeignClient(BeanDefinitionRegistry registry,
      AnnotationMetadata annotationMetadata, Map<String, Object> attributes) {
   String className = annotationMetadata.getClassName();
    //创建FeignClientFactoryBean，包含将feign client的注解属性信息存入FeignClientFactoryBean中
   BeanDefinitionBuilder definition = BeanDefinitionBuilder
         .genericBeanDefinition(FeignClientFactoryBean.class);
    //校验feign client的配置，配置的fallback及fallbackFatory必须是实现类
   validate(attributes);
    //将@FeignClient注解配置的属性放入FeignClientFactoryBean的BeanDefinitionBuilder中
   definition.addPropertyValue("url", getUrl(attributes));
   definition.addPropertyValue("path", getPath(attributes));
   String name = getName(attributes);
   definition.addPropertyValue("name", name);
   definition.addPropertyValue("type", className);
   definition.addPropertyValue("decode404", attributes.get("decode404"));
   definition.addPropertyValue("fallback", attributes.get("fallback"));
   definition.addPropertyValue("fallbackFactory", attributes.get("fallbackFactory"));
   definition.setAutowireMode(AbstractBeanDefinition.AUTOWIRE_BY_TYPE);

   String alias = name + "FeignClient";
   AbstractBeanDefinition beanDefinition = definition.getBeanDefinition();

   boolean primary = (Boolean)attributes.get("primary"); // has a default, won't be null

   beanDefinition.setPrimary(primary);

   String qualifier = getQualifier(attributes);
   if (StringUtils.hasText(qualifier)) {
      alias = qualifier;
   }

   BeanDefinitionHolder holder = new BeanDefinitionHolder(beanDefinition, className,
         new String[] { alias });
    //注册bean到spring容器中
   BeanDefinitionReaderUtils.registerBeanDefinition(holder, registry);
}
```

在spring容器启动时会调用FeignClientFactoryBean的getObject()方法（只有在其他bean注入feign client时才会调用），看下FeignClientFactoryBean的getObject()方法做了哪些处理。

```java
public Object getObject() throws Exception {
    //直接调用了getTarget()方法
   return getTarget();
}

/**
 * @param <T> the target type of the Feign client
 * @return a {@link Feign} client created with the specified data and the context information
 */
<T> T getTarget() {
   //这个FeignContext在FeignAutoConfiguration配置中已经声明了，所以可以直接用applicationContext获取bean
   FeignContext context = applicationContext.getBean(FeignContext.class);
    //配置feign 的decoder、encoder、retryer、contract、RequestInterceptor等
    //这些有默认配置，在FeignAutoConfiguration及FeignClientsConfiguration中有默认配置
   Feign.Builder builder = feign(context);

   if (!StringUtils.hasText(this.url)) {
      //如果@FeignClient注解上指定了url，其实除非本地调试，一般不建议指定URL
      String url;
      if (!this.name.startsWith("http")) {
         url = "http://" + this.name;
      }
      else {
         url = this.name;
      }
       //处理URL，没配置URL时，这里的URL形式为http://name+/path
      url += cleanPath();
       //使用负载均衡处理feign 请求
      return (T) loadBalance(builder, context, new HardCodedTarget<>(this.type,
            this.name, url));
   }
    //配置了FeignClient的具体URL
   if (StringUtils.hasText(this.url) && !this.url.startsWith("http")) {
      this.url = "http://" + this.url;
   }
   String url = this.url + cleanPath();
   Client client = getOptional(context, Client.class);
   if (client != null) {
      if (client instanceof LoadBalancerFeignClient) {
         // not load balancing because we have a url,
         // but ribbon is on the classpath, so unwrap
         client = ((LoadBalancerFeignClient)client).getDelegate();
      }
      builder.client(client);
   }
   Targeter targeter = get(context, Targeter.class);
   return (T) targeter.target(this, builder, context, new HardCodedTarget<>(
         this.type, this.name, url));
}
```

- decoder：将http请求的response转换成对象
- encoder：将http请求的对象转换成http request body
- contract：校验Feign Client上的注解及value值是否合法
- retryer：定义http请求如果失败了是否应该重试以及重试间隔、方式等等
- RequestInterceptor：feign发起请求前的拦截器，可以全局定义basic auth、发起请求前自动添加header等等

从@FeignClient注解上是否指定URL，feign的处理分成了两部分，如果未指定URL，则使用负载均衡去发送请求，指定URL，只会向指定的URL发送请求。

一般是不指定URL的，接下来先看下，不指定具体URL时，feign的处理。

```java
protected <T> T loadBalance(Feign.Builder builder, FeignContext context,
      HardCodedTarget<T> target) {
    //默认client为LoadBalancerFeignClient，为啥？参见DefaultFeignLoadBalancedConfiguration
   Client client = getOptional(context, Client.class);
   if (client != null) {
      builder.client(client);
       //这个Targeter默认为DefaultTargeter，参见FeignAutoConfiguration
      Targeter targeter = get(context, Targeter.class);
      return targeter.target(this, builder, context, target);
   }

   throw new IllegalStateException(
         "No Feign Client for loadBalancing defined. Did you forget to include spring-cloud-starter-netflix-ribbon?");
}
```

Targeter默认为DefaultTargeter，client为LoadBalancerFeignClient。再看下DefaultTargeter.target()方法

```java
public <T> T target(FeignClientFactoryBean factory, Feign.Builder feign, FeignContext context,
               Target.HardCodedTarget<T> target) {
   return feign.target(target);
}
```

Feign.target()方法。

```java
public <T> T target(Target<T> target) {
  return build().newInstance(target);
}
```

ReflectiveFeign.newInstance()方法。这里为什么是ReflectiveFeign？参考Feign.build()方法

```java
public <T> T newInstance(Target<T> target) {
    //这个apply方法就是ReflectiveFeign中的apply方法，返回了每个方法的调用包装类SynchronousMethodHandler
  Map<String, MethodHandler> nameToHandler = targetToHandlersByName.apply(target);
  Map<Method, MethodHandler> methodToHandler = new LinkedHashMap<Method, MethodHandler>();
  List<DefaultMethodHandler> defaultMethodHandlers = new LinkedList<DefaultMethodHandler>();
//这个target.type()返回的就是声明@FeignClient注解所在的class
  for (Method method : target.type().getMethods()) {
    if (method.getDeclaringClass() == Object.class) {
      continue;
    } else if(Util.isDefault(method)) {
      DefaultMethodHandler handler = new DefaultMethodHandler(method);
      defaultMethodHandlers.add(handler);
      methodToHandler.put(method, handler);
    } else {
      methodToHandler.put(method, nameToHandler.get(Feign.configKey(target.type(), method)));
    }
  }
  //返回了ReflectiveFeign.FeignInvocationHandler对象，这个对象的invoke方法其实就是调用了SynchronousMethodHandler.invoke方法
  InvocationHandler handler = factory.create(target, methodToHandler);
  T proxy = (T) Proxy.newProxyInstance(target.type().getClassLoader(), new Class<?>[]{target.type()}, handler);

  for(DefaultMethodHandler defaultMethodHandler : defaultMethodHandlers) {
    defaultMethodHandler.bindTo(proxy);
  }
  return proxy;
}
```



```java
public Map<String, MethodHandler> apply(Target key) {
    //获取类上的方法的元数据，如返回值类型，参数类型，注解数据等等
  List<MethodMetadata> metadata = contract.parseAndValidatateMetadata(key.type());
  Map<String, MethodHandler> result = new LinkedHashMap<String, MethodHandler>();
  for (MethodMetadata md : metadata) {
    BuildTemplateByResolvingArgs buildTemplate;
    if (!md.formParams().isEmpty() && md.template().bodyTemplate() == null) {
      buildTemplate = new BuildFormEncodedTemplateFromArgs(md, encoder, queryMapEncoder);
    } else if (md.bodyIndex() != null) {
      buildTemplate = new BuildEncodedTemplateFromArgs(md, encoder, queryMapEncoder);
    } else {
      buildTemplate = new BuildTemplateByResolvingArgs(md, queryMapEncoder);
    }
      //这个factory是SynchronousMethodHandler.Factory，create方法返回了一个SynchronousMethodHandler对象
    result.put(md.configKey(),
               factory.create(key, md, buildTemplate, options, decoder, errorDecoder));
  }
  return result;
}
```



简单总结下启动时Feign所做的处理：

- 获取@EnableFeignClients注解配置的扫描包路径，如果没配置，默认为启动类的包路径。
- 获得扫描包路径下@FeignClient修饰的类
- 校验@FeignClient修饰的类，包括类必须是interface，以及@FeignClient的fallback及fallbackFactory配置的必须是接口的实现类等
- 将@FeignClient修饰的类交由spring管理，声明为bean，其他bean注入FeignClient时注入的其实是当前FeignClient的代理类，这个代理类包装在Targeter内部，Targeter被注入到引用的bean中。

这样做的好处是：在程序中使用Feign Client时就可以像其他spring 管理的bean一样直接注入即可。

例如：

```java
@Autowired
private CartFeignClient cartFeignClient;

@PostMapping("/toCart/{productId}")
public ResponseEntity addCart(@PathVariable("productId") Long productId) throws InterruptedException {
    cartFeignClient.addCart(productId);
    return ResponseEntity.ok(productId);
}
```

## 调用Feign Client时的feign的处理

刚分析了应用启动及bean注入FeignClient时feign的处理，知道注入的其实是Targeter类，Targetr类包装了FeignCLient的proxy，proxy内部绑定了methodHandler为SynchronousMethodHandler。接下来仔细分析下整个实际调用过程的处理。

前面提到feign实际处理方法调用的methodHandler是SynchronousMethodHandler。

实际上，首先调用的是ReflectiveFeign的静态内部类FeignInvocationHandler，这个类实现了JDK的InvocationHandler接口，在调用代理类的方法时会被调用FeignInvocationHandler的invoke方法。

FeignInvocationHandler的invoke方法。

```java
public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
  if ("equals".equals(method.getName())) {
    try {
      Object
          otherHandler =
          args.length > 0 && args[0] != null ? Proxy.getInvocationHandler(args[0]) : null;
      return equals(otherHandler);
    } catch (IllegalArgumentException e) {
      return false;
    }
  } else if ("hashCode".equals(method.getName())) {
    return hashCode();
  } else if ("toString".equals(method.getName())) {
    return toString();
  }
  //除了equals、hashCode、toString方法外，其他方法都走dispatch.get(method).invoke(args)方法。
  //点击这个方法的实现类，就可以追到  SynchronousMethodHandler的invoke方法了。
  return dispatch.get(method).invoke(args);
}
```

可以看到除了equals、hashCode、toString方法外，其他方法都走dispatch.get(method).invoke(args)方法。
点击这个方法的实现类，就可以追到  SynchronousMethodHandler的invoke方法了。所以这里其实只是简单起到转发的作用。

SynchronousMethodHandler的invoke方法。

```java
public Object invoke(Object[] argv) throws Throwable {
  //根据调用参数创建一个RequestTemplate，用来具体处理http调用请求
  RequestTemplate template = buildTemplateFromArgs.create(argv);
  //克隆出一个一模一样的Retryer，用来处理调用失败后的重试
  Retryer retryer = this.retryer.clone();
  while (true) {
    try {
      //发送http request以及处理response等  
      return executeAndDecode(template);
    } catch (RetryableException e) {
      //处理重试次数、重试间隔等等  
      retryer.continueOrPropagate(e);
      continue;
    }
  }
}
```

先来看下如何创建的RequestTemplate。

ReflectiveFeign的内部静态类BuildTemplateByResolvingArgs的create方法。

```java
public RequestTemplate create(Object[] argv) {
  //获取methodMetada的template，这个RequestTemplate是可变的，跟随每次调用参数而变。
  RequestTemplate mutable = new RequestTemplate(metadata.template());
  if (metadata.urlIndex() != null) {
    //处理@PathVariable在URL上插入的参数  
    int urlIndex = metadata.urlIndex();
    checkArgument(argv[urlIndex] != null, "URI parameter %s was null", urlIndex);
    mutable.insert(0, String.valueOf(argv[urlIndex]));
  }
  //处理调用方法的param参数，追加到URL ？后面的参数
  Map<String, Object> varBuilder = new LinkedHashMap<String, Object>();
  for (Entry<Integer, Collection<String>> entry : metadata.indexToName().entrySet()) {
    int i = entry.getKey();
    Object value = argv[entry.getKey()];
    if (value != null) { // Null values are skipped.
      if (indexToExpander.containsKey(i)) {
        value = expandElements(indexToExpander.get(i), value);
      }
      for (String name : entry.getValue()) {
        varBuilder.put(name, value);
      }
    }
  }
  //处理query参数以及body内容	
  RequestTemplate template = resolve(argv, mutable, varBuilder);
  if (metadata.queryMapIndex() != null) {
    // add query map parameters after initial resolve so that they take
    // precedence over any predefined values
    //当  RequestTemplate处理完参数后，再处理@QueryMap注入的参数，以便优先于任意值。
    Object value = argv[metadata.queryMapIndex()];
    Map<String, Object> queryMap = toQueryMap(value);
    template = addQueryMapQueryParameters(queryMap, template);
  }

  if (metadata.headerMapIndex() != null) {
    //处理RequestTemplate的header内容  
    template = addHeaderMapHeaders((Map<String, Object>) argv[metadata.headerMapIndex()], template);
  }

  return template;
}
```

可以看到，第一步是根据调用时的参数等构造了RequestTemplate的param、body、header等内容。

再看executeAndDecode方法。

SynchronousMethodHandler的executeAndDecode方法。

```java
Object executeAndDecode(RequestTemplate template) throws Throwable {
  //构造Request，将RequestTemplate中的参数等放入Request中
  Request request = targetRequest(template);
  Response response;
  try {
    //这个client默认实现是Client接口中的Defalut，实现是通过HttpURLConnection发送请求
    //另一种是LoadBalancerFeignClient，默认也是Client接口中的Defalut，可以通过配置指定为Apache的HTTPClient，也可以指定为OKhttp来发送请求，在每个具体实现中来通过ribbon实现负载均衡，负载到集群中不同的机器，这里不再发散  
    response = client.execute(request, options);
    // ensure the request is set. TODO: remove in Feign 10
    response.toBuilder().request(request).build();
  } catch (IOException e) {
    throw errorExecuting(request, e);
  }
  boolean shouldClose = true;
  try {
    //处理response的返回值
    if (Response.class == metadata.returnType()) {
      if (response.body() == null) {
        return response;
      }
      // Ensure the response body is disconnected
      byte[] bodyData = Util.toByteArray(response.body().asInputStream());
      return response.toBuilder().body(bodyData).build();
    }
    //根据状态码处理下response
    if (response.status() >= 200 && response.status() < 300) {
      if (void.class == metadata.returnType()) {
        return null;
      } else {
        Object result = decode(response);
        shouldClose = closeAfterDecode;
        return result;
      }
    } else if (decode404 && response.status() == 404 && void.class != metadata.returnType()) {
      Object result = decode(response);
      shouldClose = closeAfterDecode;
      return result;
    } else {
      throw errorDecoder.decode(metadata.configKey(), response);
    }
  } 
}
```

总结一下：

- 代理类先调用到FeignInvocationHandler的invoke方法，而这个invoke方法相当于直接调用了SynchronousMethodHandler的invoke方法。
- SynchronousMethodHandler的invoke方法主要是构造了RequestTemplate以及出现异常重试的Retryer，最后根据构造的RequestTemplate发起了http请求以及decode。
- 构造RequestTemplate时，根据传入的参数动态构建URL中的参数（@PathVarible）以及URL ？追加的参数，还有body等等，最后再处理@QueryMap注入的参数，以保证优先级最高。
- 发起http请求时，没有负载均衡时，默认是通过JDK的HttpURLConnection发送请求，另一种就是LoadBalancerFeignClient各种实现类，如Apache的HTTPClient，以及OKhttp等，这些实现也是通过ribbon动态指定服务器IP地址，以达到负载均衡的作用。
- 最后将response处理成需要的返回值类型，以及根据状态码进行decode。