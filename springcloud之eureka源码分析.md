## eureka server端启动分析

eureka server在启动时会打印日志，追踪日志发现，打印“Initializing …”的类为DefaultEurekaServerContext的initialize()方法。

```java
@PostConstruct
@Override
public void initialize() {
    logger.info("Initializing ...");
    peerEurekaNodes.start();
    try {
        registry.init(peerEurekaNodes);
    } catch (Exception e) {
        throw new RuntimeException(e);
    }
    logger.info("Initialized");
}
```

先看start()方法

```java
public void start() {
    //创造一个拥有单线程的线程池
    taskExecutor = Executors.newSingleThreadScheduledExecutor(
            new ThreadFactory() {
                @Override
                public Thread newThread(Runnable r) {
                    Thread thread = new Thread(r, "Eureka-PeerNodesUpdater");
                    thread.setDaemon(true);
                    return thread;
                }
            }
    );
    try {
        //先更新一下所有eureka节点的状态，包括新增eureka节点，下线eureka节点等
        updatePeerEurekaNodes(resolvePeerUrls());
        //新建一个线程，线程做的事情就是更新所有eureka节点的状态
        Runnable peersUpdateTask = new Runnable() {
            @Override
            public void run() {
                try {
                    updatePeerEurekaNodes(resolvePeerUrls());
                } catch (Throwable e) {
                    logger.error("Cannot update the replica Nodes", e);
                }

            }
        };
        //启动定时线程池执行任务，默认是10分钟一次
        taskExecutor.scheduleWithFixedDelay(
                peersUpdateTask,
                serverConfig.getPeerEurekaNodesUpdateIntervalMs(),
                serverConfig.getPeerEurekaNodesUpdateIntervalMs(),
                TimeUnit.MILLISECONDS
        );
    } catch (Exception e) {
        throw new IllegalStateException(e);
    }
    for (PeerEurekaNode node : peerEurekaNodes) {
        logger.info("Replica node URL:  {}", node.getServiceUrl());
    }
}
```

总结一下PeerEurekaNodes的start()方法：

- 更新所有eureka server节点的状态，新增或者下线部分eureka server节点。
- 创建一个拥有单个线程的线程池，定时更新所有eureka server节点的状态。默认情况下，是15分钟一次。

再看一下registry.init()方法。

这个registry实例为InstanceRegistry，init方法实际上是父类PeerAwareInstanceRegistryImpl的init()方法。

```java
public void init(PeerEurekaNodes peerEurekaNodes) throws Exception {
    this.numberOfReplicationsLastMin.start();
    this.peerEurekaNodes = peerEurekaNodes;
    //初始化缓存注册实例信息
    initializedResponseCache();
    //定时修改更新注册信息的阈值，防止短时间内下线太多注册服务
    scheduleRenewalThresholdUpdateTask();
    //初始化远程区域注册，region、zone、cluster区别联系，请看https://github.com/Netflix/eureka/issues/881
    initRemoteRegionRegistry();
    try {
        Monitors.registerObject(this);
    } catch (Throwable e) {
        logger.warn("Cannot register the JMX monitor for the InstanceRegistry :", e);
    }
}
```



先研究下initializedResponseCache()方法。

```java
public synchronized void initializedResponseCache() {
    if (responseCache == null) {
        responseCache = new ResponseCacheImpl(serverConfig, serverCodecs, this);
    }
}
```

这里如果为null的时候，创建了一个新的ResponseCacheImpl实例，我们看下它的构造方法。

这个ResponseCacheImpl其实就是一个实例注册信息的缓存类，可能会被客户端访问，支持gzip压缩。

这个ResponseCacheImpl在eureka-core工程的resource包里会被访问到。

