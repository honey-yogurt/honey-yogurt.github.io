---
title: '通道'
date: 2024-03-05T14:14:53+08:00
---
> Don’t communicate by sharing memory; share memory by communicating.
> 
> 不要通过共享内存来通信，而应该通过通信来共享内存。

通道类型的值本身就是**并发安全**的，这也是 Go 语言自带的、唯一一个可以满足并发安全性的类型。

## 数据结构

```go
type hchan struct {
	qcount   uint           // 当前队列中剩余的元素个数
	dataqsiz uint           // 环形队列长度，即可以存放的元素个数
	buf      unsafe.Pointer // 环形队列指针
	elemsize uint16         // 每个元素的大小
	closed   uint32         // 标识关闭状态
	elemtype *_type         // 元素类型 
	sendx    uint           // 队列下标，指示元素写入时存放到队列中的位置
	recvx    uint           // 队列下标，指示下一个被读取的元素在队列中位置 
	recvq    waitq          // 等待读消息的协程队列
	sendq    waitq          // 等待写消息的协程队列
	lock mutex              // 互斥锁，chan不允许并发读写
}
```

从数据结构可以看出通道由**环形队列**、**类型信息**、**协程等待队列**组成。

> 由此看出为什么通道是并发安全的，因为通道自带互斥锁。

**环形队列**：数据缓冲区，队列的长度是在创建chan时指定。

sendx和recvx分别表示队尾和队首。sendx 指示元素写入时存放到队列中的位置，recvx指示下一个被读取的元素在队列中位置。

**协程等待队列**：

**一般情况下，recvq和sendq至少有一个为空，只有一个例外，那就是同一个协程使用select语句向通道一边写入数据，一边读取数据，此时协程会分别位于两个等待队列中。**

> 为什么至少有一个为空？
>
> **只有通道内部没有数据时，读协程等待队列才不为空。只有通道数据满了，写等待协程才不为空。**
>
> 所以一般来说只有既满足没有数据，又要满足数据满了，才可能都为空，显然难以成立。所以一般来说，至少有一个为空。甚至多数情况下，二者都为空。即通道有数据，又没有满。
>
> 一个例外：
>
> 注意是**同一个协程，这个通道还得是非缓冲通道**。
>
>
> Q： 为什么 只有通道内部没有数据时，读协程等待队列才不为空。只有通道数据满了，写等待协程才不为空。以及 如果一个通道里面数据没有满，多个goroutine同时写，只有一个拿的到锁，在这一瞬间难道其他 goroutine不会放入 sendq ？即 一个 goroutine 放入等待队列的条件是什么？
>
> A：一个协程是否放入读等待队列还是写等待队列中，其实是**先等到获取到锁之后，再进行一系列状态判断来决定的**。即使一瞬间有无数个goroutine操作通道，只有一个 goroutine 能够获取到锁，并执行相应操作。其他未能获取锁的 goroutine 会等待，只是普通的等待获取锁的状态。所以假设一个通道内部的数据没有满的话，任何一个写协程获取到锁之后，一定是直接写数据成功然后释放锁，不可能加入写等待协程。所以**只有通道数据满了，写等待协程才不为空**。如果一个通道还有数据，一个读协程获取锁之后，会直接读走一个数据，然后释放锁，不可能放入读等待队列。所以**只有通道内部没有数据时，读协程等待队列才不为空**。



elemsize 指示的是每个元素大小，用于在 buf 中定位元素的位置。

> Q：可是如果元素类型是interface{}，怎么设置elemsize大小以及定位元素位置？
>
> A：所有的接口类型在 Go 语言中都是由两个指针组成的：类型指针和数据指针。对于`interface{}`类型的通道，`elemsize`会设置为存储这两个指针所需的大小。因为无论实际存储的数据类型是什么，接口的存储结构都是固定的，即两个指针的大小。在 64 位系统上，通常每个指针占用 8 字节，因此`interface{}`类型的`elemsize`通常是 16 字节（8 字节类型指针 + 8 字节数据指针）。在 32 位系统上，指针大小为 4 字节，因此`elemsize`会是 8 字节。

从数据结构我们知道其实并没有单向通道。

**单向通道只是对通道的一种使用限制**，最主要的用途就是**约束其他代码的行为**，这和C语言使用 const 修饰函数参数为只读是一个道理。

## 通道的类型和值

通道可以是双向的，也可以是单向的。

- 字面形式`chan T`表示一个元素类型为`T`的双向通道类型。 编译器允许从此类型的值中接收和向此类型的值中发送数据。
- 字面形式`chan<- T`表示一个元素类型为`T`的单向写通道类型。 编译器不允许从此类型的值中读数据。
- 字面形式`<-chan T`表示一个元素类型为`T`的单向读通道类型。 编译器不允许向此类型的值中写数据。

双向通道`chan T`的值可以被隐式转换为单向通道类型`chan<- T`和`<-chan T`，但反之不行（即使显式也不行）。 类型`chan<- T`和`<-chan T`的值也不能相互转换。

每个通道值有一个容量属性。 一个容量为0的通道值称为一个非缓冲通道（unbuffered channel），一个容量不为0的通道值称为一个缓冲通道（buffered channel）。

通道类型的零值也使用预声明的`nil`来表示。 一个非零通道值必须通过内置的`make`函数来创建。 比如`make(chan int, 10)`将创建一个元素类型为`int`的通道值。 第二个参数指定了欲创建的通道的容量。此第二个实参是可选的，它的默认值为`0`。

## 通道值的比较

**所有通道类型均为可比较类型。**

从值部一文，我们了解到一个通道值可能含有底层部分。 当一个通道值被赋给另一个通道值后，这两个通道值将**共享**相同的底层部分。 换句话说，这两个通道引用着同一个底层的内部通道对象。 比较这两个通道的结果为`true`。

## 通道的操作

### 声明以及初始化

```go
var ch chan int // 声明通道
```

这种声明方式得到的 channel 值为 nil。

```go
ch1 := make(chan string)  //无缓冲
ch2 := make(chan string, 5) //带缓冲
```

make方式会在 heap 上分配一个 hchan 结构体，返回的是一个指针。

```go
close(ch) // 关闭通道
ch <- v // 写
<-ch // 读
cap(ch) // 容量
len(ch) // 长度
```

+ 关闭通道：通道不能为只读。
+ 在大多数场合下，一个数据接收操作可以被认为是一个单值表达式。它可以返回第二个可选的类型不确定的布尔值返回值从而成为一个多值表达式。 **此类型不确定的布尔值表示第一个返回值是否是在通道被关闭之前被发送的**。
+ 如果被查询的通道为一个 nil 零值通道，则`cap`和`len`函数调用都返回`0`

> 第二个返回值表示第一个返回值是否是在通道被关闭之前被发送的，具体来说即是否成功读取了（有效）数据。
>
> 如果为 false，即代表不是在channel关闭前写进去的数据，那就是默认的零值，通常来说这并不是一个有效的数据。

由于通道结构体自带了一个锁，所以上述操作都是同步操作，即并发安全的。但是Go中大多数的基本操作都是未同步的。换句话说，它们都不是并发安全的。 这些操作包括**赋值、传参、和各种容器值操作**等。

## 通道操作详解

为了让解释简单清楚，在本文后续部分，通道将被归为三类：

1. 零值（nil）通道；
2. 非零值但已关闭的通道；
3. 非零值并且尚未关闭的通道。

下表简单地描述了三种通道操作施加到三类通道的结果。

| 操作   | 一个零值nil通道 | 一个非零值但已关闭的通道 | 一个非零值且尚未关闭的通道 |
| ------ | --------------- | ------------------------ | -------------------------- |
| 关闭   | 产生panic       | 产生panic                | 成功关闭(C)                |
| 写数据 | 永久阻塞        | 产生panic                | 阻塞或者成功写(B)          |
| 读数据 | 永久阻塞        | 永不阻塞(D)              | 阻塞或者成功读(A)          |

> Q：为什么用永久阻塞而不用 panic？
>
> A：在 Go 语言中，向一个`nil`通道读写数据会导致`goroutine`永久阻塞，这是一种特意设计的行为。这种行为允许`select`语句在运行时**动态地启用或禁用特定的`case`**，因为在`select`中，向`nil`通道的读或写操作将被忽略，使其他非`nil`通道的操作有机会执行。这提供了一种灵活的控制并发行为的机制。
>
> Q：如果一个goroutine在读/写一个nil值的channel 时候，永久阻塞了，此时另外一个 goroutine 给这个 nil 的channel 重新赋值，使它不是nil了，原先被永久阻塞的goroutine是否还会被唤醒？
>
> A：既然叫做永久阻塞，肯定不会被唤醒。这是因为阻塞发生在尝试操作（读或写）`nil`通道时，而不是基于通道变量当前引用的通道实体。阻塞的`goroutine`不会“回过头来”检查通道变量是否已经更新。

