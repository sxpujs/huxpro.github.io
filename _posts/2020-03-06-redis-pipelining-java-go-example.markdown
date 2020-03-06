---
layout: post
title: "Redis管道之Java与Go代码示例"
author: "BJ大鹏"
header-style: text
tags:
  - Java
  - Go
  - Redis
---

最近读了Redis官网一篇关于使用管道加速Redis查询 的文章，原文：[Using pipelining to speedup Redis queries](https://redis.io/topics/pipelining)，中文翻译可参考：[管道（Pipelining）](http://www.redis.cn/topics/pipelining.html)

一个请求/响应服务器能处理新的请求即使客户端还未读取旧的响应。这样就可以将多个命令发送到服务器，而不用等待回复，最后在一个步骤中读取该答复。

管道不仅仅是为了减少往返时间(Round Trip Time，简称RTT)引起的延迟成本，它实际上大大提高了给定 Redis 服务器中每秒可执行的操作总量。 这是因为在不使用管道的情况下，从访问数据结构和生成应答的角度来看，为每个命令提供服务非常廉价，但从执行套接字I/O的角度来看，这是非常昂贵的。 这涉及到调用 read() 和 write()系统调用，这意味着从用户区到内核区。 上下文切换是一个巨大的速度损失。

原文给的代码示例是基于ruby语言，下面我分别展示Java语言和Go语言的Redis管道示例。

### Redis管道Java示例

语言版本是：
- JDK：13.0.2
- Redis客户端：Jedis 3.2.0

#### pom.xml（Maven包依赖）

```xml
<dependency>
  <groupId>redis.clients</groupId>
  <artifactId>jedis</artifactId>
  <version>3.2.0</version>
  <type>jar</type>
  <scope>compile</scope>
</dependency>
```

#### JedisPipelineDemo.java
```java
import com.google.common.base.Stopwatch;
import redis.clients.jedis.Jedis;
import redis.clients.jedis.Pipeline;

import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;

public class JedisPipelineDemo {

    Jedis jedis = new Jedis("localhost");

    void withoutPipeline() {
        for (int i = 0; i < 10000; i++)
            jedis.ping();
    }

    void withPipeline() {
        Pipeline p = jedis.pipelined();
        for (int i = 0; i < 10000; i++)
            p.ping();
        p.sync(); // 获取所有的响应
    }

    void serialTask() {
        printTime(() -> withoutPipeline(), "withoutPipeline"); // 346.4 ms
        printTime(() -> withPipeline(), "withPipeline");       // 18.34 ms
    }

    // 虽然传入的是Runnable对象，直接调用run方法并没有创建线程，而是使用当前线程同步执行方法。
    void printTime(Runnable task, String taskName) {
        Stopwatch stopwatch = Stopwatch.createStarted();
        task.run();
        System.out.println(taskName + " took: " + stopwatch.stop());
    }

    public static void main(String[] args) {
        JedisPipelineDemo demo = new JedisPipelineDemo();
        demo.serialTask();
    }
}
```

### Redis管道Go示例

语言版本：
- Go: 1.13.5
- Redis客户端：[redigo](https://github.com/gomodule/redigo), [API参考文档](https://pkg.go.dev/github.com/gomodule/redigo/redis?tab=doc)

#### redis_pipeline.go
```
package main

import (
    "github.com/gomodule/redigo/redis"
    "log"
    "time"
)

var (
    c     redis.Conn
    err   error
    reply interface{}
)

func init() {
    c, err = redis.Dial("tcp", "127.0.0.1:6379")
    if err != nil {
        log.Fatal(err)
    }
}

func main() {
    defer c.Close()
    withoutPipelining() // 256.53 ms
    withPipelining()    // 8.69 ms
}

func withoutPipelining() {
    defer timeTrack(time.Now(), "withoutPipelining")
    for i := 0; i < 10000; i++ {
        c.Do("PING")
    }
}

func withPipelining() {
    defer timeTrack(time.Now(), "withPipelining")
    c.Send("MULTI")
    for i := 0; i < 10000; i++ {
        c.Send("PING")
    }
    c.Do("EXEC")
}

func timeTrack(start time.Time, name string) {
    elapsed := time.Since(start)
    log.Printf("%s took %s", name, elapsed)
}
```