```java
ResponseCacheImpl(EurekaServerConfig serverConfig, ServerCodecs serverCodecs, AbstractInstanceRegistry registry) {
        this.serverConfig = serverConfig;
        this.serverCodecs = serverCodecs;
    	//是否使用只读缓存
        this.shouldUseReadOnlyResponseCache = serverConfig.shouldUseReadOnlyResponseCache();
        this.registry = registry;
		//只读缓存更新间隔
        long responseCacheUpdateIntervalMs = serverConfig.getResponseCacheUpdateIntervalMs();
    	//读写缓存map
        this.readWriteCacheMap =
                CacheBuilder.newBuilder().initialCapacity(1000)
                        .expireAfterWrite(serverConfig.getResponseCacheAutoExpirationInSeconds(), TimeUnit.SECONDS)
                        .removalListener(new RemovalListener<Key, Value>() {
                            @Override
                            public void onRemoval(RemovalNotification<Key, Value> notification) {
                                Key removedKey = notification.getKey();
                                if (removedKey.hasRegions()) {
                                    Key cloneWithNoRegions = removedKey.cloneWithoutRegions();
                                    regionSpecificKeys.remove(cloneWithNoRegions, removedKey);
                                }
                            }
                        })
                        .build(new CacheLoader<Key, Value>() {
                            @Override
                            public Value load(Key key) throws Exception {
                                if (key.hasRegions()) {
                                    Key cloneWithNoRegions = key.cloneWithoutRegions();
                                    regionSpecificKeys.put(cloneWithNoRegions, key);
                                }
                                Value value = generatePayload(key);
                                return value;
                            }
                        });

        if (shouldUseReadOnlyResponseCache) {
            //使用只读缓存的话，创建定时任务定时更新
            timer.schedule(getCacheUpdateTask(),
                    new Date(((System.currentTimeMillis() / responseCacheUpdateIntervalMs) * responseCacheUpdateIntervalMs)
                            + responseCacheUpdateIntervalMs),
                    responseCacheUpdateIntervalMs);
        }

        try {
            Monitors.registerObject(this);
        } catch (Throwable e) {
            logger.warn("Cannot register the JMX monitor for the InstanceRegistry", e);
        }
    }
```

总结一下initializedResponseCache()方法：

- 创建读写缓存
- 创建定时任务，每隔一定时间同步读写缓存到只读缓存中

再看一下scheduleRenewalThresholdUpdateTask()方法：

```java
private void scheduleRenewalThresholdUpdateTask() {
    timer.schedule(new TimerTask() {
                       @Override
                       public void run() {
                           updateRenewalThreshold();
                       }
                   }, serverConfig.getRenewalThresholdUpdateIntervalMs(),
            serverConfig.getRenewalThresholdUpdateIntervalMs());
}
```



```java
private void updateRenewalThreshold() {
    try {
        Applications apps = eurekaClient.getApplications();
        int count = 0;
        for (Application app : apps.getRegisteredApplications()) {
            for (InstanceInfo instance : app.getInstances()) {
                //如果没有指定datacenter，或者datacenter不是AWS，那么都是可注册的
                if (this.isRegisterable(instance)) {
                    ++count;
                }
            }
        }
        synchronized (lock) {
            // Update threshold only if the threshold is greater than the
            // current expected threshold or if self preservation is disabled.
            if ((count * 2) > (serverConfig.getRenewalPercentThreshold() * expectedNumberOfRenewsPerMin)
                    || (!this.isSelfPreservationModeEnabled())) {
                this.expectedNumberOfRenewsPerMin = count * 2;
                this.numberOfRenewsPerMinThreshold = (int) ((count * 2) * serverConfig.getRenewalPercentThreshold());
            }
        }
        logger.info("Current renewal threshold is : {}", numberOfRenewsPerMinThreshold);
    } catch (Throwable e) {
        logger.error("Cannot update renewal threshold", e);
    }
}
```

scheduleRenewalThresholdUpdateTask()方法，就是创建一个定时任务，定时更新每分钟注册的实例的阈值。

再看一下EurekaServerInitializerConfiguration，这个类实现了SmartLifecycle，会在spring容器初始化时调用。

```java
public void start() {
   new Thread(new Runnable() {
      @Override
      public void run() {
         try {
            //这里就是调用了EurekaServerBootstrap的contextInitialized()方法
            eurekaServerBootstrap.contextInitialized(EurekaServerInitializerConfiguration.this.servletContext);
            log.info("Started Eureka Server");
			//发布eureka服务可用事件
            publish(new EurekaRegistryAvailableEvent(getEurekaServerConfig()));
            EurekaServerInitializerConfiguration.this.running = true;
            //发布eureka服务启动事件
            publish(new EurekaServerStartedEvent(getEurekaServerConfig()));
         }
         catch (Exception ex) {
            // Help!
            log.error("Could not initialize Eureka servlet context", ex);
         }
      }
   }).start();
}
```

EurekaServerBootstrap。

```java
public void contextInitialized(ServletContext context) {
		try {
            //读取配置文件，初始化环境信息
			initEurekaEnvironment();
            //初始化eureka server 上下文
			initEurekaServerContext();

			context.setAttribute(EurekaServerContext.class.getName(), this.serverContext);
		}
		catch (Throwable e) {
			log.error("Cannot bootstrap eureka server :", e);
			throw new RuntimeException("Cannot bootstrap eureka server :", e);
		}
	}
```

