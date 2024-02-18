---
title: Singleflight：深入解析go缓存击穿方案
seo_title: Singleflight：深入解析go缓存击穿方案
date: 2024-01-01 00:45:00
description: 几乎所有的现代化面向web编程的系统都在关注高并发情况下如何降低下游服务的承接压力，go语言也不例外。go语言在并发方面的发力体现在很多个方面，如更轻量化的协程，原子性操作的atomic，抑制函数重复调用的singleflight等。在常规的业务开发中singleflight使用并不多见，本文将会详细介绍singleflight，以深入理解和探索singleflight更多的技术细节和适用场景。
categories: # 分类
- go语言系列
book: Strcpy
book_title: Singleflight：深入解析go缓存击穿方案
github: https://github.com/jinfeijie/posts
github_page: https://github.com/jinfeijie/posts/blob/master/go/Singleflight%EF%BC%9A%E6%B7%B1%E5%85%A5%E8%A7%A3%E6%9E%90go%E7%BC%93%E5%AD%98%E5%87%BB%E7%A9%BF%E6%96%B9%E6%A1%88.md
id: post-20240101004500
---

几乎所有的现代化面向web编程的系统都在关注高并发情况下如何降低下游服务的承接压力，go语言也不例外。go语言在并发方面的发力体现在很多个方面，如更轻量化的协程，原子性操作的atomic，抑制函数重复调用的singleflight等。在常规的业务开发中singleflight使用并不多见，本文将会详细介绍singleflight，以深入理解和探索singleflight更多的技术细节和适用场景。

# 设计原理
`singleflight` 主要围绕着在高并发环境下对相同 key 的函数调用进行重复抑制。以下是其设计原理的核心要点：
1. Group 结构：
  - `singleflight` 使用 `Group` 结构表示一个函数调用的组。每个 Group 对应于一类工作，其中的函数调用可能会有重复的 key。
2. call 结构：
  - `call` 结构用于表示一个正在进行中或已完成的函数调用。它包含一个 sync.WaitGroup 用于等待调用的完成，以及相关的结果和错误信息。
3. 重复调用抑制：
  - 在 `Do` 方法中，首先检查是否已经存在相同 key 的函数调用。如果存在，新的调用将等待已有调用完成，然后共享相同的结果。这样可以有效地避免对相同 key 的重复计算。
4. 结果共享：
  - 在 `Do` 方法中，如果发现已有相同 key 的调用正在进行，新的调用将等待并共享相同的结果。这是通过在 call 结构中维护一个结果值和一个错误值来实现的。等待的调用将等待 `sync.WaitGroup` 的完成，并获取共享的结果。
5. 并发安全：
  - 通过使用 `sync.Mutex` 来保护对 Group 结构的访问，确保在并发环境中对调用的重复抑制和结果共享是线程安全的。
6. Panic 处理：
  - `singleflight` 考虑了函数调用中发生 panic 的情况。当发生 panic 时，singleflight 会通过 `recover` 恢复 panic，并将其包装成一个 `panicError` 结构。这有助于在发生 panic 时确保等待的调用不会永久阻塞。
7. Forget 方法：
  - `Group` 结构提供了 Forget 方法，用于忘记特定 key 的调用。这样，未来对该 key 的调用将重新执行函数，而不是等待之前的调用完成。
总体而言，singleflight 的设计原理通过在高并发环境下对相同 key 的函数调用进行控制，避免了重复计算，提高了系统效率。同时，它通过结果共享和并发安全性，确保在并发场景下正确处理函数调用。


# 接口签名
`singleflight` 提供了两个主要的接口，分别是 `Do` 方法和 `DoChan` 方法，以及一个辅助方法 `Forget`。下面是对这些接口的简要说明：
1. Do 方法:
``` $golang
func (g *Group) Do(key string, fn func() (interface{}, error)) (v interface{}, err error, shared bool)
```
  - 作用： 执行并返回给定 key 的函数调用的结果。
  - 参数：
    - key: 标识函数调用的唯一键。
    - fn: 要执行的函数，返回一个 interface{} 类型的结果和一个 error。
  - 返回值：
    - v: 函数调用的结果值。
    - err: 函数调用的错误，如果没有错误则为 nil。
    - shared: 一个布尔值，指示结果是否被多个调用共享。
