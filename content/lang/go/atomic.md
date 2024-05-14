+++
title = '原子操作'
date = 2024-05-13T21:19:52+08:00
+++

## 原子操作

package `sync/atomic` 实现了同步算法底层的原子的内存操作原语，我们把它叫做**原子操作原语**，它提供了一些实现原子操作的方法。

> 原子操作：一个原子操作在执行的时候，其它线程不会看到执行一半的操作结果。在其它线程看来，原子操作要么执行完了，要么还没有执行。

CPU 提供了基础的原子操作，不过，不同架构的系统的原子操作是不一样的。

> 处理器（CPU）：中央处理单元（物理芯片），处理器可以包含一个或多个核心，每个核心可以独立执行任务。简而言之，核心是CPU内部的一个独立的计算单元。
>
> 核心（Core）：处理器内部的一个**独立单元**，能**独立执行**计算任务。每个核心都可以视为一个小型的处理器，但它们**共享某些资源**，如缓存（Cache）和输入/输出系统等。
>
> 理论上核数代表**并行度**。

对于单处理器单核系统来说，如果一个操作是由一个 CPU 指令来实现的，那么它就是原子操作，比如它的 XCHG 和 INC 等指令。如果操作是基于多条指令来实现的，那么，执行的过程中可能会被中断，并执行上下文切换，这样的话，原子性的保证就被打破了，因为这个时候，操作可能只执行了一半。

在多处理器多核系统中，原子操作的实现就比较复杂了。

由于 cache 的存在，单个核上的单个指令进行原子操作的时候，你要**确保其它处理器或者核不访问此原子操作的地址**，或者是确保其它处理器或者核总是访问原子操作之后的**最新的值**。x86 架构中提供了指令前缀 LOCK，LOCK 保证了指令（比如 LOCK CMPXCHG op1、op2）不会受其它处理器或 CPU 核的影响，有些指令（比如 XCHG）本身就提供 Lock 的机制。不同的 CPU 架构提供的原子操作指令的方式也是不同的，比如对于多核的 MIPS 和 ARM，提供了 LL/SC（Load Link/Store Conditional）指令，可以帮助实现原子操作（ARMLL/SC 指令 LDREX 和 STREX）。

