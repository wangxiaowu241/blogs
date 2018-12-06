springboot工程注册到eureka server非常简单，只需要引入spring-cloud-starter-netflix-eureka-client依赖，在启动类上加上@EnableDiscoveryClient注解。

例如：

```java
@EnableDiscoveryClient
@SpringBootApplication
public class CartApplication {

    public static void main(String[] args) {
        SpringApplication.run(CartApplication.class, args);
    }
}
```

再在application.yml或者application.properties中指定eureka server地址即可。

例如：

```java
eureka:
  client:
    serviceUrl:
      defaultZone: http://localhost:8761/eureka/
```

![image-20181206105425600](/Users/wangwangxiaoteng/work/code/github/blogs/eureka server控制台.png)

启动，可以看到eureka server控制台已经有应用的注册信息了。那么eureka client 是如何做到的呢？

我们先从@EnableDiscoveryClient注解入手。

```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Inherited
@Import(EnableDiscoveryClientImportSelector.class)
public @interface EnableDiscoveryClient {

   /**
    * If true, the ServiceRegistry will automatically register the local server.
    */
   boolean autoRegister() default true;
}
```

发现引入了EnableDiscoveryClientImportSelector类，也没有发现有用的东西。

换个思路，既然注解是@EnableDiscoveryClient，那就研究一下DiscoveryClient。

```java
DiscoveryClient(ApplicationInfoManager applicationInfoManager, EurekaClientConfig config, AbstractDiscoveryClientOptionalArgs args,
                Provider<BackupRegistry> backupRegistryProvider) {
    
   	//...省略部分代码
    try {
        // default size of 2 - 1 each for heartbeat and cacheRefresh
        scheduler = Executors.newScheduledThreadPool(2,
                new ThreadFactoryBuilder()
                        .setNameFormat("DiscoveryClient-%d")
                        .setDaemon(true)
                        .build());

        heartbeatExecutor = new ThreadPoolExecutor(
                1, clientConfig.getHeartbeatExecutorThreadPoolSize(), 0, TimeUnit.SECONDS,
                new SynchronousQueue<Runnable>(),
                new ThreadFactoryBuilder()
                        .setNameFormat("DiscoveryClient-HeartbeatExecutor-%d")
                        .setDaemon(true)
                        .build()
        );  // use direct handoff

	//.....省略部分代码
}
```

可以看出是在DiscoveryClient类中的构造方法中初始化了心跳消息的线程池。再追踪下，构造方法会在哪里调用，发现会在CloudEurekaClient类的构造函数中调用。

```java
public CloudEurekaClient(ApplicationInfoManager applicationInfoManager,
                   EurekaClientConfig config,
                   AbstractDiscoveryClientOptionalArgs<?> args,
                   ApplicationEventPublisher publisher) {
   super(applicationInfoManager, config, args);
   this.applicationInfoManager = applicationInfoManager;
   this.publisher = publisher;
   this.eurekaTransportField = ReflectionUtils.findField(DiscoveryClient.class, "eurekaTransport");
   ReflectionUtils.makeAccessible(this.eurekaTransportField);
}
```

再追踪，发现会在EurekaClientAutoConfiguration类进行调用。

```java
@Configuration
@EnableConfigurationProperties
@ConditionalOnClass(EurekaClientConfig.class)
@Import(DiscoveryClientOptionalArgsConfiguration.class)
@ConditionalOnBean(EurekaDiscoveryClientConfiguration.Marker.class)
@ConditionalOnProperty(value = "eureka.client.enabled", matchIfMissing = true)
@AutoConfigureBefore({ NoopDiscoveryClientAutoConfiguration.class,
      CommonsClientAutoConfiguration.class, ServiceRegistryAutoConfiguration.class })
@AutoConfigureAfter(name = {"org.springframework.cloud.autoconfigure.RefreshAutoConfiguration",
      "org.springframework.cloud.netflix.eureka.EurekaDiscoveryClientConfiguration",
      "org.springframework.cloud.client.serviceregistry.AutoServiceRegistrationAutoConfiguration"})
public class EurekaClientAutoConfiguration {

   private ConfigurableEnvironment env;

   public EurekaClientAutoConfiguration(ConfigurableEnvironment env) {
      this.env = env;
   }

   @Bean
   public HasFeatures eurekaFeature() {
      return HasFeatures.namedFeature("Eureka Client", EurekaClient.class);
   }
    
    @Bean
	@ConditionalOnMissingBean(value = EurekaClientConfig.class, search = SearchStrategy.CURRENT)
	public EurekaClientConfigBean eurekaClientConfigBean(ConfigurableEnvironment env) {
		EurekaClientConfigBean client = new EurekaClientConfigBean();
		if ("bootstrap".equals(this.env.getProperty("spring.config.name"))) {
			// We don't register during bootstrap by default, but there will be another
			// chance later.
			client.setRegisterWithEureka(false);
		}
		return client;
	}
    //这里初始化了DiscoveryClient
    @Bean
	public DiscoveryClient discoveryClient(EurekaInstanceConfig config, EurekaClient client) {
		return new EurekaDiscoveryClient(config, client);
	}
	//....省略部分代码
}    
```