2. DoChan 方法:
```$golang
func (g *Group) DoChan(key string, fn func() (interface{}, error)) <-chan Result
```
  - 作用： 类似于 Do 方法，但返回一个结果channel，通过该channel可以异步获取函数调用的结果。
  - 参数：
    - `key`: 标识函数调用的唯一键。
    - `fn`: 要执行的函数，返回一个 `interface{}` 类型的结果和一个 `error`。
  - 返回值：
    - 返回一个channel (`<-chan Result`)，通过该channel可以获取函数调用的结果。
3. Forget 方法:
func (g *Group) Forget(key string)
  - 作用： 告诉 singleflight 忘记特定 key 的调用，使得未来对该 key 的调用将重新执行函数，而不是等待之前的调用完成。
  - 参数：
    - key: 要忘记的函数调用的键。
这些接口提供了一种机制，可以对相同 key 的函数调用进行重复抑制，并在结果就绪时共享结果。通过提供异步获取结果的通道 (`DoChan`)，`singleflight` 还支持更灵活的结果处理方式。

# 解析源码
包内定义了如下3个关键的结构体
```$golang
type Group struct {
    mu sync.Mutex       // 用于保护 m
    m  map[string]*call // 存储每个 key 对应的调用信息
}

type call struct {
    wg sync.WaitGroup   // 用于等待调用的完成
    val interface{}     // 函数调用的结果值
    err error           // 函数调用的错误
    dups int            // 重复调用的计数
    chans []chan<- Result // 等待结果的通道列表
}

type Result struct {
    Val    interface{} // 函数调用的结果值
    Err    error       // 函数调用的错误
    Shared bool        // 结果是否被多个调用共享
}
```

Group 结构：
- `Group` 结构用于表示一组相关的函数调用，每个调用都有一个唯一的 key。它包含一个互斥锁 mu 用于保护对调用信息的访问，以及一个映射 m 用于存储每个 key 对应的调用信息。

call 结构：
- `call` 结构表示一个函数调用，包含了等待组 wg 用于等待调用的完成，以及相关的结果值 val、错误值 err、重复调用计数 dups 和等待结果的通道列表 chans。
  
Result结构：
- `Result` 结构用于接手函数的调用结果，包括结果值、异常信息，以及是否被多个函数调用是否被共享结果。

## Do 方法
```$golang
func (g *Group) Do(key string, fn func() (interface{}, error)) (v interface{}, err error, shared bool) {
    g.mu.Lock()
    if g.m == nil {
        g.m = make(map[string]*call)
    }
    if c, ok := g.m[key]; ok {
        c.dups++
        g.mu.Unlock()
        c.wg.Wait()

        // ... 省略了对 panic 和 errGoexit 的处理

        return c.val, c.err, true
    }
    c := new(call)
    // ... 省略了初始化等操作
    // 使用双 defer 来区分 panic 和 runtime.Goexit
    defer func() {
        // 处理 runtime.Goexit 的情况
        if !normalReturn && !recovered {
            c.err = errGoexit
        }

        g.mu.Lock()
        defer g.mu.Unlock()
        c.wg.Done()
        if g.m[key] == c {
            delete(g.m, key)
        }

        // 处理 panic 的情况
        if e, ok := c.err.(*panicError); ok {
            // ... 处理 panic 的情况
        } else if c.err == errGoexit {
            // ... 处理 runtime.Goexit 的情况
        } else {
            // 处理正常返回的情况
            for _, ch := range c.chans {
                ch <- Result{c.val, c.err, c.dups > 0}
            }
        }
    }()

    // 执行函数调用
    func() {
        defer func() {
            if !normalReturn {
                if r := recover(); r != nil {
                    c.err = newPanicError(r)
                }
            }
        }()

        c.val, c.err = fn()
        normalReturn = true
    }()

    if !normalReturn {
        recovered = true
    }

    return c.val, c.err, false
}
```

设计细节解释：
1. WaitGroup 的使用：
  - call 结构中的 wg 是一个 sync.WaitGroup，用于等待函数调用的完成。在 Do 方法中，首先通过 g.mu.Lock() 获取对 Group 结构的锁，然后检查是否已经存在相同 key 的调用。如果存在，新的调用会等待现有调用完成，通过 c.wg.Wait() 来实现。这确保了对相同 key 的函数调用只有一个在执行，避免了重复计算。
2. 处理 panic：
  - 使用 defer 和 recover 处理函数调用中的 panic。在 Do 方法中，通过 defer 和 recover 机制，可以处理函数调用中发生的 panic，将其包装成 panicError 结构，并确保在发生 panic 时等待的调用不会永久阻塞。
