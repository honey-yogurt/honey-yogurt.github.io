+++
title = 'gmp'
date = 2024-05-16T09:19:53+08:00
+++

## 进程、线程、协程（Goroutine）

在仅支持进程的操作系统中，进程是拥有资源和独立调度的基本单位。在引入线程的操作系统中，**线程是独立调度的基本单位，进程是资源拥有的基本单位**。在同一进程中，线程的切换不会引起进程切换。在不同进程中进行线程切换,如从一个进程内的线程切换到另一个进程中的线程时，会引起进程切换。

即操作系统内核最小的调度单元是内核级线程。

多进程、多线程已经提高了系统的并发能力，但是在当今互联网高并发场景下，为每个任务都创建一个线程是不现实的，因为会消耗大量的内存(进程虚拟内存会占用4GB[32位操作系统], 而线程也要大约4MB)。

大量的进程/线程出现了新的问题

+ 高内存占用 
+ 调度的高消耗CPU

**用户级线程即协程，由应用程序创建与管理，协程必须与内核级线程绑定之后才能执行**。**线程由 CPU 调度是抢占式的，协程由用户态调度是协作式的**，一个协程让出 CPU 后，才执行下一个协程。

> 可以把多个协程当做是多个并发的“业务”，它不受操作系统（cpu）调度，CPU并不知道有“用户态线程”的存在，它只知道它运行的是一个“内核态线程”(Linux的PCB进程控制块)。应用程序负责把并发的“业务”绑定到线程（内核级）中，由CPU调用线程从而执行这些并发的“业务”。

![img.png](/images/lang/go/gmp-1.png)

既然一个协程(co-routine)可以绑定一个线程(thread)，那么能不能多个协程(co-routine)绑定一个或者多个线程(thread)上呢。

于是，就有了三种协程和线程的映射关系：

### N:1

N个协程绑定1个线程，优点就是**协程在用户态线程即完成切换，不会陷入到内核态，这种切换非常的轻量快速**。但也有很大的缺点，1个进程的所有协程都绑定在1个线程上。

缺点：

+ 某个程序用不了硬件的多核加速能力 
+ 一旦某协程阻塞，造成线程阻塞，本进程的其他协程都无法执行了，根本就没有并发的能力了。

![img.png](/images/lang/go/gmp-2.png)

### 1:1

1个协程绑定1个线程，这种最容易实现。协程的调度都由CPU完成了，不存在N:1缺点。

缺点：

+ 协程的创建、删除和切换的代价都由CPU完成，有点略显昂贵了。

![img.png](/images/lang/go/gmp-3.png)

### M:N

M个协程绑定N个线程，是N:1和1:1类型的结合，克服了以上2种模型的缺点，但实现起来最为复杂。

![img.png](/images/lang/go/gmp-4.png)

## 什么是 Goroutine

Goroutine = Golang + Coroutine。**Goroutine是golang实现的协程，是用户级线程**。

**Goroutines 是在同一个用户地址空间里并行独立执行 functions（Go 函数或方法）**，一个运行的程序由一个或更多个 goroutine 组成。channels 则用于 goroutines 间的通信和同步访问控制。

Goroutine 拥有运行函数的指针、栈、上下文（指的是sp、bp、pc等寄存器上下文以及垃圾回收的标记上下文）。

goroutine 和 thread 的区别？

+ 内存占用：创建一个goroutine的栈内存消耗为2KB，可自动扩容。创建一个 thread 分配一个较大的栈内存（1-8MB），另外线程和线程之间还需要一个被称为 `guard page`（防止线程栈溢出污染临近的栈）的区域进行隔离，栈内存空间一旦创建和初始化完成后其大小不能变化，存在栈溢出风险。
  ![img.png](/images/lang/go/gmp-5.png)
+ 创建/销毁：线程创建和销毁都会有巨大的消耗，是内核级的交互（trap）。goroutine 是用户态线程，是由 `go runtime` 管理，创建和销毁的消耗非常小。
+ 调度切换：抛开陷入内核，线程切换会消耗1000-1500纳秒(上下文保存成本高，较多寄存器，公平性，复杂时间计算统计)，一个纳秒平均可以执行 12-18 条指令。所以由于线程切换，执行指令的条数会减少 12000-18000。goroutine 的切换约为 200 ns(用户态、3个寄存器)，相当于 2400-3600 条指令。因此，goroutines 切换成本比 threads 要小得多。
+ 复杂性：线程的创建和退出复杂，多个 thread 间通讯复杂(share memory)。不能大量创建线程(参考早期的 httpd)，成本高，使用网络多路复用，存在大量callback(参考twemproxy、nginx 的代码)。对于应用服务线程门槛高，例如需要做第三方库隔离，需要考虑引入线程池等。



## GMP 概念

内核线程(M)，goroutine(G)，G的上下文环境（P）。

+ G：goroutine协程，每次 go func() 都代表一个 G，使用的是 `runtime.g` 结构体
+ M：工作线程（OS thread，操作系统线程的一个封装）也被称为 Machine，使用 `runtime.m` 结构体，所有 M 是有线程栈的。如果不对该线程栈提供内存的话，系统会给该线程栈提供内存(不同操作系统提供的线程栈大小不同)。当指定了线程栈，则 M.stack→G.stack，M 的 PC 寄存器指向 G 提供的函数，然后去执行。
+ P：processor处理器，是一个抽象的概念，并不是真正的物理 CPU。使用的是 `runtime.p`。 它代表了 **M 所需的上下文环境**，也是处理用户级代码逻辑的处理器。它负责衔接 M 和 G 的调度上下文，将等待执行的 G 与 M 对接。**当 P 有任务时需要创建或者唤醒一个 M 来执行它队列里的任务。所以 P/M 需要进行绑定**，构成一个执行单元。P 决定了并行任务的数量，可通过 runtime.GOMAXPROCS 来设定。在 Go1.5 之后GOMAXPROCS 被默认设置可用的核数。**它维护一个局部可运行的 G 的队列（max=256），可以通过 CAS 的方式无锁访问**。

## GM 模型

Go 创建 M 个线程(CPU 执行调度的单元，内核的 task_struct)，之后创建的 N 个 goroutine 都会依附在这 M 个线程上执行，即 M:N 模型。它们能够同时运行，与线程类似，但相比之下非常轻量。因此，程序运行时，Goroutines 的个数应该是**远大于**线程的个数的。