源头找到了，接下来仔细分析下DiscoveryClient的构造函数，这里是整个eureka client的 启动核心。

```java
@Inject
DiscoveryClient(ApplicationInfoManager applicationInfoManager, EurekaClientConfig config, AbstractDiscoveryClientOptionalArgs args,
                Provider<BackupRegistry> backupRegistryProvider) {
   
    //....

    logger.info("Initializing Eureka in region {}", clientConfig.getRegion());

    if (!config.shouldRegisterWithEureka() && !config.shouldFetchRegistry()) {
        //配置了不注册到eureka且不获取eureka server上的实例注册信息
        logger.info("Client configured to neither register nor query for data.");
        scheduler = null;
        heartbeatExecutor = null;
        cacheRefreshExecutor = null;
        eurekaTransport = null;
        instanceRegionChecker = new InstanceRegionChecker(new PropertyBasedAzToRegionMapper(config), clientConfig.getRegion());

        // This is a bit of hack to allow for existing code using DiscoveryManager.getInstance()
        // to work with DI'd DiscoveryClient
        DiscoveryManager.getInstance().setDiscoveryClient(this);
        DiscoveryManager.getInstance().setEurekaClientConfig(config);

        initTimestampMs = System.currentTimeMillis();
        logger.info("Discovery Client initialized at timestamp {} with initial instances count: {}",
                initTimestampMs, this.getApplications().size());

        return;  // no need to setup up an network tasks and we are done
    }

    try {
        // default size of 2 - 1 each for heartbeat and cacheRefresh
        //创建定时线程池，线程数量为2个，分别用来维持心跳连接和刷新其他eureka client实例缓存
        scheduler = Executors.newScheduledThreadPool(2,
                new ThreadFactoryBuilder()
                        .setNameFormat("DiscoveryClient-%d")
                        .setDaemon(true)
                        .build());
		//创建一个线程池，线程池大小默认为2个，用来维持心跳连接
        heartbeatExecutor = new ThreadPoolExecutor(
                1, clientConfig.getHeartbeatExecutorThreadPoolSize(), 0, TimeUnit.SECONDS,
                new SynchronousQueue<Runnable>(),
                new ThreadFactoryBuilder()
                        .setNameFormat("DiscoveryClient-HeartbeatExecutor-%d")
                        .setDaemon(true)
                        .build()
        );  // use direct handoff
		//创建一个线程池，线程池大小默认为2个，用来刷新其他eureka client实例缓存
        cacheRefreshExecutor = new ThreadPoolExecutor(
                1, clientConfig.getCacheRefreshExecutorThreadPoolSize(), 0, TimeUnit.SECONDS,
                new SynchronousQueue<Runnable>(),
                new ThreadFactoryBuilder()
                        .setNameFormat("DiscoveryClient-CacheRefreshExecutor-%d")
                        .setDaemon(true)
                        .build()
        );  // use direct handoff

        //创建及初始化维持心跳连接(注册)、获取注册实例信息的httpClient
        eurekaTransport = new EurekaTransport();
        scheduleServerEndpointTask(eurekaTransport, args);
        //...
    } catch (Throwable e) {
        throw new RuntimeException("Failed to initialize DiscoveryClient!", e);
    }
	//抓取远程实例注册信息，fetchRegistry()方法里的参数，这里为false，意思是要不要强制抓取所有实例注册信息
    //这里获取注册信息，分两种方式，一种是全量获取，另一种是增量获取，默认是增量获取
    if (clientConfig.shouldFetchRegistry() && !fetchRegistry(false)) {
        //如果配置的是要获取实例注册信息，但是从远程获取失败，从备份获取实例注册信息
        fetchRegistryFromBackup();
    }

    // call and execute the pre registration handler before all background tasks (inc registration) is started
    //注册到eureka之前，调用的前置处理方法，这里eureka提供了接口，需要自己实现。
    if (this.preRegistrationHandler != null) {
        this.preRegistrationHandler.beforeRegistration();
    }

    if (clientConfig.shouldRegisterWithEureka() && clientConfig.shouldEnforceRegistrationAtInit()) {
        //如果client配置注册到eureka server 且 强制 初始化就注册到eureka 那么就注册到eureka server，默认是不初始化就注册到eureka
        try {
            if (!register() ) {
                throw new IllegalStateException("Registration error at startup. Invalid server response.");
            }
        } catch (Throwable th) {
            logger.error("Registration error at startup: {}", th.getMessage());
            throw new IllegalStateException(th);
        }
    }

    // finally, init the schedule tasks (e.g. cluster resolvers, heartbeat, instanceInfo replicator, fetch
    //最后，初始化维持心跳连接、更新注册信息缓存的定时任务
    initScheduledTasks();
    // This is a bit of hack to allow for existing code using DiscoveryManager.getInstance()
    // to work with DI'd DiscoveryClient
    DiscoveryManager.getInstance().setDiscoveryClient(this);
    DiscoveryManager.getInstance().setEurekaClientConfig(config);

    initTimestampMs = System.currentTimeMillis();
    logger.info("Discovery Client initialized at timestamp {} with initial instances count: {}",
            initTimestampMs, this.getApplications().size());
}
```



