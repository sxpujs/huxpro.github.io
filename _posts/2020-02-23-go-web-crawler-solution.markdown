---
layout: post
title: "Go Web爬虫并发实现"
author: "Sxp"
header-style: text
tags:
  - Go
  - 并发
---

题目：[Exercise: Web Crawler](https://tour.golang.org/concurrency/10)

直接参考了 <https://github.com/golang/tour/blob/master/solutions/webcrawler.go> 的实现，不过该代码使用了chan bool来存放子协程是否执行完成，我的代码是使用WaitGroup来让主协程等待子协程执行完成。

完整代码请参考 <https://github.com/sxpujs/go-example/blob/master/crawl/web-crawler.go>

请注意对于WaitGroup的处理参考了[Golang中WaitGroup使用的一点坑](https://liudanking.com/golang/golang-waitgroup-usage/)

相对原程序增加的代码如下：
```go
var fetched = struct {
	m map[string]error
	sync.Mutex
}{m: map[string]error{}}

// Crawl uses fetcher to recursively crawl
// pages starting with url, to a maximum of depth.
func Crawl(url string, depth int, fetcher Fetcher) {
	if _, ok := fetched.m[url]; ok || depth <= 0 {
		return
	}
	body, urls, err := fetcher.Fetch(url)

	fetched.Lock()
	fetched.m[url] = err
	fetched.Unlock()

	if err != nil {
		return
	}
	fmt.Printf("Found: %s %q\n", url, body)
	var wg sync.WaitGroup
	for _, u := range urls {
		wg.Add(1)
		go func(url string) {
			defer wg.Done()
			Crawl(url, depth-1, fetcher)
		}(u)
	}
	wg.Wait()
}
```