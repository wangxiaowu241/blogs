## Feign的目标

feign是声明式的web service客户端，它让微服务之间的调用变得更简单了，类似controller调用service。Spring Cloud集成了Ribbon和Eureka，可在使用Feign时提供负载均衡的http客户端。

## 引入Feign 

项目中使用了gradle作为依赖管理，maven类似。

```groovy
dependencies {
    //feign
    implementation('org.springframework.cloud:spring-cloud-starter-openfeign:2.0.2.RELEASE')
	//web
    implementation('org.springframework.boot:spring-boot-starter-web')
    //eureka client
    implementation('org.springframework.cloud:spring-cloud-starter-netflix-eureka-client:2.1.0.M1')
	//test
    testImplementation('org.springframework.boot:spring-boot-starter-test')
}
```

因为feign底层是使用了ribbon作为负载均衡的客户端，而ribbon的负载均衡也是依赖于eureka 获得各个服务的地址，所以要引入eureka-client。

SpringbootApplication启动类加上@FeignClient注解，以及@EnableDiscoveryClient。

```java
@EnableFeignClients
@EnableDiscoveryClient
@SpringBootApplication
public class ProductApplication {

    public static void main(String[] args) {
        SpringApplication.run(ProductApplication.class, args);
    }
}
```

yaml配置：

```yaml
server:
  port: 8082

#配置eureka
eureka:
  client:
    service-url:
      defaultZone: http://localhost:8761/eureka
  instance:
    status-page-url-path: /info
    health-check-url-path: /health

#服务名称
spring:
  application:
    name: product
  profiles:
    active: ${boot.profile:dev}
#feign的配置，连接超时及读取超时配置
feign:
  client:
    config:
      default:
        connectTimeout: 5000
        readTimeout: 5000
        loggerLevel: basic
```

## Feign的使用

```java
@FeignClient(value = "CART")
public interface CartFeignClient {

    @PostMapping("/cart/{productId}")
    Long addCart(@PathVariable("productId")Long productId);
}
```

上面是最简单的feign client的使用，声明完为feign client后，其他spring管理的类，如service就可以直接注入使用了，例如：

```java
//这里直接注入feign client
@Autowired
private CartFeignClient cartFeignClient;

@PostMapping("/toCart/{productId}")
public ResponseEntity addCart(@PathVariable("productId") Long productId){
    Long result = cartFeignClient.addCart(productId);
    return ResponseEntity.ok(result);
}
```

可以看到，使用feign之后，我们调用eureka 注册的其他服务，在代码中就像各个service之间相互调用那么简单。

## FeignClient注解的一些属性

| 属性名        | 默认值     | 作用                                                         | 备注                                        |
| ------------- | ---------- | ------------------------------------------------------------ | ------------------------------------------- |
| value         | 空字符串   | 调用服务名称，和name属性相同                                 |                                             |
| serviceId     | 空字符串   | 服务id，作用和name属性相同                                   | 已过期                                      |
| name          | 空字符串   | 调用服务名称，和value属性相同                                |                                             |
| url           | 空字符串   | 全路径地址或hostname，http或https可选                        |                                             |
| decode404     | false      | 配置响应状态码为404时是否应该抛出FeignExceptions             |                                             |
| configuration | {}         | 自定义当前feign client的一些配置                             | 参考FeignClientsConfiguration               |
| fallback      | void.class | 熔断机制，调用失败时，走的一些回退方法，可以用来抛出异常或给出默认返回数据。 | 底层依赖hystrix，启动类要加上@EnableHystrix |
| path          | 空字符串   | 自动给所有方法的requestMapping前加上前缀，类似与controller类上的requestMapping |                                             |
| primary       | true       |                                                              |                                             |

此外，还有qualifier及fallbackFactory，这里就不再赘述。

## Feign自定义处理返回的异常

这里贴上GitHub上openFeign的wiki给出的自定义errorDecoder例子。

