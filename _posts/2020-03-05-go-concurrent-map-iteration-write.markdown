---
layout: post
title: "Go map并发读写异常导致服务崩溃"
author: "BJ大鹏"
header-style: text
tags:
  - Go
  - 并发
  - 线上案例
---

昨天突然接到报警说服务端口丢失，也就是服务崩溃，看了错误日志，发现是map并发读写问题，记录下来，避免再犯类似错误。

### 分析错误日志

发现是调用json.Marshal时出错了，错误统计如下，都是并发读写map之类的异常。
```
229次错误：fatal error: concurrent map iteration and map write
193次错误：fatal error: concurrent map writes
129次错误：fatal error: concurrent map read and map write
```

### xxx/xxx.go文件的114行：
```
func getFeatureMap(ctx context.Context, filters []*model.FiltersType, driverId int64) map[string]string {
    var features []string
    for _, filterRule := range filters {
        param := filterRule.FilterMap
        param["driver_id"] = strconv.FormatInt(driverId, 10)
        featureKeys, _ := ufs.BuildKeyList(domain, []string{key}, param) // 114行
        features = append(features, featureKeys...)
    }
    ...
}
```

看起来就是传入的param这个map被多个协程并发写了，filterRule.FilterMap是全局变量，所以这里需要修改为深拷贝。

### 深拷贝的函数
```go
// DeepCopyMap map[string]string 类型实现深拷贝
func DeepCopyMap(params map[string]string) map[string]string {
    result := map[string]string{}
    for k, v := range params {
        result[k] = v
    }
    return result
}
```

### 调用深拷贝实现

把```param := filterRule.FilterMap```修改为 ```param := DeepCopyMap(filterRule.FilterMap)```

---

### 一个简单的模拟示例
```go
package main

import (
    "encoding/json"
    "fmt"
    "math/rand"
    "strconv"
    "sync"
    "time"
)

func main() {

    param := map[string]string{
        "name": "andy",
        "age":  "30",
    }

    f := func(i int) {
        tempMap := param
        //tempMap := DeepCopyMap(m) // 使用深度拷贝可以解决并发读写map的问题
        tempMap["idx"] = strconv.Itoa(i)
        time.Sleep(time.Millisecond * time.Duration(rand.Intn(10)))
        json.Marshal(tempMap)
    }

    var wg sync.WaitGroup
    for i := 0; i < 1000; i++ {
        wg.Add(1)
        go func(idx int) {
            defer wg.Done()
            f(idx)
        }(i)
    }
    wg.Wait()

    fmt.Println("Finish")
}

// DeepCopyMap map[string]string 类型实现深拷贝
func DeepCopyMap(params map[string]string) map[string]string {
    result := map[string]string{}
    for k, v := range params {
        result[k] = v
    }
    return result
}
```

执行上述代码，很容易报以下其中一个或多个的错误。
- fatal error: concurrent map iteration and map write
- fatal error: concurrent map writes
- fatal error: concurrent map read and map write

