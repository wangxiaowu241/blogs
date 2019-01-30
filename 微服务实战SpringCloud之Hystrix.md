## Hystrix是什么？

在微服务架构中，微服务之间互相依赖较大，相互之间调用必不可免的会失败。但当下游服务A因为瞬时流量导致服务崩溃，其他依赖于A服务的B、C服务由于调用A服务超时耗费了大量的资源，长时间下去，B、C服务也会崩溃。Hystrix就是用来解决服务之间相互调用失败，避免产生蝴蝶效应的熔断器，以及提供降级选项。Hystrix通过隔离服务之间的访问点，阻止它们之间的级联故障以及提供默认选项来实现这一目标，以提高系统的整体健壮性。

## 用来解决什么问题？

用来避免由于服务之间依赖较重，出现个别服务宕机、停止服务导致大面积服务雪崩的情况。

## 小试牛刀

### 服务降级

目前有eureka、zuul、product、cart四个服务。

目的是通过zuul调用product接口，product接口通过feign调用cart服务的接口。product项目调用cart项目超时触发hystrix熔断及服务降级。

现将关键配置及代码陈列如下：

#### eureka：

```yaml
server:
  port: 8761

eureka:
  client:
    service-url:
      defaultZone: http://${eureka.instance.hostname:localhost}:${server.port:8761}/eureka

spring:
  application:
    name: eureka
  profiles:
    active: ${boot.profile:dev}
```

#### zuul

服务配置：

```yaml
spring:
  application:
    name: zuul #应用名称
server:
  port: 8080 #应用服务端口

eureka:
  client:
    service-url:
      defaultZone: http://localhost:8761/eureka  #配置eureka 默认分区的地址

ribbon:
  okhttp:
    enabled: true #开启ribbon 使用OKhttp发送http请求

  ReadTimeout: 5000
  ConnectTimeout: 5000

zuul:
  prefix: /api #定义全局路由前缀
  strip-prefix: true #路由到下游服务时开启去除前缀开关
```

路由配置：

```java
@Configuration
public class ZuuPatternServiceRouteMapperConfiguration {

    /**
     * 获取没有版本号的路由匹配规则bean
     *
     * @return {@link PatternServiceRouteMapper}
     * @date 10:27 AM 2019/1/17
     **/
    @Bean
    public PatternServiceRouteMapper patternServiceRouteMapper() {

        return new PatternServiceRouteMapper("(?<version>v.*$)", "${name}");
    }
}
```

webMvc配置：

```java
@Configuration
public class WebMvcConfig implements WebMvcConfigurer {

    @Override
    public void addCorsMappings(CorsRegistry registry) {
        registry.addMapping("/*")
                .allowedOrigins("*");
    }
}
```

#### product

```yaml
server:
  port: 8082

eureka:
  client:
    service-url:
      defaultZone: http://localhost:8761/eureka
spring:
  application:
    name: product
  profiles:
    active: ${boot.profile:dev}

feign:
  client:
    config:
      default:
        connectTimeout: 1000 #feign调用连接超时时间
        readTimeout: 1000 #feign调用读取超时时间
        loggerLevel: basic #feign调用的log 等级
```

启动类代码：

```java
@EnableFeignClients
@EnableDiscoveryClient
@SpringBootApplication
@EnableCircuitBreaker
public class ProductApplication {

    public static void main(String[] args) {
        SpringApplication.run(ProductApplication.class, args);
    }
}
```

controller代码：

```java
@RequestMapping("/product")
@RestController
public class ProductController {

    @Autowired
    private CartFeignClient cartFeignClient; //调用cart服务的feign client

    @HystrixCommand(fallbackMethod = "getDefaultValue") //指定出现异常的默认回退方法
    @PostMapping("/toCart/{productId}")
    public ResponseEntity addCart(@PathVariable("productId") Long productId) throws InterruptedException {
        Thread.sleep(5000); //特意让线程休眠一段时间以触发熔断及服务降级
        Long aLong = cartFeignClient.addCart(productId);
        return ResponseEntity.ok(productId);
    }
	//默认降级方法
    private ResponseEntity getDefaultValue(Long productId) {
        return ResponseEntity.ok(0);
    }
}
```

