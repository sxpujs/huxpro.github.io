---
layout: post
title: "01 Spring Cloud Config 实现配置中心"
author: "BJ大鹏"
header-style: text
tags:
  - Java
  - Spring Cloud
  - 微服务
---

Spring Cloud官网: <https://spring.io/projects/spring-cloud>

本篇主要讲[Spring Cloud Config](https://spring.io/projects/spring-cloud-config)，参考内容如下：
- [Spring Cloud Config 2.2.1.RELEASE参考文档](https://cloud.spring.io/spring-cloud-static/spring-cloud-config/2.2.1.RELEASE/reference/html/)
- [Spring Cloud Config 实现配置中心，看这一篇就够了](https://www.cnblogs.com/fengzheng/p/11242128.html)

# 实现简单的配置中心
配置文件就在Spring官方提供的配置仓库：https://github.com/spring-cloud-samples/config-repo

## 1 创建配置中心服务端
完整代码参考：<https://github.com/sxpujs/spring-cloud-examples/tree/master/config/config-server>

1 新建Spring Boot项目，引入config-server
```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-config-server</artifactId>
</dependency>
```

2 配置config相关的配置项

bootstrap.yml
```
spring:
  application:
    name: foo # 应用名
  profiles:
    active: dev,mysql
```

application.yml
```
server:
  port: 8888

spring:
  cloud:
    config:
      server:
        git:
          uri: https://github.com/spring-cloud-samples/config-repo
          basedir: target/config
```

3 Application启动类，增加相关注解：

```java
@SpringBootApplication
@EnableConfigServer
public class ConfigServerApplication {
    public static void main(String[] args) {
        SpringApplication.run(ConfigServerApplication.class, args);
    }
}
```
4 启动服务，进行相关测试
```
curl localhost:8888/foo/development

{
   "name" : "foo",
   "profiles" : ["development"],
   "propertySources" : [
      {"name":"https://github.com/spring-cloud-samples/config-repo/foo-development.properties",
       "source":{"bar":"spam",
                 "foo":"from foo development"}},
      {"name":"https://github.com/spring-cloud-samples/config-repo/foo.properties", 
       "source":{"foo":"from foo props",
                 "democonfigclient.message":"hello spring io"}},
      {"name":"https://github.com/spring-cloud-samples/config-repo/application.yml (document #0)", 
       "source":{"info.url" : "https://github.com/spring-cloud-samples",
                 "info.description":"Spring Cloud Samples",
                 "foo":"baz",
                 "eureka.client.serviceUrl.defaultZone":"http://localhost:8761/eureka/"}}
   ]
}


# 访问不存在的profile（mysql）
curl http://localhost:8888/foo/mysql
{
  "name":"foo",
  "profiles":["mysql"],
  "propertySources":[
    {"name":"https://github.com/spring-cloud-samples/config-repo/foo.properties",
     "source":{"foo":"from foo props",
               "democonfigclient.message":"hello spring io"}},
    {"name":"https://github.com/spring-cloud-samples/config-repo/application.yml (document #0)",
     "source":{"info.description":"Spring Cloud Samples",
               "info.url":"https://github.com/spring-cloud-samples",
               "eureka.client.serviceUrl.defaultZone":"http://localhost:8761/eureka/",
               "foo":"baz"}}
  ]
}


curl http://localhost:8888/foo-dev.yml  # 从结果来看，包含了

bar: spam
democonfigclient:
  message: hello from dev profile
eureka:
  client:
    serviceUrl:
      defaultZone: http://localhost:8761/eureka/
foo: from foo development
info:
  description: Spring Cloud Samples
  url: https://github.com/spring-cloud-samples
my:
  prop: from application-dev.yml
```

## 2 创建配置中心客户端，使用配置
完整代码参考：<https://github.com/sxpujs/spring-cloud-examples/tree/master/config/config-client>

1 引用相关的maven依赖。
```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-config</artifactId>
</dependency>

<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
</dependency>

<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-actuator</artifactId>
</dependency>
```

2 配置文件

bootstrap.yml
```yaml
spring:
  profiles:
    active: dev
  cloud:
    config:
      uri: http://localhost:8888
  application:
    name: myapp
```

application.yml
```
management:
  endpoint:
    shutdown:
      enabled: false
  endpoints:
    web:
      exposure:
        include: "*"   # * 在yaml 文件属于关键字，所以需要加引号
```

3 Application启动类
```java
@SpringBootApplication
@RestController
public class ConfigClientApplication {

    @Value("${foo}")
    private String foo;

    @Value("${my.prop}")
    private String myProp;

    @RequestMapping("/")
    public String home() {
        return "Hello World!" + myProp + "," + foo;
    }

    public static void main(String[] args) {
        SpringApplication.run(ConfigClientApplication.class, args);
    }
}
```

4 验证数据

访问env端口，注意：从Spring Boot 2.x开始，不能直接访问 <http://localhost:8080/env>，需要添加actuator。
```
curl localhost:8080    # 可以看出my.prop这个属于是来自application-dev.yml，foo来自application.yml

Hello World!from application-dev.yml,baz


curl localhost:8080/actuator/env|json_pp

{
   "activeProfiles" : [],
   "propertySources" : [
      {"name":"server.ports","properties":{"local.server.port":{"value":8080}}},
      {
         "name" : "bootstrapProperties-https://github.com/spring-cloud-samples/config-repo/application.yml (document #0)",
         "properties" : {
            "info.url" : {"value" : "https://github.com/spring-cloud-samples"},
            "eureka.client.serviceUrl.defaultZone" : {"value" : "http://localhost:8761/eureka/"},
            "foo" : {"value" : "baz"},
            "info.description" : {"value" : "Spring Cloud Samples"}
         }
      }
   ]
}
```
