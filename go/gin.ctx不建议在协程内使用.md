---
title: gin.ctx不建议在协程内使用
seo_title: gin.ctx不建议在协程内使用
date: 2020-12-31 00:45:00
description: 千万不要再请求里面，把上下文直接传递到野生goroutine里面
categories: # 分类
- go语言系列
book: Strcpy
book_title: gin.ctx不建议在协程内使用
github: https://github.com/jinfeijie/posts
github_page: https://github.com/jinfeijie/posts/blob/master/go/%E4%B8%8D%E5%BB%BA%E8%AE%AE%E5%9C%A8Gonroutine%E5%86%85%E4%BD%BF%E7%94%A8req.ctx.md
id: post-777
---

> 首先给个结论：千万不要再请求里面，把上下文直接传递到野生goroutine里面

# 先来看一个现象
代码如下：
```golang
func(ctx *gin.Context) {
	_uuid := uuid.New().String()
	ctx.Set("uuid", _uuid)
	go func(c *gin.Context, u string) {
		time.Sleep(time.Second)
		cu := c.GetString("uuid")
		if cu == u {
			log.Printf("一致 %p, %s\n", c, u)
		} else {
			log.Printf("不一致 %p, uuid = %s, cuuid = %s\n", c, u, cu)
		}
	}(ctx, _uuid)

	ctx.JSON(200, _uuid)
	return
}
```

请求开始生成一个uuid，设置进上下文中。开启一个协程，传入上下文和生成的uuid。协程内sleep一秒后读取上下文中的的uuid是否与与传入的uuid一致。

① 模拟请求，1秒1次
![](https://image.baidu.com/search/down?url=https://tva1.sinaimg.cn/large/e6c9d24ely1h4ebbmdp45j216y0d4428.jpg)
② 模拟请求，1秒2次
![](https://image.baidu.com/search/down?url=https://tva1.sinaimg.cn/large/e6c9d24ely1h4ebbm9sv4j210j0jbwlt.jpg)
③ 模拟请求，压测模式
![](https://image.baidu.com/search/down?url=https://tva1.sinaimg.cn/large/e6c9d24ely1h4ebbm1e1xj20zh085q53.jpg)
①中全部是一致
②中不一致少于一致
③出现大量不一致，压测尾部会有少量一致

# 造成的原因

Gin框架在gin.Run里面实现了调用http.ListenAndServe方法。因为gin.Engine实现了接口http.Handler，并且在http.ListenAndServe的第二位参数将engine传入，所以服务启动后的请求都由gin.ServeHTTP处理。

```golang
func (engine *Engine) ServeHTTP(w http.ResponseWriter, req *http.Request) {
	c := engine.pool.Get().(*Context)
	c.writermem.reset(w)
	c.Request = req
	c.reset()

	engine.handleHTTPRequest(c)

	engine.pool.Put(c)
}
```
解释一下上面的代码。
1. 当请求进来时，会从上下文的池中获取到一个上下文。当池中没有上下文，则会创建一个新的。
2. 拿到上下文后，会给上下文初始化responseWriter
3. 给上下文设置请求体数据
4. 初始化上下文中的所有数据（除掉第3已设置的参数）
5. 将上下文传入业务逻辑中
6. 将上下文放回连接池中

上下文被放回sync.Pool时，没有被重置掉请求的数据。原请求一直持有该上下文的地址，当该上下文没有被二次取出时，原请求去获取上下文中的数据，可以取到一致的数据。

当请求频率间隔小于goroutine退出的时间，上下文在放回池中后又被下一个请求取出，设置上新的uuid，原goroutine拿着上下文的地址去取上下文中的数据，就导致了原goroutine读到了下一个请求的数据，导致出现了不一致。

随着并发量不断加大到一定的量，大量上下文不断被创建，被放进上下文池中，因为sync.Pool的分配策略，可能存在少部分上下文在一段时间内一直不会被取出来二次利用。就会导致，在高并发下，也会有少量的请求是读取到一致的数据。

当高并发接近尾声，不再有新增的请求进去，上下文池中的上下文不再被取出覆盖，又会出现一批一致的请求。

# 什么场景下可以往goroutine中传递上下文
goroutine非野生goroutine，goroutine在主请求中可以被管控到。在主请求的生命周期内，goroutine创建后退出。
比如下面这种

```golang
var wg sync.WaitGroup
wg.Add(2)
go func(ctx *gin.Context) {
	defer wg.Done()
	fmt.Println("1")
}(ctx)

go func(ctx *gin.Context) {
	defer wg.Done()
	fmt.Println("2")
}(ctx)
wg.Wait()
```

# 如何避免上下文被野生goroutine使用
Gin框架在充分利用资源的同时也给服务带来了风险。所以应该尽量避免往goroutine中传递上下文。

# 必须要使用该怎么做
gin框架提供了copy方法为不得不使用上下文传递的场景提供支持。
复制后的上下文可以安全地在请求外的使用。

```golang
// Copy returns a copy of the current context that can be safely used outside the request's scope.
// This has to be used when the context has to be passed to a goroutine.
func (c *Context) Copy() *Context {
   cp := Context{
      writermem: c.writermem,
      Request:   c.Request,
      Params:    c.Params,
      engine:    c.engine,
   }
   cp.writermem.ResponseWriter = nil
   cp.Writer = &cp.writermem
   cp.index = abortIndex
   cp.handlers = nil
   cp.Keys = map[string]interface{}{}
   for k, v := range c.Keys {
      cp.Keys[k] = v
   }
   paramCopy := make([]Param, len(cp.Params))
   copy(paramCopy, cp.Params)
   cp.Params = paramCopy
   return &cp
}
```