同一个时刻，一个线程只能跑一个 goroutine。当 goroutine 发生阻塞 (chan 阻塞、mutex等等) 时，Go 会把当前的 goroutine 调度走，让其他 goroutine 来继续执行，**而不是让线程阻塞休眠（线程不会阻塞，一直忙碌，减少线程调度的开销）**，尽可能多的分发任务出去，让 CPU 忙。

![img](/images/lang/go/gmp_18.jpg)

但是 GM 模型也有一些明显的缺点：

+ 全局锁和中心化状态带来锁竞争（所有的M都去全局的G队列竞争G），导致性能下降 ：*所有 goroutine 相关操作，比如：创建、结束、重新调度等都要上锁。*
+ 每个 M 都能执行任意可执行状态的 G，M 频繁地和 G 交接，导致额外开销 ：*刚创建的* *G* *放到了全局队列，而不是本地* *M* *执行，不必要的开销和延迟*
+ 每个 M 都需要处理内存缓存，导致大量内存占用，并影响数据局部性 ：*每个* *M* *持有* *mcache* *和* *stackalloc**，然而只有在* *M* *运行* *Go* *代码时才需要使用的内存(每个* *mcache* *可以高达**2mb**)，当* *M* *在处于* *syscall* *时并不需要。运行* *Go* *代码和阻塞在* *syscall* *的* *M* *的比例高达**1:100**，造成了很大的浪费。同时内存亲缘性也较差，G 当前在 M 运行后对 M 的内存进行了预热，因为现在 G 调度到同一个 M 的概率不高，数据局部性不好。*
+ 系统调用频繁阻塞和解除阻塞线程，增加了额外开销：*在系统调用的情况下，工作线程经常被阻塞和取消阻塞**，**这增加了很多开销**。比如 M 找不到G，此时 M 就会进入频繁阻塞/唤醒来进行检查的逻辑，以便及时发现新的 G 来执行。*

## GMP 模型

### 整体调度流程

![img](/images/lang/go/gmp_19.jpg)

+ global queue: 全局队列，存放等待运行的 g
+ local queue: P 的本地队列，存放等待运行的 g，数量有限，不超过 256 个。新建 g 时候， **g 优先加入 p 的本地队列**。如果队列满了（超过256个），则会把本地队列中一半的 g 移动到全局队列。
+ P 列表： **所有的 P 都是在程序启动时创建**，可通过 runtime.GOMAXPROCS 来设定。
+ M ：内核线程，M的数量会多于P，但不会太多。

**Goroutine调度器和OS调度器是通过M结合起来的，每个M都代表了1个内核线程，OS调度器负责把内核线程分配到CPU的核上执行**。

GMP模型大致调用流程如下：

+ 线程M想运行任务就需得获取 P，即与P关联绑定。
+ 然从 P 的本地队列获取 G
+ 若P的本地队列中没有可运行的G，M 会尝试从全局队列拿一批G放到P的本地队列，若全局队列也未找到可运行的G时候，M会随机从其他 P 的本地队列**偷一半**放到自己 P 的本地队列。如果还是没有 g，就会从 Network Poller 上拿一个。
+ 拿到可运行的G之后，M 运行 G，G 执行之后，M 会从 P 获取下一个 G，不断重复下去。

![img](/images/lang/go/gmp_20.jpeg)

### 设计理念

#### P 的引入

引入了 local queue，因为 P 的存在，runtime 并不需要做一个集中式的 goroutine 调度，每一个 M 都会在 P's local queue、global queue 或者其他 P 队列中找 G 执行，**减少全局锁对性能的影响**（如果只有全局队列的话，所有的P去全局队列取g都要加锁）。这也是 GMP Work-stealing 调度算法的核心。注意 P 的本地 G 队列还是可能面临一个并发访问的场景（会被窃取），为了避免加锁，这里 P 的本地队列是一个 **LockFree** 的队列，**窃取 G 时使用 CAS 原子操作来完成**。

#### Work Stealing 机制

> 以下链接是使用的master分支（此时是 go1.22），可能会在今后版本变动。