cart服务的feign client代码：

```java
@FeignClient(value = "CART") //指定服务名称
public interface CartFeignClient {

    @PostMapping("/cart/{productId}") //指定接口路径
    Long addCart(@PathVariable("productId")Long productId); //参数
}
```

cart服务没什么特殊配置及代码，只要注册到eureka以及提供接口就可以了。

服务配置：

```yaml
server:
  port: 8081

eureka:
  client:
    service-url:
      defaultZone: http://localhost:8761/eureka
  instance:
    status-page-url-path: /info
    health-check-url-path: /health

spring:
  application:
    name: cart
  profiles:
    active: ${boot.profile:dev}
```

启动类代码：

```java
@EnableDiscoveryClient
@SpringBootApplication
public class CartApplication {

    public static void main(String[] args) {
        SpringApplication.run(CartApplication.class, args);
    }

}
```

接口：

```java
@RestController
@RequestMapping("/cart")
@Api(value = "购物车", tags = {"购物车"}) //swagger
public class CartControllrr {

    @ApiOperation("添加商品到购物车") //swagger 
    @ApiImplicitParam(name = "productId",value = "productId",required = true,paramType ="path",dataType = "String")
    @PostMapping("/{productId}")
    public ResponseEntity addCart(@PathVariable("productId") Long productId) {
        System.out.println(productId);
        return ResponseEntity.ok(productId);
    }
}
```

现在将服务全部启动，并通过zuul调用接口即可。

原先直接调用product接口地址为：http://localhost:8082/product/toCart/1

经过zuul后的地址为：http://localhost:8080/api/product/product/toCart/1

正常调用结果返回为1，

由于product处理代码中加入了sleep，导致product服务调用超时，触发hystrix熔断及服务降级，调用了指定的回退方法，getDefaultValue，返回了0.

![image-20190121105728184](/Users/wangwangxiaoteng/Library/Application Support/typora-user-images/image-20190121105728184.png)

这里需要注意的有几点：

1. 通过网关zuul调用时，zuul的超时时间要大于等于hystrix的超时时间配置，否则在zuul层转发时就已经触发了zuul的超时，返回 GATEWAY TIME OUT

2. 无重试机制时，通过feign加ribbon进行服务之间调用时，hystrix配置超时时间要小于ribbon超时时间，否则在ribbon调用其他服务时就已经超时了，hystrix无法进行熔断及降级

3. 如果有重试时，如有组件跟Hystrix配合使用，一般来讲，建议Hystrix的超时 > 其他组件的超时，否则将可能导致重试特性失效。例如，如果ribbon超时时间为1秒，重试3次，hystrix超时时间应略大于3秒。

4. 定义一个fallback方法需要注意以下几点：

   - fallback方法必须和指定fallback方法的主方法在一个类中。

   - fallback方法的参数必须要和主方法的参数一致，否则不生效。

   - 使用fallback方法需要根据依赖服务设置合理的超时时间，即execution.isolation.thread.timeoutInMilliseconds的设置，可以在@HystrixCommand注解上通过HystrixProperty指定。

5. 如果要在fallback方法中获取异常信息，只需要在fallback方法中，加上一个参数Throwable throwable就可以了。

GitHub上Netflix给的例子是通过继承HystrixCommand或者HystrixObservableCommand，然后实现execute()或者queue()方法来运行，但是这种方式代码侵入性较大。使用注解@HystrixCommand方式的侵入性小一点。



刚才的hystrix的服务降级示例是同步模式的，也是通常情况下我们所使用的模式。

- 同步command，同步fallback

```java
@HystrixCommand(fallbackMethod = "getDefaultValue")
@PostMapping("/toCart/{productId}")
public ResponseEntity addCart(@PathVariable("productId") Long productId) throws InterruptedException {
    Long aLong = cartFeignClient.addCart(productId);
    System.out.println(aLong);
    return ResponseEntity.ok(productId);
}

private ResponseEntity getDefaultValue(Long productId) {
    return ResponseEntity.ok(0);
}
```

