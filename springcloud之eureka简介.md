## 简介

服务发现与治理是微服务体系结构的重要组成之一，在生产环境下由于服务器较多，手动配置很难完成，且一旦变化，如弹性扩容与下线，就会变得很复杂。eureka是Netflix开源的服务发现与治理框架，在Netflix等公司历经实战，拥有极高的可用性。

## 服务端使用eureka

### 搭建eureka服务

在springboot项目中，引入eureka-server的依赖group ID 为 `org.springframework.cloud` ， artifact ID 为 `spring-cloud-starter-netflix-eureka-server`。

```java
@SpringBootApplication
@EnableEurekaServer
public class Application {

    public static void main(String[] args) {
        new SpringApplicationBuilder(Application.class).web(true).run(args);
    }

}
```

### 高可用性

eureka没有后端存储，但是每个eureka客户端都要向eureka服务端注册自己的实例，并获得其他实例信息，所以，eureka客户端其他实例信息保存在了本地内存中，eureka微服务客户端彼此之间通信不必每次都先调用eureka server端，默认，每30秒，客户端去服务端发送一次心跳，并跟新注册信息。

### 多个eureka server实例

生产情况下，eureka server是有多个的，eureka server本身也是eureka client，每一个eureka server需要向其他eureka server注册自己，配置示例如下：

`peer1的配置`

```yaml
---
spring:
  profiles: peer1
eureka:
  instance:
    hostname: peer1
  client:
    serviceUrl:
      defaultZone: http://peer2/eureka/
```

`peer2的配置`

```yaml
spring:
  profiles: peer2
eureka:
  instance:
    hostname: peer2
  client:
    serviceUrl:
      defaultZone: http://peer1/eureka/
```

在本地开发时，eureka客户端最好不要注册到eureka服务端，以免其他人在开发时，其他微服务负载到你本机启动的实例中，eureka支持不注册到eureka server。

```yaml
eureka:
  client:
    registerWithEureka: false
    fetchRegistry: false
    serviceUrl:
      defaultZone: http://peer1/eureka/
```

### eureka server的CSRF支持

eureka 需要通过spring security来防范CSRF攻击，引入spring-security依赖spring-boot-starter-security，再加上Bean配置。

```java
@EnableWebSecurity
class WebSecurityConfig extends WebSecurityConfigurerAdapter {

    @Override
    protected void configure(HttpSecurity http) throws Exception {
        http.csrf().ignoringAntMatchers("/eureka/**");
        super.configure(http);
    }
}
```

### eureka自我保护模式

eureka客户端向服务端注册，并默认情况下每30秒发送一次心跳消息以向eureka server说明自己的实例运行正常，也称之为服务续约。当某个实例在连续3次未能注册心跳消息时，eureka会将该实例标注为“DOWN”，如果在短时间内有超过15%的服务down掉，eureka认为可能是网络原因，其他服务未能向自己注册，而并不是服务已经down掉。因此，eureka会默认启动自我保护模式，不会下线这些实例。

但是当我们本地开发时，由于服务本身就很少，很容易就能达到阈值，因此有必要关掉自我保护模式，关闭的配置如下：eureka.enableSelfPreservation=false

## 客户端使用eureka-client

### 客户端引入eureka-client

项目中引入eureka-client，使用maven或gradle引入eureka-client依赖。group ID 是 `org.springframework.cloud` ， artifact ID 为 `spring-cloud-starter-netflix-eureka-client`.

### 注册到eureka服务端

当一个客户端注册到eureka服务端时，它会将自身的IP、端口、主页等其他信息提供给eureka服务端。eureka服务端接收客户端的心跳消息，当心跳故障超过一定时间（可配置），通常会将该实例从已注册实例中移除。

以下示例显示了最小的Eureka客户端应用程序：

```java
@SpringBootApplication
@RestController
public class Application {

    @RequestMapping("/")
    public String home() {
        return "Hello world";
    }

    public static void main(String[] args) {
        new SpringApplicationBuilder(Application.class).web(true).run(args);
    }

}
```

客户端需增加以下配置：这里以yml文件为例

```yaml
eureka:
  client:
    serviceUrl:
      defaultZone: http://localhost:8761/eureka/
```

在上面的示例中，defaultZone是一个魔法值，上面的值是它默认的URL。eureka客户端的配置通过eureka.instance.*来指定，spring.application.name要配置。