[窃取最多会尝试窃取四次](https://github.com/golang/go/blob/master/src/runtime/proc.go#L3659-L3715)。

[当一个 P 执行完本地所有的 G 之后，如果全局队列不为空的话，会从全局队列中获取(当前个数/GOMAXPROCS)个 G。如果全局队列为空的时候，会尝试挑选一个 P'，从它的 G 队列中**窃取一半**的 G，如果还是没有取到 g，会从 network poller 中拿一个 g](https://github.com/golang/go/blob/master/src/runtime/proc.go#L3317-L3380)。

为了保证公平性，从随机位置上的 P 开始，而且遍历的顺序也随机化了(选择一个小于 GOMAXPROCS，且和它互为质数的步长)，保证遍历的顺序也随机化了。

光在本地队列没有G时候再去全局队列中取G是不够的，可能会导致**全局队列饥饿**。P 的调度算法中还会**每个 N 轮调度之后就去全局队列拿一个 G**。

如果所有的 p 本地队列一直都有 g，它们一直从本地队列中取 g 执行，那么全局队列中的 g 就会饥饿。
![img.png](/images/lang/go/gmp-7.png)

> Q: 什么情况下会把 g 放到 全局队列中呢？
>
> A: 
>
> + 新建 G 时 P 的本地 队列放不下已满并达到256个的时候会放**半数** G 到全局队列去
> + **阻塞的系统调用返回时找不到空闲 P** 也会放到全局队列。

#### 主动调度

协程通过调用`runtime.Goshed`方法主动让渡自己的执行权利，之后这个协程会被放到全局队列中，等待后续被执行。

#### 被动调度

当G阻塞时（mutex，chan阻塞、network I/O等），不会阻塞M，M会寻找其他runnable的G；当阻塞的G恢复后会重新进入runnable进入P队列等待执行。

> go 已经用 netpoller 实现了goroutine网络I/O阻塞不会导致M被阻塞，仅阻塞G

#### Network poller

Go 基于 I/O multiplexing 和 goroutine scheduler 构建了一个简洁而高性能的原生网络模型(基于 Go 的 I/O 多路复用 `netpoller` )，提供了 `goroutine-per-connection` 这样简单的网络编程模式。在这种模式下，开发者使用的是同步的模式去编写异步的逻辑，极大地降低了开发者编写网络应用时的心智负担，且借助于 Go runtime scheduler 对 goroutines 的高效调度，这个原生网络模型不论从适用性还是性能上都足以满足绝大部分的应用场景。

Go netpoller 在不同的操作系统，其底层使用的 I/O 多路复用技术也不一样。事实上 `Go netpoller` 底层就是基于 epoll/kqueue/iocp 这些 I/O 多路复用技术来做封装的，最终暴露出 `goroutine-per-connection` 这样的极简的开发模式给使用者。

G 发起网络 I/O 操作不会导致 M 被阻塞(仅阻塞G)，从而不会导致大量 M 被创建出来。将**异步 I/O 转换为阻塞 I/O** （不需要考虑回调）的部分称为 netpoller。打开或接受连接都被设置为非阻塞模式。如果你试图对其进行 I/O 操作，并且文件描述符数据还没有准备好，G 会进入 gopark 函数（waiting状态），将当前正在执行的 G 状态保存起来，然后切换到新的堆栈上执行新的 G。

那什么时候 G 被调度回来呢？

+ sysmon
+ schedule()：M 找 G 的调度函数
+ GC：start the world

调用 netpoll() 在某一次调度 G 的过程中，处于就绪状态的 fd 对应的 G 就会被调度回来。

![image-20240809221537159](/images/lang/go/gmp_42.png)

G 的 gopark 状态：G 置为 waiting 状态，等待显示 goready 唤醒，在 poller 中用得较多，还有锁、chan 等。

#### Hand Off 交接机制

当 **M 阻塞**时，会将 M 上的 P 的运行队列交给其它 M 执行：

![img.png](/images/lang/go/gmp-8.png)
调用 syscall 后会解绑 P (不知道这个 线程syscall 需要多久)，**然后 M 和 G（等M的syscall的返回) 进入阻塞**，而 P 此时的状态就是 syscall，表明这个 P 的 G 正在 syscall 中，这时的 P 是不能被调度给别的 M 的。如果在**短时间内阻塞(10ms)**的 M 就唤醒了，那么 M 会优先来重新获取这个 P，能获取到就继续绑回去，这样有利于数据的**局部性**。

系统监视器 (system monitor)，称为 sysmon，会定时扫描。在执行 syscall 时, 如果某个 P 的 G 执行超过一个 sysmon tick(10ms)，就会把他设为 idle，**重新调度给需要的 M，强制解绑**。

![img.png](/images/lang/go/gmp-9.png)

P1 和 M 脱离后目前在 idle list 中等待被绑定（处于 syscall 状态）。而 syscall 结束后 M 按照如下规则执行直到满足其中一个条件：

+ 尝试获取同一个 P(P1)，恢复执行 G
+ 尝试获取 idle list 中的其他空闲 P，恢复执行 G
+ **找不到空闲 P，把 G 放回 global queue，M 放回到 m' idle list**

当使用了 syscall，Go 无法限制 Blocked OS threads 的数量。

> The GOMAXPROCS variable limits the number of operating system threads that can execute user-level Go code simultaneously. There is no limit to the number of threads that can be blocked in system calls on behalf of Go code; those do not count against the GOMAXPROCS limit. This package’s GOMAXPROCS function queries and changes the limit.

#### 抢占式调度 与 sysmon

![img.png](/images/lang/go/gmp-10.png)
sysmon 也叫监控线程，它**无需 P 也可以运行**，他是一个死循环，每20us~10ms循环一次，循环完一次就 sleep 一会，为什么会是一个变动的周期呢，主要是避免空转，如果每次循环都没什么需要做的事，那么 sleep 的时间就会加大。

+ 释放闲置超过5分钟的 span 物理内存；
+ 如果超过2分钟没有垃圾回收，强制执行；
+ 将长时间未处理的 netpoll 添加到全局队列；
+ 向长时间运行的 G 任务发出**抢占调度**；
+ 收回因 syscall 长时间阻塞的 P；

![img.png](/images/lang/go/gmp-11.png)
**当 G 在 M 上执行时间超过10ms**，sysmon 调用 preemptone 将 G 标记为 stackPreempt（一个很大的常数） 。因此需要在某个地方触发检测逻辑，Go 当前是在检查栈是否溢出的地方判定(morestack())，M 会保存当前 G 的上下文，重新进入调度逻辑。

但是这种抢占是有限制的，比如一个for死循环，这个 g 就一直执行，但是无法将他给抢占调度。

信号抢占：[go1.14基于信号的抢占式调度实现原理](https://xiaorui.cc/archives/6535)

异步抢占，注册 sigurg 信号，通过 sysmon 检测，对 M 对应的线程发送信号，触发注册的 handler，它往当前 G 的 PC 中插入一条指令(调用某个方法)，在处理完 handler，G 恢复后，自己把自己推到了 global queue 中。

#### spin thread

**线程自旋**（死循环）是相对于线程阻塞（阻塞就是休眠）而言的，表象就是循环执行一个指定逻辑(**调度逻辑，目的是不停地寻找 G**)。这样做的问题显而易见，如果 G 迟迟不来，CPU 会白白浪费在这无意义的计算上。但好处也很明显，降低了 M 的上下文切换成本，提高了性能。在两个地方引入自旋：

+ 类型1：M 不带 P 的找 P 挂载（一有 P 释放就结合）
+ 类型2：M 带 P 的找 G 运行（一有 runable 的 G 就执行）

**本质是不能让非休眠的线程空闲，因为线程是真正的执行单元。**

为了避免过多浪费 CPU 资源，自旋的 M 最多只允许 GOMAXPROCS (Busy P)。同时当有类型1的自旋 M 存在时，类型2的自旋 M 就不阻塞，阻塞会释放 P，一释放 P 就马上被类型1的自旋 M 抢走了，没必要。

在新 G 被创建、M 进入系统调用、M 从空闲被激活这三种状态变化前，调度器会确保至少有一个自旋 M 存在（唤醒或者创建一个 M），除非没有空闲的 P。

+ 当新 G 创建，如果有可用 P，就意味着新 G 可以被立即执行，即便不在同一个 P 也无妨，所以我们保留一个自旋的 M（这时应该不存在类型1的自旋只有类型2的自旋）就可以保证新 G 很快被运行。
+ 当 M 进入系统调用，意味着 M 不知道何时可以醒来，那么 M 对应的 P 中剩下的 G 就得有新的 M 来执行，所以我们保留一个自旋的 M 来执行剩下的 G（这时应该不存在类型2的自旋只有类型1的自旋）。
+ 如果 M 从空闲变成活跃，意味着可能一个处于自旋状态的 M 进入工作状态了，这时要检查并确保还有一个自旋 M 存在，以防还有 G 或者还有 P 空着的。

自旋线程是为了尽量避免P的空闲，从而尽可能不浪费并行能力（执行 g）。

#### scheduler affinity（亲缘性调度）

![img.png](/images/lang/go/gmp-12.png)

在 chan 来回通信的 goroutine 会导致频繁的 blocks，即**频繁地在本地队列中重新排队**。然而，由于本地队列是 FIFO 实现，如果另一个 goroutine 占用线程，**unblock goroutine 不能保证尽快运行**。

goroutine #9 在 chan 被阻塞后恢复。但是，它必须等待#2、#5和#4之后才能运行。goroutine #5将阻塞其线程，从而延迟goroutine #9，并使其面临被另一个 P 窃取的风险。

![img.png](/images/lang/go/gmp-13.png)
针对 communicate-and-wait 模式，进行了亲缘性调度的优化。Go 1.5 在 P 中**引入了 `runnext` 特殊的一个字段**，可以**高优先级**执行 unblock G。

goroutine #9现在被标记为下一个可运行的。这种新的优先级排序允许 goroutine 在再次被阻塞之前快速运行。这一变化对运行中的标准库产生了总体上的积极影响，提高了一些包的性能。

### 抢占式调度详解

sysmon是一个Go里面的一个特殊的线程，不与任何P绑定，不参与调度，主要用于监控整个Go进程，主要有如下作用：

+ 释放闲置超过5分钟的span物理内存
+ 超过2分钟没有垃圾回收，强制启动垃圾回收
+ 将长时间没有处理的netpoll结果添加到任务队列
+ 向长时间执行的G任务发起抢占调度
+ 收回因syscall而长时间阻塞的P

sysmon线程在runtime.main函数里面创建：

```go
func main() {
    ...
      if GOARCH != "wasm" { // no threads on wasm yet, so no sysmon
      // 启动sysmon的代码
      // 在系统栈内生成一个新的M来启动sysmon
      atomic.Store(&sched.sysmonStarting, 1)
        systemstack(func() {
          newm(sysmon, nil, -1)
      })
    }
  ...
}
```

以下为sysmon函数循环检查Go进程的过程：

```go
func sysmon() {
    lasttrace := int64(0)
    idle := 0 // 每次扫描无需抢占的计数器，无须抢占次数越多，后续sysmon线程休眠时间越高
    delay := uint32(0)
    for {
        // delay按idel值来判断休眠时间，首次20us，1ms之后sleep逐步翻倍，最高10ms
        // ......
        usleep(delay)
        // poll网络监听，处理超过10ms以上的netpoll
        lastpoll := sched.lastpoll.Load()
        if netpollinited() && lastpoll != 0 && lastpoll+10*1000*1000 < now {
            sched.lastpoll.CompareAndSwap(lastpoll, now)
            list := netpoll(0) // non-blocking - returns list of goroutines
```

sysmon监控线程判断是否需要抢占**主要通过retake函数进行检查**，遍历所有的P，如果某个P经过10ms没有切换都没有协程，那么就需要被抢占了。

```go
const forcePreemptNS = 10 * 1000 * 1000 // 10ms


func retake(now int64) uint32 {
    // ......
    for i := 0; i < len(allp); i++ {
        pp := allp[i]
        pd := &pp.sysmontick
        s := pp.status
        sysretake := false
        if s == _Prunning || s == _Psyscall {
            // Preempt G if it's running for too long.
            t := int64(pp.schedtick)
```

**找到需要抢占的P后，调用preemptone(pp)对P当前运行的协程进行抢占**。抢占的方式有两种，一种是基于协作的抢占，一种是基于信号的抢占。

```go
const(
    // 0xfffffade in hex.
    stackPreempt = 0xfffffade // (1<<(8*sys.PtrSize) - 1) & -1314
)
func preemptone(pp *p) bool {
    mp := pp.m.ptr()
    gp := mp.curg
    gp.preempt = true
    // 基于协程的抢占
    // stackPreempt是一个很大的常量，其他地方会检查这个变量，当go的stackgurad0栈为这个数值时，标明需要被抢占
    gp.stackguard0 = stackPreempt
    if preemptMSupported && debug.asyncpreemptoff == 0 {
        pp.preempt = true
```

 **stackPreempt是一个很大的常量，其他地方会检查这个变量，当go的stackgurad0栈为这个数值时，标明需要被抢占**。

#### 基于协作的抢占式调度

在1.14版本之前，只有基于协作的抢占式调度，即preemptone函数中只有设置`gp.stackguard0 = stackPreempt`，而没有后面的`preemptM(mp)`过程。

由于goroutine初始栈桢很小（2kb），为了避免栈溢出，go语言编译期会**在函数头部加上栈增长检测代码**，如果在**函数调用时编译器发现栈不够用了或者g.stackguard0 = stackPreempt**，将会调用runtime.morestack来进行**栈增长**，而runtime.morestack是个汇编函数，又会调用runtime.newstack。

在morestack中首先要**保存好当前协程的上下文，之后该协程继续从这个位置执行**。保存完成之后调用newstack。

```go
TEXT runtime·morestack(SB),NOSPLIT,$0-0
    ...
    MOVQ    0(SP), AX // f's PC
    MOVQ    AX, (g_sched+gobuf_pc)(SI)
    MOVQ    SI, (g_sched+gobuf_g)(SI)
    LEAQ    8(SP), AX // f's SP
    MOVQ    AX, (g_sched+gobuf_sp)(SI)
    MOVQ    BP, (g_sched+gobuf_bp)(SI)
    MOVQ    DX, (g_sched+gobuf_ctxt)(SI)
    ...
    CALL    runtime·newstack(SB)
```

newstack函数执行的栈扩张逻辑，在栈**扩张之前，首先会检查是否是要协程抢占**：

```go
func newstack() {
    thisg := getg()
    ...


    gp := thisg.m.curg
    ...
    // 判断是否执行抢占
    preempt := atomic.Loaduintptr(&gp.stackguard0) == stackPreempt


    // 保守的对用户态代码进行抢占，而非抢占运行时代码
    // 如果正持有锁、分配内存或抢占被禁用，则不发生抢占
```

当newstack判断是发生了抢占时，会调用到goschedImpl函数，可以看到，会先**把当前的g放到全局队列，然后开始调度**。

```go
func goschedImpl(gp *g) {
    casgstatus(gp, _Grunning, _Grunnable)
    dropg()
    lock(&sched.lock)
    // 将被抢占的协程g，放到全局的sched.runq队列中，等被其他m执行
    globrunqput(gp)
    unlock(&sched.lock)
    // 执行调度
    schedule()
}
```

#### 基于信号的抢占式调度

基于协作的抢占式调度不管是陷入到大量计算还是系统调用，大多可被sysmon扫描到并进行抢占。但有些场景是无法抢占成功的。比如轮询计算 `for { i++ }` 等，这类操作无法进行newstack、morestack、syscall，所以无法检测 `stackguard0 = stackpreempt`。

所以在1.14中加入了基于信号的协程调度抢占。原理是这样的，**首先注册绑定 SIGURG 信号及处理方法runtime.doSigPreempt，sysmon会间隔性检测超时的p，然后发送信号，m收到信号后休眠执行的goroutine并且进行重新调度**。

```go
// src/runtime/signal_unix.go
func preemptM(mp *m) {
    if GOOS == "darwin" || GOOS == "ios" {
        execLock.rlock()
    }
    if atomic.Cas(&mp.signalPending, 0, 1) {
        if GOOS == "darwin" || GOOS == "ios" {
            atomic.Xadd(&pendingPreemptSignals, 1)
        }
        // const sigPreempt=0x17该信号表示抢占信号
        signalM(mp, sigPreempt)
    }
    if GOOS == "darwin" || GOOS == "ios" {
```

**基于信号的抢占式调度是通过监控线程sysmon发现有10ms以上未调度的P时，通过执行signalM对Go进程发送抢占信号（0x17）**。

Go进程收到该信号之后是如何执行抢占的呢，我们先来看信号是如何注册的：

**在初始化时注册信号**：

```go
// src/runtime/signal_unix.go
func initsig(preinit bool) {
    // 预初始化
    if !preinit {
        signalsOK = true
    }
    //遍历信号数组 _NSIG=65
    for i := uint32(0); i < _NSIG; i++ {
        t := &sigtable[i]
        //略过信号：SIGKILL、SIGSTOP、SIGTSTP、SIGCONT、SIGTTIN、SIGTTOU
        if t.flags == 0 || t.flags&_SigDefault != 0 {
            continue
        }
```

在initsig中先遍历信号数组调用setsig进行注册，setsig中会执行系统调用来安装信号和信号处理函数。我们继续看**信号处理函数**。

```go
// src/runtime/signal_unix.go


func sighandler(sig uint32, info *siginfo, ctxt unsafe.Pointer, gp *g) {
    // The g executing the signal handler. This is almost always
    // mp.gsignal. See delayedSignal for an exception.
    gsignal := getg()
    mp := gsignal.m
    c := &sigctxt{info, ctxt}
    // const sigPreempt = _SIGURG = 0x17
    if sig == sigPreempt {
        // 是一个抢占信号
        doSigPreempt(gp, c)
```

在信号处理函数sighandler中，对于抢占信号，会执行doSigPreempt函数，其中会**通过pushcall插入syncPreempt（异步抢占）函数调用**。

```go
// src/runtime/preempt_arm.s
TEXT ·asyncPreempt(SB),NOSPLIT|NOFRAME,$0-0
    ...
    CALL ·asyncPreempt2(SB)
    ...

```

syncPreempt最终调用了asyncPreempt2()函数

```go
// src/runtime/preempt.go
func asyncPreempt2() {
    gp := getg()
    gp.asyncSafePoint = true
    if gp.preemptStop {
        mcall(preemptPark)
    } else {
        mcall(gopreempt_m)
    }
    gp.asyncSafePoint = false
}
```

最终跟基于协作的信号抢占一样，执行preemptPark或gopreempt_m函数来执行schedule。

信号抢占的整体逻辑如下：

1. M 注册一个 SIGURG 信号的处理函数：sighandler。
2. sysmon 线程检测到**执行时间过长的 goroutine、GC stw 时**，会向相应的 M（或者说线程，每个线程对应一个 M）发送 SIGURG 信号。
3. 收到信号后，内核执行 sighandler 函数，**通过 pushCall 插入 asyncPreempt 函数调用**。
4. 回到当前 goroutine 执行 asyncPreempt 函数，通过 mcall **切到 g0 栈执行 gopreempt_m**。
5. 将当前 goroutine 插入到**全局可运行队列**，M 则继续寻找其他 goroutine 来运行。
6. 被抢占的 goroutine 再次调度过来执行时，会继续原来的执行流。



[golang 基于信号的抢占式调度的设计实现原理](https://xiaorui.cc/archives/6535)

### Goroutine Lifecycle

#### m0 & g0

M0 是启动程序后的编号为 0 的**主线程**，这个 M 对应的实例会在全局变量 runtime.m0 中，不需要在 heap 上分配，**M0 负责执行初始化操作和启动第一个 G， 在之后 M0 就和其他的 M 一样了**。

G0 是每次启动一个 M 都会第一个创建的 gourtine，**G0 仅用于负责调度的 G**，G0 不指向任何可执行的函数，所以不能被抢占也不会被调度，每个 M 都会有一个自己的 G0。在调度或系统调用时会使用 G0 的栈空间，**全局变量的 G0 是 M0 的 G0**。G0的栈是系统分配的，比普通的G栈（2KB）要大，不能扩容也不能缩容。

#### m 的工作机制

在 runtime 中有三种线程，一种是主线程（m0），一种是用来跑 sysmon 的线程，一种是普通的用户线程。主线程在 runtime 由对应的全局变量` runtime.m0` 来表示。用户线程就是普通的线程了，和 p 绑定，执行 g 中的任务。虽然说是有三种，实际上前两种线程整个 runtime 就只有一个实例。用户线程才会有很多实例。

#### g0 的调度

![img.png](/images/lang/go/gmp-15.png)
Go 基于两种断点将 G 调度到线程上：

+ 当 **G 阻塞**时：系统调用、互斥锁或 chan。阻塞的 G 进入睡眠模式/进入队列，并允许 Go 安排和运行等待其他的 G。
+ **在函数调用期间，如果 G 必须扩展其堆栈**。这个断点允许 Go 调度另一个 G 并避免运行 G 占用 CPU。

在这两种情况下，运行调度程序的 g0 将当前 G 替换为另一个 G，即 ready to run。然后，选择的 G 替换 g0 并在线程上运行。与常规 G 相反，g0 有一个固定和更大的栈。

+ Defer 函数的分配
+ GC 收集，比如 STW、扫描 G 的堆栈和标记、清除操作
+ 栈扩容，当需要的时候，由 g0 进行扩栈操作

在 Go 中，G 的切换相当轻便，其中需要保存的状态仅仅涉及以下两个：

+ Goroutine 在停止运行前执行的指令，程序当前要运行的指令是记录在程序计数器（PC）中的， G 稍后将在同一指令处恢复运行；
+ G 的堆栈，以便在再次运行时还原局部变量；

![img.png](/images/lang/go/gmp-16.png)
从 g 到 g0 或从 g0 到 g 的切换是相当迅速的，它们只包含少量固定的指令。相反，对于调度阶段，调度程序需要检查许多资源以便确定下一个要运行的 G。

1. 当前 g 阻塞在 chan 上并切换到 g0：
   1. **PC 和堆栈指针一起保存在内部结构中**；
   2. **将 g0 设置为正在运行的 goroutine**；
   3. **g0 的堆栈替换当前堆栈**；

2. g0 寻找新的 Goroutine 来运行

3. g0 使用所选的 Goroutine 进行切换： 
   1. **PC 和堆栈指针是从其内部结构中获取的**；
   2. **程序跳转到对应的 PC 地址**；



G 很容易创建，栈很小以及快速的上下文切换。然而，一个产生许多 shortlive 的 g 的程序将花费相当长的时间来创建和销毁它们。

每个 P 维护一个 freelist G（空闲 g），保持这个列表是本地的，这样做的好处是不使用任何锁来 push/get 一个空闲的 G。**当 G 退出当前工作时**，它将被 push 到这个空闲列表中。

> 本质是为了复用g，避免大量的 shortlive（短生命周期）的  g 创建销毁开销大。

![image-20240809161534220](/images/lang/go/gmp_31.png)

为了更好地分发**空闲的 G** ，调度器也有自己的列表（全局）。它实际上有两个列表：一个包含已分配栈的 G，另一个包含释放过堆栈的 G（无栈）。

> 1、有stack的 g 是刚运行完的协程；2、没有stack的 g 是运行完后经过一次gc了的，在并发标记阶段归还的；

![img.png](/images/lang/go/gmp-17.png)

锁保护 central list，因为任何 M 都可以访问它。当本地列表长度超过64时，调度程序持有的列表从 P 获取 G。然后一半的 G 将移动到中心列表。回收 G 是一种节省分配成本的好方法。但是，由于堆栈是动态增长的，现有的G 最终可能会有一个大栈。因此，当堆栈增长（即超过2K）时，Go 不会保留这些栈。即使是空闲的g，因为会浪费内存。

> Q: 不是说超过256个，P才会推送到全局G队列吗？这里怎么又是64？
>
> A: G 其实是有状态的，P 中最多保存 256 个**可运行**的 g，而 p 本地的 freelist 都是**空闲的** g，超过64时，会推一半的 g 进入全局的 freelist。



#### 生命周期大致流程

![img](/images/lang/go/gmp_30.png)

从一段代码开始分析：

```go
package main

import "fmt"

func main() {
    fmt.Println("Hello world")
}
```

1. runtime创建最初的线程m0和goroutine g0，并把2者关联。
2. 调度器初始化：初始化**m0、栈、垃圾回收**，设置M的最大数量，以及创建和初始化由**GOMAXPROCS**个`P`构成的`P列表`
3. 示例代码中的main函数是`main.main`，`runtime`中也有1个main函数——`runtime.main`，代码经过编译后，`runtime.main`会调用`main.main`，程序启动时会为`runtime.main`创建goroutine，称它为main goroutine吧，然后把main goroutine加入到P的本地队列。
4. 启动m0，m0已经绑定了P，会从P的本地队列获取G，获取到main goroutine。
5. G拥有栈，M根据G中的栈信息和调度信息设置运行环境
6. M运行G
7. G退出，再次回到M获取可运行的G，这样重复下去，直到`main.main`退出，`runtime.main`执行Defer和Panic处理，或调用`runtime.exit`退出程序。

`runtime.main()`: 启动 sysmon 线程；启动 GC 协程；执行 init，即代码中的各种 init 函数；执行 `main.main` 函数。

> 调度器的生命周期几乎占满了一个Go程序的一生，`runtime.main`的goroutine执行之前都是为调度器做准备工作，`runtime.main`的goroutine运行，才是调度器的真正开始，直到`runtime.main`结束而结束

### Goroutine  调度场景

#### 创建 g

正在 M1 上运行的 P，有一个 G1。通过 go func() 创建 G2 后，为了局部性，G2 会被优先放入 P 的本地队列。

![img](/images/lang/go/gmp_32.jpg)

#### g 运行完成

M1 上的 G1 运行完成后（调用 goexit() 函数），M1 上运行的 G 会切换为 G0，然后从 M1 绑定的 P 的本地队列中获取 G2 来执行。

![img](/images/lang/go/gmp_33.jpg)

#### G个数大于本地队列（可运行的 g ）长度

P 的本地队列最多能存 256 个 G，这里以最多能存 4 个为例说明。 

正在 M1 上运行的 G2 要通过 go func() 创建 6 个 G。在前 4 个 G 放入 P 的本地队列中后，由于本地队列已满，创建第 5 个 G（G7）时，会将 P 的**本地队列中前一半**和 G7 一起**打乱顺序**放入全局队列中，P 的本地队列剩下的 G 则往前移动。然后创建第 6 个 G（G8），这时 P 的本地队列还未满，将 G8 放入本地队列中。

![img](/images/lang/go/gmp_34.jpg)

#### M 的自旋状态

创建新的 G 时，运行的 G2 会尝试**唤醒其它空闲的 M** 绑定 P 去执行，如果 G2 唤醒了 M2，M2 绑定了一个 P2，会先运行 M2 的 G0。

这时候的 M2 没有从 P2 的本地队列中找到可用的 G，会进去**自旋状态**（spinning），处于自旋状态的 M2 会尝试**从全局空闲线程队列里获取 G**，放入 P2 的本地队列去执行。

获取的数量计算公式如下，每个 P 应该从全局队列承担一定数量的 G，但又不能太多，要给其他 P 留一些，提高并发执行的效率。

> n = min(len(globrunqsize)/GOMAXPROCS + 1, len(localrunsize/2))

![img](/images/lang/go/gmp_35.jpg)

#### 任务窃取机制

一个处于自旋状态的 M 会尝试先从全局队列寻找可运行的 G，如果全局队列为空，则会从其他 P 偷取一些 G 放到自己绑定的 P 的本地队列，数量是那个 P 运行队列的一半。

![img](/images/lang/go/gmp_36.jpg)

#### G M 发生阻塞

当 G2 发生系统调用进入阻塞，其所在的 M1 也会阻塞（当然，也有 m 不阻塞的情况），进入内核状态等待系统资源。

为了提高并发运行效率，和 M1 绑定的 P1 会从休眠线程队列中寻找空闲的 M3 执行，以避免 P1 本地队列的 G 因为所在的 M1 进入阻塞状态而全部无法执行。

![img](/images/lang/go/gmp_37.jpg)

#### G解除阻塞

当刚才进入系统调用的 G2 解除了阻塞，其所在的 M1 会重新寻找 P 去执行，优先会去找原来的 P。

如果找不到一个 P 进行绑定，则将解除阻塞的 G2 **放入全局队列**，等待其他的 P 获取和调度执行，然后将 M1 放回休眠线程队列中。

![img](/images/lang/go/gmp_38.jpg)

## 源码分析

### 数据结构与状态

#### g

goroutine 的内部数据结构可以在 `runtime.runtime2.go` 文件中

```go
// stack 描述的是 Go 的执行栈，下界和上界分别为 [lo, hi]
// 如果从传统内存布局的角度来讲，Go 的栈实际上是分配在 C 语言中的堆区的
// 所以才能比 ulimit -s 的 stack size 还要大(1GB)
type stack struct {
    lo uintptr
    hi uintptr
}
// g 的运行现场
type gobuf struct {
    sp   uintptr    // sp 寄存器
    pc   uintptr    // pc 寄存器
    g    guintptr   // g 指针
    ctxt unsafe.Pointer // 这个似乎是用来辅助 gc 的
    ret  sys.Uintreg
    lr   uintptr    // 这是在 arm 上用的寄存器，不用关心
    bp   uintptr    // 开启 GOEXPERIMENT=framepointer，才会有这个
}
type g struct {
    // 简单数据结构，lo 和 hi 成员描述了栈的下界和上界内存地址
    stack       stack
	// 在函数的栈增长 prologue 中用 sp 寄存器和 stackguard0 来做比较
	// 如果 sp 比 stackguard0 小(因为栈向低地址方向增长)，那么就触发栈拷贝和调度
	// 正常情况下 stackguard0 = stack.lo + StackGuard
	// 不过 stackguard0 在需要进行调度时，会被修改为 StackPreempt
	// 以触发抢占s
	stackguard0 uintptr
	// stackguard1 是在 C 栈增长 prologue 作对比的对象
	// 在 g0 和 gsignal 栈上，其值为 stack.lo+StackGuard
	// 在其它的栈上这个值是 ~0(按 0 取反)以触发 morestack 调用(并 crash)
	stackguard1 uintptr
	m              *m             // 当前与 g 绑定的 m
    sched          gobuf          // goroutine 的现场
    goid         uint64 // Goroutine的唯一标识符。
    param          unsafe.Pointer // wakeup 时的传入参数
    atomicstatus atomic.Uint32 // Goroutine状态的原子化版本，如运行、阻塞等，用于并发访问。
	preempt        bool     // 抢占标记，这个为 true 时，stackguard0 是等于 stackpreempt 的
    lockedm        muintptr // 如果调用了 LockOsThread，那么这个 g 会绑定到某个 m 上
    gopc           uintptr // 创建该 goroutine 的语句的指令地址
    startpc        uintptr // goroutine 函数的指令地址
	selectDone     uint32         // 该 g 是否正在参与 select，是否已经有人从 select 中胜出
}
```

主要字段如下： 

+ stack：描述了当前协程的栈内存范围 [stack.lo, stack.hi) 
+ stackguard0：用于调度器抢占式调度 
+ 结构体 m：记录当前 G 占用的线程 M，可能为空 
+ sched：存储 G 的调度相关的数据 
+ atomicstatus：表示 G 的状态 
+ goid：表示 G 的 id，对开发者不可见

此外，Goroutine的内部数据结构还包含其他一些字段，用于管理调度、垃圾回收等。这些字段的具体细节可能会因Golang版本和运行时实现而有所不同。

sched 字段的结构体为 runtime.gobuf，会在调度器将当前 G 切换离开 M，以及将当前 G 进入 M 执行时用到，栈指针 sp 和程序计数器 pc 用于存放和恢复寄存器中的值，改变程序执行的指令。

```go
type gobuf struct {
	sp  uintptr  // 栈指针
	pc  uintptr  // 程序计数器，记录G要执行的下一条指令位置
	g   guintptr // 持有 runtime.gobuf 的 G
	ret uintptr  // 系统调用的返回值
	......
}
```

atomicstatus 字段储存当前 G 的状态，枚举如下：

```go
const (
	// _Gidle 表示 G 刚刚被分配并且还没有被初始化
	_Gidle = iota // 0
	// _Grunnable 表示 G  没有执行代码，没有栈的所有权，存储在运行队列中
	_Grunnable // 1
	// _Grunning 可以执行代码，拥有栈的所有权，被赋予了内核线程 M 和处理器 P
	_Grunning // 2
	// _Gsyscall 正在执行系统调用，拥有栈的所有权，没有执行用户代码，被赋予了内核线程 M 但是不在运行队列上
	_Gsyscall // 3
	// _Gwaiting 由于运行时而被阻塞，没有执行用户代码并且不在运行队列上，但是可能存在于 Channel 的等待队列上
	_Gwaiting // 4
	// _Gdead 没有被使用，没有执行代码，可能有分配的栈
	_Gdead // 6
	// _Gcopystack 栈正在被拷贝，没有执行代码，不在运行队列上
	_Gcopystack // 8
	// _Gpreempted 由于抢占而被阻塞，没有执行用户代码并且不在运行队列上，等待唤醒
	_Gpreempted // 9
	// _Gscan GC 正在扫描栈空间，没有执行代码，可以与其他状态同时存在
	_Gscan          = 0x1000
	......
)

```

主要状态有： 

+ _Gidle：G 被创建但还未完全被初始化 
+ _Grunnable：当前 G 可运行，正在等待被运行，当程序中的 G 非常多时，每个 G 就会有更多时间处于当前状态 
+ _Grunning：当前 G 正在某个 M 上运行 
+ _Gsyscall：当前 G 正在执行系统调用 
+ _Gwaiting：当前 G 正在因为某个原因而等待 
+ _Gdead：当前 G 完成了运行

G 从创建到结束的生命周期经历的状态变化如下图。

![img](/images/lang/go/gmp_40.jpg)

当 g 遇到阻塞，或需要等待的场景时，会被打包成 sudog 这样一个结构。一个 g 可能被打包为多个 sudog 分别挂在不同的等待队列上:

```go
// sudog 代表在等待列表里的 g，比如向 channel 发送/接收内容时
// 之所以需要 sudog 是因为 g 和同步对象之间的关系是多对多的
// 一个 g 可能会在多个等待队列中，所以一个 g 可能被打包为多个 sudog
// 多个 g 也可以等待在同一个同步对象上
// 因此对于一个同步对象就会有很多 sudog 了
// sudog 是从一个特殊的池中进行分配的。用 acquireSudog 和 releaseSudog 来分配和释放 sudog
type sudog struct {

    // 之后的这些字段都是被该 g 所挂在的 channel 中的 hchan.lock 来保护的
    // shrinkstack depends on
    // this for sudogs involved in channel ops.
    g *g

    // isSelect 表示一个 g 是否正在参与 select 操作
    // 所以 g.selectDone 必须用 CAS 来操作，以胜出唤醒的竞争
    isSelect bool
    next     *sudog
    prev     *sudog
    elem     unsafe.Pointer // data element (may point to stack)

    // 下面这些字段则永远都不会被并发访问
    // 对于 channel 来说，waitlink 只会被 g 访问
    // 对于信号量来说，所有的字段，包括上面的那些字段都只在持有 semaRoot 锁时才可以访问
    acquiretime int64
    releasetime int64
    ticket      uint32
    parent      *sudog // semaRoot binary tree
    waitlink    *sudog // g.waiting list or semaRoot
    waittail    *sudog // semaRoot
    c           *hchan // channel
}
```

	#### m

M 的数据结构定义在 Go 源码的 src/runtime/runtime2.go 中。

```go
type m struct {
	g0       *g                // 持有调度栈的 G
	gsignal  *g                // 处理 signal 的 g
	tls      [tlsSlots]uintptr // 线程本地存储
	mstartfn func()            // M 的起始函数，go语句携带的那个函数
	curg     *g                // 在当前线程上运行的 G
	p        puintptr          // 执行 go 代码时持有的 p (如果没有执行则为 nil)
	nextp    puintptr          // 用于暂存与当前 M 有潜在关联的 P
	oldp     puintptr          // 执行系统调用前绑定的 P
	spinning bool              // 表示当前 M 是否正在寻找 G，在寻找过程中 M 处于自旋状态
	lockedg  guintptr          // 表示与当前 M 锁定的那个 G
	.....
}
```

主要字段如下：

+ g0：在 Go 程序启动之初创建，用来调度其他 G 到 此 M 上
+ mstartfn：M 的起始函数，也就是使用 go 语句创建协程指定的函数
+ curg：存放当前正在运行的 G 的指针
+ p：指向当前和 M 绑定的 P
+ nextp：暂存和 M 有潜在关联的 P 
+ spinning：当前 M 是否处于自旋状态，也就是处于寻找 G 的状态 
+ lockedg：与当前 M 锁定的那个 G，当系统把一个 M 和一个 G 锁定，就只能双方相互作用，不再接受别的 G

#### p

P 的数据结构定义在 Go 源码的 src/runtime/runtime2.go 中。

```go
type p struct {
	status      uint32    // p 的状态 pidle/prunning/...
	schedtick   uint32    // 每次执行调度器调度 +1
	syscalltick uint32    // 每次执行系统调用 +1
	m           muintptr  // 关联的 m 
	mcache      *mcache   // 用于 P 所在的线程 M 的内存分配的 mcache
	deferpool   []*_defer // 本地 P 队列的 defer 结构体池
	// 可运行的 G 队列，可无锁访问
	runqhead uint32
	runqtail uint32
	runq     [256]guintptr
	// 线程下一个需要执行的 G
	runnext guintptr
	// 空闲的 G 队列，G 状态 status 为 _Gdead，可重新初始化使用
	gFree struct {
		gList
		n int32
	}
	......
}
```

主要字段如下： 

+ status：P 的状态 
+ runqhead、runqtail、runq：P 的本地队列，是一个长度为256的环形队列，储存着待执行的 G 的列表 
+ runnext：线程下一个需要执行的 G 
+ gFree：P 的本地队列中状态为 _Gdead 的空闲的 G，可重新初始化使用 

status 字段的主要状态有： 

+ _Pidle：P 没有运行用户代码或者调度器，运行队列为空 
+ _Prunning：和 M 绑定，正在执行用户代码或调度器 
+ _Psyscall：M 陷入系统调用，没有执行用户代码 
+ _Pgcstop：和 M 绑定，当前处理器由于垃圾回收被停止 
+ _Pdead：当前 P 已经不被使用

![img](/images/lang/go/gmp_41.jpg)


#### 调度源码

[Go GMP调度器的设计与原理](https://huanglianjing.com/posts/go-gmp%E8%B0%83%E5%BA%A6%E5%99%A8%E7%9A%84%E8%AE%BE%E8%AE%A1%E4%B8%8E%E5%8E%9F%E7%90%86/#4-%e6%ba%90%e7%a0%81%e5%88%86%e6%9e%90)

[Golang 调度器 GMP 原理与调度全分析](https://learnku.com/articles/41728)