- 异步command，同步fallback

```java
@HystrixCommand(fallbackMethod = "getDefaultAsyncAddCart")
public Future<ResponseEntity<Long>> asyncAddCart(@PathVariable("productId") Long productId) throws InterruptedException {
    log.info("异步command：run。。。");
    Thread.sleep(5000);//触发降级逻辑
    return new AsyncResult<ResponseEntity<Long>>() {
        @Override
        public ResponseEntity<Long> invoke() {
            return ResponseEntity.ok(cartFeignClient.addCart(productId));
        }
    };
}

private ResponseEntity<Long> getDefaultAsyncAddCart(Long productId, Throwable throwable) {
    log.info("异步command，同步fallback：run。。。");
    return ResponseEntity.ok(0L);
}
```

- 异步command，异步fallback

```java
@HystrixCommand(fallbackMethod = "getDefaultAsyncAddCart2")
public Future<ResponseEntity<Long>> asyncAddCart2(@PathVariable("productId") Long productId) throws InterruptedException {
    log.info("异步command：run。。。");
    Thread.sleep(5000);
    return new AsyncResult<ResponseEntity<Long>>() {
        @Override
        public ResponseEntity<Long> invoke() {
            return ResponseEntity.ok(cartFeignClient.addCart(productId));
        }
    };
}

@HystrixCommand //注意，异步fallback 这里必须加@HystixCommand注解，否则运行时报错
private Future<ResponseEntity<Long>> getDefaultAsyncAddCart2(Long productId, Throwable throwable) {
    log.info("异步command，同步fallback：run。。。");
    log.warn("", throwable);
    return new AsyncResult<ResponseEntity<Long>>() {
        @Override
        public ResponseEntity<Long> invoke() {
            return ResponseEntity.ok(0L);
        }
    };
}
```

注意，hystrix不支持同步command，异步fallback。

#### hystrix配置

我们先看下@HystrixCommand注解中有哪些配置。

| 属性                    | 类型                    | 描述                                                         |
| ----------------------- | ----------------------- | ------------------------------------------------------------ |
| groupKey                | String                  | 用于报告、告警、大盘展示时的分组key                          |
| commandKey              | String                  | hystrix 命令的key值，默认是方法名                            |
| threadPoolKey           | String                  | 用于区分不同线程池的key值，hystrix线程池是用来监控，缓存，避免个别服务出现异常导致拖累所有线程都被占用的key值。 |
| fallbackMethod          | String                  | 指定回退/降级的方法，此方法必须要定义在相同的类中，参数也应该相同 |
| commandProperties       | HystrixProperty数组     | 指定hystrix command 属性值                                   |
| threadPoolProperties    | HystrixProperty数组     | 指定hystrix线程池的属性值                                    |
| ignoreExceptions        | Throwable及子类         | 定义应该忽略的异常                                           |
| observableExecutionMode | ObservableExecutionMode | 指定hystrix用于执行观察者命令的模式，默认饥饿加载            |

其他配置：详情见https://github.com/Netflix/Hystrix/wiki/Configuration

@HystixCommand属性配置官方示例：https://github.com/Netflix/Hystrix/tree/master/hystrix-contrib/hystrix-javanica

命令属性配置：

执行时：

