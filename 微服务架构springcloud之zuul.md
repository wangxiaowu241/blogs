注意：本文的前提是基于zuul的1.3.X版本来解析的，2.0版本采用了netty作为底层框架重新设计了整个zuul的架构，将在后面进行分析。

#zuul是什么

zuul是Netflix设计用来为所有面向设备、web网站提供服务的所有应用的门面，zuul可以提供动态路由、监控、弹性扩展、安全认证等服务，他还可以根据需求将请求路由到多个应用中。

# zuul是用来解决什么问题的

在使用网关之前，动态的路由是通过Nginx的配置来做的，但是一旦发生改变，比如IP地址发生改变，加入其它路由，就要重新配置Nginx，重启Nginx。安全认证是放在每一个应用中，应用中包含了非业务强相关的内容，看起来也是不够优雅。

在目前的应用中，zuul主要用来做如下几件事情：

- 动态路由：APP、web网站通过zuul来访问不同的服务提供方，且与ribbon结合，还可以负载均衡的路由到同一个应用不同的实例中。

- 安全认证：zuul作为互联网服务架构中的网关，可以用来校验非法访问、授予token、校验token等。

- 限流：zuul通过记录每种请求的类型来达到限制访问过多导致服务down掉的目的。

- 静态响应处理：直接在zuul就处理一些请求，返回响应内容，不转发到微服务内部。

- 区域弹性：主要是针对AWS上的应用做一些弹性扩展。

# zuul的最佳实践是怎样的

##Netflix是如何使用zuul的？

