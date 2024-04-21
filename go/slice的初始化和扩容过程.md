---
title: slice的初始化和扩容过程
seo_title: slice的初始化和扩容过程
categories:
  - go语言系列
book: 柠芒技术博客
book_title: slice的初始化和扩容过程
date: 2022-01-16 19:59:30
description:
github: https://github.com/jinfeijie/posts
github_page: https://github.com/jinfeijie/posts/blob/master/go/slice%E7%9A%84%E5%88%9D%E5%A7%8B%E5%8C%96%E5%92%8C%E6%89%A9%E5%AE%B9%E8%BF%87%E7%A8%8B.md
id: post-20220116195930
---

切片和数组在go语言中都是非常常见的结构，很多刚开始使用go语言的开发者会混淆这两者的概念而留下不少隐藏的bug。本篇会从slice编译期、运行时来分析，以更好理解和掌握slice。

### slice概述

slice表示一个拥有相同数据类型的可变长度的序列。
该序列通过一个较为简单的数据结构实现：[https://github.com/golang/go/blob/master/src/runtime/slice.go#L15](https://github.com/golang/go/blob/master/src/runtime/slice.go#L15)

```go
type slice struct {
  array unsafe.Pointer
  len   int
  cap   int
}
```

### 编译期逻辑

> 编译的整个过程另外开文章说，下面直接讲slice的编译过程。 感兴趣可以先看 [`src/cmd/compile/main.go`](https://go.dev/src/cmd/compile/main.go)

在slice的编译过程中，会调用`NewSlice`方法，该方法需传入类型为[`*Type`](https://github.com/golang/go/blob/go1.17/src/cmd/compile/internal/types/type.go#L137)的参数`elem`，并且返回一个`*Type`类型的结构体。

```go
// NewSlice returns the slice Type with element type elem.
func NewSlice(elem *Type) *Type {
	if t := elem.cache.slice; t != nil {
		if t.Elem() != elem {
			base.Fatalf("elem mismatch")
		}
		if elem.HasShape() != t.HasShape() {
			base.Fatalf("Incorrect HasShape flag for cached slice type")
		}
		return t
	}

	t := newType(TSLICE)
	t.extra = Slice{Elem: elem}
	elem.cache.slice = t
	if elem.HasShape() {
		t.SetHasShape(true)
	}
	return t
}
```

进入`Slice{Elem: elem}`，slice的数据结构如下：

```go
// Slice contains Type fields specific to slice types.
type Slice struct {
  Elem *Type // element type
}

type Type struct {
	// extra contains extra etype-specific fields.
	// As an optimization, those etype-specific structs which contain exactly
	// one pointer-shaped field are stored as values rather than pointers when possible.
	//
	// TMAP: *Map
	// TFORW: *Forward
	// TFUNC: *Func
	// TSTRUCT: *Struct
	// TINTER: *Interface
	// TFUNCARGS: FuncArgs
	// TCHANARGS: ChanArgs
	// TCHAN: *Chan
	// TPTR: Ptr
	// TARRAY: *Array
	// TSLICE: Slice
	// TSSA: string
	extra interface{}
 ……
}
```

从代码中可知，返回的`*Type`类型的结构体中 `extra` 字段是一个只包含切片内元素类型的结构（针对包含特定类型字段的结构体进行优化存储。如果这些结构体中包含恰好一个指针类型的字段，那么在可能的情况下会将其存储为值而不是指针）。所以在编译期时切片内元素的类型就确定下来了，在运行中往切片中塞入其他类型的数据就引发异常。

### 运行时（runtime）

#### 创建一个新的切片

slice在运行时，会通过`runtime.makeslice`在内存上开辟一块连续的空间。

```diff
func makeslice(et *_type, len, cap int) unsafe.Pointer {
	mem, overflow := math.MulUintptr(et.Size_, uintptr(cap))
	if overflow || mem > maxAlloc || len < 0 || len > cap {
+		// NOTE: Produce a 'len out of range' error instead of a
+		// 'cap out of range' error when someone does make([]T, bignumber).
+		// 'cap out of range' is true too, but since the cap is only being
+		// supplied implicitly, saying len is clearer.
+		// See golang.org/issue/4085.
		mem, overflow := math.MulUintptr(et.Size_, uintptr(len))
		if overflow || mem > maxAlloc || len < 0 {
			panicmakeslicelen()
		}
		panicmakeslicecap()
	}

	return mallocgc(mem, et, true)
}
```

如代码所示，通过[`math.MulUintptr`](https://github.com/golang/go/blob/master/src/runtime/internal/math/math.go#L13)函数计算出占用空间的大小，以及是否内存溢出。
他内部的计算公式为 `et.size * cap`。翻译一下就是`开辟的内存大小 = slice内部元素的大小 * 容量`作为起始的内存。

从代码里我们可以得知三个信息：

1. 当数据长度超过slice容量时 或者长度小于0 会抛异常
2. 当内存开辟超过最大可分配的内容时会抛异常
3. 当内存溢出时会抛异常

所以在交叉编译时可能会引入一些跨平台/机器的问题。举个例子，假设以上三点在编译期就固定了。在32G内存的机器上编译了代码，输出了可执行文件。文件被复制到8G内存的机器上运行，当占用内存不断增长，超过8g将会出现内存溢出的问题。

在完成前置判断后程序开始开辟内存。开辟内存的方式逻辑如下，先初始化一个P，如果对象需要开辟的空间小于32768（32K），则直接在初始化在P上。而超过32K的，就在堆上开辟空间。（go里面内存开辟几乎都使用这个逻辑）
> 感兴趣可以查看 [https://github.com/golang/go/blob/master/src/runtime/malloc.go#L909](https://github.com/golang/go/blob/master/src/runtime/malloc.go#L909)

另外，与`makeslice`类似的函数`makeslice64`，会在开辟较大空间的时候使用（较大空间是指超出了int范围）。需要注意的是`makeslice64`虽然在函数名上标记了int64的长度，实际上`make([]int64,math.MaxInt64,math.MaxInt64)`是会抛 `panic: runtime error: makeslice: len out of range`异常的。实际容量需要根据系统架构和元素大小情况计算。

#### 切片扩容

slice是一个可变长度的序列的，支持`append`方法往里面新增数据。当数据量达到了当前切片最大容量时，就不得不进行扩容。

在进行数据新增并且新增后超出容量时就会调用函数[`runtime.growslice`](https://github.com/golang/go/blob/master/src/runtime/slice.go#L155-L264)。

截取关键代码，梳理一下扩容逻辑。

```golang

// 小于1.22版本 
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
2. 当前容量小于1024，则进行双倍扩容(***注意1.18后时256***)
3. 当当前容量大于1024（既>=2048），则新容量是当前容量的1.25倍(***注意1.18后时256，>=512***)
4. 当新容量溢出了，则不进行扩容

确认了最新容量后，则进行内存对齐。通过对元素的大小（et.size）的判断，对了内存对齐。通过数组`class_to_size`拿到对齐的值。

然后通过对齐后的值进行内存开辟，复制原内存内的数据到新开辟的内存上，再内存地址放到新的Slice结构体上返回。

> review代码看到1.18版本后将1024调整为256。在1.22版本抽象了计算新容量的方法 [`nextslicecap`](https://github.com/golang/go/commit/1d84b02b228cbd35660e168d26fd2801daed08fe)

```
// 1.22 版本抽象了计算新容量的方法
func nextslicecap(newLen, oldCap int) int {
	newcap := oldCap
	doublecap := newcap + newcap
	if newLen > doublecap {
		return newLen
	}

	const threshold = 256
	if oldCap < threshold {
		return doublecap
	}
	for {
		// Transition from growing 2x for small slices
		// to growing 1.25x for large slices. This formula
		// gives a smooth-ish transition between the two.
		newcap += (newcap + 3*threshold) >> 2

		// We need to check `newcap >= newLen` and whether `newcap` overflowed.
		// newLen is guaranteed to be larger than zero, hence
		// when newcap overflows then `uint(newcap) > uint(newLen)`.
		// This allows to check for both with the same comparison.
		if uint(newcap) >= uint(newLen) {
			break
		}
	}

	// Set newcap to the requested cap when
	// the newcap calculation overflowed.
	if newcap <= 0 {
		return newLen
	}
	return newcap
}

```
在1.22版本后切片的扩容机制变更为

1. 初始化变量：函数接受两个参数，newLen 表示切片的新长度，oldCap 表示切片的旧容量。开始时将 newcap 初始化为 oldCap。

2. 判断是否需要直接扩容至新长度：首先计算 doublecap，即旧容量的两倍。如果新长度大于 doublecap，则直接返回新长度，因为此时直接扩容到新长度即可。

3. 阈值判断：定义了一个阈值常量 threshold，其值为 256。如果旧容量小于该阈值，那么新容量直接设置为 doublecap。

4. 循环计算新容量：对于大于等于阈值的旧容量，采用一种新的扩容策略，即每次增加 newcap 的 1.25 倍，直到 newcap 大于等于 newLen。

5. 溢出检查：通过将 newcap 强制转换为 uint 类型进行溢出检查。如果 newcap 溢出，则直接返回新长度。

6. 返回新容量：最后返回新容量，如果新容量小于等于 0，则返回新长度，以防溢出。

这个机制在处理切片扩容时，尤其是针对大容量的切片，可以更加有效地管理内存，避免频繁的内存分配和拷贝操作，从而提高性能。


#### 切片拷贝

切片拷贝在切片的几个方法中算使用频率较低的一个方式。`func copy(dst, src []Type) int`。

在数据发生复制时，通过调用`slicecopy`就可以实现数据复制。而`slicecopy`方法的实现，也比初始化和扩容简单很多。如果原数据大小只有1，则直接进行了指针赋值。其他情况则时调用了`memmove`执行了内存数据复制。

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