```java
/**
 * Users can override to initialize the environment themselves.
 */
protected void initEurekaEnvironment() throws Exception {
    logger.info("Setting the eureka configuration..");
	//EUREKA_DATACENTER===eureka.datacenter 这个GitHub上eureka描述是为了在AWS云上部署，自动初始化一些特定信息的。暂时无需关注
    String dataCenter = ConfigurationManager.getConfigInstance().getString(EUREKA_DATACENTER);
    if (dataCenter == null) {
      ConfigurationManager.getConfigInstance().setProperty(ARCHAIUS_DEPLOYMENT_DATACENTER, DEFAULT);
    } else {
      ConfigurationManager.getConfigInstance().setProperty(ARCHAIUS_DEPLOYMENT_DATACENTER, dataCenter);
    }
    //...EUREKA_ENVIRONMENT=eureka.environment，这个是为了说明环境是test,prod等等，用来指定eureka-client.properties配置文件，不过一般我们会将这些配置放入application.properties，这个貌似也用不着
    String environment = ConfigurationManager.getConfigInstance().getString(EUREKA_ENVIRONMENT);
    if (environment == null) {
     ConfigurationManager.getConfigInstance().setProperty(ARCHAIUS_DEPLOYMENT_ENVIRONMENT, TEST);
        logger.info("Eureka environment value eureka.environment is not set, defaulting to test");
    }else {
			ConfigurationManager.getConfigInstance()
					.setProperty(ARCHAIUS_DEPLOYMENT_ENVIRONMENT, environment);
		}
}
```

从上面代码可以看出initEurekaEnvironment()方法主要是初始化和环境一些相关的信息，比如设置了eureka.datacenter（只针对AWS云部署）以及eureka.environment（用来指定eureka-client.properties，一般用不着)等。

接下来分析initEurekaServerContext()方法。

```java
protected void initEurekaServerContext() throws Exception {
   //...
   log.info("Initialized server context");

   // Copy registry from neighboring eureka node 从其他eureka节点复制实例注册信息，并注册到自己上
   int registryCount = this.registry.syncUp();
    //
   this.registry.openForTraffic(this.applicationInfoManager, registryCount);

   // Register all monitoring statistics.
   EurekaMonitors.registerAllStats();
}
```

PeerAwareInstanceRegistryImpl.syncUp()方法

```java
public int syncUp() {
    // Copy entire entry from neighboring DS node
    int count = 0;

    for (int i = 0; ((i < serverConfig.getRegistrySyncRetries()) && (count == 0)); i++) {
        //....从eurekaClient获取所有应用信息，遍历，注册到当前eureka server上
        Applications apps = eurekaClient.getApplications();
        for (Application app : apps.getRegisteredApplications()) {
            for (InstanceInfo instance : app.getInstances()) {
                try {
                    if (isRegisterable(instance)) {
                        register(instance, instance.getLeaseInfo().getDurationInSecs(), true);
                        count++;
                    }
                } catch (Throwable t) {
                    logger.error("During DS init copy", t);
                }
            }
        }
    }
    return count;
}
```

看一下具体的注册到eureka server的 方法，AbstractInstanceRegistry.register()方法

