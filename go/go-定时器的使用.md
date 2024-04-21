---
title: go 定时器的使用
seo_title: go 定时器的使用
categories:
  - go语言系列
book: 柠芒技术博客
book_title: go 定时器的使用
date: 2021-12-20 02:42:21
description:
github: https://github.com/jinfeijie/posts
github_page: https://github.com/jinfeijie/posts/blob/master/go/go-%E5%AE%9A%E6%97%B6%E5%99%A8%E7%9A%84%E4%BD%BF%E7%94%A8.md
id: post-842
---

在执行定时任务的场景，每隔几秒钟就会执行一次，触发任务。
go提供了time包，用来实现延时，只需要循环执行延时，就可以实现定时器功能。
但是关于它的使用和实现，需要具体分析一下。

### 实现原理
go的time包里面提供了两个函数用于延时，分别是`func After(d Duration) <-chan Time `和`func Tick(d Duration) <-chan Time`。而他们分别调用`func NewTimer(d Duration) *Timer`和`func NewTicker(d Duration) *Ticker`来实现这个延时的。两个函数的实现差异只在变量`t`的赋值上，`NewTimer`使用`Timer`实现，`NewTicker`使用`Ticker`实现。两个结构体的数据结构完全一样，因此他们实际实现的效果也是完全一致的。


`NewTicker`的实现如下
```golang
func NewTicker(d Duration) *Ticker {
	if d <= 0 {
		panic(errors.New("non-positive interval for NewTicker"))
	}
	// Give the channel a 1-element time buffer.
	// If the client falls behind while reading, we drop ticks
	// on the floor until the client catches up.
	c := make(chan Time, 1)
	t := &Ticker{
		C: c,
		r: runtimeTimer{
			when:   when(d),
			period: int64(d),
			f:      sendTime,
			arg:    c,
		},
	}
	startTimer(&t.r)
	return t
}
```

Ticker实现
```golang
// A Ticker holds a channel that delivers `ticks' of a clock
// at intervals.
type Ticker struct {
	C <-chan Time // The channel on which the ticks are delivered.
	r runtimeTimer
}
```

Ticker实现只有两个参数：
C: 管道，上层应用根据此管道接收事件；
r: runtime定时器，该定时器即系统管理的定时器，对上层应用不可见；
这里应该按照层次来理解Timer数据结构，Timer.C即面向Timer用户的，Timer.r是面向底层的定时器实现。

`Ticker.C`管道每隔一个延时周期就会接收到一个事件，用于延时任务的触发。

### 实现一个延时
```golang
func main() {
	fmt.Println(time.Now().Unix())
	<-time.Tick(1*time.Second)
	fmt.Println(time.Now().Unix())
}
```

### 实现一个定时器
```golang
func main() {
	timer := time.NewTicker(time.Second)
	for   {
		<-timer.C
		fmt.Println(time.Now().Unix())
	}
}
```

### 错误示例
```golang
func main() {
	for   {
		select {
		case <-time.After(time.Second):
			fmt.Println(time.Now().Unix())
		}
	}
}
```

把`time.After`内的方法实现一遍，输出每次的定时器的内存地址
```golang
func main() {
	for   {
		select {
		case <-(func() <-chan time.Time {
			t := time.NewTicker(time.Second)
			fmt.Printf("%p\n", t)
			return t.C
		})():

		}
	}
}
```
可以看到，每一次调用方法都会生成一个定时器，不断开辟内存空间。

### 正确示例
```golang
func main() {
	t := time.NewTicker(time.Second)
	for   {
		select {
		case <-t.C:
			fmt.Printf("%p\n", t)
		}
	}
}
```

## 结尾
重启可以解决99.9%的问题？！