```java
public class StashErrorDecoder implements ErrorDecoder {

    @Override
    public Exception decode(String methodKey, Response response) {
        if (response.status() >= 400 && response.status() <= 499) {
            //这里是给出的自定义异常
            return new StashClientException(
                    response.status(),
                    response.reason()
            );
        }
        if (response.status() >= 500 && response.status() <= 599) {
            //这里是给出的自定义异常
            return new StashServerException(
                    response.status(),
                    response.reason()
            );
        }
        //这里是其他状态码处理方法
        return errorStatus(methodKey, response);
    }
}
```

自定义好异常处理类后，要在@Configuration修饰的配置类中声明此类。

## Feign使用OKhttp发送request

Feign底层默认是使用jdk中的HttpURLConnection发送HTTP请求，feign也提供了OKhttp来发送请求，具体配置如下：

```java
feign:
  client:
    config:
      default:
        connectTimeout: 5000
        readTimeout: 5000
        loggerLevel: basic
  okhttp:
    enabled: true
  hystrix:
    enabled: true
```

## Feign原理简述

- 启动时，程序会进行包扫描，扫描所有包下所有@FeignClient注解的类，并将这些类注入到spring的IOC容器中。当定义的Feign中的接口被调用时，通过JDK的动态代理来生成RequestTemplate。
- RequestTemplate中包含请求的所有信息，如请求参数，请求URL等。
- RequestTemplate声场Request，然后将Request交给client处理，这个client默认是JDK的HTTPUrlConnection，也可以是OKhttp、Apache的HTTPClient等。
- 最后client封装成LoadBaLanceClient，结合ribbon负载均衡地发起调用。

详细原理请参考源码解析。

Feign、hystrix与retry的关系请参考https://xli1224.github.io/2017/09/22/configure-feign/

## Feign开启GZIP压缩

Spring Cloud Feign支持对请求和响应进行GZIP压缩，以提高通信效率。

application.yml配置信息如下：

```yaml
feign:
  compression:
    request: #请求
      enabled: true #开启
      mime-types: text/xml,application/xml,application/json #开启支持压缩的MIME TYPE
      min-request-size: 2048 #配置压缩数据大小的下限
    response: #响应
      enabled: true #开启响应GZIP压缩
```

注意：

由于开启GZIP压缩之后，Feign之间的调用数据通过二进制协议进行传输，返回值需要修改为ResponseEntity<byte[]>才可以正常显示，否则会导致服务之间的调用乱码。

示例如下：

```java
@PostMapping("/order/{productId}")
ResponseEntity<byte[]> addCart(@PathVariable("productId") Long productId);
```

## 作用在所有Feign Client上的配置方式

方式一：通过java bean 的方式指定。

@EnableFeignClients注解上有个defaultConfiguration属性，可以指定默认Feign Client的一些配置。

```java
@EnableFeignClients(defaultConfiguration = DefaultFeignConfiguration.class)
@EnableDiscoveryClient
@SpringBootApplication
@EnableCircuitBreaker
public class ProductApplication {

    public static void main(String[] args) {
        SpringApplication.run(ProductApplication.class, args);
    }
}
```

DefaultFeignConfiguration内容：

```java
@Configuration
public class DefaultFeignConfiguration {

    @Bean
    public Retryer feignRetryer() {
        return new Retryer.Default(1000,3000,3);
    }
}
```

方式二：通过配置文件方式指定。

```yaml
feign:
  client:
    config:
      default:
        connectTimeout: 5000 #连接超时
        readTimeout: 5000 #读取超时
        loggerLevel: basic #日志等级
```

## Feign Client开启日志

日志配置和上述配置相同，也有两种方式。

方式一：通过java bean的方式指定

```java
@Configuration
public class DefaultFeignConfiguration {
    @Bean
    public Logger.Level feignLoggerLevel(){
        return Logger.Level.BASIC;
    }
}
```

方式二：通过配置文件指定

```yaml
logging:
  level:
    com.xt.open.jmall.product.remote.feignclients.CartFeignClient: debug
```