**因为不同的 CPU 架构甚至不同的版本提供的原子操作的指令是不同的，所以，要用一种编程语言实现支持不同架构的原子操作是相当有难度的。**不过，还好这些都不需要你操心，因为 Go 提供了一个通用的原子操作的 API，将更底层的不同的架构下的实现封装成 atomic 包，提供了修改类型的原子操作（[atomic read-modify-write](https://preshing.com/20150402/you-can-do-any-kind-of-atomic-read-modify-write-operation/)，RMW）和加载存储类型的原子操作（[Load 和 Store](https://preshing.com/20130618/atomic-vs-non-atomic-operations/)）的 API。

有的代码也会因为架构的不同而不同。有时看起来貌似一个操作是原子操作，但实际上，对于不同的架构来说，情况是不一样的。比如下面的代码的第 4 行，是将一个 64 位的值赋值给变量 i：

```go
const x int64 = 1 + 1<<33

func main() {
    var i = x  // 第 4 行
    _ = i
}
```

如果你使用 GOARCH=386 的架构去编译这段代码，那么，第 5 行其实是被拆成了两个指令，分别操作低 32 位和高 32 位（使用 GOARCH=386 go tool compile -N -l test.go；GOARCH=386 go tool objdump -gnu test.o 反编译试试）：

![img.png](/images/lang/go/atomic-1.png)

如果 GOARCH=amd64 的架构去编译这段代码，那么，第 5 行其中的赋值操作其实是一条指令：

![img.png](/images/lang/go/atomic-2.png)

所以，如果要想保证原子操作，**切记一定要使用 atomic 提供的方法**。

## 应用场景

使用 atomic 的一些方法，我们可以实现更底层的一些优化。如果使用 Mutex 等并发原语进行这些优化，虽然可以解决问题，但是这些并发原语的实现逻辑比较复杂，对**性能还是有一定的影响**的。

假设你想在程序中使用一个标志（flag，比如一个 bool 类型的变量），来标识一个定时任务是否已经启动执行了。

我们先来看看加锁的方法。如果使用 Mutex 和 RWMutex，在读取和设置这个标志的时候加锁，是可以做到互斥的、保证同一时刻只有一个定时任务在执行的，所以使用 Mutex 或者 RWMutex 是一种解决方案。

其实，这个场景中的问题**不涉及到对资源复杂的竞争逻辑，只是会并发地读写这个标志，这类场景就适合使用 atomic 的原子操作**。具体怎么做呢？你可以使用一个 `uint32` 类型的变量，如果这个变量的值是 0，就标识没有任务在执行，如果它的值是 1，就标识已经有任务在完成了。

> Q：为什么都是使用 `uint32` 类型？`uint8`占用的空间不是更加小吗？
>
> A：
>
> + **原子操作的支持**：atomic 对 `uint32` 有直接支持，没有对 `uint8` 提供原子操作支持
> + **内存对齐和效率**：`uint32` 类型通常可以保证在多种平台上良好的内存对齐，这对于有效的原子操作是必要的。不正确的内存对齐可能会导致效率低下甚至在某些平台上无法正常工作。

假设你在开发应用程序的时候，需要从配置服务器中读取一个节点的配置信息。而且，在这个节点的配置发生变更的时候，你需要重新从配置服务器中拉取一份新的配置并更新。你的程序中可能有多个 goroutine 都依赖这份配置，涉及到对这个配置对象的并发读写，你可以使用读写锁实现对配置对象的保护。在大部分情况下，你也可以利用 atomic 实现配置对象的更新和加载。

**有时候，你也可以使用 atomic 实现自己定义的基本并发原语**。比如 Go issue 有人提议的 CondMutex、Mutex.LockContext、WaitGroup.Go 等，我们可以使用 atomic 或者基于它的更高一级的并发原语去实现。包括 Mutex 也是使用 atomic 实现的。

除此之外，**atomic 原子操作还是实现 lock-free 数据结构的基石**。

> Q: 什么是 lock-free 数据结构？
>
> A：Lock-free数据结构是**一种设计目标是在并发环境中无需使用锁（Lock）就能保证线程安全的数据结构**。

在实现 lock-free 的数据结构时，我们可以不使用互斥锁，这样就**不会让线程因为等待互斥锁而阻塞休眠**，而是让线程保持继续处理的状态。另外，不使用互斥锁的话，lock-free 的数据结构还可以提供并发的性能。

不过，lock-free 的数据结构实现起来比较复杂，需要考虑的东西很多，这里是微软专家写的一篇经验分享：[Lockless Programming Considerations for Xbox 360 and Microsoft Windows](https://learn.microsoft.com/zh-cn/windows/win32/dxtecharts/lockless-programming)

## atomic 提供的方法

**atomic 操作的对象是一个地址，你需要把可寻址的变量的地址作为参数传递给方法，而不是把变量的值传递给方法。**

### Go1.19之前的版本支持的原子操作概述

对于一个整数类型`T`，`sync/atomic`标准库包提供了下列原子操作函数。 其中`T`可以是内置`int32`、`int64`、`uint32`、`uint64`和`uintptr`类型。

```go
func AddT(addr *T, delta T)(new T)
func LoadT(addr *T) (val T)
func StoreT(addr *T, val T)
func SwapT(addr *T, new T) (old T)
func CompareAndSwapT(addr *T, old, new T) (swapped bool)
```

#### Add

Add 方法就是给第一个参数地址中的值增加一个 delta 值。

对于有符号的整数来说，delta 可以是一个负数，相当于减去一个值。对于无符号的整数和 uinptr 类型来说，怎么实现减去一个值呢？毕竟，atomic 并没有提供单独的减法操作。

1. 第二个实参为类型为`T`的一个**变量**值`v`。 因为`-v`在Go中是合法的，所以`-v`可以直接被用做`AddT`调用的第二个实参。
2. 第二个实参为一个正整数常量`c`，这时`-c`在Go中是编译不通过的，所以它不能被用做`AddT`调用的第二个实参。 这时我们可以使用`^T(c-1)`（仍为一个正数）做为`AddT`调用的第二个实参。

此`^T(v-1)`小技巧对于无符号类型的变量`v`也是适用的，但是**`^T(v-1)`比`T(-v)`的效率要低**。

对于这个`^T(c-1)`小技巧，如果`c`是一个类型确定值并且它的类型确实就是`T`，则它的表示形式可以简化为`^(c-1)`。

```go
	var a uint32 = 10
	var d uint32 = 4
	atomic.AddUint32(&a, -d)
	fmt.Println(a)
```

**利用计算机补码的规则，把减法变成加法**。也就是第二种方式，以 uint32 类型为例：

```go
AddUint32(&x, ^uint32(c-1))
// C 就是减几
```

如果是对 uint64 的值进行操作，那么，就把上面的代码中的 uint32 替换成 uint64。

尤其是减 1 这种特殊的操作，我们可以简化为：

```go
AddUint32(&x, ^uint32(0))
```

#### CAS

在 CAS 的方法签名中，需要提供要操作的地址、原数据值、新值，如下所示：

这个方法会**比较当前 addr 地址里的值是不是 old**，如果不等于 old，就返回 false；如果等于 old，就把此地址的值替换成 new 值，返回 true。这就相当于**判断相等才替换**。

#### Swap

如果**不需要比较旧值**，只是比较粗暴地替换的话，就可以使用 Swap 方法，它**替换后还可以返回旧值**。

#### Load

Load 方法会取出 addr 地址中的值，即使在多处理器、多核、有 CPU cache 的情况下，这个操作也能保证 Load 是一个原子操作。

#### Store

Store 方法会把一个值存入到指定的 addr 地址中，即使在多处理器、多核、有 CPU cache 的情况下，这个操作也能保证 Store 是一个原子操作。别的 goroutine 通过 Load 读取出来，不会看到存取了一半的值。



#### 指针

下列四个原子操作函数提供给了（安全）指针类型。 因为这几个函数被引入标准库的时候，Go还不支持自定义泛型，所以这些函数是**通过非类型安全指针`unsafe.Pointer`来实现的**。

```go
func LoadPointer(addr *unsafe.Pointer) (val unsafe.Pointer)
func StorePointer(addr *unsafe.Pointer, val unsafe.Pointer)
func SwapPointer(addr *unsafe.Pointer, new unsafe.Pointer,
				) (old unsafe.Pointer)
func CompareAndSwapPointer(addr *unsafe.Pointer,
				old, new unsafe.Pointer) (swapped bool)
```

因为Go（安全）指针不支持算术运算，所以相对于整数类型，指针类型的原子操作少了一个`AddPointer`函数。

#### Value 类型

`sync/atomic`标准库包也提供了一个`Value`类型，以它为基的指针类型`*Value`拥有四个方法（见下，其中后两个是从Go 1.17开始才支持的）。 **`Value`值用来原子读取和修改任何类型的Go值**。

```go
func (*Value) Load() (x interface{})
func (*Value) Store(x interface{})
func (*Value) Swap(new interface{}) (old interface{})
func (*Value) CompareAndSwap(old, new interface{}) (swapped bool)
```

### Go1.19+ 版本新增的原子操作概述

Go 1.19引入了几个各自拥有若干方法的类型用来实现上一节中列出的函数提供的同样的功能。

在这些类型中，`Int32`、`Int64`、`Uint32`、`Uint64`和`Uintptr`用来实现整数原子操作。 下面列出的是`atomic.Int32`类型的方法。其它四个类型的方法是类似的。

```go
func (*Int32) Add(delta int32) (new int32)
func (*Int32) Load() int32
func (*Int32) Store(val int32)
func (*Int32) Swap(new int32) (old int32)
func (*Int32) CompareAndSwap(old, new int32) (swapped bool)
```

从Go 1.19开始，一些标准库包开始使用自定义泛型，这其中包括`sync/atomic`标准库包。 此包在Go 1.19中引入的`atomic.Pointer[T any]`类型就是一个泛型类型。 下面列出了它的方法：

```go
(*Pointer[T]) Load() *T
(*Pointer[T]) Store(val *T)
(*Pointer[T]) Swap(new *T) (old *T)
(*Pointer[T]) CompareAndSwap(old, new *T) (swapped bool)
```

Go 1.19也引入了一个`Bool`类型来进行布尔原子操作。



## 代码示例

使用`Add`原子操作来并发地递增一个`int32`值。

```go
func main() {
    var n int32
	var wg sync.WaitGroup
	for i := 0; i < 1000; i++ {
		wg.Add(1)
		go func() {
			atomic.AddInt32(&n, 1)
			wg.Done()
		}()
	}
	wg.Wait()
	fmt.Println(atomic.LoadInt32(&n)) // 1000
}
```

```go
func main() {
	var n atomic.Int32
	var wg sync.WaitGroup
	for i := 0; i < 1000; i++ {
		wg.Add(1)
		go func() {
			n.Add(1)
			wg.Done()
		}()
	}
	wg.Wait()
	fmt.Println(n.Load()) // 1000
}
```

`StoreT`和`LoadT`原子操作函数或者方法经常被用来需要并发运行的实现setter和getter方法。

```go
type Page struct {
	views uint32
}

func (page *Page) SetViews(n uint32) {
	atomic.StoreUint32(&page.views, n)
}

func (page *Page) Views() uint32 {
	return atomic.LoadUint32(&page.views)
}
```

```go
type Page struct {
	views atomic.Uint32
}

func (page *Page) SetViews(n uint32) {
	page.views.Store(n)
}

func (page *Page) Views() uint32 {
	return page.views.Load()
}
```

`SwapT`函数调用和`StoreT`函数调用类似，但是返回修改之前的旧值（因此称为置换操作）。

一个`CompareAndSwapT`函数调用传递的旧值和目标值的当前值匹配的情况下才会将目标值改为新值，并返回`true`；否则立即返回`false`。

```go
func main() {
	var n int64 = 123
	var old = atomic.SwapInt64(&n, 789)
	fmt.Println(n, old) // 789 123
	swapped := atomic.CompareAndSwapInt64(&n, 123, 456)
	fmt.Println(swapped) // false
	fmt.Println(n)       // 789
	swapped = atomic.CompareAndSwapInt64(&n, 789, 456)
	fmt.Println(swapped) // true
	fmt.Println(n)       // 456
}
```

```go
func main() {
	var n atomic.Int64
	n.Store(123)
	var old = n.Swap(789)
	fmt.Println(n.Load(), old) // 789 123
	swapped := n.CompareAndSwap(123, 456)
	fmt.Println(swapped)  // false
	fmt.Println(n.Load()) // 789
	swapped = n.CompareAndSwap(789, 456)
	fmt.Println(swapped)  // true
	fmt.Println(n.Load()) // 456
}
```

请注意，到目前为止（Go 1.22），**一个64位字（int64或uint64值）的原子操作要求此64位字的内存地址必须是8字节对齐的**。 对于Go 1.19引入的**原子方法**操作，此要求无论在32-bit还是64-bit架构上**总是会得到满足**，但是对于**32-bit**架构上的**原子函数**操作，此要求**并非总会得到满足**。

> Q: 为什么要对齐？为什么原子方法操作满足，原子函数可能不满足？
>
> A:



上面已经提到了`sync/atomic`标准库包为指针值的原子操作提供了四个函数，并且指针值的原子操作是通过非类型安全指针来实现的。

**在Go中， 任何指针类型的值可以被显式转换为非类型安全指针类型`unsafe.Pointer`，反之亦然。 所以指针类型`*unsafe.Pointer`的值也可以被显式转换为类型`unsafe.Pointer`，反之亦然。**

```go
type T struct {x int}

func main() {
	var pT *T
    // pT 本身是一个有效的变量，不过里面存的指针是nil，所以 &pT 是没有问题的，但是 *pT 会panic
    // unsafePPT 的值是 pT 的地址
	var unsafePPT = (*unsafe.Pointer)(unsafe.Pointer(&pT))
	var ta, tb = T{1}, T{2}
	// 修改
	atomic.StorePointer(
		unsafePPT, unsafe.Pointer(&ta))
	fmt.Println(pT) // &{1}
	// 读取
	pa1 := (*T)(atomic.LoadPointer(unsafePPT))
	fmt.Println(pa1 == &ta) // true
	// 置换
	pa2 := atomic.SwapPointer(
		unsafePPT, unsafe.Pointer(&tb))
	fmt.Println((*T)(pa2) == &ta) // true
	fmt.Println(pT) // &{2}
	// 比较置换
	b := atomic.CompareAndSwapPointer(
		unsafePPT, pa2, unsafe.Pointer(&tb))
	fmt.Println(b) // false
	b = atomic.CompareAndSwapPointer(
		unsafePPT, unsafe.Pointer(&tb), pa2)
	fmt.Println(b) // true
}
```

**指针的原子操作需要引入`unsafe`标准库包，所以这些操作函数不在[Go 1兼容性保证](https://golang.google.cn/doc/go1compat)之列。**

> TODO: 学习 unsafe.Pointer 之后好好理解这段代码

```go
type T struct {x int}

func main() {
	var pT atomic.Pointer[T]
	var ta, tb = T{1}, T{2}
	// store
	pT.Store(&ta)
	fmt.Println(pT.Load()) // &{1}
	// load
	pa1 := pT.Load()
	fmt.Println(pa1 == &ta) // true
	// swap
	pa2 := pT.Swap(&tb)
	fmt.Println(pa2 == &ta) // true
	fmt.Println(pT.Load())  // &{2}
	// compare and swap
	b := pT.CompareAndSwap(&ta, &tb)
	fmt.Println(b) // false
	b = pT.CompareAndSwap(&tb, &ta)
	fmt.Println(b) // true
}
```

更为重要的是，上面这段代码没有引入`unsafe`标准库包，所以Go 1会保证它的向后兼容性。

`sync/atomic`标准库包中提供的`Value`类型可以用来读取和修改任何类型的值。这些方法均以`interface{}`做为参数类型，所以传递给它们的实参可以是任何类型的值。

但是对于一个可寻址的`Value`类型的值`v`，一旦`v.Store`方法（`(&v).Store`的简写形式）被曾经调用一次，则传递给值`v`的后续方法调用的实参的具体类型必须和传递给它的第一次调用的实参的具体类型一致； 否则，将产生一个 panic 。`nil`接口类型实参也将导致`v.Store()`方法调用产生 panic 。

> Q: 是不是底层指针（地址）已经被类型化了？

事实上，我们也可以使用上一节介绍的指针原子操作来对任何类型的值进行原子读取和修改，不过需要多一级指针的间接引用。 两种方法有各自的好处和缺点。在实践中需要根据具体需要选择合适的方法。

## 原子操作相关的顺序保证

见 内存顺序保证 一文。

## 第三方库拓展

[uber-go/atomic](https://github.com/uber-go/atomic) 定义和封装了几种与常见类型相对应的原子操作类型，这些类型提供了原子操作的方法。这些类型包括 Bool、Duration、Error、Float64、Int32、Int64、String、Uint32、Uint64 等。

比如 Bool 类型，提供了 CAS、Store、Swap、Toggle 等原子方法，还提供 String、MarshalJSON、UnmarshalJSON 等辅助方法，确实是一个精心设计的 atomic 扩展库。

```go
    var running atomic.Bool
    running.Store(true)
    running.Toggle()
    fmt.Println(running.Load()) // false
```

## Lock-Free queue

atomic 常常用来实现 Lock-Free 的数据结构。

Lock-Free queue 最出名的就是 Maged M. Michael 和 Michael L. Scott 1996 年发表的[论文](https://www.cs.rochester.edu/u/scott/papers/1996_PODC_queues.pdf)中的算法，算法比较简单，容易实现，伪代码的每一行都提供了注释。

我们使用 Go 实现这个数据结构的代码几乎和伪代码一样：

```go
// lock-free的queue
type LKQueue struct {
	// 辅助头指针，头指针不包含有意义的数据
	head unsafe.Pointer
	tail unsafe.Pointer
}

// 通过链表实现，这个数据结构代表链表中的节点
type node struct {
	value interface{}
	next  unsafe.Pointer
}

func NewLKQueue() *LKQueue {
	n := unsafe.Pointer(&node{})
	return &LKQueue{head: n, tail: n}
}

// 入队
func (q *LKQueue) Enqueue(v interface{}) {
	n := &node{value: v}
	for {
		// 拿到队尾
		tail := load(&q.tail)
		// 队尾的next
		next := load(&tail.next)
		// 因为是并发支持，所以下面的每一步都有可能其他线程/协程插入新的数据
		if tail == load(&q.tail) { // 尾还是尾（没有其他协程入队）
			// 判断是否有其他协程入队了
			if next == nil { // 还没有其他协程入队新数据
				// 判断是否有其他协程入队，比较交换（没有变动才交换）
				if cas(&tail.next, next, n) { //增加到队尾
					// 这一步会 return false 吗？ 可能会，但是没有影响
					// 假设从上一个 cas(&tail.next, next, n)  到这一个cas之间，有其他协程入列，也就是 n 已经插入队列了，但是还没有更新尾指针
					// 另外的协程在判断 next==nil 时候就已经失败了，因为尾指针（尚未更新的）next 是 刚刚插入的 n，这个另外的协程只能走else分支
					// 也就是这两个协程都会更新尾指针，而且 n 和 next 是一个node地址，所以这里更新失败无所谓
					cas(&q.tail, tail, n) //入队成功，移动尾巴指针
					return
				}
			} else { // 已有新数据加到队列后面，需要移动尾指针
				cas(&q.tail, tail, next)
			}
		}
	}
}

// 出队，没有元素则返回nil
func (q *LKQueue) Dequeue() interface{} {
	for {
		head := load(&q.head)
		tail := load(&q.tail)
		next := load(&head.next)
		if head == load(&q.head) { // head还是那个head
			if head == tail { // head和tail一样
				if next == nil { // 说明是空队列
					return nil
				}
				// 只是尾指针还没有调整，尝试调整它指向下一个
				cas(&q.tail, tail, next)
			} else {
				// 读取出队的数据
				v := next.value
				// 既然要出队了，头指针移动到下一个
				if cas(&q.head, head, next) {
					return v // Dequeue is done. return
				}
			}
		}
	}
}

// 将unsafe.Pointer原子加载转换成node
func load(p *unsafe.Pointer) (n *node) {
	return (*node)(atomic.LoadPointer(p))
}

// 封装CAS,避免直接将*node转换成unsafe.Pointer
func cas(p *unsafe.Pointer, old, new *node) (ok bool) {
	return atomic.CompareAndSwapPointer(p, unsafe.Pointer(old), unsafe.Pointer(new))
}

```

## 对一个地址的赋值是原子操作吗？

在现在的系统中，write 的地址基本上都是对齐的（aligned）。 比如，32 位的操作系统、CPU 以及编译器，write 的地址总是 4 的倍数，64 位的系统总是 8 的倍数（还记得 WaitGroup 针对 64 位系统和 32 位系统对 state1 的字段不同的处理吗）。对齐地址的写，不会导致其他人看到只写了一半的数据，因为它通过一个指令就可以实现对地址的操作。如果地址不是对齐的话，那么，处理器就需要分成两个指令去处理，如果执行了一个指令，其它人就会看到更新了一半的错误的数据，这被称做撕裂写（torn write） 。所以，你可以认为赋值操作是一个原子操作，这个“原子操作”可以认为是保证数据的完整性。

但是，对于现代的多处理多核的系统来说，由于 cache、指令重排，可见性等问题，我们对原子操作的意义有了更多的追求。在多核系统中，一个核对地址的值的更改，在更新到主内存中之前，是在多级缓存中存放的。这时，多个核看到的数据可能是不一样的，其它的核可能还没有看到更新的数据，还在使用旧的数据。

多处理器多核心系统为了处理这类问题，使用了一种叫做内存屏障（memory fence 或 memory barrier）的方式。一个写内存屏障会告诉处理器，必须要等到它管道中的未完成的操作（特别是写操作）都被刷新到内存中，再进行操作。此操作还会让相关的处理器的 CPU 缓存失效，以便让它们从主存中拉取最新的值。

**atomic 包提供的方法会提供内存屏障的功能**，所以，a**tomic 不仅仅可以保证赋值的数据完整性，还能保证数据的可见性**，一旦一个核更新了该地址的值，其它处理器总是能读取到它的最新值。但是，需要注意的是，因为需要处理器之间保证数据的一致性，atomic 的操作也是会降低性能的。

