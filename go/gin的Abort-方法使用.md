---
title: gin的Abort()方法使用
seo_title: gin的Abort()方法使用
categories:
  - go语言系列
book: Strcpy
book_title: gin的Abort()方法使用
date: 2021-12-20 02:46:12
description:
github:
github_page:
id: post-837
---

gin的`Abort()`方法使用

食用本篇文章，请先看完[Gin的Next()方法使用](https://jinfeijie.cn/post-822.html "Gin的Next()方法使用")

<!--more-->

### 简单的Http服务的实现

先来实现一个简单的http服务

```
package main

import (
	"github.com/gin-gonic/gin"
	"github.com/jinfeijie/blog-demo/post-837/middleware/md1"
	"github.com/jinfeijie/blog-demo/post-837/middleware/md2"
	"github.com/jinfeijie/blog-demo/post-837/middleware/md3"
	"github.com/jinfeijie/blog-demo/post-837/middleware/md4"
	"net/http"
)

func main() {
	router := gin.New()
	router.Use(gin.Recovery(), md1.Md1, md2.Md2, md3.Md3, md4.Md4)

	fmt.Println(math.MaxInt8 / 2)
	router.GET("/", func(ctx *gin.Context) {
		ctx.JSON(http.StatusOK, []int{1, 2, 3})
	})

	if err := http.ListenAndServe(":1080", router); err != nil {
		panic(err.Error())
	}
}
```

我们在程序中引入了4个中间件，来实现非嵌入式的业务处理。
简化后的中间件代码如下（demo代码中，4个中间件只有数字的差异）
```
package md1

import (
	"fmt"
	"github.com/gin-gonic/gin"
)

func Md1(ctx *gin.Context) {
	fmt.Println("md1 start")
	ctx.Next()
	fmt.Println("md1 end")
}
```
运行起来后，请求，可以在日志中看到如下输出：
```
[GIN-debug] GET    /                         --> main.main.func1 (6 handlers)
md1 start
md2 start
md3 start
md4 start
md4 end
md3 end
md2 end
md1 end
```

依旧是非常典型的洋葱形式的输出。

### 使用`Abort()`的效果

#### md1 是风控模块
假设中间1是一个风控模块，当检测到请求参数异常时，立即进行拦截，不再继续执行后续操作。
我们可以在A中加入`Abort()`，让他跳过后续步骤的执行。中间件1的实现代码
```
func Md1(ctx *gin.Context) {
	fmt.Println("md1 start")
	if ctx.Query("danger") == "1" {
		ctx.Abort()
	}
	ctx.Next()
	fmt.Println("md1 end")
}
```
此时我们通过风控链接访问到服务，就会触发风控机制，直接被abort。
```
[GIN-debug] GET    /                         --> main.main.func1 (6 handlers)
md1 start
md1 end
```
#### md2是登录验证模块
假设中间2是一个登录验证模块，未登录的用户会在这一层被拦截
```
func Md2(ctx *gin.Context) {
	fmt.Println("md2 start")
	if ctx.Query("login") == "0" {
		ctx.Abort()
	}
	ctx.Next()
	fmt.Println("md2 end")
}
```
当一个未登录的用户访问过来，就会被这一层拦截，不再执行后续的代码
```
[GIN-debug] GET    /                         --> main.main.func1 (6 handlers)
md1 start
md2 start
md2 end
md1 end
```

### `Abort()`如何实现它的功能
从上一篇`Next`的文章中我们可以知道中间件的执行过程是 `Md1.start` -> `Md2.start` -> `Md3.start` -> `service` -> `Md3.end` -> `Md2.end` -> `Md1.end`。当中间件中遇到异常执行了abort时，他会中止执行后续的中间件的逻辑，直接结束掉。流程就是`Md1.start` -> `Md2.start` -> `Md2.end` -> `Md1.end`。
那`Abort()`是如何实现他的这个功能的呢？查看代码后会发现，它的实现非常简单，只有一行代码。
```
c.index = abortIndex
```
它的作用就是把中间件的索引位放置到第64位，直接跳过前置的中间件的运行。` (const abortIndex = (1<<7 - 1) / 2 )`
与`Abort()`方法对应的`IsAborted()`就是通过当前索引位是否大于等于63。

### 更优雅地使用`Abort()`
从上文日志中可以发现，当使用`Abort()`时，不会有任何的输出。而一个在线服务就算是报错了也需要一个内容返回。那如何更优雅地给出提示？我们可以调用`AbortWithStatusJSON(code int, jsonObj interface{})`来输出json内容。如果不满足输出可以，可以自己封装一个`Abort()`。


### 中间件+`Abort()`组合有何作用
1. 风控拦截，在中间件层实现风控拦截，可以避免业务代码与风控代码耦合度过高
2. 登录验证，在中间件层做完权鉴，通过上下文传递，实现业务代码与权鉴的解耦
3. 黑名单拦截，在中间件层实现黑名单拦截，可以使业务代码可以更加专注于实现业务
4. 限流，在中间件层处理限流逻辑，可以提升系统的横向扩展能力

### 潜在的问题
常规情况下一个系统不会有超过64个中间件。一旦中间件超过64个，使用`isAbort()`就会出现不在预期的结果。

### 引用
【1】[文章相关代码](https://github.com/jinfeijie/blog-demo/tree/master/post-837 "文章相关代码")