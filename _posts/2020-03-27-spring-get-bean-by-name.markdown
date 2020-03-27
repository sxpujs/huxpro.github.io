---
layout: post
title: "Spring通过名称获取Bean示例"
author: "BJ大鹏"
header-style: text
tags:
  - Java
  - Spring
---

摘要：本文主要演示通过继承自抽象类ApplicationObjectSupport获取Bean实例。

参考文档：
- [Building a RESTful Web Service](https://spring.io/guides/gs/rest-service/)
- [Spring在代码中获取bean的几种方式](https://www.cnblogs.com/bjlhx/p/7043651.html)

[Spring在代码中获取bean的几种方式](https://www.cnblogs.com/bjlhx/p/7043651.html)提到共有以下几种方式：
- 方法一：在初始化时保存ApplicationContext对象
- 方法二：通过Spring提供的utils类获取ApplicationContext对象
- 方法三：继承自抽象类ApplicationObjectSupport
- 方法四：继承自抽象类WebApplicationObjectSupport
- 方法五：实现接口ApplicationContextAware
- 方法六：通过Spring提供的ContextLoader

完整示例代码请参考：<https://github.com/sxpujs/spring-cloud-examples/tree/master/rest-service>

软件环境：
- 操作系统：MacOS Catalina 10.15.3
- JDK 13.0.2
- spring-boot-starter-parent: 2.2.5.RELEASE
- Maven: 3.6.3

目录：
- [文件布局](#文件布局)
- [pom.xml](#pomxml)
- [Task接口](#task接口)
	- [Task实现类BarTask](#task实现类bartask)
	- [Task实现类FooTask](#task实现类footask)
- [Spring上下文工具类（继承ApplicationObjectSupport）](#spring上下文工具类继承applicationobjectsupport)
- [用于测试的GreetingController](#用于测试的greetingcontroller)
- [测试](#测试)

### 文件布局

```
localhost:rest-service didi$ tree .
.
├── pom.xml
├── src
│   └── main
│       └── java
│           └── com
│               └── example
│                   ├── RestServiceApplication.java
│                   ├── controller
│                   │   ├── Greeting.java
│                   │   └── GreetingController.java
│                   ├── service
│                   │   ├── BarTask.java
│                   │   ├── FooTask.java
│                   │   └── Task.java
│                   └── util
│                       └── SpringContextHolder.java

```

### pom.xml

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 https://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>
    <parent>
        <groupId>com.example</groupId>
        <artifactId>springcloud-parent</artifactId>
        <version>1.0-SNAPSHOT</version>
    </parent>
    <groupId>com.example</groupId>
    <artifactId>rest-service</artifactId>
    <version>1.0-SNAPSHOT</version>
    <name>rest-service</name>
    <description>Demo project for Spring Boot</description>

    <properties>
        <java.version>13</java.version>
    </properties>

    <dependencies>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>

        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-test</artifactId>
            <scope>test</scope>
            <exclusions>
                <exclusion>
                    <groupId>org.junit.vintage</groupId>
                    <artifactId>junit-vintage-engine</artifactId>
                </exclusion>
            </exclusions>
        </dependency>
    </dependencies>

    <build>
        <plugins>
            <plugin>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-maven-plugin</artifactId>
            </plugin>
        </plugins>
    </build>

</project>
```

### Task接口

文件路径：src/main/java/com/example/service/Task.java

```java
package com.example.service;

public interface Task {
    void execute();
}
```
#### Task实现类BarTask

文件路径：src/main/java/com/example/service/BarTask.java

```java
package com.example.service;

import org.springframework.stereotype.Service;

@Service("barTask")
public class BarTask implements Task {

    @Override
    public void execute() {
        System.out.println("Run BarTask");
    }

}
```

#### Task实现类FooTask

文件路径：src/main/java/com/example/service/FooTask.java
```java
package com.example.service;

import org.springframework.stereotype.Service;

@Service("fooTask")
public class FooTask implements Task {

    @Override
    public void execute() {
        System.out.println("Run FooTask");
    }

}
```

### Spring上下文工具类（继承ApplicationObjectSupport）

文件路径：src/main/java/com/example/util/SpringContextHolder.java
```java
package com.example.util;

import com.example.service.Task;
import org.springframework.context.support.ApplicationObjectSupport;
import org.springframework.stereotype.Component;

@Component
public class SpringContextHolder extends ApplicationObjectSupport {

    public Task getTask(String beanName){
        return super.getApplicationContext().getBean(beanName , Task.class);
    }
}
```

### 用于测试的GreetingController

文件路径：src/main/java/com/example/controller/GreetingController.java
```java
package com.example.controller;

import java.util.concurrent.atomic.AtomicLong;

import com.example.service.Task;
import com.example.util.SpringContextHolder;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RequestParam;
import org.springframework.web.bind.annotation.RestController;

@RestController
public class GreetingController {

    private static final String template = "Hello, %s!";
    private final AtomicLong counter = new AtomicLong();

    @Autowired
    SpringContextHolder holder;


    @GetMapping("/greeting")
    public Greeting greeting(@RequestParam(value = "name", defaultValue = "World") String name) {

        Task task1 = holder.getTask("fooTask");
        Task task2 = holder.getTask("barTask");

        task1.execute();
        task2.execute();

        return new Greeting(counter.incrementAndGet(), String.format(template, name));
    }
}
```

其它如src/main/java/com/example/RestServiceApplication.java和src/main/java/com/example/controller/Greeting.java等文件请参考[Github](https://github.com/sxpujs/spring-cloud-examples/tree/master/rest-service)

### 测试
启动该服务后，在浏览器中输入：<http://localhost:8080/greeting>，会得到如下响应：
```
{"id":1,"content":"Hello, World!"}
```

在后台日志会看到：
```
Run FooTask
Run BarTask
```
