---
layout: post
title:  "Golang Memory Model"
date:   2021-08-29 16:12:53 +0800
categories: golang server
---
## Golang内存模型是什么？

提到内存模型，一般直觉上会和运行时的内存分配关联起来，但其实这两者并不是同一个东西，内存模型定义的是语言在运行时，多线程（在Golang中是协程）对于内存的读写操作行为，并描述了这些行为产生的现象。Golang的内存模型主要描述的就是Goroutine之间能够读到对方写操作的条件。

（比较有意思的是，Golang官网的memory model文章作者似乎并不希望读者来看，他认为如果读者需要通过memory model才能理解程序的运行方式是不合理的）

## Happens-Before 关系

在单个Goroutine中，程序的运行结果是符合程序定义的读写操作顺序的。但实际在不影响运行结果的前提下，编译器和处理器都可能对程序中的读写指令进行重排序。这导致了，当你想要用一个Goroutine去观测另一个Goroutine的运行时，结果可能会和程序定义的不同。

例如，一个Goroutine中定义以下行为：

```go
a = 1
b = 2
```

另一个Goroutine可能会看到b相比a先被进行了赋值。

为了描述内存读写操作的规律，happens-before关系就被定义出来，描述内存操作的偏序关系。

> 以下全文用 ‘<’ 来代表happes-before关系，用 '>' 来代表happens-before关系，用 '=' 来代表并发关系（无法确定执行顺序先后），e（{op}, {variable}) 代表针对variable执行op的事件

在这样的框架下，e(r, v) 想要观测到 e(w, v)，需要满足以下条件：

1. e(r, v) >= e(w, v)
2. 不存在e2(w, v)，满足e2(w, v) < e(r, v) and e2(w, v) > e(w, v)

但这个观测结果是不确定的，因为存在e(r, v) = e(w, v)，e2(w, v) = e(w, v), e2(w, v) = e(r, v)的可能。
如果想确保e(r, v)观测到e(w, v)，需要满足以下条件：

1. e(r, v) > e(w, v)
2. 不存在e2(w, v)，满足e2(w, v) <= e(r, v) and e2(w, v) >= e(w, v)

Tips：

1. 在golang中，申明变量也会被视为一次写入，

2. 针对超过一个寄存器长度的数据读写，会分为多次无序的操作完成。

## 同步

### 初始化

Golang的初始化动作是由一个routine完成的。
包中的init函数执行顺序可以理解成是一个栈，每次一个包被引用，就向栈内压入这个包，所有包都被压入栈，就开始出栈和执行init函数，因此被引用包的初始化会先于引用包的初始化。
main函数会在所有init函数执行完成后被运行。
如果在init()函数中创建了一个新的routine，它会和执行初始化动作的routine并发运行。

### routine
如果一个routine的创建没有跟着任何的同步动作（wait、unlock、channel等），那它就可能不被执行（比如程序提前退出），甚至一些激进的编译器会直接去掉这个routine的创建动作。