我们可以认为一个通道内部维护了三个队列（均可被视为**先进先出**（FIFO）队列）：

1. 读数据协程队列（可以看做是先进先出队列但其实并不完全是，见下面解释）。此队列是一个**没有长度限制**的链表。 此队列中的协程均处于**阻塞状态**，它们正等待着从此通道读数据。
2. 写数据协程队列（可以看做是先进先出队列但其实并不完全是，见下面解释）。此队列也是一个**没有长度限制**的链表。 此队列中的协程亦均处于**阻塞状态**，它们正等待着向此通道写数据。 **此队列中的每个协程将要写的值（或者此值的指针，取决于具体编译器实现）和此协程一起存储在此队列中**。
3. 数据缓冲队列。这是一个循环队列（绝对先进先出），它的长度为此通道的容量。此队列中存放的值的类型都为此通道的元素类型。 如果此队列中当前存放的值的个数已经达到此通道的容量，则我们说此通道已经处于满槽状态。 如果此队列中当前存放的值的个数为零，则我们说此通道处于空槽状态。 **对于一个非缓冲通道（容量为零），它总是同时处于满槽状态和空槽状态**。

每个通道内部维护着一个互斥锁用来在各种通道操作中防止数据竞争。

当一个协程 R 尝试从一个非零且尚未关闭的通道**读数据**的时候，此协程R将**首先尝试获取此通道的锁**，**成功**之后将执行下列步骤，直到其中一个步骤的条件得到满足。

1. 如果此通道的缓冲队列不为空（这种情况下，读数据协程队列必为空），此协程R将从缓冲队列**取出一个值**。 如果写数据协程队列不为空（更进一步，即这个缓冲队列不仅不为空，甚至还是满的），**一个写协程将从此队列中弹出**，此协程欲写的值将被推入缓冲队列。**此写协程将恢复至运行状态**。 读数据协程R继续运行，不会阻塞。对于这种情况，**此读操作为一个非阻塞操作**。
2. 否则（即此通道的缓冲队列为空），如果写数据协程队列不为空（这种情况下，此通道必为一个非缓冲通道）， 一个**写数据协程将从此队列中弹出**，**此协程欲写的值将被读协程R接收**。此**写协程将恢复至运行状态**。 读**协程R继续运行，不会阻塞**。对于这种情况，此读操作为一个非阻塞操作。
3. 对于剩下的情况（即此通道的缓冲队列和写数据协程队列均为空，非缓冲通道在第二种情况已经阐述过），此**读协程R将被推入读协程队列，并进入阻塞状态**。 它以后**可能会被另一个写数据协程唤醒而恢复运行**。 对于这种情况，此数据接收操作为一个阻塞操作。

当一个协程 S 尝试向一个非零且尚未关闭的通道**写数据**的时候，此协程S将**首先尝试获取此通道的锁**，**成功**之后将执行下列步骤，直到其中一个步骤的条件得到满足。

1. 如果此通道的读数据协程队列不为空（这种情况下，缓冲队列必为空）， 一个**读数据协程将从此队列中弹出**，**此协程将接收到写协程S发送的值**。**此读协程将恢复至运行状态。 写数据协程S继续运行，不会阻塞**。对于这种情况，此数据发送操作为一个非阻塞操作。
2. 否则（读数据协程队列为空），如果缓冲队列未满（这种情况下，写数据协程队列必为空）， 写协程S欲发送的值将被推入缓冲队列，发送数据协程S继续运行，不会阻塞。 对于这种情况，此数据发送操作为一个非阻塞操作。
3. 对于剩下的情况（读数据协程队列为空，并且缓冲队列已满），此**写协程S将被推入写数据协程队列**，并进入**阻塞**状态。 它以后**可能会被另一个读数据协程唤醒而恢复运行**。 对于这种情况，此数据发送操作为一个阻塞操作。

当一个协程**成功获取到一个非零且尚未关闭的通道的锁并且准备关闭此通道**时，下面两步将依次执行：

1. 如果此通道的**读数据协程队列不为空**（这种情况下，缓冲队列必为空），**此队列中的所有协程将被依个弹出**，并且每个协程将接收到此通道的元素类型的一个**零值**，然后**恢复至运行状态**。
2. 如果此通道的**写数据协程队列不为空**，**此队列中的所有协程将被依个弹出**，并且每个协程中都将产生一个 **panic**（因为向已关闭的通道发送数据）。这就是并发地关闭一个通道和向此通道发送数据这种情形属于不良设计的原因。 事实上，在数据竞争侦测编译选项（race）打开时，Go官方标准运行时将很可能会对并发地关闭一个通道和向此通道发送数据这种情形报告成数据竞争。

当一个缓冲队列不为空的通道被关闭之后，它的**缓冲队列不会被清空**，其中的数据仍然可以被后续的数据读操作所接收到。

一个非零通道被关闭之后，**此通道上的后续数据读操作将永不会阻塞** 。 此通道的缓冲队列中存储**数据仍然可以被读出来**。 伴随着这些接收出来的缓冲数据的第二个可选返回值**仍然是true**。一旦此缓冲队列**变为空**，后续的数据接收操作将永不阻塞并且总会返回此通道的元素类型的**零值和值为false**的第二个可选返回结果。 上面已经提到了，一个读操作的第二个可选返回结果表示一个接收到的值**是否是在此通道被关闭之前写**的。 如果此返回值为false ，则第一个返回值必然是一个此通道的元素类型的零值。

> 也就是，关闭的瞬间（缓冲队列不为空）所有的阻塞的读协程拿到的都是零值，但是之后的协程读已经关闭的通道时候，不会阻塞，该有值还是有值。读完了才是零值。
>
> Q：关闭后，并发的读，协程还会阻塞在读协程队列中吗？
> A：前文所知，并发的读只有一个goroutine可以拿到锁，然后发现有数据，直接读数据然后释放锁。下一个goroutine 拿锁，读数据，释放锁，所以**不可能会阻塞**。

知道哪些通道操作是阻塞的和哪些是非阻塞的对正确理解后面将要介绍的`select`流程控制机制非常重要。

如果一个协程被从一个通道的某个队列中（不论写数据协程队列还是读数据协程队列）弹出，并且此协程是在一个`select`控制流程中推入到此队列的，那么此协程将在下面将要讲解的`select`控制流程的执行步骤中的第*9*步中恢复至运行状态，并且同时它会被从相应的`select`控制流程中的相关的若干通道的协程队列中移除掉。

根据上面的解释，我们可以得出如下的关于一个通道的内部的三个队列的各种事实：

- 如果一个通道被关闭了，则它的写数据协程队列和读数据协程队列肯定都为空，但是它的缓冲队列可能不为空。
- 在任何时刻，如果缓冲队列不为空，则读数据协程队列必为空。
- 在任何时刻，如果缓冲队列未满，则写数据协程队列必为空。
- 如果一个通道是缓冲的，则在任何时刻，它的写数据协程队列和读数据协程队列之一必为空。
- 如果一个通道是非缓冲的，则在任何时刻，一般说来，它的写数据协程队列和读数据协程队列之一必为空， 但是有一个例外：一个协程可能在一个 **select** 流程控制中同时被推入到此通道的写数据协程队列和读数据协程队列中。

## 通道的值传递方式

从通道中接受数据或者是向通道中写入数据，均是通过 **copy** 方式传递值的。

在一个值被从一个协程传递到另一个协程的过程中，此值将被**复制至少一次**。 如果此传递值曾经在某个通道的缓冲队列中停留过，则它在此传递过程中将被复制两次。 一次复制发生在从写协程向缓冲队列推入此值的时候，另一个复制发生在读协程从缓冲队列取出此值的时候。 和赋值以及函数调用传参一样，当一个值被传递时，**只有它的直接部分被复制**。

对于官方标准编译器，最大支持的通道的元素类型的尺寸为`65535`。 但是，一般说来，为了在数据传递过程中避免过大的复制成本，我们**不应该使用尺寸很大的通道元素类型**。 如果欲传送的值的尺寸较大，应该改用**指针类型**做为通道的元素类型。

具体如下：

元素值从外界进入通道时会被复制。更具体地说，**进入通道的并不是在接收操作符右边的那个元素值，而是它的副本**。

元素值从通道进入外界时会被移动。这个移动操作实际上包含了两步，第一步是**生成正在通道中的这个元素值的副本**，并准备给到接收方，第二步是**删除在通道中的这个元素值**。

写操作要么还没复制元素值，要么已经复制完毕，绝不会出现只复制了一部分的情况。

读操作在准备好元素值的副本之后，一定会删除掉通道中的原值，绝不会出现通道中仍有残留的情况。

## channel & select-case

### 一些规则

