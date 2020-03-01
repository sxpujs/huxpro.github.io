---
layout: post
title: "02 Spring Cloud Netflix Eureka实现服务注册与发现"
author: "BJ大鹏"
header-style: text
tags:
  - Java
  - Spring Cloud
  - 微服务
---

Spring Cloud官网: <https://spring.io/projects/spring-cloud>

本篇主要讲[Spring Cloud Netflix](https://spring.io/projects/spring-cloud-netflix)中的Eureka，参考内容如下
- [Spring Cloud Netflix 2.2.1.RELEASE参考文档](https://cloud.spring.io/spring-cloud-static/spring-cloud-netflix/2.2.1.RELEASE/reference/html/)
- [Spring Cloud 系列之 Eureka 实现服务注册与发现](https://www.cnblogs.com/fengzheng/p/10603672.html)

文章内容会尽量参考官方文档。

## 1 注册中心（Eureka Server）
完整代码地址：<https://github.com/sxpujs/spring-cloud-examples/tree/master/netflix/netflix-eureka-server>

1 maven依赖增加 netflix-eureka-server
```xml
<dependency>
  <groupId>org.springframework.cloud</groupId>
  <artifactId>spring-cloud-starter-netflix-eureka-server</artifactId>
</dependency>
```

2 配置文件 application.yml
```yaml
server:
  port: 8761

eureka:
  instance:
    hostname: localhost
  client:
    registerWithEureka: false
    fetchRegistry: false
    serviceUrl:
      defaultZone: http://${eureka.instance.hostname}:${server.port}/eureka/
```

3 Application启动类，增加@EnableEurekaServer注解

```java
@SpringBootApplication
@EnableEurekaServer
public class NetflixEurekaServerApplication {

  public static void main(String[] args) {
    SpringApplication.run(NetflixEurekaServerApplication.class, args);
  }
}
```
4 启动服务，在浏览器打开如下地址：<http://localhost:8761/>，页面如下：
![Eureka Home](/img/eureka1.png)


## 2 创建服务提供者
完整代码参考：<https://github.com/sxpujs/spring-cloud-examples/tree/master/netflix/netflix-eureka-client-provider>

1 maven依赖增加 netflix-eureka-client
```xml
<dependency>
  <groupId>org.springframework.cloud</groupId>
  <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
</dependency>
<dependency>
  <groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-starter-web</artifactId>
</dependency>
```

2 配置application.yml
```yaml
eureka:
  client:
    serviceUrl:
      defaultZone: http://localhost:8761/eureka/
spring:
  application:
    name: provider  ## 应用程序名称，后面会在消费者中用到
server:
  port: 8000
```

3 Application启动类与hello接口
```java
@SpringBootApplication
@RestController
@Slf4j
public class NetflixEurekaClientProviderApplication {

  @RequestMapping("/")
  public String home() {
    return "Hello world";
  }

  @Autowired
  private DiscoveryClient discoveryClient;

  @RequestMapping(value = "/hello")
  public String hello(){
    List<String> services = discoveryClient.getServices();
    for(String s : services){
      log.info(s);
    }
    return "hello spring cloud!";
  }

  public static void main(String[] args) {
    SpringApplication.run(NetflixEurekaClientProviderApplication.class, args);
  }
}
```

4 启动项目，正常情况下就注册到了 Eureka 注册中心，打开 Eureka 控制台，会看到已经出现了这个服务。
![Eureka Home](/img/eureka2.png)

```
curl localhost:8000/hello
结果: hello spring cloud!
```

## 3 创建服务消费者
完整代码参考：<https://github.com/sxpujs/spring-cloud-examples/tree/master/netflix/netflix-eureka-client-consumer>

1 maven依赖增加 netflix-eureka-client, spring-cloud-starter-openfeign等
```xml
<dependency>
  <groupId>org.springframework.cloud</groupId>
  <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
</dependency>
<dependency>
  <groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-starter-web</artifactId>
</dependency>
<dependency>
  <groupId>org.springframework.cloud</groupId>
  <artifactId>spring-cloud-starter-openfeign</artifactId>
</dependency>
<dependency>
  <groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-starter-actuator</artifactId>
</dependency>
```

2 配置application.yml
```yaml
eureka:
  client:
    serviceUrl:
      defaultZone: http://localhost:8761/eureka/
spring:
  application:
    name: consumer  ## 应用程序名称

server:
  port: 9000
```

3 Application启动类
```java
@SpringBootApplication
@RestController
@EnableEurekaClient
@EnableFeignClients
public class NetflixEurekaClientConsumerApplication {

  /**
   * 注入 RestTemplate
   * 并用 @LoadBalanced 注解，用负载均衡策略请求服务提供者
   * 这是 Spring Ribbon 的提供的能力
   * @return
   */
  @LoadBalanced
  @Bean
  public RestTemplate restTemplate() {
    return new RestTemplate();
  }

  @RequestMapping("/")
  public String home() {
    return "Hello world";
  }

  public static void main(String[] args) {
    SpringApplication.run(NetflixEurekaClientConsumerApplication.class, args);
  }

}
```

4 创建一个服务接口类，这是 Feign 的使用方式，详细的用法可以查一下 Spring Cloud Feign 相关文档
```java
/**
 * IHelloService
 * 配置服务提供者：provider 是服务提供者的 application.name
 */
@FeignClient("provider")
public interface IHelloService {

    @RequestMapping(value = "/hello")
    String hello();
}
```

5 创建一个 Controller 用于调用服务
```java
@RestController
public class ConsumerController {

    @Autowired
    private RestTemplate restTemplate;

    @Autowired
    private IHelloService helloService;

    private static final String applicationName = "provider";

    @RequestMapping(value = "commonRequest")
    public Object commonRequest(){
        String url = "http://"+ applicationName +"/hello";
        return restTemplate.getForObject(url,String.class);
    }

    @RequestMapping(value = "feignRequest")
    public Object feignRequest(){
        return helloService.hello();
    }
}
```
其中 feignRequest 方法是使用了 Feign 的方式调用服务接口；
commonRequest 方法是用 RestTemplate 提供的方法调用服务接口；

6 启动服务，测试接口。
```shell
curl localhost:9000/commonRequest
结果: hello spring cloud!

curl localhost:9000/feignRequest
结果: hello spring cloud!
```