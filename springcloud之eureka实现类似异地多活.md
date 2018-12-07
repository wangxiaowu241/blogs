## zone&region

![eureka_region_zone](https://raw.githubusercontent.com/wangxiaowu241/blogs/master/eureka_region_zone.png)

上图是eureka高可用架构，也是Netflix推荐的用法。

上图的us-east-1c、us-east-1d、us-east-1e各自是一个zone，每个zone内都有各自的eureka server & eureka client，就是说每个zone内都有服务注册中心及微服务的提供者和消费者。

那么zone和region在eureka中的概念是什么呢？

![AWS_regions](https://raw.githubusercontent.com/wangxiaowu241/blogs/master/AWS_regions.png)

- region和zone其实是来自于AWS亚马逊云的概念，AWS有很多大的区域，比如说亚太区，北美区，欧洲去
- region：区域，同一地理地区中的命名 AWS 资源集。一个区域包含至少两个可用区。

- zone：可用区。

AWS中，region有us-east-1、us-east-2等，分别表示为美国东部（弗吉尼亚北部）、美国东部（俄亥俄州）表示不同区域。

而us-east-1可能有us-east-1a、us-east-1b等不同的zone（可用区），类似我们的机房的概念。

## eureka中的zone的使用

eureka server -zone1

```yaml
server:
  port: 8761

eureka:
  instance:
    hostname: localhost
    metadata-map.zone: zone1
  client:
    region: cn-east
    service-url:
      defaultZone: http://localhost:8761/eureka,http://localhost:8762/eureka

spring:
  application:
    name: eureka
  profiles:
    active: ${boot.profile:dev}
```

eureka server -zone2

```yaml
server:
  port: 8762

eureka:
  instance:
    hostname: localhost
    metadata-map.zone: zone2
  client:
    region: cn-east
    service-url:
      defaultZone: http://localhost:8761/eureka,http://localhost:8762/eureka

spring:
  application:
    name: eureka
  profiles:
    active: ${boot.profile:dev}
```

Zuul-zone1

```yaml
spring:
  application:
    name: zuul
server:
  port: 8081

eureka:
  instance:
    metadata-map.zone: zone1
  client:
  	region: cn-east
    service-url:
      defaultZone: http://localhost:8761/eureka,http://localhost:8762/eureka
```

Zuul-zone2

```yaml
spring:
  application:
    name: zuul
server:
  port: 8082

eureka:
  instance:
    metadata-map.zone: zone2
  client:
  	region: cn-east
    service-url:
      defaultZone: http://localhost:8761/eureka,http://localhost:8762/eureka
```

cart-zone1

```yaml
server:
  port: 8091

eureka:
  client:
    service-url:
      defaultZone: http://localhost:8761/eureka,http://localhost:8762/eureka
    region: cn-east  
  instance:
    status-page-url-path: /info
    health-check-url-path: /health
    metadata-map.zone: zone1    

spring:
  application:
    name: cart
  profiles:
    active: ${boot.profile:dev}
```

Cart-zone2

```yaml
server:
  port: 8092

eureka:
  client:
    service-url:
      defaultZone: http://localhost:8761/eureka,http://localhost:8762/eureka
    region: cn-east  
  instance:
    status-page-url-path: /info
    health-check-url-path: /health
    metadata-map.zone: zone2    

spring:
  application:
    name: cart
  profiles:
    active: ${boot.profile:dev}
```

上面的配置概括来说就是，有三个应用，分别是eureka、zuul、cart。每个应用通过指定不同的profile启动，共启动六个微服务。zone1有eureka server、zuul、cart，zone2有eureka server、zuul、cart。

通过zuul来调用cart应用的controller，原cart应用zone1的接口地址为：POST http://localhost:8091/cart/1

通过zuul网关来访问，地址是POST   http://localhost:8081/cart/1.

持续通过访问zone1的zuul暴露出来的接口 http://localhost:8081/cart/1.发现始终路由到zone1的cart应用。

这时候，我们将zone1的cart应用下线，再次访问，报错，访问出错，过会再访问，路由到zone2的cart应用。

总结：

- 当一个region有多个zone是，微服务调用应用时优先调用同一个zone内的应用。原因是eureka有个配置prefer-same-zone-eureka，默认为true。
- 当同一个zone内的某个微服务下线时，其他微服务调用这个下线的应用，首先会报错，那是因为eureka server中还保留这个应用的注册信息（eureka client及server都有本地缓存），过一会再次访问，发现已经路由到同一个region的其他zone内的应用了。

## eureka中region的使用

目前eureka server提供配置remoteRegionUrlsWithName，key为region的名称，value为远程eureka server list。仅当本地服务不可用时，从远程（其他region）获取服务列表。
eureka client 也提供了fetchRemoteRegionsRegistry用于从远程获取，它的值为region list 用逗号分隔开。

使用以上两个配置可以实现类似于异地多活的功能。