![](https://camo.githubusercontent.com/5e596c573110bffb608614a09c97611107205d0d/687474703a2f2f6e6574666c69782e6769746875622e696f2f7a75756c2f696d616765732f7a75756c2d706879736963616c2d617263682e706e67)

可以看到，在Netflix的架构中，亚马逊的弹性负载均衡作为第一层，zuul作为第二层，为所有应用的网关，请求经过AWS的负载均衡先发送到zuul，zuul再将流量转发到各个API、website等等，然后各个API、website应用再调用各个微服务。

当然在Netflix的使用中，zuul也结合了Netflix的其他微服务组件一起使用。

- Hystrix：用来服务降级及熔断

- Ribbon：用来作为软件的负载均衡，微服务之间相互调用是通过ribbon作为软件负载均衡使用负载到微服务集群内的不同的实例
- Turbin：监控服务的运行状况
- Feign：用作微服务之间发送rest请求的组件，可以将rest调用类似spring的其他bean一样直接注入使用功能
- Eureka：服务注册中心

## 入门示例

### 引入zuul

```groovy
dependencies {
    //spring-cloud-starter-netflix-zuul模块
    implementation('org.springframework.cloud:spring-cloud-starter-netflix-zuul')
    //test模块
    testImplementation('org.springframework.boot:spring-boot-starter-test')
    //引入eureka-client模块
    implementation('org.springframework.cloud:spring-cloud-starter-netflix-eureka-client')
}
```

这里引入eureka-client是为了将zuul也注册到eureka中，作为微服务中的一个应用，那么zuul就能将eureka中注册的所有微服务应用的注册信息都拿到，从而做到动态路由。

启动类

```java
@EnableDiscoveryClient
@EnableZuulProxy
@SpringBootApplication
public class ZuulApplication {

    public static void main(String[] args) {
        SpringApplication.run(ZuulApplication.class, args);
    }
}
```

### zuul的路由配置

zuul的配置类为ZuulProperties，配置前缀为zuul。

下面为路由的一些示例配置（未结合eureka）：

```yaml
zuul:
  routes:
    product: http://localhost:8082/product/**
    cart: http://localhost:8081/cart/**
```

zuul服务引用了eureka-client后配置可以改成如下：

```yaml
zuul:
  routes:
    product: /product/**
    cart: /cart/**
```

或者更细致的配置：

```java
zuul:
  routes:
    cart:
      path: /cart/**
      serviceId: cart
```

原访问地址：http://localhost:8081/cart/1

经过zuul后的访问地址：http://localhost:8080/cart/cart/1

访问效果：

![](/Users/wangwangxiaoteng/work/code/github/blogs/zuul-访问效果1.png)



注意：zuul只有结合了eureka，才会有ribbon作为软件负载均衡，直接配置逻辑URL，不会起到负载均衡的效果，也不会有hystrix作为熔断器使用。

如下：

```yaml
zuul:
  routes:
  users:
  path: /product/**
  url: http://localhost:8082/product
```

如果想指定多个服务的列表且需要通过ribbon实现负载均衡，配置可参考如下：

```yaml
zuul:
  routes:
    users:
      path: /product/**
      serviceId: product

ribbon:
  eureka:
    enabled: false

product:
  ribbon:
    listOfServers: example.com,google.com
```

当整合eureka时，配置简单，服务太多，不想一个个配置的时候，可以通过定义PatternServiceRouteMapper来全局定义匹配规则。

个人推荐这种方式，简单。

示例如下：

```java
@Configuration
public class ZuuPatternServiceRouteMapperConfiguration {

    //注意！！！这两者保留一个即可。
    /**
     * 没有版本号的路由匹配规则bean
     * @retu没rn 路由匹配规则bean
     */
    @Bean
    public PatternServiceRouteMapper patternServiceRouteMapper(){

        return new PatternServiceRouteMapper("(?<version>v.*$)","${name}");
    }

    /**
     * 有版本号的路由匹配规则bean
     * @return 路由匹配规则bean
     */
    @Bean
    public PatternServiceRouteMapper patternServiceRouteMapperWithVersion(){
        return new PatternServiceRouteMapper("(?<name>.*)-(?<version>v.*$)","${version}/${name}");
    }

}
```

###全局添加映射前缀

通过zuul.prefix可以指定全局映射前缀，可以避免服务URL直接暴露出去，如前缀为/api（注意，这个”/“要加上，否则可能会出现404），默认情况下，代理前缀会在请求转发前从请求中删除前缀，可以通过zuul.stripPrefix来配置，默认是true。

```yaml
zuul:
  prefix: /api
  strip-prefix: true
```

这样原来直接访问比如说cart服务的地址为http://localhost:8081/cart/1，通过zuul网关，且添加了全局映射前缀/api后的访问路径变为http://localhost:8080/api/cart/cart/1。

如果在局部路由想关闭这个删除前缀映射，可以通过以下配置指定：

```yaml
zuul:
   routes:
     cart:
       path: /cart/**
       stripPrefix: false
```

### 不走路由的服务或者映射配置

ZuulProperties配置中，有两个属性ignoredServices，ignoredPatterns，属性类型均为LinkedHashSet。

ignoredServices：指定忽略某些服务的路由

ignoredPatterns：配置不走路由的某些路由匹配规则

### zuul的filter

1.X版本的zuul实现是依赖于servlet的，所以对过滤器支持较好。zuul本身提供了较多的filter，如SendForwardFilter、DebugFilter、SendResponseFilter、SendErrorFilter等。

zuul本身也提供了抽象类ZuulFilter，供自定义filter。

####自定义ZuulFilter

自定义ZuulFilter，需要实现几个方法，下面为前置过滤器的简单示例。

```java
public class AuthenticationFilter extends ZuulFilter {
    @Override
    public String filterType() {
        //fiterType,有pre、route、post、error四种类型，分别代表路由前、路由时
        //路由后以及异常时的过滤器
        return FilterConstants.PRE_TYPE;
    }

    @Override
    public int filterOrder() {
        //排序，指定相同filterType时的执行顺序，越小越优先执行，这里指定顺序为路由过滤器的顺序-1，即在路由前执行
        return return FilterConstants.PRE_DECORATION_FILTER_ORDER - 1;
    }

    @Override
    public boolean shouldFilter() {
        //是否应该调用run()方法
        return false;
    }

    @Override
    public Object run() throws ZuulException {
        HttpServletRequest request = RequestContext.getCurrentContext().getRequest();
        String token = request.getHeader("X-Authentication");
        if (StringUtils.isBlank(token)) {
            throw new TokenNotValidException("token不存在！");
        }
        //校验token正确性
        return null;
    }
}
```

多个过滤器之间可能想要传输内容，那么可以通过zuul提供的RequestContext来完成，RequestContext本身也是依赖于ThreadLocal来完成的。RequestContext本身也携带了HttpServletRequest和HttpServletResponse，可以获取请求或响应中的内容。

####zuul的过滤器也可以配置为不启用

通过zuul.<SimpleClassName>.<filterType>.disable=false来关闭指定过滤器。

例如：

`zuul.SendResponseFilter.post.disable=true`

### @EnableZuulServer与@EnableZuulProxy的区别

在zuul的@EnableZuulProxy注解的源代码中，是这么说的，@EnableZuulProxy提供了一些基本的反向代理过滤器，@EnableZuulServer只是将zuul指定为一个zuul的server，并不提供任何反向代理的过滤器。一般我们推荐使用@EnableZuulProxy，如果不想用zuul自带的过滤器，可以通过上面的方式关闭指定的过滤器。

@EnableZuulProxy自带的filter如下：

1. pre类型：

   - ServletDetectionFilter

   - FormBodyWrapperFilter：解析请求form表单并在转发到下游请求前重新编码

   - DebugFilter：会打印执行每个过滤器的执行日志，但仅仅是打印”Debug.addRoutingDebug("Filter " + filter.filterType() + " " + filter.filterOrder() + " " + filterName);“

   - SendForwardFilter：将请求转发（路由）到下游服务

2. post类型：

   - SendResponseFilter:将下游服务对请求的响应写入当前response中。

3. error类型：

   - SendErrorFilter：处理异常情况的过滤器

### zuul的http客户端

zuul实现路由，是通过http方式转发到其他服务的，那么就需要http客户端。默认情况下是通过Apache HTTP Client来发送http请求的，如果想使用restClient或者OKhttp可以通过配置指定。

```yaml
ribbon:
  okhttp:
    enabled: true
```

或者

```yaml
ribbon:
  restclient:
    enabled: true
```

当然可以自定义Apache HTTP Client和OkHttpClient并声明为Bean。

### 敏感header内容

zuul支持将一些敏感的header内容不转发到下游的其他服务中，通过zuul.sensitiveHeaders将这些header内容对下游服务屏蔽。默认情况下，header中的Cookie、Set-Cookie、Authorization将不被传递到下游服务器中，可以通过zuul.sensitiveHeaders指定全局的敏感header内容。所以在下游服务器想要获取到cookie内容时，这里需要重新配置下。个人建议，cookie、Authorization内容应该在zuul内将其转换成下游服务需要的userId、user等，再传递到下游服务器。

### 超时配置

zuul支持配置超时时间，如果使用的是eureka作为服务注册中心，那么只要指定ribbon的超时时间即可。即ribbon.ReadTimeout和ribbon.SocketTimeout。如果使用的是具体的URL路由，那么通过zuul.host.connect-timeout-millis和zuul.host.socket-timeout-millis指定。zuulProperties内部类Host中还有一些诸如maxTotalConnections最大连接数等的一些配置。

### CORS配置

默认情况下，zuul是允许所有cors请求的，如果要自定义cors，可以通过自定义WebMvcConfigurer来完成。

如：

```java
@Configuration
public class WebMvcConfig implements WebMvcConfigurer {

    @Override
    public void addCorsMappings(CorsRegistry registry) {
        registry.addMapping("/*")
                .allowedOrigins("http://baidu.com");
    }
}
```

### 其他配置

#### 重试

zuul.retryable可以配置zuul是否使用ribbon的重试机制。

#### 代理请求header

zuul.addProxyHeaders可以配置是否将X-Forwarded-Host放入转发的请求中

# zuul的实现原理及设计是怎样的

了解了zuul的使用方式，我们要开始了解下zuul的源码以及zuul的整个架构设计。

先从启动类ZuulApplication入手。

```java
@EnableDiscoveryClient
@EnableZuulProxy
@SpringBootApplication
public class ZuulApplication {

    public static void main(String[] args) {
        SpringApplication.run(ZuulApplication.class, args);
    }
}
```

可以看到启动类上加了注解@EnableZuulProxy，我们看下这个注解的源码。

```java
@EnableCircuitBreaker
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Import(ZuulProxyMarkerConfiguration.class)
public @interface EnableZuulProxy {
}
```

这个注解导入了一个配置类ZuulProxyMarkerConfiguration。

```java
/**
 * Responsible for adding in a marker bean to trigger activation of 
 * {@link ZuulProxyAutoConfiguration}
 *
 * @author Biju Kunjummen
 */

@Configuration
public class ZuulProxyMarkerConfiguration {
   @Bean
   public Marker zuulProxyMarkerBean() {
      return new Marker();
   }

   class Marker {
   }
}
```

可以看到这个类什么都没做，但是注释上说明了这个类的作用是触发另一个配置类ZuulProxyAutoConfiguration。

```java
@Configuration
//导入了发送http请求的配置类，分别是对restClient、httpClient、OKhttp等封装的配置类
@Import({ RibbonCommandFactoryConfiguration.RestClientRibbonConfiguration.class,
      RibbonCommandFactoryConfiguration.OkHttpRibbonConfiguration.class,
      RibbonCommandFactoryConfiguration.HttpClientRibbonConfiguration.class,
      HttpClientConfiguration.class })
@ConditionalOnBean(ZuulProxyMarkerConfiguration.Marker.class)
public class ZuulProxyAutoConfiguration extends ZuulServerAutoConfiguration {

   @SuppressWarnings("rawtypes")
   @Autowired(required = false)
   private List<RibbonRequestCustomizer> requestCustomizers = Collections.emptyList();

   @Autowired(required = false)
   private Registration registration;

   @Autowired
   private DiscoveryClient discovery;

   @Autowired
   private ServiceRouteMapper serviceRouteMapper;

   @Override
   public HasFeatures zuulFeature() {
      return HasFeatures.namedFeature("Zuul (Discovery)",
            ZuulProxyAutoConfiguration.class);
   }
   //依赖服务发现的路由定向类
   @Bean
   @ConditionalOnMissingBean(DiscoveryClientRouteLocator.class)
   public DiscoveryClientRouteLocator discoveryRouteLocator() {
      return new DiscoveryClientRouteLocator(this.server.getServlet().getContextPath(), this.discovery, this.zuulProperties,
            this.serviceRouteMapper, this.registration);
   }

   // pre filters 前置过滤器，基于RouteLocator决定如何路由以及路由到哪个服务
   @Bean
   @ConditionalOnMissingBean(PreDecorationFilter.class)
   public PreDecorationFilter preDecorationFilter(RouteLocator routeLocator, ProxyRequestHelper proxyRequestHelper) {
      return new PreDecorationFilter(routeLocator, this.server.getServlet().getContextPath(), this.zuulProperties,
            proxyRequestHelper);
   }

   // route filters 路由过滤器，和ribbon结合的路由过滤器，决定路由到服务的具体哪个实例
   @Bean
   @ConditionalOnMissingBean(RibbonRoutingFilter.class)
   public RibbonRoutingFilter ribbonRoutingFilter(ProxyRequestHelper helper,
         RibbonCommandFactory<?> ribbonCommandFactory) {
      RibbonRoutingFilter filter = new RibbonRoutingFilter(helper, ribbonCommandFactory,
            this.requestCustomizers);
      return filter;
   }
   //route filter 路由过滤器，针对预先决定的(配置了URL的)服务，执行简单的路由功能，并且此时也没有配置	CloseableHttpClient的bean
   @Bean
   @ConditionalOnMissingBean({SimpleHostRoutingFilter.class, CloseableHttpClient.class})
   public SimpleHostRoutingFilter simpleHostRoutingFilter(ProxyRequestHelper helper,
         ZuulProperties zuulProperties,
         ApacheHttpClientConnectionManagerFactory connectionManagerFactory,
         ApacheHttpClientFactory httpClientFactory) {
      return new SimpleHostRoutingFilter(helper, zuulProperties,
            connectionManagerFactory, httpClientFactory);
   }
//route filter 路由过滤器，针对预先决定的(配置了URL的)服务，执行简单的路由功能，并且配置	CloseableHttpClient的bean
   @Bean
   @ConditionalOnMissingBean({SimpleHostRoutingFilter.class})
   public SimpleHostRoutingFilter simpleHostRoutingFilter2(ProxyRequestHelper helper,
                                             ZuulProperties zuulProperties,
                                             CloseableHttpClient httpClient) {
      return new SimpleHostRoutingFilter(helper, zuulProperties,
            httpClient);
   }

   @Bean
   @ConditionalOnMissingBean(ServiceRouteMapper.class)
   public ServiceRouteMapper serviceRouteMapper() {
      return new SimpleServiceRouteMapper();
   }

   @Configuration
   @ConditionalOnMissingClass("org.springframework.boot.actuate.health.Health")
   protected static class NoActuatorConfiguration {

      //请求代理工具类，处理请求的URL转换、参数、请求头等等
      @Bean
      public ProxyRequestHelper proxyRequestHelper(ZuulProperties zuulProperties) {
         ProxyRequestHelper helper = new ProxyRequestHelper(zuulProperties);
         return helper;
      }

   }

   @Configuration
   @ConditionalOnClass(Health.class)
   protected static class EndpointConfiguration {

      @Autowired(required = false)
      private HttpTraceRepository traces;

      //列出所有路由信息的endpoint 
      @Bean
      @ConditionalOnEnabledEndpoint
      public RoutesEndpoint routesEndpoint(RouteLocator routeLocator) {
         return new RoutesEndpoint(routeLocator);
      }
       
      //列出所有zuul的过滤器的endpoint 
      @ConditionalOnEnabledEndpoint
      @Bean
      public FiltersEndpoint filtersEndpoint() {
         FilterRegistry filterRegistry = FilterRegistry.instance();
         return new FiltersEndpoint(filterRegistry);
      }

      //增强了log的请求代理工具类
      @Bean
      public ProxyRequestHelper proxyRequestHelper(ZuulProperties zuulProperties) {
         TraceProxyRequestHelper helper = new TraceProxyRequestHelper(zuulProperties);
         if (this.traces != null) {
            helper.setTraces(this.traces);
         }
         return helper;
      }
   }
}
```

从源码中可以看到，ZuulProxyAutoConfiguration配置了一些诸如DiscoveryClientRouteLocator（依赖服务发现的路由定向类）、ProxyRequestHelper（请求代理工具类）、zuul的几个pre、route、post过滤器等bean。

ZuulProxyAutoConfiguration还导入了兼容了几种兼容发送http请求的配置类，如HTTPClient、OKhttp、restClient等。

除此之外，ZuulProxyAutoConfiguration还继承了ZuulProxyAutoConfiguration类。我们看下这个类做了哪些配置操作。

```java
@Configuration
@EnableConfigurationProperties({ ZuulProperties.class })
@ConditionalOnClass(ZuulServlet.class)//基于ZuulServlet的配置
@ConditionalOnBean(ZuulServerMarkerConfiguration.Marker.class)
// Make sure to get the ServerProperties from the same place as a normal web app would
// FIXME @Import(ServerPropertiesAutoConfiguration.class)
public class ZuulServerAutoConfiguration {

   //zuul的配置类，之前配置的zuul的prefix、routes等都是配置在这个类中
   @Autowired
   protected ZuulProperties zuulProperties;

   //server的配置类，主要是获取contextPath 
   @Autowired
   protected ServerProperties server;

   //错误异常处理controller，springboot自带BasicErrorController 
   @Autowired(required = false)
   private ErrorController errorController;

   private Map<String, CorsConfiguration> corsConfigurations;

   @Autowired(required = false)
   private List<WebMvcConfigurer> configurers = emptyList();

   @Bean
   public HasFeatures zuulFeature() {
      return HasFeatures.namedFeature("Zuul (Simple)", ZuulServerAutoConfiguration.class);
   }

   //组合的路由定向类
   @Bean
   @Primary
   public CompositeRouteLocator primaryRouteLocator(
         Collection<RouteLocator> routeLocators) {
      return new CompositeRouteLocator(routeLocators);
   }

   //简单的路由定向类，基于ZuulProperties 
   @Bean
   @ConditionalOnMissingBean(SimpleRouteLocator.class)
   public SimpleRouteLocator simpleRouteLocator() {
      return new SimpleRouteLocator(this.server.getServlet().getContextPath(),
            this.zuulProperties);
   }

   //声明ZuulController，将ZuulServlet交给spring管理，springMvc mapping到路由，会路由到此controller 
   @Bean
   public ZuulController zuulController() {
      return new ZuulController();
   }

   //配置zuul handlerMapping，将请求的地址与远程服务匹配 
   @Bean
   public ZuulHandlerMapping zuulHandlerMapping(RouteLocator routes) {
      ZuulHandlerMapping mapping = new ZuulHandlerMapping(routes, zuulController());
      mapping.setErrorController(this.errorController);
      mapping.setCorsConfigurations(getCorsConfigurations());
      return mapping;
   }

   protected final Map<String, CorsConfiguration> getCorsConfigurations() {
      if (this.corsConfigurations == null) {
         ZuulCorsRegistry registry = new ZuulCorsRegistry();
         this.configurers
               .forEach(configurer -> configurer.addCorsMappings(registry));
         this.corsConfigurations = registry.getCorsConfigurations();
      }
      return this.corsConfigurations;
   }

   //zuul的应用监听器，用于监听下游服务有没有刷新applicationContext 
   @Bean
   public ApplicationListener<ApplicationEvent> zuulRefreshRoutesListener() {
      return new ZuulRefreshListener();
   }

   //注册zuulServlet到spring中 
   @Bean
   @ConditionalOnMissingBean(name = "zuulServlet")
   public ServletRegistrationBean zuulServlet() {
      ServletRegistrationBean<ZuulServlet> servlet = new ServletRegistrationBean<>(new ZuulServlet(),
            this.zuulProperties.getServletPattern());
      // The whole point of exposing this servlet is to provide a route that doesn't
      // buffer requests.
      servlet.addInitParameter("buffer-requests", "false");
      return servlet;
   }

   // pre filters，决定请求是走DispatcherServlet还是ZuulServlet
   @Bean
   public ServletDetectionFilter servletDetectionFilter() {
      return new ServletDetectionFilter();
   }
   //解析form表单数据，并重新编码		
   @Bean
   public FormBodyWrapperFilter formBodyWrapperFilter() {
      return new FormBodyWrapperFilter();
   }

   @Bean
   public DebugFilter debugFilter() {
      return new DebugFilter();
   }

   //支持servlet 3.0的包装过滤器，对请求进行包装 
   @Bean
   public Servlet30WrapperFilter servlet30WrapperFilter() {
      return new Servlet30WrapperFilter();
   }

   // post filters，后置过滤器，将代理请求中的响应写入response中
   @Bean
   public SendResponseFilter sendResponseFilter(ZuulProperties properties) {
      return new SendResponseFilter(zuulProperties);
   }

   //处理error的filter，执行顺序为0 
   @Bean
   public SendErrorFilter sendErrorFilter() {
      return new SendErrorFilter();
   }

   //重定向的filter，执行顺序为500 
   @Bean
   public SendForwardFilter sendForwardFilter() {
      return new SendForwardFilter();
   }

   //饥饿加载时，从zuulPropertis中获取所有路由信息 
   @Bean
   @ConditionalOnProperty(value = "zuul.ribbon.eager-load.enabled")
   public ZuulRouteApplicationContextInitializer zuulRoutesApplicationContextInitiazer(
         SpringClientFactory springClientFactory) {
      return new ZuulRouteApplicationContextInitializer(springClientFactory,
            zuulProperties);
   }

   @Configuration
   protected static class ZuulFilterConfiguration {

      @Autowired
      private Map<String, ZuulFilter> filters;

      //初始化zuul filter各种各样组件
      @Bean
      public ZuulFilterInitializer zuulFilterInitializer(
            CounterFactory counterFactory, TracerFactory tracerFactory) {
         FilterLoader filterLoader = FilterLoader.getInstance();
         FilterRegistry filterRegistry = FilterRegistry.instance();
         return new ZuulFilterInitializer(this.filters, counterFactory, tracerFactory, filterLoader, filterRegistry);
      }

   }
//....
}
```

zuul提供的过滤器执行顺序如下：

![zuul filters执行顺序](/Users/wangwangxiaoteng/work/code/github/blogs/zuul filters执行顺序.jpg)

zuul的关键逻辑在ZuulServlet中。

```java
public class ZuulServlet extends HttpServlet {

    private ZuulRunner zuulRunner;

    @Override
    public void init(ServletConfig config) throws ServletException {
        super.init(config);
		//从初始化方法参数中，获取是否缓存请求的配置
        String bufferReqsStr = config.getInitParameter("buffer-requests");
        boolean bufferReqs = bufferReqsStr != null && bufferReqsStr.equals("true") ? true : false;
		//构造ZuulRunner，初始化request & response，也是执行filter的门面
        zuulRunner = new ZuulRunner(bufferReqs);
    }

    @Override
    public void service(javax.servlet.ServletRequest servletRequest, javax.servlet.ServletResponse servletResponse) throws ServletException, IOException {
        try {
            //初始化request、response
            init((HttpServletRequest) servletRequest, (HttpServletResponse) servletResponse);

            // Marks this request as having passed through the "Zuul engine", as opposed to servlets
            // explicitly bound in web.xml, for which requests will not have the same data attached
            RequestContext context = RequestContext.getCurrentContext();
            context.setZuulEngineRan();

            try {
                //1.先执行前置过滤器
                preRoute();
            } catch (ZuulException e) {
                //发生异常执行error过滤器
                error(e);
                //执行后置过滤器
                postRoute();
                return;
            }
            try {
                //2.执行路由过滤器
                route();
            } catch (ZuulException e) {
                //发生异常执行error过滤器
                error(e);
                //执行后置过滤器
                postRoute();
                return;
            }
            try {
                //3.执行后置过滤器
                postRoute();
            } catch (ZuulException e) {
                //发生异常执行error过滤器
                error(e);
                return;
            }

        } catch (Throwable e) {
            error(new ZuulException(e, 500, "UNHANDLED_EXCEPTION_" + e.getClass().getName()));
        } finally {
            RequestContext.getCurrentContext().unset();
        }
    }

    /**
     * executes "post" ZuulFilters
     *
     * @throws ZuulException
     */
    void postRoute() throws ZuulException {
        //通过ZuulRunner执行后置过滤器
        zuulRunner.postRoute();
    }

    /**
     * executes "route" filters
     *
     * @throws ZuulException
     */
    void route() throws ZuulException {
        //通过ZuulRunner执行路由过滤器
        zuulRunner.route();
    }

    /**
     * executes "pre" filters
     *
     * @throws ZuulException
     */
    void preRoute() throws ZuulException {
        //通过ZuulRunner执行前缀过滤器
        zuulRunner.preRoute();
    }

    /**
     * initializes request
     *
     * @param servletRequest
     * @param servletResponse
     */
    void init(HttpServletRequest servletRequest, HttpServletResponse servletResponse) {
        //通过ZuulRunner执行init方法，初始化request & response
        zuulRunner.init(servletRequest, servletResponse);
    }

    /**
     * sets error context info and executes "error" filters
     *
     * @param e
     */
    void error(ZuulException e) {
        //通过ZuulRunner执行error过滤器
        RequestContext.getCurrentContext().setThrowable(e);
        zuulRunner.error();
    }
}    
```

先看下init()方法。

```java
void init(HttpServletRequest servletRequest, HttpServletResponse servletResponse) {
    //直接调用了ZuulRunner的init()方法
    zuulRunner.init(servletRequest, servletResponse);
}
```

ZuulRunner的init()方法。

```java
public void init(HttpServletRequest servletRequest, HttpServletResponse servletResponse) {
	//获取请求上下文
    RequestContext ctx = RequestContext.getCurrentContext();
    if (bufferRequests) {
        //如果缓存请求内容，包装下request，再将包装后的request放入请求上下文中，供其他类使用，例如过滤器
        ctx.setRequest(new HttpServletRequestWrapper(servletRequest));
    } else {
        //将request放入请求上下文中，供其他类使用，例如过滤器
        ctx.setRequest(servletRequest);
    }
	 //将response放入请求上下文中，供其他类使用，例如过滤器
    ctx.setResponse(new HttpServletResponseWrapper(servletResponse));
}
```

再看下preRoute()方法。

```java
/**
 * executes "pre" filters
 *
 * @throws ZuulException
 */
void preRoute() throws ZuulException {
    //通过ZuulRunner执行前缀过滤器
    zuulRunner.preRoute();
}
```

ZuulRunner的preRoute()方法。

```java
/**
 * executes "pre" filterType  ZuulFilters
 *
 * @throws ZuulException
 */
public void preRoute() throws ZuulException {
    //通过FilterProcessor执行preRoute方法
    FilterProcessor.getInstance().preRoute();
}
```

FilterProcessor的preRoute()方法。

```java
/**
 * runs all "pre" filters. These filters are run before routing to the orgin.
 *
 * @throws ZuulException
 */
public void preRoute() throws ZuulException {
    //...
    runFilters("pre");
    //...
}

public Object runFilters(String sType) throws Throwable {
        //...
    	//定义全局结果布尔值
        boolean bResult = false;
    	//从FilterLoader中根据过滤器类型获取该类型的所有zuulFilters
        List<ZuulFilter> list = FilterLoader.getInstance().getFiltersByType(sType);
        if (list != null) {
            for (int i = 0; i < list.size(); i++) {
                //逐个获取过滤器实例
                ZuulFilter zuulFilter = list.get(i);
                //执行过滤器
                Object result = processZuulFilter(zuulFilter);
                if (result != null && result instanceof Boolean) {
                    bResult |= ((Boolean) result);
                }
            }
        }
        return bResult;
}

public Object processZuulFilter(ZuulFilter filter) throws ZuulException {
		//获取请求上下文
        RequestContext ctx = RequestContext.getCurrentContext();
        long execTime = 0;
        try {
            //记录执行过滤器时开始时间
            long ltime = System.currentTimeMillis();
            Object o = null;
            Throwable t = null;
			//执行过滤器
            ZuulFilterResult result = filter.runFilter();
            //过滤器执行结果状态
            ExecutionStatus s = result.getStatus();
            //记录执行过滤器所耗时间
            execTime = System.currentTimeMillis() - ltime;

            switch (s) {
                case FAILED:
                    //如果过滤器执行结果状态是FAILED，获取执行时异常，向请求上下文中塞入执行结果概要
                    t = result.getException();
                    ctx.addFilterExecutionSummary(filterName, ExecutionStatus.FAILED.name(), execTime);
                    break;
                case SUCCESS:
                    //如果过滤器执行结果状态是SUCCESS，获取执行结果，向请求上下文中塞入执行结果概要
                    o = result.getResult();
                    ctx.addFilterExecutionSummary(filterName, ExecutionStatus.SUCCESS.name(), execTime);
                    
                    break;
                default:
                    break;
            }
            //如果执行过滤器失败，有异常，这里再抛出异常
            if (t != null) throw t;
            return o;
        } 
    //...
    }
```

接下来看如何执行过滤器，追踪ZuulFilter的runFilter()方法。

```java
public ZuulFilterResult runFilter() {
    //构造zuul 过滤器执行结果类
    ZuulFilterResult zr = new ZuulFilterResult();
    //先判断过滤器是否关闭了
    if (!isFilterDisabled()) {
        //判断是否要执行过滤器，通过调用该过滤器的shouldFilter()方法
        if (shouldFilter()) {
            //....
            try {
                //调用该过滤器的run()方法
                Object res = run();
                //封装run()方法执行结果到过滤器执行结果类中
                zr = new ZuulFilterResult(res, ExecutionStatus.SUCCESS);
            } catch (Throwable e) {
                //如果执行run()方法异常了，封装过滤器执行结果类
                zr = new ZuulFilterResult(ExecutionStatus.FAILED);
                zr.setException(e);
            }
            //....
        } else {
            //不执行过滤器，封装过滤器执行结果类
            zr = new ZuulFilterResult(ExecutionStatus.SKIPPED);
        }
    }
    return zr;
}
```

在FilterProcessor的runFilters()方法中，有一行代码值得说一下。

​        `List<ZuulFilter> list = FilterLoader.getInstance().getFiltersByType(sType);`

这句代码的意思是根据zuul的过滤器类型，"pre"、"route"、"post"、"error"等获取过滤器列表。

追踪下FilterLoader的getFiltersByType()方法。

```java
/**
 * Returns a list of filters by the filterType specified
 *
 * @param filterType
 * @return a List<ZuulFilter>
 */
public List<ZuulFilter> getFiltersByType(String filterType) {
	//这里hashFiltersByType是一个ConcurrentHashMap，key为filterType，value为zuul的所有filterType类型的filters
    //先从内存缓存中获取该过滤器类型的过滤器列表
    List<ZuulFilter> list = hashFiltersByType.get(filterType);
    if (list != null) return list;

    list = new ArrayList<ZuulFilter>();
	//如果内存缓存中没有这些过滤器list，再从FilterRegistry中获取所有过滤器，遍历，获取所有该filterType的filter list
    Collection<ZuulFilter> filters = filterRegistry.getAllFilters();
    for (Iterator<ZuulFilter> iterator = filters.iterator(); iterator.hasNext(); ) {
        ZuulFilter filter = iterator.next();
        if (filter.filterType().equals(filterType)) {
            list.add(filter);
        }
    }
    //按照filterOrder排序，从小到大
    Collections.sort(list); // sort by priority
	//从FilterRegistry中获取的所有过滤器再放入内存缓存中
    hashFiltersByType.putIfAbsent(filterType, list);
    return list;
}
```

FilterRegistr整个类比较简单，这里直接列出这个类。

```java
public class FilterRegistry {

    private static final FilterRegistry INSTANCE = new FilterRegistry();

    public static final FilterRegistry instance() {
        return INSTANCE;
    }

    private final ConcurrentHashMap<String, ZuulFilter> filters = new ConcurrentHashMap<String, ZuulFilter>();

    private FilterRegistry() {
    }

    public ZuulFilter remove(String key) {
        return this.filters.remove(key);
    }

    public ZuulFilter get(String key) {
        return this.filters.get(key);
    }

    public void put(String key, ZuulFilter filter) {
        this.filters.putIfAbsent(key, filter);
    }

    public int size() {
        return this.filters.size();
    }

    public Collection<ZuulFilter> getAllFilters() {
        //可以看到getAllFilters()方法就是直接将这个类中的ConcurrentHashMap缓存的值返回出去了。
        return this.filters.values();
    }

}
```

既然FilterRegistry中的这个ConcurrentHashMap有所有过滤器的数据，那么这个数据是什么时候放进去的呢？

我们看下FilterRegistry的put()方法在什么时候被调用。

追踪到了ZuulFilterInitializer的contextInitialized()方法中。前面说过，这个ZuulFilterInitializer会在zuul启动时初始化，那么就在启动的时候就已经将FilterRegistry缓存的所有过滤器数据塞进去了。

```java
@PostConstruct
public void contextInitialized() {
   log.info("Starting filter initializer");

   TracerFactory.initialize(tracerFactory);
   CounterFactory.initialize(counterFactory);

   for (Map.Entry<String, ZuulFilter> entry : this.filters.entrySet()) {
      filterRegistry.put(entry.getKey(), entry.getValue());
   }
}
```

FilterProcessor的route()、postRoute()、error()方法和preRoute()方法类似，都是调用runFilters()方法，传入不同的filterType，这里不再赘述。

总结一下：

- ZuulProperties：zuul的配置类，yaml文件中配置的zuul的prefix、routes等都是映射到这个类中

- ZuulProxyMarkerConfiguration：配置类，引入了ZuulProxyAutoConfiguration配置。

- ZuulProxyAutoConfiguration：引入了restClient、HTTPClient、OKhttp等配置类，以及PreDecorationFilter、RibbonRoutingFilter等内置过滤器，还有ProxyRequestHelper请求代理类等。
- ZuulServerAutoConfiguration：配置了CompositeRouteLocator（组合的路由定向类）、SimpleRouteLocator（简单的路由定向类）、声明了ZuulController、ZuulHandlerMapping等，注册了ZuulRefreshListener、ServletRegistrationBean等，配置了ServletDetectionFilter、FormBodyWrapperFilter、SendResponseFilter、SendErrorFilter、SendForwardFilter等filter。
- ZuulFilterInitializer：监听整个zuul Filter的生命周期，初始化zuul的各种组件，包括过滤器等，并初始化了FilterRegistry、FilterLoader，保存了各个类型的过滤器的列表。以及生命周期结束时时清除FilterRegistry、FilterLoader的过滤器缓存数据。
- ZuulServlet：zuul的1.X版本是依赖于servlet的，是zuul的所有访问的入口，包括对请求的过滤器执行。
- ZuulRunner：初始化request & response，以及将zuulServlet的调用转发到FilterProcessor中去。
- FilterProcessor：执行过滤器的核心类。

![image-20190118150429728](/Users/wangwangxiaoteng/work/code/github/blogs/zuul 几种过滤器类型执行顺序.png)

# 能不能有更好的方式解决这个问题

待完善。

# zuul存在的一些问题是什么

- 基于多线程+阻塞IO的网络模型，注定在面对大并发时处理能力相对较弱，不过2.X版本已经重新设计了架构，采用netty作为底层网络模型，并发连接数有了大幅度提升，性能也有了一定提升。
- 权限认证、限流等需要自己从头开发，无开箱即用的框架或接口。