```java
public void register(InstanceInfo registrant, int leaseDuration, boolean isReplication) {
    try {
        read.lock();
        //获取相同appName的已注册实例信息，集群内的其他实例注册信息
        Map<String, Lease<InstanceInfo>> gMap = registry.get(registrant.getAppName());
        REGISTER.increment(isReplication);
        if (gMap == null) {
            //如果是第一个注册的实例，新建一个concurrentHashMap用来存放相同appName的实例信息
            final ConcurrentHashMap<String, Lease<InstanceInfo>> gNewMap = new ConcurrentHashMap<String, Lease<InstanceInfo>>();
            //在并发情况下，可能会有两个同时操作
            gMap = registry.putIfAbsent(registrant.getAppName(), gNewMap);
            if (gMap == null) {
                //如果并发时，只会有一个key为appName的concurrentHashMap创建
                gMap = gNewMap;
            }
        }
        //获取这个map下以instanceInfo的id为key的Lease(租约)
        //这里说明一下，registrant.getId()如果是非AWS应用，就是InstanceInfo的instanceId
        Lease<InstanceInfo> existingLease = gMap.get(registrant.getId());
        // Retain the last dirty timestamp without overwriting it, if there is already a lease
        if (existingLease != null && (existingLease.getHolder() != null)) {
            //如果已存在相同InstanceInfo id的Lease租约，比较两者的LastDirtyTimestamp，选择最新的Lease关联的InstanceInfo
            Long existingLastDirtyTimestamp = existingLease.getHolder().getLastDirtyTimestamp();
            Long registrationLastDirtyTimestamp = registrant.getLastDirtyTimestamp();
            logger.debug("Existing lease found (existing={}, provided={}", existingLastDirtyTimestamp, registrationLastDirtyTimestamp);

            // this is a > instead of a >= because if the timestamps are equal, we still take the remote transmitted
            // InstanceInfo instead of the server local copy.
            if (existingLastDirtyTimestamp > registrationLastDirtyTimestamp) {
                logger.warn("There is an existing lease and the existing lease's dirty timestamp {} is greater" +
                        " than the one that is being registered {}", existingLastDirtyTimestamp, registrationLastDirtyTimestamp);
                logger.warn("Using the existing instanceInfo instead of the new instanceInfo as the registrant");
                registrant = existingLease.getHolder();
            }
        } else {
            //不存在相同instanceInfo  id的Lease租约，更新expectedNumberOfRenewsPerMin和阈值
            // The lease does not exist and hence it is a new registration
            synchronized (lock) {
                if (this.expectedNumberOfRenewsPerMin > 0) {
                    // Since the client wants to cancel it, reduce the threshold
                    // (1
                    // for 30 seconds, 2 for a minute)
                    this.expectedNumberOfRenewsPerMin = this.expectedNumberOfRenewsPerMin + 2;
                    this.numberOfRenewsPerMinThreshold =
                            (int) (this.expectedNumberOfRenewsPerMin * serverConfig.getRenewalPercentThreshold());
                }
            }
            logger.debug("No previous lease information found; it is new registration");
        }
        //重新构造一个Lease，并放入相同appName的map中，key为InstanceInfo的id，value为Lease本身
        Lease<InstanceInfo> lease = new Lease<InstanceInfo>(registrant, leaseDuration);
        if (existingLease != null) {
            lease.setServiceUpTimestamp(existingLease.getServiceUpTimestamp());
        }
        gMap.put(registrant.getId(), lease);
        synchronized (recentRegisteredQueue) {
            recentRegisteredQueue.add(new Pair<Long, String>(
                    System.currentTimeMillis(),
                    registrant.getAppName() + "(" + registrant.getId() + ")"));
        }
        //更新注册实例的状态
        // This is where the initial state transfer of overridden status happens
        if (!InstanceStatus.UNKNOWN.equals(registrant.getOverriddenStatus())) {
            logger.debug("Found overridden status {} for instance {}. Checking to see if needs to be add to the "
                            + "overrides", registrant.getOverriddenStatus(), registrant.getId());
            if (!overriddenInstanceStatusMap.containsKey(registrant.getId())) {
                logger.info("Not found overridden id {} and hence adding it", registrant.getId());
                overriddenInstanceStatusMap.put(registrant.getId(), registrant.getOverriddenStatus());
            }
        }
        InstanceStatus overriddenStatusFromMap = overriddenInstanceStatusMap.get(registrant.getId());
        if (overriddenStatusFromMap != null) {
            logger.info("Storing overridden status {} from map", overriddenStatusFromMap);
            registrant.setOverriddenStatus(overriddenStatusFromMap);
        }

        // Set the status based on the overridden status rules
        InstanceStatus overriddenInstanceStatus = getOverriddenInstanceStatus(registrant, existingLease, isReplication);
        registrant.setStatusWithoutDirty(overriddenInstanceStatus);

        // If the lease is registered with UP status, set lease service up timestamp
        if (InstanceStatus.UP.equals(registrant.getStatus())) {
            lease.serviceUp();
        }
        registrant.setActionType(ActionType.ADDED);
        recentlyChangedQueue.add(new RecentlyChangedItem(lease));
        registrant.setLastUpdatedTimestamp();
        invalidateCache(registrant.getAppName(), registrant.getVIPAddress(), registrant.getSecureVipAddress());
        logger.info("Registered instance {}/{} with status {} (replication={})",
                registrant.getAppName(), registrant.getId(), registrant.getStatus(), isReplication);
    } finally {
        read.unlock();
    }
}
```

总结一下register()方法：

- 从注册信息缓存map中获取以注册的实例的appName为key的实例集合map，如果没有则新建一个map
- 从相同appName的实例map中获取以当前InstanceInfo的id的租约信息，如果有，和当前要注册的实例信息比较，选择最新的实例信息
- 更新实例信息的状态

再看一下openForTraffic()方法

