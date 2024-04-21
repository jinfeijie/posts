---
title: gin的Next()方法使用
seo_title: gin的Next()方法使用
categories:
  - go语言系列
book: 柠芒技术博客
book_title: gin的Next()方法使用
date: 2021-12-20 02:47:57
description:
github: https://github.com/jinfeijie/posts
github_page: https://github.com/jinfeijie/posts/blob/master/go/gin%E7%9A%84Next()%E6%96%B9%E6%B3%95%E4%BD%BF%E7%94%A8.md
id: post-822
---

使用gin这个web框架开发http服务，少不了使用中间件。

在调用gin.Default()方法的时候就会自动使用`Logger`和`Recovery()`两个中间件。

仔细查看两个中间件的实现，你会发现，两个中间件都有`c.Next()`这个方法的调用。


很多人把Next放在中间的最后一行，然后发现加不加next似乎并没有什么区别。所以，它到底是怎么用的？

<!--more-->

来自SF的引用文章：[goalng框架Gin中间件的c.Next()有什么作用？](https://segmentfault.com/q/1010000020256918 "goalng框架Gin中间件的c.Next()有什么作用？")

![洋葱模型](https://image.baidu.com/search/down?url=https://tva1.sinaimg.cn/large/e6c9d24ely1h4ebbn59jrj20da0c33zj.jpg)

上图可以说是Next()方法的一个形象化的解释

### 不使用 `Next()`
假设我们去编写这样的一段代码
```golang
route := gin.Default()
route.Use(middleware.RequestID)
route.Use(middleware.Log)
route.GET("/", app.Index)
```
而`middleware.RequestID`的实现如下
```golang
func RequestID(ctx *gin.Context) {
	log.Println("request start")
	ctx.Set("request_id", Uid())
	log.Println("request end")
}
```
`middleware.Log`的实现如下
```golang
func Log(ctx *gin.Context) {
	log.Println("log start")
	log.Println("log end")
}
```
`app.Index`的实现如下
```golang
func Index(ctx *gin.Context) {
	log.Println(ctx.Get("request_id"))
	ctx.JSON(http.StatusOK, "ok")
}
```
请求后可以在日志中信息中获取到
```
2020/02/09 23:53:45 request start
2020/02/09 23:53:45 request end
2020/02/09 23:53:45 log start
2020/02/09 23:53:45 log end
2020/02/09 23:53:45 18c6da3b-3e8c-4925-a0ec-76382eb15a10 true
```
日志显示，程序按照代码中定义的顺序`request` => `log` => `index` 执行了。

### 使用 `Next()`
那加入了`Next()`后怎么执行呢
如下为两个中间加入后的实际代码
```golang
func RequestID(ctx *gin.Context) {
	log.Println("request start")
	ctx.Set("request_id", Uid())
	ctx.Next()
	log.Println("request end")
}
```

```golang
func Log(ctx *gin.Context) {
	log.Println("log start")
	ctx.Next()
	log.Println("log end")
}
```
请求后，从日志中可以得知如下信息
```
2020/02/09 23:58:45 request start
2020/02/09 23:58:45 log start
2020/02/09 23:58:45 ecffea3b-0e46-4ce0-b55f-f0bceef07e92 true
2020/02/09 23:58:45 log end
2020/02/09 23:58:45 request end
```
程序先执行到` request start`，遇到`Next`，就去执行`log start`，又遇到`Next`。去执行`app.Index`，返回执行`log end`，再是`request end`。

出现了一种先进后出的情况。而这个就是上面图片上画的洋葱模型。

### 分析Next()的实现
`Next()`方法的实现非常简单
```golang
	route.Use(middleware.RequestID)
	route.Use(middleware.Log)
```
代码入口加载中间件就是往`c.handlers`中append这个方法。
```golang
func (c *Context) Next() {
	c.index++
	for c.index < int8(len(c.handlers)) {
		c.handlers[c.index](c)
		c.index++
	}
}
```
而每一次调用`Next`都会调用当前index的下一个索引位的方法，开始自增循环，直到index已经比所有的中间件数量还要大了，就会开始执行业务代码。当业务代码执行完后，就会一层一层的往外执行每个中间件未执行完的代码。


### 洋葱模式在实际业务中如何使用？

1. 计算业务程序实际耗时
我们来写一个`ExecTime`的中间件

```golang
func ExecTime(ctx *gin.Context) {
		log.Println("exec_time start")
		startTime := time.Now()
		ctx.Next()
		log.Println(time.Now().Sub(startTime).Seconds())
		log.Println("exec_time end")
}
```
放在离路由开始最近的位置
```golang
	route := gin.Default()
	route.Use(middleware.RequestID)
	route.Use(middleware.Log)
	route.Use(middleware.ExecTime)
	route.GET("/", app.Index)
```
请求后，从日志中可以得知如下信息
```
2020/02/10 00:06:51 request start
2020/02/10 00:06:51 log start
2020/02/10 00:06:51 exec_time start
2020/02/10 00:06:51 7b943cb5-d2f7-4e0e-ab0f-1b70cd800a8f true
2020/02/10 00:06:51 0.000198358
2020/02/10 00:06:51 exec_time end
2020/02/10 00:06:51 log end
2020/02/10 00:06:51 request end
```
执行耗时在业务代码执行完后就输出了

2. 程序异常的捕捉和报警
3. 链路跟踪

