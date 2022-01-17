---
title: slice的初始化和扩容过程
seo_title: slice的初始化和扩容过程
categories:
  - go语言系列
book: Strcpy
book_title: slice的初始化和扩容过程
date: 2022-01-16 19:59:30
description:
github: https://github.com/jinfeijie/posts
github_page: https://github.com/jinfeijie/posts/blob/master/go/slice%E7%9A%84%E5%88%9D%E5%A7%8B%E5%8C%96%E5%92%8C%E6%89%A9%E5%AE%B9%E8%BF%87%E7%A8%8B.md
id: post-20220116195930
---

切片和数组数组在go语言中都是非常常见的结构。很多刚开始使用go语言的开发者会混淆这两者的概念。

本篇会从slice的编译期、运行时来分析slice。其中会包含slice的初始化，数据新增，动态扩容，数据拷贝等几个常见的操作。

### slice概述

slice表示一个拥有相同数据类型的可变长度的序列。
该序列通过一个较为简单的数据结构实现：<https://github.com/golang/go/blob/master/src/runtime/slice.go#L15>

```go
type slice struct {
 array unsafe.Pointer
 len   int
 cap   int
}
```

### 编译期逻辑

> 编译的整个过程另外开文章说，下面直接讲slice的编译过程。 感兴趣可以先看 [`src/cmd/compile/main.go`](https://go.dev/src/cmd/compile/main.go)

在slice的编译过程中，会调用到`NewSlice`方法，这个方法支持传入一个参数`elem`，指定类型为[`*Type`](https://github.com/golang/go/blob/go1.17/src/cmd/compile/internal/types/type.go#L137)，并且返回一个`*Type`类型的结构体。

```go
// NewSlice returns the slice Type with element type elem.
func NewSlice(elem *Type) *Type {
 if t := elem.Cache.slice; t != nil {
  if t.Elem() != elem {
   Fatalf("elem mismatch")
  }
  return t
 }

 t := New(TSLICE)
 t.Extra = Slice{Elem: elem}
 elem.Cache.slice = t
 return t
}
```

进入`Slice{Elem: elem}`，可以知道在编译期，slice的数据结构如下：

```go
// Slice contains Type fields specific to slice types.
type Slice struct {
 Elem *Type // element type
}
```

由此可知，返回的`*Type`类型的结构体中的 `Extra` 字段是一个只包含切片内元素类型的结构。所以在编译期时，切片内元素的类型就确定下来了，而不是在运行时数据被添加进去的时候。

### 运行时（runtime）

#### 创建一个新的切片

slice在运行时，会通过`runtime.makeslice`在内存上开辟一块连续的空间。

```go
func makeslice(et *_type, len, cap int) unsafe.Pointer {
 mem, overflow := math.MulUintptr(et.size, uintptr(cap))
 if overflow || mem > maxAlloc || len < 0 || len > cap {
  mem, overflow := math.MulUintptr(et.size, uintptr(len))
  if overflow || mem > maxAlloc || len < 0 {
   panicmakeslicelen()
  }
  panicmakeslicecap()
 }

 return mallocgc(mem, et, true)
}
```

如代码所示，通过[`math.MulUintptr`](https://github.com/golang/go/blob/go1.17/src/runtime/internal/math/math.go#L13)函数计算出占用空间的大小，以及是否内存溢出。
他内部的计算公式为 `et.size * cap`。翻译一下就是`开辟的内存大小 = slice内部元素的大小 * 容量作为起始的内存`。

从代码里我们可以得知三个信息：

1. 当数据长度超过slice容量时 或者长度小于0 会抛异常
2. 当内存开辟超过最大可分配的内容时会抛异常
3. 当内存溢出时会抛异常

所以并不是什么问题都是在编译期就可以解决掉的。
举个例子，假设以上三点在编译期就固定了。在32G内存的机器上编译了代码，输出了可执行文件。文件被复制到8G内存的机器上运行，当占用内存不断增长，超过8g将会出现无法预估的问题。

在做完一系列判断后就开始开辟内存。开辟内存的方式大概逻辑如下，先初始化一个P，如果对象需要开辟的空间小于32768（32K），则直接在初始化在P上。而超过32K的，就在堆上开辟空间。（go里面内存开辟几乎都使用这个逻辑）
> 感兴趣可以查看 [https://github.com/golang/go/blob/master/src/runtime/malloc.go#L909](https://github.com/golang/go/blob/master/src/runtime/malloc.go#L909)

另在与`makeslice`类似的函数`makeslice64`，会在开辟较大空间的时候使用。较大空间时指超出了int范围。需要注意的是，`makeslice64`虽然在函数名上有int64的长度，实际上`make([]int64,math.MaxInt64,math.MaxInt64)`是会抛 `panic: runtime error: makeslice: len out of range`异常的。实际容量需要根据系统架构和元素大小情况计算。

#### 切片扩容

slice是一个可变长度的序列的，支持`append`方法往里面新增呢个数据。当数据量达到了当前容量时，就不得不进行扩容。

在进行数据新增，并且新增后超出容量时就会调用函数[`runtime.growslice`](https://github.com/golang/go/blob/master/src/runtime/slice.go#L166-L288)。

截取关键代码，梳理一下扩容逻辑。

```go
func growslice(et *_type, old slice, cap int) slice {
 newcap := old.cap
 doublecap := newcap + newcap
 if cap > doublecap {
  newcap = cap
 } else {
  if old.cap < 1024 {
   newcap = doublecap
  } else {
   // Check 0 < newcap to detect overflow
   // and prevent an infinite loop.
   for 0 < newcap && newcap < cap {
    newcap += newcap / 4
   }
   // Set newcap to the requested cap when
   // the newcap calculation overflowed.
   if newcap <= 0 {
    newcap = cap
   }
  }
 }
}
```

扩容逻辑如下：

1. 当目标容量大于当前容量的2倍，则使用目标容量
2. 当前容量小于1024，则进行双倍扩容
3. 当当前容量大于1024（既>=2048），则新容量是当前容量的1.25倍
4. 当新容量溢出了，则不进行扩容

确认了最新容量后，则进行内存对齐。通过对元素的大小（et.size）的判断，对了内存对齐。通过数组`class_to_size`拿到对齐的值。

然后通过对齐后的值进行内存开辟，复制原内存内的数据到新开辟的内存上，再内存地址放到新的Slice结构体上返回。

#### 切片拷贝

切片拷贝在切片的几个方法中算使用频率教低的一个方式。`func copy(dst, src []Type) int`。

在数据发生复制时，通过调用`slicecopy`就可以实现数据复制。而`slicecopy`方法的实现，也比初始化和扩容简单很多。如果远数据大小只有1，则直接进行了指针赋值。其他情况则时调用了`memmove`执行了内存数据复制。

```go
 if size == 1 { // common case worth about 2x to do here
  // TODO: is this still worth it with new memmove impl?
  *(*byte)(toPtr) = *(*byte)(fromPtr) // known to be a byte pointer
 } else {
  memmove(toPtr, fromPtr, size)
 }
```

相比于循环遍历切片，再往新的切片上复制，memmove性能上占据了更大的优势。但是在较大容量的切片复制上，通过此方法会开辟出大量的内存空间。在使用的时候需要注意对内存性能的影响。

### 总结

slice因为是动态序列，不管是初始化切片，新增切片，扩容切片，复制切片都是依赖于运行时(runtime)。在运行时发生的行为，需要注意切片在扩容和拷贝过程中发生内存大规模的开辟，避免对程序的性能造成太大的影响。