```java
@Override
public void openForTraffic(ApplicationInfoManager applicationInfoManager, int count) {
    // Renewals happen every 30 seconds and for a minute it should be a factor of 2.
    //计算每分钟最大续约次数
    this.expectedNumberOfRenewsPerMin = count * 2;
    //计算每分钟最小续约次数=最大续约次数*启动自我保护模式的百分比阈值
    this.numberOfRenewsPerMinThreshold =
            (int) (this.expectedNumberOfRenewsPerMin * serverConfig.getRenewalPercentThreshold());
    logger.info("Got {} instances from neighboring DS node", count);
    logger.info("Renew threshold is: {}", numberOfRenewsPerMinThreshold);
    this.startupTime = System.currentTimeMillis();
    if (count > 0) {
        this.peerInstancesTransferEmptyOnStartup = false;
    }
    DataCenterInfo.Name selfName = applicationInfoManager.getInfo().getDataCenterInfo().getName();
    boolean isAws = Name.Amazon == selfName;
    if (isAws && serverConfig.shouldPrimeAwsReplicaConnections()) {
        //如果是AWS亚马逊云服务，做一些兼容
        logger.info("Priming AWS connections for all replicas..");
        primeAwsReplicas(applicationInfoManager);
    }
    logger.info("Changing status to UP");
    //更新状态
    applicationInfoManager.setInstanceStatus(InstanceStatus.UP);
    super.postInit();
}
```



```java
protected void postInit() {
    renewsLastMin.start();
    if (evictionTaskRef.get() != null) {
        evictionTaskRef.get().cancel();
    }
    //重点在于EvictionTask.run()方法
    evictionTaskRef.set(new EvictionTask());
    //创建定时任务，定时清理过期的注册实例
    evictionTimer.schedule(evictionTaskRef.get(),
            serverConfig.getEvictionIntervalTimerInMs(),
            serverConfig.getEvictionIntervalTimerInMs());
}

public void run() {
            try {
                long compensationTimeMs = getCompensationTimeMs();
                logger.info("Running the evict task with compensationTime {}ms", compensationTimeMs);
                //下线已过期的实例
                evict(compensationTimeMs);
            } catch (Throwable e) {
                logger.error("Could not run the evict task", e);
            }
  }
```

到这里基本就分析完了。

最后补充下，不从日志分析，如何确定启动的流程。

从@EnableEurekaServer入手，发现@Import(EurekaServerMarkerConfiguration.class)，import一个配置类。

```java
/**
 * Responsible for adding in a marker bean to activate
 * {@link EurekaServerAutoConfiguration}
 *
 * @author Biju Kunjummen
 */
@Configuration
public class EurekaServerMarkerConfiguration {

   @Bean
   public Marker eurekaServerMarkerBean() {
      return new Marker();
   }

   class Marker {
   }
}
```

可以看到这个类link到了EurekaServerAutoConfiguration,这里声明了EurekaServerBootstrap、peerAwareInstanceRegistry等Bean。

```java
@Bean
public PeerAwareInstanceRegistry peerAwareInstanceRegistry(
      ServerCodecs serverCodecs) {
   this.eurekaClient.getApplications(); // force initialization
   return new InstanceRegistry(this.eurekaServerConfig, this.eurekaClientConfig,
         serverCodecs, this.eurekaClient,
         this.instanceRegistryProperties.getExpectedNumberOfRenewsPerMin(),
         this.instanceRegistryProperties.getDefaultOpenForTrafficCount());
}

@Bean
@ConditionalOnMissingBean
public PeerEurekaNodes peerEurekaNodes(PeerAwareInstanceRegistry registry,
      ServerCodecs serverCodecs) {
   return new RefreshablePeerEurekaNodes(registry, this.eurekaServerConfig,
         this.eurekaClientConfig, serverCodecs, this.applicationInfoManager);
}

@Bean
	public EurekaServerContext eurekaServerContext(ServerCodecs serverCodecs,
			PeerAwareInstanceRegistry registry, PeerEurekaNodes peerEurekaNodes) {
		return new DefaultEurekaServerContext(this.eurekaServerConfig, serverCodecs,
				registry, peerEurekaNodes, this.applicationInfoManager);
	}

	@Bean
	public EurekaServerBootstrap eurekaServerBootstrap(PeerAwareInstanceRegistry registry,
			EurekaServerContext serverContext) {
		return new EurekaServerBootstrap(this.applicationInfoManager,
				this.eurekaClientConfig, this.eurekaServerConfig, registry,
				serverContext);
	}
```