3. 正常返回的处理：
  - 在正常返回的情况下，通过 defer 中的处理来释放资源、更新状态，并通知等待结果的通道。这样，等待结果的调用就能够在结果就绪时获取共享的结果。
4. 重复调用的计数：
  - 在 call 结构中，使用 dups 来记录重复调用的计数。在结果就绪时，通过 Result 结构的 Shared 字段来指示结果是否被多个调用共享。

## DoChan 方法
```$golang
func (g *Group) DoChan(key string, fn func() (interface{}, error)) <-chan Result {
    ch := make(chan Result, 1)
    g.mu.Lock()
    if g.m == nil {
        g.m = make(map[string]*call)
    }
    if c, ok := g.m[key]; ok {
        c.dups++
        g.m.Unlock()
        c.wg.Add(1)
        c.chans = append(c.chans, ch)
        return ch
    }
    c := new(call)
    c.wg.Add(1)
    c.chans = append(c.chans, ch)
    g.m[key] = c
    g.mu.Unlock()

    go g.doCall(c, key, fn)
    return ch
}
```

- DoChan 方法类似于 Do 方法，但返回一个通道 (chan Result)，通过该通道可以异步获取函数调用的结果。如果发现相同 key 的调用正在进行，新的调用将等待并共享相同的结果。与 Do 方法不同，DoChan 不会等待结果，而是立即返回一个通道，可以在以后获取结果。

1. doCall 方法：
```$golang
func (g *Group) doCall(c *call, key string, fn func() (interface{}, error)) {
    normalReturn := false
    recovered := false

    defer func() {
        if !normalReturn {
            if r := recover(); r != nil {
                c.err = newPanicError(r)
            }
        }

        g.mu.Lock()
        defer g.mu.Unlock()
        c.wg.Done()
        if g.m[key] == c {
            delete(g.m, key)
        }

        // ... 省略了对 panic 和 errGoexit 的处理

        if e, ok := c.err.(*panicError); ok {
            // ... 处理 panic 的情况
        } else if c.err == errGoexit {
            // ... 处理 runtime.Goexit 的情况
        } else {
            // 处理正常返回的情况
            for _, ch := range c.chans {
                ch <- Result{c.val, c.err, c.dups > 0}
            }
        }
    }()

    func() {
        defer func() {
            if !normalReturn {
                if r := recover(); r != nil {
                    c.err = newPanicError(r)
                }
            }
        }()

        c.val, c.err = fn()
        normalReturn = true
    }()

    if !normalReturn {
        recovered = true
    }
}
```

- doCall 方法用于处理单个 key 对应的函数调用。在该方法中，通过 defer 和 recover 机制来处理函数调用中的 panic。在函数执行结束后，根据结果的正常与否，通过 normalReturn 和 recovered 来判断是否发生了 panic，并进行相应的处理。最终，在结果就绪时通知等待结果的通道。

这些设计细节保证了 singleflight 在高并发环境下对相同 key 的函数重复调用进行抑制，并在结果就绪时进行合适的处理，同时处理了 panic 和 runtime.Goexit 等情况，确保代码的正确性和稳定性。

# 代码使用
singleflight的实现代码算上注释才215行。但是在实际使用时并不容易使用正确。

## 如何获取包
singleflight虽然已经发布多年，但依然在golang.org/x/sync/singleflight 项目下以便及时获取社区反馈和快速演进。
因此如需使用singleflight包，需要执行go get -u golang.org/x/sync/singleflight 来获取最新的包。

## 如何使用singleflight
### 场景
C端商品页面需要查询某热门商品信息，SPUID是P1002，高峰期QPS5000。

### 调用链路
C端页面 -> 商品中台 -> 商品详情

### 思路
#### 优化前
所有请求从C端页面到商品中台再全部透传给商品详情，商品详情服务需要承接住5000QPS的请求。
#### 优化后
请求从C端页面到商品中台，商品中台聚合请求放行至多X个请求到商品详情（X数量为商品中台节点数量）

### 代码DEMO
#### ⚠️ 非预期用法
```$golang
func main() {
    var g singleflight.Group
    for i := 0; i < 10; i++ {
       v, _, b := g.Do("spuId", func() (interface{}, error) {
          res := fetchData()
          return res, nil
       })
       fmt.Println(v, b)
    }
}

func fetchData() string {
    time.Sleep(time.Second * 1) // 模拟耗时操作
    return uuid.New().String()
}
```