| 属性                                                | 默认   | 描述                                                         | 全局配置                                                     |
| --------------------------------------------------- | ------ | ------------------------------------------------------------ | ------------------------------------------------------------ |
| execution.isolation.strategy                        | THREAD | 执行hystrix命令模式时的隔离模式，默认是THREAD。有两种选项，THREAD线程隔离，SEMAPHORE信号量隔离。THREAD模式会在有限线程的线程池内选择一个单独的线程执行，SEMAPHORE是直接在调用线程上执行，并发请求受信号量计数的限制 | hystrix.command.default.execution.isolation.strategy         |
| execution.isolation.thread.timeoutInMilliseconds    | 1000   | 设置执行hystrix命令包裹方法的超时时间，超过这个时间则执行回退逻辑 | hystrix.command.default.execution.isolation.thread.timeoutInMilliseconds |
| execution.timeout.enabled                           | true   | 设置是否开启执行hystrix命令包裹方法的超时。                  | hystrix.command.default.execution.timeout.enabled            |
| execution.isolation.thread.interruptOnTimeout       | true   | 此属性设置`HystrixCommand.run()`在发生超时时是否应中断执行。 | hystrix.command.default.execution.isolation.thread.interruptOnTimeout |
| execution.isolation.thread.interruptOnCancel        | false  | 此属性设置`HystrixCommand.run()`在发生取消时是否应中断执行。 | hystrix.command.default.execution.isolation.thread.interruptOnCancel |
| execution.isolation.semaphore.maxConcurrentRequests | 10     | 当使用SEMAPHORE信号量模式时，设置允许HystrixCommand.run()同时并发执行的最大请求数。达到最大值时，后面的请求将被拒绝执行，并执行回退逻辑，如果没有回退方法，则会抛出异常。生产环境可根据实际情况调整 | hystrix.command.default.fallback.isolation.semaphore.maxConcurrentRequests |

回退时：

| 属性                                               | 默认值 | 描述                                                         | 全局配置                                                     |
| -------------------------------------------------- | ------ | ------------------------------------------------------------ | ------------------------------------------------------------ |
| fallback.isolation.semaphore.maxConcurrentRequests | 10     | 设置允许回退方法同时执行的最大并发数，超过这个值，则将拒绝后续请求并抛出异常（无第二级回退时） | hystrix.command.default.fallback.isolation.semaphore.maxConcurrentRequests |
| fallback.enabled                                   | true   | 是否执行回退                                                 | hystrix.command.default.fallback.enabled                     |

断路器配置：

| 属性                                     | 默认值 | 描述                                                         | 全局配置                                                     |
| ---------------------------------------- | ------ | ------------------------------------------------------------ | ------------------------------------------------------------ |
| circuitBreaker.enabled                   | true   | 断路器开关，此配置决定断路器是否用于跟踪运行状况，以及在其跳闸时是否用于短路请求。 | hystrix.command.default.circuitBreaker.enabled               |
| circuitBreaker.requestVolumeThreshold    | 20     | 设置用于触发跳闸的滚动窗口的最小失败请求数的阈值             | hystrix.command.default.circuitBreaker.requestVolumeThreshold |
| circuitBreaker.sleepWindowInMilliseconds | 5000   | 设置触发断路器后允许再次执行请求前，拒绝请求的时间，单位为毫秒 | hystrix.command.default.circuitBreaker.sleepWindowInMilliseconds |
| circuitBreaker.errorThresholdPercentage  | 50     | 设置触发断路器，走降级逻辑的默认百分比最小阈值               | hystrix.command.default.circuitBreaker.errorThresholdPercentage |
| circuitBreaker.forceOpen                 | false  | 是否强制进入断路器状态，进入该状态将拒绝所有请求             | hystrix.command.default.circuitBreaker.forceOpen             |
| circuitBreaker.forceClosed               | false  | 是否强制关闭断路器状态，关闭后，将允许所有请求进入           | hystrix.command.default.circuitBreaker.forceClosed           |

Metrics：

