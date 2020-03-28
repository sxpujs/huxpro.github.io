---
layout: post
title: "Spring Controller单例与线程安全那些事儿"
author: "BJ大鹏"
header-style: text
tags:
  - Java
  - Spring
  - 设计模式
---

### 目录
- [单例（singleton）作用域](#单例singleton作用域)
- [原型（Prototype）作用域](#原型prototype作用域)
- [多个HTTP请求在Spring控制器内部串行还是并行执行方法？](#多个http请求在spring控制器内部串行还是并行执行方法)
- [实现单例模式并模拟大量并发请求，验证线程安全](#实现单例模式并模拟大量并发请求验证线程安全)
- [附录：Spring Bean作用域](#附录spring-bean作用域)

### 单例（singleton）作用域

每个添加@RestController或@Controller的控制器，默认是单例(singleton)，这也是Spring Bean的默认作用域。

下面代码示例参考了[Building a RESTful Web Service](https://spring.io/guides/gs/rest-service/)，该教程搭建基于Spring Boot的web项目，源代码可参考[spring-guides/gs-rest-service](https://github.com/spring-guides/gs-rest-service)

GreetingController.java代码如下：
```java
package com.example.controller;

import java.util.concurrent.atomic.AtomicLong;

import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RequestParam;
import org.springframework.web.bind.annotation.RestController;

@RestController
public class GreetingController {

    private static final String template = "Hello, %s!";
    private final AtomicLong counter = new AtomicLong();

    @GetMapping("/greeting")
    public Greeting greeting(@RequestParam(value = "name", defaultValue = "World") String name) {
        Greeting greet =  new Greeting(counter.incrementAndGet(), String.format(template, name));
        System.out.println("id=" + greet.getId() + ", instance=" + this);
        return greet;
    }
}
```

我们使用HTTP基准工具[wrk](https://github.com/wg/wrk)来生成大量HTTP请求。在终端输入如下命令来测试：
```shell
wrk -t12 -c400 -d10s http://127.0.0.1:8080/greeting
```

在服务端的标准输出中，可以看到类似日志。
```
id=162440, instance=com.example.controller.GreetingController@368b1b03
id=162439, instance=com.example.controller.GreetingController@368b1b03
id=162438, instance=com.example.controller.GreetingController@368b1b03
id=162441, instance=com.example.controller.GreetingController@368b1b03
id=162442, instance=com.example.controller.GreetingController@368b1b03
id=162443, instance=com.example.controller.GreetingController@368b1b03
id=162444, instance=com.example.controller.GreetingController@368b1b03
id=162445, instance=com.example.controller.GreetingController@368b1b03
id=162446, instance=com.example.controller.GreetingController@368b1b03
```

日志中所有GreetingController实例的地址都是一样的，说明多个请求对同一个 GreetingController 实例进行处理，并且它的AtomicLong类型的counter字段正按预期在每次调用时递增。

### 原型（Prototype）作用域

如果我们在@RestController注解上方增加@Scope("prototype")注解，使bean作用域变成原型作用域，其它内容保持不变。

```java
...

@Scope("prototype")
@RestController
public class GreetingController {
    ...
}
```
服务端的标准输出日志如下，说明改成原型作用域后，每次请求都会创建新的bean，所以返回的id始终是1，bean实例地址也不同。
```
id=1, instance=com.example.controller.GreetingController@2437b9b6
id=1, instance=com.example.controller.GreetingController@c35e3b8
id=1, instance=com.example.controller.GreetingController@6ea455db
id=1, instance=com.example.controller.GreetingController@3fa9d3a4
id=1, instance=com.example.controller.GreetingController@3cb58b3
```

### 多个HTTP请求在Spring控制器内部串行还是并行执行方法？

如果我们在greeting()方法中增加休眠时间，来看下每个http请求是否会串行调用控制器里面的方法。

```java
@RestController
public class GreetingController {

    private static final String template = "Hello, %s!";
    private final AtomicLong counter = new AtomicLong();

    @GetMapping("/greeting")
    public Greeting greeting(@RequestParam(value = "name", defaultValue = "World") String name) throws InterruptedException {
        Thread.sleep(1000); // 休眠1s
        Greeting greet =  new Greeting(counter.incrementAndGet(), String.format(template, name));
        System.out.println("id=" + greet.getId() + ", instance=" + this);
        return greet;
    }
}
```

还是使用wrk来创建大量请求，可以看出即使服务端的方法休眠1秒，导致每个请求的平均延迟达到1.18s，但每秒能处理的请求仍达到166个，证明HTTP请求在Spring MVC内部是并发调用控制器的方法，而不是串行。
```
wrk -t12 -c400 -d10s http://127.0.0.1:8080/greeting

Running 10s test @ http://127.0.0.1:8080/greeting
  12 threads and 400 connections
  Thread Stats   Avg      Stdev     Max   +/- Stdev
    Latency     1.18s   296.41ms   1.89s    85.22%
    Req/Sec    37.85     38.04   153.00     80.00%
  1664 requests in 10.02s, 262.17KB read
  Socket errors: connect 155, read 234, write 0, timeout 0
Requests/sec:    166.08
Transfer/sec:     26.17KB
```

### 实现单例模式并模拟大量并发请求，验证线程安全

#### 单例类的定义：Singleton.java
```java
package com.demo.designpattern;

import java.util.concurrent.atomic.AtomicInteger;

public class Singleton {

    private volatile static Singleton singleton;
    private int counter = 0;
    private AtomicInteger atomicInteger = new AtomicInteger(0);

    private Singleton() {
    }

    public static Singleton getSingleton() {
        if (singleton == null) {
            synchronized (Singleton.class) {
                if (singleton == null) {
                    singleton = new Singleton();
                }
            }
        }
        return singleton;
    }

    public int getUnsafeNext() {
        return ++counter;
    }

    public int getUnsafeCounter() {
        return counter;
    }

    public int getSafeNext() {
        return atomicInteger.incrementAndGet();
    }

    public int getSafeCounter() {
        return atomicInteger.get();
    }

}
```

#### 测试单例类并创建大量请求并发调用：SingletonTest.java
```java
package com.demo.designpattern;

import java.util.*;
import java.util.concurrent.*;

public class SingletonTest {

    public static void main(String[] args) {
        // 定义可返回计算结果的非线程安全的Callback实例
        Callable<Integer> unsafeCallableTask = () -> Singleton.getSingleton().getUnsafeNext();
        runTask(unsafeCallableTask);
        // unsafe counter may less than 1000, i.e. 984
        System.out.println("current counter = " + Singleton.getSingleton().getUnsafeCounter());

        // 定义可返回计算结果的线程安全的Callback实例（基于AtomicInteger）
        Callable<Integer> safeCallableTask = () -> Singleton.getSingleton().getSafeNext();
        runTask(safeCallableTask);
        // safe counter should be 1000
        System.out.println("current counter = " + Singleton.getSingleton().getSafeCounter());

    }

    public static void runTask(Callable<Integer> callableTask) {
        int cores = Runtime.getRuntime().availableProcessors();
        ExecutorService threadPool = Executors.newFixedThreadPool(cores);
        List<Callable<Integer>> callableTasks = new ArrayList<>();
        for (int i = 0; i < 1000; i++) {
            callableTasks.add(callableTask);
        }
        Map<Integer, Integer> frequency = new HashMap<>();
        try {
            List<Future<Integer>> futures = threadPool.invokeAll(callableTasks);
            for (Future<Integer> future : futures) {
                frequency.put(future.get(), frequency.getOrDefault(future.get(), 0) + 1);
                //System.out.printf("counter=%s\n", future.get());
            }
        } catch (InterruptedException | ExecutionException e) {
            e.printStackTrace();
        }
        threadPool.shutdown();
    }
}
```

### 附录：[Spring Bean作用域](https://docs.spring.io/spring/docs/current/spring-framework-reference/core.html#beans-factory-scopes)

| <div style="width: 100pt">范围</div> | 描述 |
|---------|----------|
| singleton（单例） |（默认值）将每个Spring IoC容器的单个bean定义范围限定为单个对象实例。<br><br> 换句话说，当您定义一个bean并且其作用域为单例时，Spring IoC容器将为该bean所定义的对象创建一个实例。该单例存储在单例beans的高速缓存中，并且对该命名bean的所有后续请求和引用都返回该高速缓存的对象。
|prototype（原型）| 将单个bean定义的作用域限定为任意数量的对象实例。<br><br>每次对特定bean发出请求时，bean原型作用域都会创建一个新bean实例。也就是说，将Bean注入到另一个Bean中，或者您可以调用容器上的getBean()方法来请求它。通常，应将原型作用域用于所有有状态Bean，将单例作用域用于无状态Bean。
|request| 将单个bean定义的范围限定为单个HTTP请求的生命周期。也就是说，每个HTTP请求都有一个在单个bean定义后创建的bean实例。仅在web-aware的Spring ApplicationContext上下文有效。
|session| 将单个bean定义的范围限定为HTTP Session的生命周期。仅在基于web的Spring ApplicationContext上下文有效。
|application| 将单个bean定义的范围限定为ServletContext的生命周期。仅在基于web的Spring ApplicationContext上下文有效。
|websocket| 将单个bean定义的作用域限定为WebSocket的生命周期。仅在基于web的Spring ApplicationContext上下文有效。