如果要禁用eureka，可以通过设置`eureka.client.enabled`为`false`

### 使用eureka server进行服务认证

如果其中一个`eureka.client.serviceUrl.defaultZone`URL中嵌入了认证，则会自动将HTTP基本身份验证添加到您的eureka客户端（如下所示:) `http://user:password@localhost:8761/eureka`。对于更复杂的需求，可以创建一个`@Bean`类型`DiscoveryClientOptionalArgs`并将`ClientFilter`实例注入其中，所有这些实例都应用于从客户端到服务器的调用。

### 状态页面和健康指标页面

eureka的状态页面和健康指标页面分别默认为/info`和`/health，如果使用非默认上下文路径或servlet路径（例如`server.servletPath=/custom`），则需要更改这些，即使对于Actuator应用程序也是如此。以下示例显示了这两个设置的默认值：

```yaml
eureka:
  instance:
    statusPageUrlPath: ${server.servletPath}/info
    healthCheckUrlPath: ${server.servletPath}/health
```

### eureka的健康检查

默认情况下，eureka是通过客户端发送的心跳消息来判断客户端是否处于运行状态。通常，eureka客户端不会根据Spring Boot Actuator来传播客户端的当前运行状况检查状态。因此在成功注册后，客户端在eureka那里始终处于“UP”状态。如果想将客户端状态传播至eureka，那么可以启用eureka的健康检查，这样其他应用程序都不会向“UP”以外的状态下的应用程序发送流量。

**application.yml.** 

```yaml
eureka:
  client:
    healthcheck:
      enabled: true
```

注意，这里是application.yml，不是bootstrap.yml，如果在bootstrap.yml中配置了此配置，可能会发生一些意向不到的情况，如eureka那里显示应用程序为“UNKNOWN”，因为bootstrap.yml的配置优先级较高。

如果想要更为复杂的健康检查，就要实现自己的健康检查类，`com.netflix.appinfo.HealthCheckHandler`

### 使用EurekaClient

在注册到eureka服务后，微服务之间调用通过负载均衡调用时，可以通过如下方式来获取服务的URL：

```java
@Autowired
private EurekaClient discoveryClient;

public String serviceUrl() {
    InstanceInfo instance = discoveryClient.getNextServerFromEureka("STORES", false);
    return instance.getHomePageUrl();
}
```

注意：不要在`@PostConstruct`方法或`@Scheduled`方法中使用（或者`ApplicationContext`可能尚未启动的任何地方）。

### 不使用jersey

默认情况下，eurekaClient是通过Jersey进行http通信的，如果不想使用Jersey，可以通过下面的方式去除Jersey的依赖，以及使用restTemplate进行http通信。

```
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
    <exclusions>
        <exclusion>
            <groupId>com.sun.jersey</groupId>
            <artifactId>jersey-client</artifactId>
        </exclusion>
        <exclusion>
            <groupId>com.sun.jersey</groupId>
            <artifactId>jersey-core</artifactId>
        </exclusion>
        <exclusion>
            <groupId>com.sun.jersey.contribs</groupId>
            <artifactId>jersey-apache-client4</artifactId>
        </exclusion>
    </exclusions>
</dependency>
```

```java
@EnableDiscoveryClient
@SpringBootApplication
public class CartApplication {

    public static void main(String[] args) {
        SpringApplication.run(CartApplication.class, args);
    }
    
    @Bean
    @LoadBalanced
    public RestTemplate restTemplate(){
        return new RestTemplate();
    }
}
```

### 为什么注册服务这么慢

在使用eureka的时候吗，有时候可能会发现明明eureka客户端已经注册到eureka服务上了，可是其他的客户端却找不到该服务。原因是每个eureka客户端在本地都会有一份eureka实例的缓存，默认情况下，每30秒客户端会向服务端发送一次心跳检查，并更新服务实例信息。如果想要加快速度，可以通过eureka.instance.leaseRenewalIntervalInSeconds 来指定时间。

#### zone区域

在生产环境下，我们可能会将eureka客户端及服务端部署到不同的区域，如华北，华南区域，相同区域的服务之间相互调用网络延迟会相对较少一些。eureka中可以通过配置，指定实例的zone区域。

如：

一区

```properties
eureka.instance.metadataMap.zone = zone1
eureka.client.preferSameZoneEureka = true
```

二区

```properties
eureka.instance.metadataMap.zone = zone2
eureka.client.preferSameZoneEureka = true
```
