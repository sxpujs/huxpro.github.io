---
layout: post
title: "golang map并发读写异常导致服务崩溃"
author: "BJ大鹏"
header-style: text
tags:
  - Go
  - 并发
  - 线上案例
---

昨天突然接到报警说服务端口丢失，也就是服务崩溃了。

#### 分析错误日志

发现是调用json.Marshal时出错了，错误原因是：concurrent map iteration and map write，即并发读写map。

```
fatal error: concurrent map iteration and map write

goroutine 543965 [running]:
runtime.throw(0xb74baf, 0x26)
	/usr/local/go1.10.1/src/runtime/panic.go:616 +0x81 fp=0xc420c90a98 sp=0xc420c90a78 pc=0x42c7a1
runtime.mapiternext(0xc42160d5c0)
	/usr/local/go1.10.1/src/runtime/hashmap.go:747 +0x55c fp=0xc420c90b28 sp=0xc420c90a98 pc=0x40a73c
runtime.mapiterinit(0xa7ff20, 0xc4205372f0, 0xc42160d5c0)
	/usr/local/go1.10.1/src/runtime/hashmap.go:737 +0x1f1 fp=0xc420c90b50 sp=0xc420c90b28 pc=0x40a0f1
reflect.mapiterinit(0xa7ff20, 0xc4205372f0, 0x15)
	/usr/local/go1.10.1/src/runtime/hashmap.go:1217 +0x54 fp=0xc420c90b80 sp=0xc420c90b50 pc=0x40b814
reflect.Value.MapKeys(0xa7ff20, 0xc4205372f0, 0x15, 0x0, 0x409322, 0xc420c90d10)
	/usr/local/go1.10.1/src/reflect/value.go:1114 +0xdd fp=0xc420c90c28 sp=0xc420c90b80 pc=0x4b4afd
encoding/json.(*mapEncoder).encode(0xc421098ee8, 0xc4230bd810, 0xa7ff20, 0xc4205372f0, 0x15, 0xa70100)
	/usr/local/go1.10.1/src/encoding/json/encode.go:668 +0xad fp=0xc420c90d88 sp=0xc420c90c28 pc=0x4f42cd
encoding/json.(*mapEncoder).(encoding/json.encode)-fm(0xc4230bd810, 0xa7ff20, 0xc4205372f0, 0x15, 0xc420530100)
	/usr/local/go1.10.1/src/encoding/json/encode.go:700 +0x64 fp=0xc420c90dc8 sp=0xc420c90d88 pc=0x4feab4
encoding/json.(*encodeState).reflectValue(0xc4230bd810, 0xa7ff20, 0xc4205372f0, 0x15, 0x100)
	/usr/local/go1.10.1/src/encoding/json/encode.go:325 +0x82 fp=0xc420c90e00 sp=0xc420c90dc8 pc=0x4f1cf2
encoding/json.(*encodeState).marshal(0xc4230bd810, 0xa7ff20, 0xc4205372f0, 0xc420780100, 0x0, 0x0)
	/usr/local/go1.10.1/src/encoding/json/encode.go:298 +0xa5 fp=0xc420c90e38 sp=0xc420c90e00 pc=0x4f19e5
encoding/json.Marshal(0xa7ff20, 0xc4205372f0, 0xc420783850, 0xc420783857, 0xc420c90f30, 0x0, 0x0)
	/usr/local/go1.10.1/src/encoding/json/encode.go:161 +0x5f fp=0xc420c90e80 sp=0xc420c90e38 pc=0x4f13ef
xxx.BuildKeyList(0xc42051a2e8, 0xc420783850, 0xb, 0xc420c910e0, 0x1, 0x1,
 0xc4205372f0, 0x0, 0x2, 0xc4214e96b0, ...)
	xxx/custom.go:85 +0x59 fp=0xc420c90f60 sp=0xc420c90e80 pc=0x774dd9
xxx.BuildKeyList(0xc420783850, 0xb, 0xc420c910e0, 0x1, 0x1, 0xc4205372f0, 0xb, 0xc4207cd0a0, 0x13, 0xc42245c8a0, ...)
	xxx/dao.go:42 +0x79 fp=0xc420c90fd0 sp=0xc420c90f60 pc=0x834b59
xxx/logic.getFeatureMap(0xbde540, 0xc4214e9410, 0xc422561700, 0x4, 0x4, 0x2100035d178a8, 0xabce00)
	xxx/calc_plan.go:114 +0x4af fp=0xc420c911f8 sp=0xc420c90fd0 pc=0x9cd4df
```

#### xxx/calc_plan.go文件的114行：
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

#### 深拷贝的函数
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

#### 调用深拷贝实现

把```param := filterRule.FilterMap```修改为 ```param := DeepCopyMap(filterRule.FilterMap)```

## 一个简单的模拟示例
```go
package main

import (
	"encoding/json"
	"fmt"
	"strconv"
)

func main() {

	m := map[string]string{
		"name": "andy",
		"age":  "30",
	}

	write := make(chan bool)
	go func() {
		for i := 0; i < 10000; i++ {
			m["idx"] = strconv.Itoa(i)
		}
		write <- true
	}()

	read := make(chan bool)
	go func() {
		for i := 0; i < 10000; i++ {
			json.Marshal(m)
		}
		read <- true
	}()

	<-write
	<-read

	fmt.Println("Finish")
}
```

执行上述代码，很容易报错：fatal error: concurrent map iteration and map write