Go中有一个专门为通道设计的`select-case`分支流程控制语法。 `select-case`流程控制代码块中也可以有若干`case`分支和最多一个`default`分支。在一个`select-case`流程控制中：

+ `select`关键字和`{`之间不允许存在任何表达式和语句。
+ `fallthrough`语句不能被使用。
+ 每个`case`关键字后必须跟随一个通道读数据操作或者一个通道写数据操作。 通道读数据操作可以做为源值出现在一条简单赋值语句中。 以后，一个`case`关键字后跟随的通道操作将被称为一个`case`操作。
+ 所有的非阻塞`case`操作中将有一个被**随机选择执行**（而不是按照从上到下的顺序），然后执行此操作对应的`case`分支代码块。
+ select的**case语句读通道时不会阻塞**，尽管通道中没有数据。这是由于case语句**编译后调用读通道时会明确传入不会阻塞的参数**（后文源码解读中可以看到这个参数），**读不到数据时不会将当前协程加入等待队列，而是直接返回**。
+ 在所有的`case`操作均为阻塞的情况下，如果`default`分支存在，则`default`分支代码块将得到执行； 否则，当前协程将被推入所有阻塞操作中相关的通道的写数据协程队列或者读数据协程队列中，并进入阻塞状态。

> Q：真的推送到所有的channel的相应的等待队列中？
>
> A：推到所有的channel对应的协程等待队列中，其中该协程在一个队列中被唤醒，也会在其他所有的channel的等待队列中弹出。

按照上述规则，一个不含任何分支的`select-case`代码块`select{}`将使当前协程处于永久阻塞状态。

### 原理