```java
//获取远程注册实例信息
private boolean fetchRegistry(boolean forceFullRegistryFetch) {
    try {
        // If the delta is disabled or if it is the first time, get all
        // applications
        Applications applications = getApplications();

        if (clientConfig.shouldDisableDelta()
                || (!Strings.isNullOrEmpty(clientConfig.getRegistryRefreshSingleVipAddress()))
                || forceFullRegistryFetch
                || (applications == null)
                || (applications.getRegisteredApplications().size() == 0)
                || (applications.getVersion() == -1)) //Client application does not have latest library supporting delta
        {
           	//全量获取实例注册信息，这里是通过HTTPClient发送的请求，HTTPClient的初始化，在前面的scheduleServerEndpointTask()方法中
            getAndStoreFullRegistry();
        } else {
            //增量获取实例注册信息
            getAndUpdateDelta(applications);
        }
    }

    // 通知本地缓存要清除了
    onCacheRefreshed();
    // 更新远程实例信息在本机的状态
    updateInstanceRemoteStatus();
    return true;
}
```

注册方法：

```java
/**
 * Register with the eureka service by making the appropriate REST call.
 */
boolean register() throws Throwable {
    logger.info(PREFIX + "{}: registering service...", appPathIdentifier);
    EurekaHttpResponse<Void> httpResponse;
    try {
        //也是通过前面初始化的HTTPClient发送注册请求
        httpResponse = eurekaTransport.registrationClient.register(instanceInfo);
    } catch (Exception e) {
        logger.warn(PREFIX + "{} - registration failed {}", appPathIdentifier, e.getMessage(), e);
        throw e;
    }
    if (logger.isInfoEnabled()) {
        logger.info(PREFIX + "{} - registration status: {}", appPathIdentifier, httpResponse.getStatusCode());
    }
    return httpResponse.getStatusCode() == 204;
}
```

维持心跳连接：

```java
boolean renew() {
    EurekaHttpResponse<InstanceInfo> httpResponse;
    try {
        //发送请求
        httpResponse = eurekaTransport.registrationClient.sendHeartBeat(instanceInfo.getAppName(), instanceInfo.getId(), instanceInfo, null);
        logger.debug(PREFIX + "{} - Heartbeat status: {}", appPathIdentifier, httpResponse.getStatusCode());
        if (httpResponse.getStatusCode() == 404) {
            REREGISTER_COUNTER.increment();
            logger.info(PREFIX + "{} - Re-registering apps/{}", appPathIdentifier, instanceInfo.getAppName());
            long timestamp = instanceInfo.setIsDirtyWithTime();
            boolean success = register();
            if (success) {
                instanceInfo.unsetIsDirty(timestamp);
            }
            return success;
        }
        return httpResponse.getStatusCode() == 200;
    } catch (Throwable e) {
        logger.error(PREFIX + "{} - was unable to send heartbeat!", appPathIdentifier, e);
        return false;
    }
}
```

总结一下：

- eureka client在启动时首先会创建一个定时任务线程池，线程池大小为2个，分别用来维持心跳链接和刷新本地缓存
- 获取远程实例信息时，有两种方式，一种是全量获取，一种是增量获取，默认是增量获取
- 维持信条连接和获取远程实例信息是通过HTTPClient发送的请求