##### 输出
```
ef3935ba-6dc2-45ea-bc01-81db30c4b01b false
11cef141-5dfd-4409-b2cc-4d43f0aaaae4 false
d81ca745-2fbd-4b71-9278-4b6eefc8552f false
4f36c79b-a779-431e-b2b3-03fcda0ca050 false
06d5dc75-4636-4d05-90c4-711e9d1f5860 false
9c091e82-9f2b-4a39-9092-b17e12175465 false
f36fa8bf-e05d-4f46-bb3c-94f72512e2be false
1922c34f-8d5b-45c4-9a4a-1852724009b5 false
da82323a-9949-4348-ade2-b6fd440799f5 false
d89e03dc-3b07-4852-9157-21584a39f6e3 false
```
从代码中可以看出，期望是多个请求可以合并走一次fetchData。但是由于上下请求是串行，并没有请求周期上的交叉，所以上一个请求的数据并不会像缓存一样给到下一个请求使用。

#### ✅ 预期用法
```$golang
func main() {
    var (
       g singleflight.Group
       w sync.WaitGroup
    )
    for i := 0; i < 10; i++ {
       w.Add(1)
       go func() {
          defer w.Done()
          v, _, b := g.Do("spuId", func() (interface{}, error) {
             res := fetchData()
             return res, nil
          })
          fmt.Println(v, b)
       }()
    }
    w.Wait()
}

func fetchData() string {
    time.Sleep(time.Second * 1)
    return uuid.New().String()
}
```

##### 输出
```
2791154e-91bc-4121-921b-035a607edee0 true
2791154e-91bc-4121-921b-035a607edee0 true
2791154e-91bc-4121-921b-035a607edee0 true
2791154e-91bc-4121-921b-035a607edee0 true
2791154e-91bc-4121-921b-035a607edee0 true
2791154e-91bc-4121-921b-035a607edee0 true
2791154e-91bc-4121-921b-035a607edee0 true
2791154e-91bc-4121-921b-035a607edee0 true
2791154e-91bc-4121-921b-035a607edee0 true
2791154e-91bc-4121-921b-035a607edee0 true
```
# 应用场景
`singleflight` 主要用于在高并发环境下对相同 key 的函数抑制重复调用，以减少重复计算和提高性能。以下是一些适合使用 singleflight 的应用场景：
1. 缓存填充：
  - 当多个 goroutine 需要同时获取某个数据的时候，可以使用 `singleflight` 来避免同时发起多次相同的数据请求。这样可以减轻后端服务的压力，避免重复计算。
2. 资源加载：
  - 当多个请求需要加载相同的资源（例如文件、图片等）时，可以使用 `singleflight` 来避免并发地重复加载相同的资源。这有助于提高系统性能，并减少资源的重复加载。
3. 网络请求合并：
  - 当多个请求需要向相同的远程服务发起请求时，可以使用 `singleflight` 来避免并发地发送相同的请求。这可以减轻远程服务的压力，提高请求的响应速度。
4. 数据库查询：
  - 在数据库查询场景中，当多个请求需要查询相同的数据时，可以使用 `singleflight` 来避免多次并发地查询相同的数据。这有助于降低数据库负载，提高查询效率。
5. 限流控制：
  - 在一些特定场景下，需要限制对某个资源的并发访问。使用 `singleflight` 可以有效地控制对资源的并发访问，确保不会同时进行多次相同的操作。

## 实际业务的使用
上文例子中讲到的商品信息可以使用singleflight，另外目前用户量很大的小说网站在此方面的应用效果是非常明显的。在请求周期内如有相同请求进入，则可以降低非常多的内网请求量和带宽。

# 注意事项
## 使用
特别需要注意，singleflight只能代替缓存获取的过程，但不能代替缓存。singleflight的生命周期是Do函数的第二位fn函数的执行周期，数据共享也仅能在这个周期内是有效的。如果理解为是缓存，则可能带来未知的问题。
## 包约束
singleflight目前归属在golang官方的x项目golang.org/x中，供 Go 语言团队和社区成员在不受标准库稳定性和向后兼容性要求的情况下，进行实验、尝试新功能和设计，一般来说会兼容支持前两个版本。所以在使用时需要关注是否有接口的变更。

[https://go.dev/wiki/X-Repositories](https://go.dev/wiki/X-Repositories)