下面列出了官方标准运行时中`select-case`流程控制的[实现步骤](https://github.com/golang/go/blob/master/src/runtime/select.go)。

1. 将所有`case`操作中涉及到的通道表达式和发送值表达式按照从上到下，从左到右的顺序一一估值。 在赋值语句中做为源值的数据读操作对应的目标值在此时刻不需要被估值。

2. 将所有分支随机排序。`default`分支总是排在最后。 所有`case`操作中相关的通道可能会有重复的。

3. 为了防止在下一步中造成（和其它协程互相）死锁，对所有`case`操作中相关的通道进行**排序**。 排序依据并不重要，官方Go标准编译器使用通道的地址顺序进行排序。 排序结果中前`N`个通道不存在重复的情况。 `N`为所有`case`操作中涉及到的不重复的通道的数量。 下面，**通道锁顺序**是针对此排序结果中的前`N`个通道来说的，**通道锁逆序**是指此顺序的逆序。

4. 按照上一步中的生成通道锁顺序**获取所有相关的通道的锁**。

5. 按照第*2*步中生成的分支顺序检查相应分支：

    1. 如果这是一个`case`分支并且相应的通道操作是一个向关闭了的通道写数据操作，则按照通道锁逆序解锁所有的通道并在当前协程中产生一个恐慌。 跳到第*12*步。

    2. 如果这是一个`case`分支并且相应的通道操作是非阻塞的，则按照通道锁逆序解锁所有的通道并执行相应的`case`分支代码块。 （此相应的通道操作可能会唤醒另一个处于阻塞状态的协程。） 跳到第*12*步。

    3. 如果这是`default`分支，则按照通道锁逆序解锁所有的通道并执行此`default`分支代码块。 跳到第*12*步。

       （到这里，`default`分支肯定是不存在的，并且所有的`case`操作均为阻塞的。）

6. **将当前协程（和对应`case`分支信息）推入到每个`case`操作中对应的通道的写数据协程队列或读数据协程队列中**。 当前协程可能会被多次推入到同一个通道的这两个队列中，因为多个`case`操作中对应的通道可能为同一个。
7. 使当前协程进入**阻塞状态**并且**按照通道锁逆序解锁**所有的通道。
8. ...，当前协程处于阻塞状态，等待其它协程通过通道操作唤醒当前协程，...
9. 当前协程被另一个协程中的一个通道操作唤醒。 此唤醒通道操作可能是一个通道关闭操作，也可能是一个数据读/写操作。 如果它是一个数据读/写操作，则（当前正被解释的`select-case`流程中）肯定有一个相应`case`操作与之配合传递数据。 在此配合过程中，当前协程将从相应`case`操作相关的通道的读/写数据协程队列中弹出。
10. 按照第*3*步中的生成的通道锁顺序获取所有相关的通道的锁。
11. 将当前协程从各个`case`操作中对应的通道的写数据协程队列或读数据协程队列中（可能以非弹出的方式）移除。
    1. 如果当前协程是被一个通道关闭操作所唤醒，则跳到第*5*步。
    2. 如果当前协程是被一个数据读/写操作所唤醒，则相应的`case`分支已经在第*9*步中知晓。 按照通道锁逆序解锁所有的通道并执行此`case`分支代码块。
12. 完毕。

从此实现中，我们得知

- 一个协程**可能同时多次处于同一个通道的写数据协程队列或读数据协程队列中**。
- 一个协程可能**同时处于不同通道的写数据协程队列或读数据协程队列中**。
- 当一个协程被阻塞在一个`select-case`流程控制中并在以后被唤醒时，它**可能会从多个通道的写数据协程队列和读数据协程队列中被移除**。

## channel & for-range

`for-range`循环控制流程也适用于通道。 此循环将不断地尝试从一个通道接收数据，**直到此通道关闭并且它的缓冲队列为空为止**。和应用于数组/切片/映射的`for-range`语法不同，应用于通道的`for-range`语法中最多只能出现一个循环变量，此循环变量用来存储接收到的值。

```go
for v := range aChannel {
	// 使用v
}
// ---------------  等价于 -------------
for {
	v, ok = <-aChannel
	if !ok {
		break
	}
	// 使用v
}
```

> 注意上文中的 **直到**。如果通道没有被关闭，`for range`循环会一直等待更多的数据写入通道，这在没有更多数据写入的情况下会阻塞该协程，如果该协程是主协程，甚至会导致死锁。

```go
func main() {
	c := make(chan int, 10)
	c <- 1
	c <- 2
	c <- 3
	c <- 4
	c <- 5
	c <- 6
	for cc := range c {
		fmt.Println(cc)
	}
	fmt.Println("I am here!") // for-range 中死锁
}

// ------------------ 修复 ------------------
func main() {
    c := make(chan int, 10)

    // 使用goroutine向通道发送数据
    go func() {
        for i := 1; i <= 6; i++ {
            c <- i
        }
        close(c) // 在发送完成后关闭通道
    }()

    // 使用for range循环从通道读取数据
    for cc := range c {
        fmt.Println(cc)
    }

    fmt.Println("I am here!")
}
```

## 使用反射操作 channel

select 语句可以处理 chan 的 send 和 recv，send 和 recv 都可以作为 case clause。如果我们同时处理两个 chan，就可以写成下面的样子：

```go
    select {
    case v := <-ch1:
        fmt.Println(v)
    case v := <-ch2:
        fmt.Println(v)
    }
```

如果需要处理三个 chan，你就可以再添加一个 case clause，用它来处理第三个 chan。可是，如果要处理 100 个 chan 呢？一万个 chan 呢？或者是，chan 的数量在编译的时候是不定的，在运行的时候需要处理一个 slice of chan，这个时候，也没有办法在编译前写成字面意义的 select。那该怎么办？

通过 `reflect.Select` 函数，你可以将一组运行时的 case clause 传入，当作参数执行。Go 的 select 是伪随机的，它可以在执行的 case 中随机选择一个 case，并把选择的这个 case 的索引（chosen）返回，如果没有可用的 case 返回，会返回一个 bool 类型的返回值，这个返回值用来表示是否有 case 成功被选择。如果是 recv case，还会返回接收的元素。Select 的方法签名如下：

```go
func Select(cases []SelectCase) (chosen int, recv Value, recvOK bool)
```

动态处理两个 chan 的情形：

首先，createCases 函数分别为每个 chan 生成了 recv case 和 send case，并返回一个 reflect.SelectCase 数组。

然后，通过一个循环 10 次的 for 循环执行 reflect.Select，这个方法会从 cases 中选择一个 case 执行。第一次肯定是 send case，因为此时 chan 还没有元素，recv 还不可用。等 chan 中有了数据以后，recv case 就可以被选择了。这样，你就可以处理不定数量的 chan 了。

```go
func main() {
    var ch1 = make(chan int, 10)
    var ch2 = make(chan int, 10)

    // 创建SelectCase
    var cases = createCases(ch1, ch2)

    // 执行10次select
    for i := 0; i < 10; i++ {
        chosen, recv, ok := reflect.Select(cases)
        if recv.IsValid() { // recv case
            fmt.Println("recv:", cases[chosen].Dir, recv, ok)
        } else { // send case
            fmt.Println("send:", cases[chosen].Dir, ok)
        }
    }
}

func createCases(chs ...chan int) []reflect.SelectCase {
    var cases []reflect.SelectCase


    // 创建recv case
    for _, ch := range chs {
        cases = append(cases, reflect.SelectCase{
            Dir:  reflect.SelectRecv,
            Chan: reflect.ValueOf(ch),
        })
    }

    // 创建send case
    for i, ch := range chs {
        v := reflect.ValueOf(i)
        cases = append(cases, reflect.SelectCase{
            Dir:  reflect.SelectSend,
            Chan: reflect.ValueOf(ch),
            Send: v,
        })
    }

    return cases
}
```



## 通道类型

在通道操作详解一节已经详细介绍了一个读或者写协程在不同情况下的具体逻辑，本小节通过简单的图示来具化一下。

### 无缓冲通道（Unbuffered Channel）

无缓冲 channel 没有容量，因此进行任何交换前需要两个 goroutine 同时准备好。当 goroutine 试图将一个资源发送到一个无缓冲的通道并且没有goroutine 等待接收该资源时，该通道将锁住发送 goroutine 并使其等待。当 goroutine 尝试从无缓冲通道接收，并且没有 goroutine 等待发送资源时，该通道将锁住接收 goroutine 并使其等待。
{{% details title="展开图片" closed="true" %}}
![Unbuffered Channel](/images/lang/go/unbuffer-channel.png)
{{% /details %}}

**无缓冲信道的本质是保证同步**。

Receive（读） 先于 Send（写） 发生。 好处: 100% 保证能收到。 代价: 延迟时间未知。

### 缓冲通道（Buffered Channel ）

buffered channel 具有容量，因此其行为可能有点不同。当 goroutine 试图将资源发送到缓冲通道，而该通道已满时，该通道将锁住 goroutine并使其等待缓冲区可用。如果通道中有空间，发送可以立即进行，goroutine 可以继续。当goroutine 试图从缓冲通道接收数据，而缓冲通道为空时，该通道将锁住 goroutine 并使其等待资源被发送。
{{% details title="展开图片" closed="true" %}}
![Buffered Channel ](/images/lang/go/buffer-channel.png)
{{% /details %}}



## 关于通道和垃圾回收

注意，**一个通道被其写数据协程队列和读数据协程队列中的所有协程引用着**。因此，如果一个通道的这两个队列只要有一个不为空，则此通道肯定不会被垃圾回收。 另一方面，如果一个协程处于一个通道的某个协程队列之中，则此协程也肯定不会被垃圾回收，即使此通道仅被此协程所引用。 事实上，一个协程只有在退出后才能被垃圾回收。

## 通道关闭的原则
一个常用的使用Go通道的原则是**不要在数据接收方或者在有多个发送者的情况下关闭通道**。 换句话说，我们只应该让**一个通道唯一的发送者关闭此通道**。 

当然，这并不是一个通用的关闭通道的原则。通用的原则是**不要关闭已关闭的通道**。 如果我们能够保证从某个时刻之后，再没有协程将向一个未关闭的非nil通道发送数据，则一个协程可以安全地关闭此通道。
### 常见的关闭方式
使用`sync.Once`来关闭通道。
```go
type MyChannel struct {
	C    chan T
	once sync.Once
}

func NewMyChannel() *MyChannel {
	return &MyChannel{C: make(chan T)}
}

func (mc *MyChannel) SafeClose() {
	mc.once.Do(func() {
		close(mc.C)
	})
}
```
使用`sync.Mutex`来防止多次关闭一个通道。
```go
type MyChannel struct {
	C      chan T
	closed bool
	mutex  sync.Mutex
}

func NewMyChannel() *MyChannel {
	return &MyChannel{C: make(chan T)}
}

func (mc *MyChannel) SafeClose() {
	mc.mutex.Lock()
	defer mc.mutex.Unlock()
	if !mc.closed {
		close(mc.C)
		mc.closed = true
	}
}

func (mc *MyChannel) IsClosed() bool {
	mc.mutex.Lock()
	defer mc.mutex.Unlock()
	return mc.closed
}
```
不过这两种方式**不能完全有效地避免数据竞争**。 目前的Go白皮书并不保证发生在一个通道上的并发关闭操作和发送操作不会产生数据竞争。 如果一个SafeClose函数和同一个通道上的发送操作同时运行，则数据竞争可能发生（虽然这样的数据竞争一般并不会带来什么危害）。
### 优雅地关闭通道的方法
上一节中介绍的`SafeSend`函数有一个弊端，它的调用不能做为case操作而被使用在select代码块中。本节下面将介绍一些在各种情形下使用纯通道操作来关闭通道的方法。

（为了演示程序的完整性，下面这些例子中使用到了sync.WaitGroup。在实践中，sync.WaitGroup并不是必需的。）
#### M个接收者和一个发送者。发送者通过关闭用来传输数据的通道来传递发送结束信号
```go
package main

import (
	"time"
	"math/rand"
	"sync"
	"log"
)

func main() {
	rand.Seed(time.Now().UnixNano()) // Go 1.20之前需要
	log.SetFlags(0)

	// ...
	const Max = 100000
	const NumReceivers = 100

	wgReceivers := sync.WaitGroup{}
	wgReceivers.Add(NumReceivers)

	// ...
	dataCh := make(chan int)

	// 发送者
	go func() {
		for {
			if value := rand.Intn(Max); value == 0 {
				// 此唯一的发送者可以安全地关闭此数据通道。
				close(dataCh)
				return
			} else {
				dataCh <- value
			}
		}
	}()

	// 接收者
	for i := 0; i < NumReceivers; i++ {
		go func() {
			defer wgReceivers.Done()

			// 接收数据直到通道dataCh已关闭
			// 并且dataCh的缓冲队列已空。
			for value := range dataCh {
				log.Println(value)
			}
		}()
	}

	wgReceivers.Wait()
}
```
#### 一个接收者和N个发送者，此唯一接收者通过关闭一个额外的信号通道来通知发送者不要再发送数据了
我们不能让接收者关闭用来传输数据的通道来停止数据传输，因为这样做违反了通道关闭原则。 但是我们可以让接收者关闭一个额外的信号通道来通知发送者不要再发送数据了。
```go
package main

import (
	"time"
	"math/rand"
	"sync"
	"log"
)

func main() {
	rand.Seed(time.Now().UnixNano()) // Go 1.20之前需要
	log.SetFlags(0)

	// ...
	const Max = 100000
	const NumSenders = 1000

	wgReceivers := sync.WaitGroup{}
	wgReceivers.Add(1)

	// ...
	dataCh := make(chan int)
	stopCh := make(chan struct{})
		// stopCh是一个额外的信号通道。它的
		// 发送者为dataCh数据通道的接收者。
		// 它的接收者为dataCh数据通道的发送者。

	// 发送者
	for i := 0; i < NumSenders; i++ {
		go func() {
			for {
				// 这里的第一个尝试接收用来让此发送者
				// 协程尽早地退出。对于这个特定的例子，
				// 此select代码块并非必需。
				select {
				case <- stopCh:
					return
				default:
				}

				// 即使stopCh已经关闭，此第二个select
				// 代码块中的第一个分支仍很有可能在若干个
				// 循环步内依然不会被选中。如果这是不可接受
				// 的，则上面的第一个select代码块是必需的。
				select {
				case <- stopCh:
					return
				case dataCh <- rand.Intn(Max):
				}
			}
		}()
	}

	// 接收者
	go func() {
		defer wgReceivers.Done()

		for value := range dataCh {
			if value == Max-1 {
				// 此唯一的接收者同时也是stopCh通道的
				// 唯一发送者。尽管它不能安全地关闭dataCh数
				// 据通道，但它可以安全地关闭stopCh通道。
				close(stopCh)
				return
			}

			log.Println(value)
		}
	}()

	// ...
	wgReceivers.Wait()
}
```
对于此额外的信号通道stopCh，它只有一个发送者，即dataCh数据通道的唯一接收者。 dataCh数据通道的接收者关闭了信号通道stopCh，这是不违反通道关闭原则的。

在此例中，数据通道dataCh并没有被关闭。是的，我们不必关闭它。 **当一个通道不再被任何协程所使用后，它将逐渐被垃圾回收掉，无论它是否已经被关闭**。 所以这里的优雅性体现在通过不关闭一个通道来停止使用此通道。

#### M个接收者和N个发送者。它们中的任何协程都可以让一个中间调解协程帮忙发出停止数据传送的信号
这是最复杂的一种情形。我们不能让接收者和发送者中的任何一个关闭用来传输数据的通道，我们也不能让多个接收者之一关闭一个额外的信号通道。 这两种做法都违反了通道关闭原则。 然而，我们可以**引入一个中间调解者角色并让其关闭额外的信号通道来通知所有的接收者和发送者结束工作**。 具体实现见下例。注意其中使用了一个尝试发送操作来向中间调解者发送信号。
```go
package main

import (
	"time"
	"math/rand"
	"sync"
	"log"
	"strconv"
)

func main() {
	rand.Seed(time.Now().UnixNano()) // Go 1.20之前需要
	log.SetFlags(0)

	// ...
	const Max = 100000
	const NumReceivers = 10
	const NumSenders = 1000

	wgReceivers := sync.WaitGroup{}
	wgReceivers.Add(NumReceivers)

	// ...
	dataCh := make(chan int)
	stopCh := make(chan struct{})
		// stopCh是一个额外的信号通道。它的发送
		// 者为中间调解者。它的接收者为dataCh
		// 数据通道的所有的发送者和接收者。
	toStop := make(chan string, 1)
		// toStop是一个用来通知中间调解者让其
		// 关闭信号通道stopCh的第二个信号通道。
		// 此第二个信号通道的发送者为dataCh数据
		// 通道的所有的发送者和接收者，它的接收者
		// 为中间调解者。它必须为一个缓冲通道。

	var stoppedBy string

	// 中间调解者
	go func() {
		stoppedBy = <-toStop
		close(stopCh)
	}()

	// 发送者
	for i := 0; i < NumSenders; i++ {
		go func(id string) {
			for {
				value := rand.Intn(Max)
				if value == 0 {
					// 为了防止阻塞，这里使用了一个尝试
					// 发送操作来向中间调解者发送信号。
					select {
					case toStop <- "发送者#" + id:
					default:
					}
					return
				}

				// 此处的尝试接收操作是为了让此发送协程尽早
				// 退出。标准编译器对尝试接收和尝试发送做了
				// 特殊的优化，因而它们的速度很快。
				select {
				case <- stopCh:
					return
				default:
				}

				// 即使stopCh已关闭，如果这个select代码块
				// 中第二个分支的发送操作是非阻塞的，则第一个
				// 分支仍很有可能在若干个循环步内依然不会被选
				// 中。如果这是不可接受的，则上面的第一个尝试
				// 接收操作代码块是必需的。
				select {
				case <- stopCh:
					return
				case dataCh <- value:
				}
			}
		}(strconv.Itoa(i))
	}

	// 接收者
	for i := 0; i < NumReceivers; i++ {
		go func(id string) {
			defer wgReceivers.Done()

			for {
				// 和发送者协程一样，此处的尝试接收操作是为了
				// 让此接收协程尽早退出。
				select {
				case <- stopCh:
					return
				default:
				}

				// 即使stopCh已关闭，如果这个select代码块
				// 中第二个分支的接收操作是非阻塞的，则第一个
				// 分支仍很有可能在若干个循环步内依然不会被选
				// 中。如果这是不可接受的，则上面尝试接收操作
				// 代码块是必需的。
				select {
				case <- stopCh:
					return
				case value := <-dataCh:
					if value == Max-1 {
						// 为了防止阻塞，这里使用了一个尝试
						// 发送操作来向中间调解者发送信号。
						select {
						case toStop <- "接收者#" + id:
						default:
						}
						return
					}

					log.Println(value)
				}
			}
		}(strconv.Itoa(i))
	}

	// ...
	wgReceivers.Wait()
	log.Println("被" + stoppedBy + "终止了")
}
```
请注意，信号通道`toStop`的容量必须至少为1。 如果它的容量为0，则在中间调解者还未准备好的情况下就已经有某个协程向`toStop`发送信号时，此信号将被抛弃。

我们也可以不使用尝试发送操作向中间调解者发送信号，但信号通道`toStop`的容量必须至少为数据发送者和数据接收者的数量之和，以防止向其发送数据时（有一个极其微小的可能）导致某些发送者和接收者协程永久阻塞。
```go
...
toStop := make(chan string, NumReceivers + NumSenders)
...
			value := rand.Intn(Max)
			if value == 0 {
				toStop <- "sender#" + id
				return
			}
...
				if value == Max-1 {
					toStop <- "receiver#" + id
					return
				}
...
```
我们也没有关闭`dataCh`,我们只要尽快让所有的接收者和发送者协程不再使用这个通道即可。

#### “M个接收者和一个发送者”情形的一个变种：用来传输数据的通道的关闭请求由第三方发出
有时，数据通道（`dataCh`）的关闭请求需要由某个第三方协程发出。对于这种情形，我们可以使用一个额外的信号通道来通知唯一的发送者关闭数据通道（dataCh）。
```go
package main

import (
	"time"
	"math/rand"
	"sync"
	"log"
)

func main() {
	rand.Seed(time.Now().UnixNano()) // Go 1.20之前需要
	log.SetFlags(0)

	// ...
	const Max = 100000
	const NumReceivers = 100
	const NumThirdParties = 15

	wgReceivers := sync.WaitGroup{}
	wgReceivers.Add(NumReceivers)

	// ...
	dataCh := make(chan int)
	closing := make(chan struct{}) // 信号通道
	closed := make(chan struct{})
	
	// 此stop函数可以被安全地多次调用。
	stop := func() {
		select {
		case closing<-struct{}{}:
			<-closed
		case <-closed:
		}
	}
	
	// 一些第三方协程
	for i := 0; i < NumThirdParties; i++ {
		go func() {
			r := 1 + rand.Intn(3)
			time.Sleep(time.Duration(r) * time.Second)
			stop()
		}()
	}

	// 发送者
	go func() {
		defer func() {
			close(closed)
			close(dataCh)
		}()

		for {
			select{
			case <-closing: return
			default:
			}

			select{
			case <-closing: return
			case dataCh <- rand.Intn(Max):
			}
		}
	}()

	// 接收者
	for i := 0; i < NumReceivers; i++ {
		go func() {
			defer wgReceivers.Done()

			for value := range dataCh {
				log.Println(value)
			}
		}()
	}

	wgReceivers.Wait()
}
```

#### “N个发送者”的一个变种：用来传输数据的通道必须被关闭以通知各个接收者数据发送已经结束了
在上面的提到的“N个发送者”情形中，为了遵守通道关闭原则，我们避免了关闭数据通道（dataCh）。 但是有时候，数据通道（dataCh）必须被关闭以通知各个接收者数据发送已经结束。 对于这种“N个发送者”情形，我们可以使用一个中间通道将它们转化为“一个发送者”情形，然后继续使用上一节介绍的技巧来关闭此中间通道，从而避免了关闭原始的dataCh数据通道。
```go
package main

import (
	"time"
	"math/rand"
	"sync"
	"log"
	"strconv"
)

func main() {
	rand.Seed(time.Now().UnixNano()) // Go 1.20之前需要
	log.SetFlags(0)

	// ...
	const Max = 1000000
	const NumReceivers = 10
	const NumSenders = 1000
	const NumThirdParties = 15

	wgReceivers := sync.WaitGroup{}
	wgReceivers.Add(NumReceivers)

	// ...
	dataCh := make(chan int)   // 将被关闭
	middleCh := make(chan int) // 不会被关闭
	closing := make(chan string)
	closed := make(chan struct{})

	var stoppedBy string

	stop := func(by string) {
		select {
		case closing <- by:
			<-closed
		case <-closed:
		}
	}
	
	// 中间层
	go func() {
		exit := func(v int, needSend bool) {
			close(closed)
			if needSend {
				dataCh <- v
			}
			close(dataCh)
		}

		for {
			select {
			case stoppedBy = <-closing:
				exit(0, false)
				return
			case v := <- middleCh:
				select {
				case stoppedBy = <-closing:
					exit(v, true)
					return
				case dataCh <- v:
				}
			}
		}
	}()
	
	// 一些第三方协程
	for i := 0; i < NumThirdParties; i++ {
		go func(id string) {
			r := 1 + rand.Intn(3)
			time.Sleep(time.Duration(r) * time.Second)
			stop("3rd-party#" + id)
		}(strconv.Itoa(i))
	}

	// 发送者
	for i := 0; i < NumSenders; i++ {
		go func(id string) {
			for {
				value := rand.Intn(Max)
				if value == 0 {
					stop("sender#" + id)
					return
				}

				select {
				case <- closed:
					return
				default:
				}

				select {
				case <- closed:
					return
				case middleCh <- value:
				}
			}
		}(strconv.Itoa(i))
	}

	// 接收者
	for range [NumReceivers]struct{}{} {
		go func() {
			defer wgReceivers.Done()

			for value := range dataCh {
				log.Println(value)
			}
		}()
	}

	// ...
	wgReceivers.Wait()
	log.Println("stopped by", stoppedBy)
}
```

## 实践（应用场景）

### 消息交流

从 chan 的内部实现看，它是以一个循环队列的方式存放数据，所以，它有时候也会被当**成线程安全的队列和 buffer 使用**。一个 goroutine 可以安全地往 Channel 中塞数据，另外一个 goroutine 可以安全地从 Channel 中读取数据，goroutine 就可以安全地实现**信息交流**了。

第一个例子是 worker 池的例子。Marcio Castilho 在 [使用 Go 每分钟处理百万请求](http://marcio.io/2015/07/handling-1-million-requests-per-minute-with-golang/) 这篇文章中，就介绍了他们应对大并发请求的设计。他们将用户的请求放在一个 chan Job 中，这个 chan Job 就相当于一个待处理任务队列。除此之外，还有一个 chan chan Job 队列，用来存放可以处理任务的 worker 的缓存队列。

dispatcher 会把待处理任务队列中的任务放到一个可用的缓存队列中，worker 会一直处理它的缓存队列。通过使用 Channel，实现了一个 worker 池的任务处理中心，并且解耦了前端 HTTP 请求处理和后端任务处理的逻辑。

第二个例子是 etcd 中的 node 节点的实现，包含大量的 chan 字段，比如 recvc 是消息处理的 chan，待处理的 protobuf 消息都扔到这个 chan 中，node 有一个专门的 run goroutine 负责处理这些消息。

{{% details title="展开图片" closed="true" %}}
![etcd-node](/images/lang/go//etcd-node.png)
{{% /details %}}

### 数据传递

“击鼓传花”的游戏很多人都玩过，花从一个人手中传给另外一个人，就有点类似流水线的操作。这个花就是数据，花在游戏者之间流转，这就类似编程中的数据传递。

> 有 4 个 goroutine，编号为 1、2、3、4。每秒钟会有一个 goroutine 打印出它自己的编号，要求你编写程序，让输出的编号总是按照 1、2、3、4、1、2、3、4……这个顺序打印出来。

为了实现顺序的数据传递，我们可以定义一个令牌的变量，谁得到令牌，谁就可以打印一次自己的编号，同时将令牌**传递**给下一个 goroutine，我们尝试使用 chan 来实现，可以看下下面的代码。

```go
type Token struct{}

func newWorker(id int, ch chan Token, nextCh chan Token) {
    for {
        token := <-ch         // 取得令牌
        fmt.Println((id + 1)) // id从1开始
        time.Sleep(time.Second)
        nextCh <- token
    }
}
func main() {
    chs := []chan Token{make(chan Token), make(chan Token), make(chan Token), make(chan Token)}

    // 创建4个worker
    for i := 0; i < 4; i++ {
        go newWorker(i, chs[i], chs[(i+1)%4])
    }

    //首先把令牌交给第一个worker
    chs[0] <- struct{}{}
  
    select {}
}
```

这类场景有一个特点，就是当前持有数据的 goroutine 都有一个信箱，信箱使用 chan 实现，goroutine 只需要关注自己的信箱中的数据，处理完毕后，就把结果发送到下一家的信箱中。

### 信号通知

chan 类型有这样一个特点：chan 如果为空，那么，receiver 接收数据的时候就会阻塞等待，直到 chan 被关闭或者有新的数据到来。利用这个机制，我们可以实现 wait/notify 的设计模式。

传统的并发原语 Cond 也能实现这个功能。但是，Cond 使用起来比较复杂，容易出错，而使用 chan 实现 wait/notify 模式，就方便多了。

除了正常的业务处理时的 wait/notify，我们经常碰到的一个场景，就是程序关闭的时候，我们需要在退出之前做一些清理（doCleanup 方法）的动作。这个时候，我们经常要使用 chan。

比如，使用 chan 实现程序的 graceful shutdown，在退出之前执行一些连接关闭、文件 close、缓存落盘等一些动作。

```go
func main() {
  go func() {
      ...... // 执行业务处理
    }()

  // 处理CTRL+C等中断信号
  termChan := make(chan os.Signal)
  signal.Notify(termChan, syscall.SIGINT, syscall.SIGTERM)
  <-termChan 

  // 执行退出之前的清理动作
  doCleanup()
  
  fmt.Println("优雅退出")
}
```

有时候，doCleanup 可能是一个很耗时的操作，比如十几分钟才能完成，如果程序退出需要等待这么长时间，用户是不能接受的，所以，在实践中，我们需要设置一个最长的等待时间。只要超过了这个时间，程序就不再等待，可以直接退出。所以，退出的时候分为两个阶段：

1. closing，代表程序退出，但是清理工作还没做；
2. closed，代表清理工作已经做完。

所以，上面的例子可以改写如下：

```go
func main() {
    var closing = make(chan struct{})
    var closed = make(chan struct{})

    go func() {
        // 模拟业务处理
        for {
            select {
            case <-closing:
                return
            default:
                // ....... 业务计算
                time.Sleep(100 * time.Millisecond)
            }
        }
    }()

    // 处理CTRL+C等中断信号
    termChan := make(chan os.Signal)
    signal.Notify(termChan, syscall.SIGINT, syscall.SIGTERM)
    <-termChan

    close(closing)
    // 执行退出之前的清理动作
    go doCleanup(closed)

    select {
    case <-closed:
    case <-time.After(time.Second):
        fmt.Println("清理超时，不等了")
    }
    fmt.Println("优雅退出")
}

func doCleanup(closed chan struct{}) {
    time.Sleep((time.Minute))
    close(closed)
}
```

**关闭通道是实践中用得最多通知实现方式**。

### 锁

在 chan 的内部实现中，就有一把互斥锁保护着它的所有字段。从外在表现上，chan 的发送和接收之间也存在着 happens-before 的关系，保证元素放进去之后，receiver 才能读取到（关于 happends-before 的关系，是指事件发生的先后顺序关系。

要想使用 chan 实现互斥锁，至少有两种方式。一种方式是先初始化一个 capacity 等于 1 的 Channel，然后再放入一个元素。这个元素就代表锁，谁取得了这个元素，就相当于获取了这把锁。另一种方式是，先初始化一个 capacity 等于 1 的 Channel，它的“空槽”代表锁，谁能成功地把元素发送到这个 Channel，谁就获取了这把锁。

```go
// 使用chan实现互斥锁
type Mutex struct {
    ch chan struct{}
}

// 使用锁需要初始化
func NewMutex() *Mutex {
    mu := &Mutex{make(chan struct{}, 1)}
    mu.ch <- struct{}{}
    return mu
}

// 请求锁，直到获取到
func (m *Mutex) Lock() {
    <-m.ch
}

// 解锁
func (m *Mutex) Unlock() {
    select {
    case m.ch <- struct{}{}:
    default:
        panic("unlock of unlocked mutex")
    }
}

// 尝试获取锁
func (m *Mutex) TryLock() bool {
    select {
    case <-m.ch:
        return true
    default:
    }
    return false
}

// 加入一个超时的设置
func (m *Mutex) LockTimeout(timeout time.Duration) bool {
    timer := time.NewTimer(timeout)
    select {
    case <-m.ch:
        timer.Stop()
        return true
    case <-timer.C:
    }
    return false
}

// 锁是否已被持有
func (m *Mutex) IsLocked() bool {
    return len(m.ch) == 0
}


func main() {
    m := NewMutex()
    ok := m.TryLock()
    fmt.Printf("locked v %v\n", ok)
    ok = m.TryLock()
    fmt.Printf("locked %v\n", ok)
}
```

利用 select+chan 的方式，很容易实现 TryLock、Timeout 的功能。具体来说就是，在 select 语句中，我们可以使用 default 实现 TryLock，使用一个 Timer 来实现 Timeout 的功能。

### 任务编排

重点介绍下多个 chan 的编排方式，总共 5 种，分别是 Or-Done 模式、扇入模式、扇出模式、Stream 和 map-reduce。

#### Or-Done 模式

Or-Done 模式是信号通知模式中更宽泛的一种模式。

> 我们会使用“信号通知”实现某个任务执行完成后的通知机制，在实现时，我们为这个任务定义一个类型为 chan struct{}类型的 done 变量，等任务结束后，我们就可以 close 这个变量，然后，其它 receiver 就会收到这个通知。

这是有一个任务的情况，如果有多个任务，只要有任意一个任务执行完，我们就想获得这个信号，这就是 Or-Done 模式。

比如，你发送同一个请求到多个微服务节点，只要任意一个微服务节点返回结果，就算成功，这个时候，就可以参考下面的实现：

```go
func or(channels ...<-chan interface{}) <-chan interface{} {
    // 特殊情况，只有零个或者1个chan
    switch len(channels) {
    case 0:
        return nil
    case 1:
        return channels[0]
    }

    orDone := make(chan interface{})
    go func() {
        defer close(orDone)

        switch len(channels) {
        case 2: // 2个也是一种特殊情况
            select {
            case <-channels[0]:
            case <-channels[1]:
            }
        default: //超过两个，二分法递归处理
            m := len(channels) / 2
            select {
            case <-or(channels[:m]...):
            case <-or(channels[m:]...):
            }
        }
    }()

    return orDone
}
```

我们可以写一个测试程序测试它：

```go
func sig(after time.Duration) <-chan interface{} {
    c := make(chan interface{})
    go func() {
        defer close(c)
        time.Sleep(after)
    }()
    return c
}


func main() {
    start := time.Now()

    <-or(
        sig(10*time.Second),
        sig(20*time.Second),
        sig(30*time.Second),
        sig(40*time.Second),
        sig(50*time.Second),
        sig(01*time.Minute),
    )

    fmt.Printf("done after %v", time.Since(start))
}
```

这里的实现使用了一个巧妙的方式，**当 chan 的数量大于 2 时，使用递归的方式等待信号**。

在 chan 数量比较多的情况下，递归并不是一个很好的解决方式，根据前面小节的反射的方法，我们也可以实现 Or-Done 模式：

```go
func or(channels ...<-chan interface{}) <-chan interface{} {
    //特殊情况，只有0个或者1个
    switch len(channels) {
    case 0:
        return nil
    case 1:
        return channels[0]
    }

    orDone := make(chan interface{})
    go func() {
        defer close(orDone)
        // 利用反射构建SelectCase
        var cases []reflect.SelectCase
        for _, c := range channels {
            cases = append(cases, reflect.SelectCase{
                Dir:  reflect.SelectRecv,
                Chan: reflect.ValueOf(c),
            })
        }

        // 随机选择一个可用的case
        reflect.Select(cases)
    }()


    return orDone
}
```

#### 扇入模式

在软件工程中，模块的扇入是指有多少个上级模块调用它。而对于我们这里的 Channel 扇入模式来说，就是指有**多个源 Channel 输入、一个目的 Channel 输出**的情况。扇入比就是源 Channel 数量比 1。

每个源 Channel 的元素都会发送给目标 Channel，相当于目标 Channel 的 receiver 只需要监听目标 Channel，就可以接收所有发送给源 Channel 的数据。

```go
func fanInReflect(chans ...<-chan interface{}) <-chan interface{} {
    out := make(chan interface{})
    go func() {
        defer close(out)
        // 构造SelectCase slice
        var cases []reflect.SelectCase
        for _, c := range chans {
            cases = append(cases, reflect.SelectCase{
                Dir:  reflect.SelectRecv,
                Chan: reflect.ValueOf(c),
            })
        }
        
        // 循环，从cases中选择一个可用的
        for len(cases) > 0 {
            i, v, ok := reflect.Select(cases)
            if !ok { // 此channel已经close
                cases = append(cases[:i], cases[i+1:]...) // 从 cases 中删除 closed 的channel
                continue
            }
            out <- v.Interface()
        }
    }()
    return out
}

// ---------------------- 二分递归 --------------------
func fanInRec(chans ...<-chan interface{}) <-chan interface{} {
    switch len(chans) {
    case 0:
        c := make(chan interface{})
        close(c)
        return c
    case 1:
        return chans[0]
    case 2:
        return mergeTwo(chans[0], chans[1])
    default:
        m := len(chans) / 2
        return mergeTwo(
            fanInRec(chans[:m]...),
            fanInRec(chans[m:]...))
    }
}
func mergeTwo(a, b <-chan interface{}) <-chan interface{} {
    c := make(chan interface{})
    go func() {
        defer close(c)
        for a != nil || b != nil { //只要还有可读的chan
            select {
            case v, ok := <-a:
                if !ok { // a 已关闭，设置为nil
                    a = nil
                    continue
                }
                c <- v
            case v, ok := <-b:
                if !ok { // b 已关闭，设置为nil
                    b = nil
                    continue
                }
                c <- v
            }
        }
    }()
    return c
}
```

#### 扇出模式

扇出模式只有一个输入源 Channel，有多个目标 Channel，扇出比就是 1 比目标 Channel 数的值，经常用在设计模式中的观察者模式中（观察者设计模式定义了对象间的一种一对多的组合关系。这样一来，一个对象的状态发生变化时，所有依赖于它的对象都会得到通知并自动刷新）。在观察者模式中，数据变动后，多个观察者都会收到这个变更信号。

下面是一个扇出模式的实现。从源 Channel 取出一个数据后，依次发送给目标 Channel。在发送给目标 Channel 的时候，可以同步发送，也可以异步发送：

```go
func fanOut(ch <-chan interface{}, out []chan interface{}, async bool) {
    go func() {
        defer func() { //退出时关闭所有的输出chan
            for i := 0; i < len(out); i++ {
                close(out[i])
            }
        }()

        for v := range ch { // 从输入chan中读取数据
            v := v
            for i := 0; i < len(out); i++ {
                i := i
                if async { //异步
                    go func() {
                        out[i] <- v // 放入到输出chan中,异步方式
                    }()
                } else {
                    out[i] <- v // 放入到输出chan中，同步方式
                }
            }
        }
    }()
}
```

#### Stream

把 Channel 当作流式管道使用的方式，也就是把 Channel 看作流（Stream），提供跳过几个元素，或者是只取其中的几个元素等方法。

首先，我们提供创建流的方法。这个方法把一个数据 slice 转换成流：

```go
func asStream(done <-chan struct{}, values ...interface{}) <-chan interface{} {
    s := make(chan interface{}) //创建一个unbuffered的channel
    go func() { // 启动一个goroutine，往s中塞数据
        defer close(s) // 退出时关闭chan
        for _, v := range values { // 遍历数组
            select {
            case <-done:
                return
            case s <- v: // 将数组元素塞入到chan中
            }
        }
    }()
    return s
}
```

流创建好以后，该咋处理呢？下面我再给你介绍下实现流的方法。

+ takeN：只取流中的前 n 个数据；
+ takeFn：筛选流中的数据，只保留满足条件的数据；
+ takeWhile：只取前面满足条件的数据，一旦不满足条件，就不再取；
+ skipN：跳过流中前几个数据；
+ skipFn：跳过满足条件的数据；
+ skipWhile：跳过前面满足条件的数据，一旦不满足条件，当前这个元素和以后的元素都会输出给 Channel 的 receiver。

```go
func takeN(done <-chan struct{}, valueStream <-chan interface{}, num int) <-chan interface{} {
    takeStream := make(chan interface{}) // 创建输出流
    go func() {
        defer close(takeStream)
        for i := 0; i < num; i++ { // 只读取前num个元素
            select {
            case <-done:
                return
            case takeStream <- <-valueStream: //从输入流中读取元素
            }
        }
    }()
    return takeStream
}
```

#### map-reduce

map-reduce 是一种处理数据的方式，最早是由 Google 公司研究提出的一种面向大规模数据处理的并行计算模型和方法，开源的版本是 hadoop，前几年比较火。

这里并不是分布式的 map-reduce，而是单机单进程的 map-reduce 方法。

map-reduce 分为两个步骤，第一步是映射（map），处理队列中的数据，第二步是规约（reduce），把列表中的每一个元素按照一定的处理方式处理成结果，放入到结果队列中。

```go
func mapChan(in <-chan interface{}, fn func(interface{}) interface{}) <-chan interface{} {
    out := make(chan interface{}) //创建一个输出chan
    if in == nil { // 异常检查
        close(out)
        return out
    }

    go func() { // 启动一个goroutine,实现map的主要逻辑
        defer close(out)
        for v := range in { // 从输入chan读取数据，执行业务操作，也就是map操作
            out <- fn(v)
        }
    }()

    return out
}

func reduce(in <-chan interface{}, fn func(r, v interface{}) interface{}) interface{} {
    if in == nil { // 异常检查
        return nil
    }

    out := <-in // 先读取第一个元素
    for v := range in { // 实现reduce的主要逻辑
        out = fn(out, v)
    }

    return out
}
```

我们可以写一个程序，这个程序使用 map-reduce 模式处理一组整数，map 函数就是为每个整数乘以 10，reduce 函数就是把 map 处理的结果累加起来：

```go
// 生成一个数据流
func asStream(done <-chan struct{}) <-chan interface{} {
    s := make(chan interface{})
    values := []int{1, 2, 3, 4, 5}
    go func() {
        defer close(s)
        for _, v := range values { // 从数组生成
            select {
            case <-done:
                return
            case s <- v:
            }
        }
    }()
    return s
}

func main() {
    in := asStream(nil)

    // map操作: 乘以10
    mapFn := func(v interface{}) interface{} {
        return v.(int) * 10
    }

    // reduce操作: 对map的结果进行累加
    reduceFn := func(r, v interface{}) interface{} {
        return r.(int) + v.(int)
    }

    sum := reduce(mapChan(in, mapFn), reduceFn) //返回累加结果
    fmt.Println(sum)
}
```



## 源码分析

基本逻辑同通道操作详解一节所说并无差异。并且经过上面学习，下文中的源码应该也是可以比较容易阅读。

### makechan

```go
func makechan(t *chantype, size int) *hchan {
    elem := t.elem
  
        // 略去检查代码
        mem, overflow := math.MulUintptr(elem.size, uintptr(size))
        
    //
    var c *hchan
    switch {
    case mem  0:
      // chan的size或者元素的size是0，不必创建buf
      c = (*hchan)(mallocgc(hchanSize, nil, true))
      c.buf = c.raceaddr()
    case elem.ptrdata  0:
      // 元素不是指针，分配一块连续的内存给hchan数据结构和buf
      c = (*hchan)(mallocgc(hchanSize+mem, nil, true))
            // hchan数据结构后面紧接着就是buf
      c.buf = add(unsafe.Pointer(c), hchanSize)
    default:
      // 元素包含指针，那么单独分配buf
      c = new(hchan)
      c.buf = mallocgc(mem, elem, true)
    }
  
        // 元素大小、类型、容量都记录下来
    c.elemsize = uint16(elem.size)
    c.elemtype = elem
    c.dataqsiz = uint(size)
    lockInit(&c.lock, lockRankHchan)

    return c
  }
```

注意：元素是指针与否，分配方式也有所不同。

### send

Go 在编译发送数据给 chan 的时候，会把 send 语句转换成 chansend1 函数，chansend1 函数会调用 chansend，我们分段学习它的逻辑：

```go
func chansend1(c *hchan, elem unsafe.Pointer) {
    chansend(c, elem, true, getcallerpc())
}
func chansend(c *hchan, ep unsafe.Pointer, block bool, callerpc uintptr) bool {
        // 第一部分
    if c  nil {
      if !block {
        return false
      }
      gopark(nil, nil, waitReasonChanSendNilChan, traceEvGoStop, 2)
      throw("unreachable")  // 第 11 行
    }
      ......
  }
```

最开始，第一部分是进行判断：如果 chan 是 nil 的话，就把调用者 goroutine park（阻塞休眠）， 调用者就永远被阻塞住了，所以，第 11 行是不可能执行到的代码。

```go
  // 第二部分，如果chan没有被close,并且chan满了，直接返回
    if !block && c.closed  0 && full(c) {
      return false
  }
```

第二部分的逻辑是当你往一个已经满了的 chan 实例发送数据时，并且想不阻塞当前调用，那么这里的逻辑是直接返回。chansend1 方法在调用 chansend 的时候设置了阻塞参数，所以不会执行到第二部分的分支里。

```go
  // 第三部分，chan已经被close的情景
    lock(&c.lock) // 开始加锁
    if c.closed != 0 {
      unlock(&c.lock)
      panic(plainError("send on closed channel"))
  }
```

第三部分显示的是，如果 chan 已经被 close 了，再往里面发送数据的话会 panic。

```go
      // 第四部分，从接收队列中出队一个等待的receiver
        if sg := c.recvq.dequeue(); sg != nil {
      // 
      send(c, sg, ep, func() { unlock(&c.lock) }, 3)
      return true
    }
```

第四部分，如果等待队列中有等待的 receiver，那么这段代码就把它从队列中弹出，然后直接把数据交给它（通过 memmove(dst, src, t.size)），而不需要放入到 buf 中，速度可以更快一些。

```go
    // 第五部分，buf还没满
      if c.qcount < c.dataqsiz {
      qp := chanbuf(c, c.sendx)
      if raceenabled {
        raceacquire(qp)
        racerelease(qp)
      }
      typedmemmove(c.elemtype, qp, ep)
      c.sendx++
      if c.sendx  c.dataqsiz {
        c.sendx = 0
      }
      c.qcount++
      unlock(&c.lock)
      return true
    }
```

第五部分说明当前没有 receiver，需要把数据放入到 buf 中，放入之后，就成功返回了。

```go
      // 第六部分，buf满。
        // chansend1不会进入if块里，因为chansend1的block=true
        if !block {
      unlock(&c.lock)
      return false
    }
        ......
```

第六部分是处理 buf 满的情况。如果 buf 满了，发送者的 goroutine 就会加入到发送者的等待队列中，直到被唤醒。这个时候，数据或者被取走了，或者 chan 被 close 了。

### recv

在处理从 chan 中接收数据时，Go 会把代码转换成 chanrecv1 函数，如果要返回两个返回值，会转换成 chanrecv2，chanrecv1 函数和 chanrecv2 会调用 chanrecv。我们分段学习它的逻辑：

```go
    func chanrecv1(c *hchan, elem unsafe.Pointer) {
    chanrecv(c, elem, true)
  }
  func chanrecv2(c *hchan, elem unsafe.Pointer) (received bool) {
    _, received = chanrecv(c, elem, true)
    return
  }

    func chanrecv(c *hchan, ep unsafe.Pointer, block bool) (selected, received bool) {
        // 第一部分，chan为nil
    if c  nil {
      if !block {
        return
      }
      gopark(nil, nil, waitReasonChanReceiveNilChan, traceEvGoStop, 2)
      throw("unreachable")
    }
```

chanrecv1 和 chanrecv2 传入的 block 参数的值是 true，都是阻塞方式，所以我们分析 chanrecv 的实现的时候，不考虑 block=false 的情况。

第一部分是 chan 为 nil 的情况。和 send 一样，从 nil chan 中接收（读取、获取）数据时，调用者会被永远阻塞。

```go
  // 第二部分, block=false且c为空
    if !block && empty(c) {
      ......
    }
```

第二部分你可以直接忽略，因为不是我们这次要分析的场景。

```go
        // 加锁，返回时释放锁
      lock(&c.lock)
      // 第三部分，c已经被close,且chan为空empty
    if c.closed != 0 && c.qcount  0 {
      unlock(&c.lock)
      if ep != nil {
        typedmemclr(c.elemtype, ep)
      }
      return true, false
    }
```

第三部分是 chan 已经被 close 的情况。如果 chan 已经被 close 了，并且队列中没有缓存的元素，那么返回 true、false。

```go
      // 第四部分，如果sendq队列中有等待发送的sender
        if sg := c.sendq.dequeue(); sg != nil {
      recv(c, sg, ep, func() { unlock(&c.lock) }, 3)
      return true, true
    }
```

第四部分是处理 buf 满的情况。这个时候，如果是 unbuffer 的 chan，就直接将 sender 的数据复制给 receiver，否则就从队列头部读取一个值，并把这个 sender 的值加入到队列尾部。

```go
  // 第五部分, 没有等待的sender, buf中有数据
if c.qcount > 0 {
  qp := chanbuf(c, c.recvx)
  if ep != nil {
    typedmemmove(c.elemtype, ep, qp)
  }
  typedmemclr(c.elemtype, qp)
  c.recvx++
  if c.recvx  c.dataqsiz {
    c.recvx = 0
  }
  c.qcount--
  unlock(&c.lock)
  return true, true
}

if !block {
  unlock(&c.lock)
  return false, false
}

    // 第六部分， buf中没有元素，阻塞
    ......
```

第五部分是处理没有等待的 sender 的情况。这个是和 chansend 共用一把大锁，所以不会有并发的问题。如果 buf 有元素，就取出一个元素给 receiver。

第六部分是处理 buf 中没有元素的情况。如果没有元素，那么当前的 receiver 就会被阻塞，直到它从 sender 中接收了数据，或者是 chan 被 close，才返回。

### close

通过 close 函数，可以把 chan 关闭，编译器会替换成 closechan 方法的调用。

下面的代码是 close chan 的主要逻辑。如果 chan 为 nil，close 会 panic；如果 chan 已经 closed，再次 close 也会 panic。否则的话，如果 chan 不为 nil，chan 也没有 closed，就把等待队列中的 sender（writer）和 receiver（reader）从队列中全部移除并唤醒。

```go
    func closechan(c *hchan) {
    if c  nil { // chan为nil, panic
      panic(plainError("close of nil channel"))
    }
  
    lock(&c.lock)
    if c.closed != 0 {// chan已经closed, panic
      unlock(&c.lock)
      panic(plainError("close of closed channel"))
    }

    c.closed = 1  

    var glist gList

    // 释放所有的reader
    for {
      sg := c.recvq.dequeue()
      ......
      gp := sg.g
      ......
      glist.push(gp)
    }
  
    // 释放所有的writer (它们会panic)
    for {
      sg := c.sendq.dequeue()
      ......
      gp := sg.g
      ......
      glist.push(gp)
    }
    unlock(&c.lock)
  
    for !glist.empty() {
      gp := glist.pop()
      gp.schedlink = 0
      goready(gp, 3)
    }
  }
```

