| 属性名                                                       | 默认值 | 描述                                                         | 全局配置                                                     |
| ------------------------------------------------------------ | ------ | ------------------------------------------------------------ | ------------------------------------------------------------ |
| metrics.rollingStats.timeInMilliseconds                      | 10000  | 设置统计滚动窗口失败请求次数的统计时间                       | hystrix.command.default.metrics.rollingStats.timeInMilliseconds |
| metrics.rollingStats.numBuckets                              | 10     | 此属性设置滚动统计窗口划分的存储桶数。必须能被metrics.rollingStats.timeInMilliseconds整除，否则抛出异常 | hystrix.command.default.metrics.rollingStats.numBuckets      |
| metrics.rollingPercentile.enabled                            | true   | 配置是否应跟踪执行延迟并将其计算为百分位数。                 | hystrix.command.default.metrics.rollingPercentile.enabled    |
| metrics.rollingPercentile.timeInMilliseconds                 | 60000  | 设置滚动窗口的持续时间，其中保持执行时间以允许百分位计算，以毫秒为单位。 | hystrix.command.default.metrics.rollingPercentile.timeInMilliseconds |
| metrics.rollingPercentile.numBucketsmetrics.rollingPercentile.numBuckets | 6      | 设置rollingPercentile窗口将分成的桶数。必须能被metrics.rollingPercentile.timeInMilliseconds整除，否则抛出异常 | hystrix.command.default.metrics.rollingPercentile.numBuckets |
| metrics.rollingPercentile.bucketSize                         | 100    | 设置每个存储桶保留的最大执行次数                             | hystrix.command.default.metrics.rollingPercentile.bucketSize |
| metrics.healthSnapshot.intervalInMilliseconds                | 500    | 设置允许执行计算成功和错误百分比的健康快照与影响断路器状态之间的等待时间（以毫秒为单位）。 | hystrix.command.default.metrics.healthSnapshot.intervalInMilliseconds |

Request Context：

| 属性                 | 默认值 | 描述                                                         | 全局配置                                     |
| -------------------- | ------ | ------------------------------------------------------------ | -------------------------------------------- |
| requestCache.enabled | true   | 请求缓存的开关，开启之后，hystrix的cacheKey会被缓存掉，当同一个请求来时，使用缓存的内容 | hystrix.command.default.requestCache.enabled |
| requestLog.enabled   | true   | 是否打印请求的log                                            | hystrix.command.default.requestLog.enabled   |

Collapser Properties：

| 属性                     | 默认值            | 描述                               | 全局配置                                           |
| ------------------------ | ----------------- | ---------------------------------- | -------------------------------------------------- |
| maxRequestsInBatch       | Integer.MAX_VALUE | 设置在批处理之前允许的最大请求数   | hystrix.collapser.default.maxRequestsInBatch       |
| timerDelayInMilliseconds | 10                | 设置在批处理创建完后多少毫秒后执行 | hystrix.collapser.default.timerDelayInMilliseconds |
| requestCache.enabled     | true              | 是否开启请求缓存                   | hystrix.collapser.default.requestCache.enabled     |

Thread Properties：

配置hystrix线程池的属性

![](https://raw.githubusercontent.com/wiki/Netflix/Hystrix/images/thread-configuration-1280.png)

| 属性                                    | 默认值 | 描述                                                         | 全局配置                                                     |
| --------------------------------------- | ------ | ------------------------------------------------------------ | ------------------------------------------------------------ |
| coreSize                                | 10     | 设置hystrix线程池核心线程池大小                              | hystrix.threadpool.default.coreSize                          |
| maximumSize                             | 10     | 设置hystrix线程池最大线程池大小                              | hystrix.threadpool.default.maximumSize                       |
| maxQueueSize                            | −1     | 设置hystrix线程池的最大队列大小                              | hystrix.threadpool.default.maxQueueSize                      |
| queueSizeRejectionThreshold             | 5      | 设置队列拒绝阈值，即使未达到maxQueueSize也会发生拒绝的最大队列大小，此属性的存在是因为无法动态更改maxQueueSize，我们希望允许您动态更改影响拒绝的队列大小。当maxQueueSize为-1时，此配置不生效 | hystrix.threadpool.default.queueSizeRejectionThreshold       |
| keepAliveTimeMinutes                    | 1      | 设置线程池线程释放之前保持活跃状态的时间，单位：分钟         | hystrix.threadpool.default.keepAliveTimeMinutes              |
| allowMaximumSizeToDivergeFromCoreSize   | false  |                                                              | hystrix.threadpool.default.allowMaximumSizeToDivergeFromCoreSize |
| metrics.rollingStats.timeInMilliseconds | 10000  | 设置统计滚动窗口的持续时间                                   | hystrix.threadpool.default.metrics.rollingStats.timeInMilliseconds |
| metrics.rollingStats.numBuckets         | 10     | 设置滚动统计窗口分为的桶数。                                 | hystrix.threadpool.default.metrics.rollingStats.numBuckets   |

### 服务容错保护（Hystrix依赖隔离）

hystrix为每一个命令创建一个独立的线程池，这样就算某个hystrix命令由于依赖其他服务导致出现延迟过高的情况，也只是对该依赖服务的调用产生影响，而不会拖累其他服务。

通过对依赖服务的线程池隔离实现，可以带来如下优势：

- 应用自身得到完全的保护，不会受不可控的依赖服务影响。即便给依赖服务分配的线程池被填满，也不会影响应用自身的额其余部分。
- 可以有效的降低接入新服务的风险。如果新服务接入后运行不稳定或存在问题，完全不会影响到应用其他的请求。
- 当依赖的服务从失效恢复正常后，它的线程池会被清理并且能够马上恢复健康的服务，相比之下容器级别的清理恢复速度要慢得多。
- 当依赖的服务出现配置错误的时候，线程池会快速的反应出此问题（通过失败次数、延迟、超时、拒绝等指标的增加情况）。同时，我们可以在不影响应用功能的情况下通过实时的动态属性刷新（后续会通过Spring Cloud Config与Spring Cloud Bus的联合使用来介绍）来处理它。
- 当依赖的服务因实现机制调整等原因造成其性能出现很大变化的时候，此时线程池的监控指标信息会反映出这样的变化。同时，我们也可以通过实时动态刷新自身应用对依赖服务的阈值进行调整以适应依赖方的改变。
- 除了上面通过线程池隔离服务发挥的优点之外，每个专有线程池都提供了内置的并发实现，可以利用它为同步的依赖服务构建异步的访问。

总之，通过对依赖服务实现线程池隔离，让我们的应用更加健壮，不会因为个别依赖服务出现问题而引起非相关服务的异常。同时，也使得我们的应用变得更加灵活，可以在不停止服务的情况下，配合动态配置刷新实现性能配置上的调整。

## 原理及设计

![](/Users/wangwangxiaoteng/work/code/github/blogs/hystrix-command-flow-chart.png)

通过翻看hystrix-javanica的ReadMe以及查看注解@HystrixCommand的引用可以发现，HystrixCommandAspect类是一个很关键的类。

接下来我们从HystrixCommandAspect来入手。

```java
@Aspect
public class HystrixCommandAspect {

    private static final Map<HystrixPointcutType, MetaHolderFactory> META_HOLDER_FACTORY_MAP;

    static {
        META_HOLDER_FACTORY_MAP = ImmutableMap.<HystrixPointcutType, MetaHolderFactory>builder()
                .put(HystrixPointcutType.COMMAND, new CommandMetaHolderFactory())
                .put(HystrixPointcutType.COLLAPSER, new CollapserMetaHolderFactory())
                .build();
    }

//指定切点为@HystrixCommand注解 @Pointcut("@annotation(com.netflix.hystrix.contrib.javanica.annotation.HystrixCommand)")

    public void hystrixCommandAnnotationPointcut() {
    }

    @Pointcut("@annotation(com.netflix.hystrix.contrib.javanica.annotation.HystrixCollapser)")
    public void hystrixCollapserAnnotationPointcut() {
    }
	//切面
    @Around("hystrixCommandAnnotationPointcut() || hystrixCollapserAnnotationPointcut()")
    public Object methodsAnnotatedWithHystrixCommand(final ProceedingJoinPoint joinPoint) throws Throwable {
        //先获取到执行的方法数据
        Method method = getMethodFromTarget(joinPoint);
        Validate.notNull(method, "failed to get method from joinPoint: %s", joinPoint);
        //...
        //根据注解类型获取不同methodHolder的构造工厂，methodHolder用来保存方法的一些数据，如注解以及注解的属性值等
        MetaHolderFactory metaHolderFactory = META_HOLDER_FACTORY_MAP.get(HystrixPointcutType.of(method));
        //根据method创建一个methodHolder
        MetaHolder metaHolder = metaHolderFactory.create(joinPoint);
        //构造hystrixCommand对象
        HystrixInvokable invokable = HystrixCommandFactory.getInstance().create(metaHolder);
        //获取方法执行类型，同步、异步还是流式。
        ExecutionType executionType = metaHolder.isCollapserAnnotationPresent() ?
                metaHolder.getCollapserExecutionType() : metaHolder.getExecutionType();

        Object result;
        try {
            if (!metaHolder.isObservable()) {
                //非流式，执行命令
                result = CommandExecutor.execute(invokable, executionType, metaHolder);
            } else {
                //流式，执行命令
                result = executeObservable(invokable, executionType, metaHolder);
            }
        } catch (HystrixBadRequestException e) {
            throw e.getCause() != null ? e.getCause() : e;
        } catch (HystrixRuntimeException e) {
            throw hystrixRuntimeExceptionToThrowable(metaHolder, e);
        }
        return result;
    }
}    
```

非流式命令执行代码：

CommandExecutor.execute()。

```java
public static Object execute(HystrixInvokable invokable, ExecutionType executionType, MetaHolder metaHolder) throws RuntimeException {
    //......
    switch (executionType) {
        case SYNCHRONOUS: {
            //同步模式执行command
            return castToExecutable(invokable, executionType).execute();
        }
        case ASYNCHRONOUS: {
            //异步模式执行command
            HystrixExecutable executable = castToExecutable(invokable, executionType);
            if (metaHolder.hasFallbackMethodCommand()
                    && ExecutionType.ASYNCHRONOUS == metaHolder.getFallbackExecutionType()) {
                return new FutureDecorator(executable.queue());
            }
            return executable.queue();
        }
        case OBSERVABLE: {
            //流式模式执行command
            HystrixObservable observable = castToObservable(invokable);
            return ObservableExecutionMode.EAGER == metaHolder.getObservableExecutionMode() ? observable.observe() : observable.toObservable();
        }
        default:
            throw new RuntimeException("unsupported execution type: " + executionType);
    }
}

private static HystrixExecutable castToExecutable(HystrixInvokable invokable, ExecutionType executionType) {
    	//转换
        if (invokable instanceof HystrixExecutable) {
            return (HystrixExecutable) invokable;
        }
        throw new RuntimeException("Command should implement " + HystrixExecutable.class.getCanonicalName() + " interface to execute in: " + executionType + " mode");
    }

    private static HystrixObservable castToObservable(HystrixInvokable invokable) {
        //转换
        if (invokable instanceof HystrixObservable) {
            return (HystrixObservable) invokable;
        }
        throw new RuntimeException("Command should implement " + HystrixObservable.class.getCanonicalName() + " interface to execute in observable mode");
    }
```

同步执行的execute()方法有两个实现类，分别是HystrixCommand以及HystrixCollapser。

通常是HystrixCommand，只有在指定合并多个请求时才会是HystrixCollapser。

我们先看下HystrixCommand的execute()方法。

```java
public R execute() {
    try {
        //直接调用queue()方法获得结果Future后再调用get()方法。
        return queue().get();
    } catch (Exception e) {
        //重新抛出异常
        throw Exceptions.sneakyThrow(decomposeException(e));
    }
}

public Future<R> queue() {
        /*
         * The Future returned by Observable.toBlocking().toFuture() does not implement the
         * interruption of the execution thread when the "mayInterrupt" flag of Future.cancel(boolean) is set to true;
         * thus, to comply with the contract of Future, we must wrap around it.
         */
        final Future<R> delegate = toObservable().toBlocking().toFuture();
    	
        final Future<R> f = new Future<R>() {

            //...
        	
        };

        /* special handling of error states that throw immediately */
        if (f.isDone()) {
            try {
                f.get();
                return f;
            } catch (Exception e) {
                //...处理异常
            }
        }

        return f;
    }
```

## Hystrix还有哪些待改进的地方？

待完善。

## 有没有更好的解决方式

待完善。