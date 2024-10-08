+++
title = '进程间通信'
date = 2024-05-28T21:59:05+08:00
+++

每个进程的用户地址空间都是独立的，一般而言是不能互相访问的，但**内核空间是每个进程都共享的**，所以进程之间要通信**必须通过内核**。

![img](/images/cs/os/procom-1.jpg)

## 管道

linux 指令 管道 `|`：

```bash
ps auxf | grep mysql
```

它的功能是将前一个命令（ps auxf）的输出，作为后一个命令（grep mysql）的输入，从这功能描述，可以看出**管道传输数据是单向的**，如果想相互通信，我们需要创建两个管道才行。

同时，我们得知上面这种管道是没有名字，所以「`|`」表示的管道称为**匿名管道**，用完了就销毁。

管道还有另外一个类型是**命名管道**，也被叫做 `FIFO`，因为数据是**先进先出**的传输方式。

在使用命名管道前，先需要通过 `mkfifo` 命令来创建，并且指定管道名字：

```bash
mkfifo myPipe
```

myPipe 就是这个管道的名称，基于 Linux 一切皆文件的理念，所以管道也是以文件的方式存在，我们可以用 ls 看一下，这个文件的类型是 p，也就是 pipe（管道） 的意思：

接下来，我们往 myPipe 这个管道写入数据：

```bash
echo "hello" > myPipe  # 将数据写入管道
					   # 停住了...
```

你操作了后，你会发现命令执行后就停在这了，这是因为管道里的内容没有被读取，**只有当管道里的数据被读完后，命令才可以正常退出**。

我们执行另外一个命令来读取这个管道里的数据：

```bash
cat < myPipe # 读取管道数据
hello
```

可以看到，管道里的内容被读取出来了，并打印在了终端上，另外一方面，echo 那个命令也正常退出了。

我们可以看出，**管道这种通信方式效率低，不适合进程间频繁地交换数据**。当然，它的好处，自然就是简单，同时也我们很容易得知管道里的数据已经被另一个进程读取了。

### 原理

匿名管道的创建，需要通过下面这个系统调用：

```c
int pipe(int fd[2])
```

这里表示创建一个匿名管道，并**返回了两个描述符**，一个是管道的**读取端描述符** fd[0]，另一个是管道的**写入端描述符** fd[1]。注意，这个**匿名管道是特殊的文件，只存在于内存**，不存于文件系统中。

![img](/images/cs/os/procom-2.jpg)

其实，**所谓的管道，就是内核里面的一串缓存**。从管道的一段写入的数据，实际上是缓存在内核中的，另一端读取，也就是从内核中读取这段数据。另外，管道传输的数据是无格式的流且大小受限。

这两个描述符都是在一个进程里面，并没有起到进程间通信的作用，怎么样才能使得管道是跨过两个进程的呢？

我们可以使用 fork 创建子进程，**创建的子进程会复制父进程的文件描述符**，这样就做到了两个进程各有两个「 fd[0] 与 fd[1]」，两个进程就可以通过各自的 fd 写入和读取同一个管道文件实现跨进程通信了。

![img](/images/cs/os/procom-3.jpg)

管道只能一端写入，另一端读出，所以上面这种模式容易造成混乱，因为父进程和子进程都可以同时写入，也都可以读出。那么，为了避免这种情况，通常的做法是：

+ 父进程关闭读取的 fd[0]，只保留写入的 fd[1]；
+ 子进程关闭写入的 fd[1]，只保留读取的 fd[0]；

![img](/images/cs/os/procom-4.jpg)

所以说如果需要双向通信，则应该创建两个管道。

到这里，我们仅仅解析了使用管道进行父进程与子进程之间的通信，但是在我们 shell 里面并不是这样的。

在 shell 里面执行 A | B命令的时候，**A 进程和 B 进程都是 shell 创建出来的子进程**，A 和 B 之间不存在父子关系，它俩的父进程都是 shell。

![img](/images/cs/os/procom-5.jpg)

所以说，在 shell 里通过「|」匿名管道将多个命令连接在一起，实际上也就是创建了多个子进程，那么在我们编写 shell 脚本时，能使用一个管道搞定的事情，就不要多用一个管道，这样可以减少创建子进程的系统开销。

我们可以得知，**对于匿名管道，它的通信范围是存在父子关系的进程**。因为管道没有实体，也就是没有管道文件，只能通过 fork 来复制父进程 fd 文件描述符，来达到通信的目的。

另外，**对于命名管道，它可以在不相关的进程间也能相互通信**。因为命令管道，**提前创建了一个类型为管道的设备文件**，在进程里只要使用这个设备文件，就可以相互通信。

不管是匿名管道还是命名管道，进程写入的数据都是**缓存在内核**中，另一个进程读取数据时候自然也是从内核中获取，同时通信数据都遵循**先进先出**原则，不支持 lseek 之类的文件定位操作。

## 消息队列

**消息队列是保存在内核中的消息链表**，在发送数据时，会分成一个一个独立的数据单元，也就是消息体（数据块），消息体是用户自定义的数据类型，**消息的发送方和接收方要约定好消息体的数据类型**，所以**每个消息体都是固定大小的存储块**，不像**管道是无格式的字节流数据**。如果进程从消息队列中读取了消息体，内核就会把这个消息体删除。

**消息队列生命周期随内核**，如果没有释放消息队列或者没有关闭操作系统，消息队列会一直存在，而前面提到的**匿名管道的生命周期，是随进程的创建而建立**，随进程的结束而销毁。

消息这种模型，两个进程之间的通信就像平时发邮件一样，你来一封，我回一封，可以频繁沟通了。

但邮件的通信方式存在不足的地方有两点，**一是通信不及时，二是附件也有大小限制**，这同样也是消息队列通信不足的点。

**消息队列不适合比较大数据的传输**，因为在**内核中每个消息体都有一个最大长度的限制**，同时所有**队列所包含的全部消息体的总长度也是有上限**。在 Linux 内核中，会有两个宏定义 MSGMAX 和 MSGMNB，它们以字节为单位，分别定义了一条消息的最大长度和一个队列的最大长度。

**消息队列通信过程中，存在用户态与内核态之间的数据拷贝开销**，因为进程**写入数据**到内核中的消息队列时，会发生从**用户态拷贝数据到内核态**的过程，同理另一进程读取内核中的消息数据时，会发生从内核态拷贝数据到用户态的过程。



## 共享内存

消息队列的读取和写入的过程，都会有发生用户态与内核态之间的消息拷贝过程。那共享内存的方式，就很好的解决了这一问题。

现代操作系统，对于内存管理，采用的是虚拟内存技术，也就是每个进程都有自己独立的虚拟内存空间，不同进程的虚拟内存映射到不同的物理内存中。所以，即使进程 A 和 进程 B 的虚拟地址是一样的，其实访问的是不同的物理内存地址，对于数据的增删查改互不影响。

**共享内存的机制，就是拿出一块虚拟地址空间来，映射到相同的物理内存中**。这样这个进程写入的东西，另外一个进程马上就能看到了，都不需要拷贝来拷贝去，传来传去，大大提高了进程间通信的速度。

![img](/images/cs/os/procom-6.jpg)

## 信号量

用了共享内存通信方式，带来新的问题，那就是**如果多个进程同时修改同一个共享内存，很有可能就冲突了**。例如两个进程都同时写一个地址，那先写的那个进程会发现内容被别人覆盖了。

为了防止多进程竞争共享资源，而造成的数据错乱，所以需要保护机制，**使得共享的资源，在任意时刻只能被一个进程访问**。正好，信号量就实现了这一保护机制。

**信号量其实是一个整型的计数器，主要用于实现进程间的互斥与同步，而不是用于缓存进程间通信的数据**。

信号量表示资源的数量，控制信号量的方式有两种原子操作：

+ 一个是 **P 操作**，这个操作会把信号量减去 1，相减后如果信号量 < 0，则表明资源已被占用，进程需阻塞等待；相减后如果信号量 >= 0，则表明还有资源可使用，进程可正常继续执行。
+ 另一个是 **V 操作**，这个操作会把信号量加上 1，相加后如果信号量 <= 0，**则表明当前有阻塞中的进程，于是会将该进程唤醒运行**；相加后如果信号量 > 0，则表明当前没有阻塞中的进程；

**P 操作是用在进入共享资源之前，V 操作是用在离开共享资源之后，这两个操作是必须成对出现的**。

接下来，举个例子，如果要使得两个进程互斥访问共享内存，我们可以初始化信号量为 1。

![img](/images/cs/os/procom-7.jpg)

具体的过程如下：

+ 进程 A 在访问共享内存前，先执行了 P 操作，由于信号量的初始值为 1，故在进程 A 执行 P 操作后信号量变为 0，表示共享资源可用，于是进程 A 就可以访问共享内存。
+ 若此时，进程 B 也想访问共享内存，执行了 P 操作，结果信号量变为了 -1，这就意味着临界资源已被占用，因此进程 B 被阻塞。
+ 直到进程 A 访问完共享内存，才会执行 V 操作，使得信号量恢复为 0，接着就会唤醒阻塞中的线程 B，使得进程 B 可以访问共享内存，最后完成共享内存的访问后，执行 V 操作，使信号量恢复到初始值 1。

可以发现，**信号初始化为 1，就代表着是互斥信号量**，它可以**保证共享内存在任何时刻只有一个进程在访问**，这就很好的保护了共享内存。

另外，在多进程里，每个进程并不一定是顺序执行的，它们基本是以各自独立的、不可预知的速度向前推进，但有时候我们又希望多个进程能密切合作，以实现一个共同的任务。

例如，进程 A 是负责生产数据，而进程 B 是负责读取数据，这两个进程是相互合作、相互依赖的，进程 A 必须先生产了数据，进程 B 才能读取到数据，所以**执行是有前后顺序**的。

那么这时候，就可以用信号量来实现**多进程同步**的方式，我们可以**初始化信号量为 0**。

![img](/images/cs/os/procom-8.jpg)

具体过程：

+ 如果进程 B 比进程 A 先执行了，那么执行到 P 操作时，由于信号量初始值为 0，故信号量会变为 -1，表示进程 A 还没生产数据，于是进程 B 就阻塞等待；
+ 接着，当进程 A 生产完数据后，执行了 V 操作，就会使得信号量变为 0，于是就会唤醒阻塞在 P 操作的进程 B；
+ 最后，进程 B 被唤醒后，意味着进程 A 已经生产了数据，于是进程 B 就可以正常读取数据了。

可以发现，信号初始化为 0，就代表着是同步信号量，它**可以保证进程 A 应在进程 B 之前执行（A执行V操作，B执行P操作）**。

## 信号

上面说的进程间通信，都是常规状态下的工作模式。**对于异常情况下的工作模式，就需要用「信号」的方式来通知进程**。

在 Linux 操作系统中， 为了响应各种各样的事件，提供了几十种信号，分别代表不同的意义。我们可以通过 kill -l 命令，查看所有的信号：

```bash
$ kill -l
 1) SIGHUP       2) SIGINT       3) SIGQUIT      4) SIGILL       5) SIGTRAP
 6) SIGABRT      7) SIGBUS       8) SIGFPE       9) SIGKILL     10) SIGUSR1
11) SIGSEGV     12) SIGUSR2     13) SIGPIPE     14) SIGALRM     15) SIGTERM
16) SIGSTKFLT   17) SIGCHLD     18) SIGCONT     19) SIGSTOP     20) SIGTSTP
21) SIGTTIN     22) SIGTTOU     23) SIGURG      24) SIGXCPU     25) SIGXFSZ
26) SIGVTALRM   27) SIGPROF     28) SIGWINCH    29) SIGIO       30) SIGPWR
31) SIGSYS      34) SIGRTMIN    35) SIGRTMIN+1  36) SIGRTMIN+2  37) SIGRTMIN+3
38) SIGRTMIN+4  39) SIGRTMIN+5  40) SIGRTMIN+6  41) SIGRTMIN+7  42) SIGRTMIN+8
43) SIGRTMIN+9  44) SIGRTMIN+10 45) SIGRTMIN+11 46) SIGRTMIN+12 47) SIGRTMIN+13
48) SIGRTMIN+14 49) SIGRTMIN+15 50) SIGRTMAX-14 51) SIGRTMAX-13 52) SIGRTMAX-12
53) SIGRTMAX-11 54) SIGRTMAX-10 55) SIGRTMAX-9  56) SIGRTMAX-8  57) SIGRTMAX-7
58) SIGRTMAX-6  59) SIGRTMAX-5  60) SIGRTMAX-4  61) SIGRTMAX-3  62) SIGRTMAX-2
63) SIGRTMAX-1  64) SIGRTMAX
```

运行在 shell 终端的进程，我们可以通过键盘输入某些组合键的时候，给进程发送信号。例如

+ Ctrl+C 产生 SIGINT 信号，表示终止该进程；
+ Ctrl+Z 产生 SIGTSTP 信号，表示停止该进程，但还未结束；

如果进程在后台运行，可以通过 kill 命令的方式给进程发送信号，但前提需要知道运行中的进程 PID 号，例如：

+ kill -9 1050 ，表示给 PID 为 1050 的进程发送 SIGKILL 信号，用来立即结束该进程；

所以，信号事件的来源主要有**硬件来源**（如键盘 Cltr+C ）和**软件来源**（如 kill 命令）。

**信号是进程间通信机制中唯一的异步通信机制**，因为**可以在任何时候发送信号给某一进程**，一旦有信号产生，我们就有下面这几种，用户进程对信号的处理方式：

+ **执行默认操作**。Linux 对每种信号都规定了默认操作，例如，上面列表中的 SIGTERM 信号，就是终止进程的意思。
+ **捕捉信号**。我们可以为信号定义一个信号处理函数。当信号发生时，我们就执行相应的信号处理函数。
+ **忽略信号**。当我们不希望处理某些信号的时候，就可以忽略该信号，不做任何处理。**有两个信号是应用进程无法捕捉和忽略的，即 SIGKILL 和 SEGSTOP，它们用于在任何时候中断或结束某一进程**。

## Socket

前面提到的管道、消息队列、共享内存、信号量和信号都是在同一台主机上进行进程间通信，那要想**跨网络与不同主机上的进程之间通信，就需要 Socket 通信了**。

实际上，Socket 通信不仅可以跨网络与不同主机的进程间通信，**还可以在同主机上进程间通信**。

我们来看看创建 socket 的系统调用：

```c
int socket(int domain, int type, int protocal)
```

三个参数分别代表：

+ domain 参数用来指定协议族，比如 AF_INET 用于 IPV4、AF_INET6 用于 IPV6、AF_LOCAL/AF_UNIX 用于本机；
+ type 参数用来指定通信特性，比如 SOCK_STREAM 表示的是**字节流**，对应 TCP、SOCK_DGRAM 表示的是**数据报**，对应 UDP、SOCK_RAW 表示的是**原始套接字**；
+ protocal 参数原本是用来指定通信协议的，但现在基本废弃。因为协议已经通过前面两个参数指定完成，protocol 目前一般写成 0 即可；

根据创建 socket 类型的不同，通信的方式也就不同：

+ 实现 TCP 字节流通信： socket 类型是 AF_INET 和 SOCK_STREAM；
+ 实现 UDP 数据报通信：socket 类型是 AF_INET 和 SOCK_DGRAM；
+ 实现本地进程间通信： 「本地字节流 socket 」类型是 AF_LOCAL 和 SOCK_STREAM，「本地数据报 socket 」类型是 AF_LOCAL 和 SOCK_DGRAM。另外，AF_UNIX 和 AF_LOCAL 是等价的，所以 AF_UNIX 也属于本地 socket；

接下来，简单说一下这三种通信的编程模式。

### TCP

![img](/images/cs/os/procom-9.jpg)

+ 服务端和客户端初始化 socket，得到文件描述符；
+ 服务端调用 bind，将绑定在 IP 地址和端口;
+ 服务端调用 listen，进行监听；
+ 服务端调用 accept，等待客户端连接；
+ 客户端调用 connect，向服务器端的地址和端口发起连接请求；
+ 服务端 accept 返回用于传输的 socket 的文件描述符；
+ 客户端调用 write 写入数据；服务端调用 read 读取数据；
+ 客户端断开连接时，会调用 close，那么服务端 read 读取数据的时候，就会读取到了 EOF，待处理完数据后，服务端调用 close，表示连接关闭。

这里需要注意的是，**服务端调用 accept 时，连接成功了会返回一个已完成连接的 socket**，后续用来传输数据。

所以，监听的 socket 和真正用来传送数据的 socket，是「两个」 socket，一个叫作**监听 socket**，一个叫作**已完成连接 socket**。

成功连接建立之后，双方开始通过 read 和 write 函数来读写数据，就像往一个文件流里面写东西一样。

> GPT4:
>
> 服务端：
>
> + 监听socket：数量1，这个 `socket` 是由服务端创建并通过 `bind()` 和 `listen()` 函数与特定的 IP 地址和端口绑定，用于监听客户端的连接请求。它**负责接收客户端的连接请求**，但不会用于实际的数据传输。
> + **已完成连接的 `socket`**：数量N（个客户端连接会创建一个新的 `socket`），当服务端调用 `accept()` 函数并成功接收一个客户端的连接请求时，会返回一个新的 `socket`，这个 `socket` 用于**与特定的客户端进行通信**。每当有新的客户端连接进来，服务端都会生成一个新的已完成连接的 `socket`。
>
> 客户端：
>
> + 客户端 socket：数量1（每个客户端有一个 `socket`），客户端通过 `socket()` 函数创建一个 `socket`，然后使用 `connect()` 函数向服务端指定的 IP 地址和端口发起连接请求。**这个 `socket` 在连接成功后，用于与服务端进行数据传输**。
>
> 数据被写入到客户端的 `socket`，然后通过网络发送。数据到达服务端的已连接 `socket`，服务端从这个 `socket` 读取数据。

### UDP

![img](/images/cs/os/procom-10.jpg)

UDP 是没有连接的，所以不需要三次握手，也就不需要像 TCP 调用 listen 和 connect，但是 UDP 的交互仍然需要 IP 地址和端口号，因此也需要 bind。

对于 UDP 来说，不需要要维护连接，那么也就没有所谓的发送方和接收方，甚至都不存在客户端和服务端的概念，**只要有一个 socket 多台机器就可以任意通信，因此每一个 UDP 的 socket 都需要 bind**。

另外，每次通信时，调用 sendto 和 recvfrom，都要传入目标主机的 IP 地址和端口。

### 本地进程间

本地 socket 被用于在同一台主机上进程间通信的场景：

+ 本地 socket 的编程接口和 IPv4 、IPv6 套接字编程接口是一致的，可以支持「字节流」和「数据报」两种协议；
+ 本地 socket 的实现效率大大高于 IPv4 和 IPv6 的字节流、数据报 socket 实现；

对于本地字节流 socket，其 socket 类型是 AF_LOCAL 和 SOCK_STREAM。

对于本地数据报 socket，其 socket 类型是 AF_LOCAL 和 SOCK_DGRAM。

本地字节流 socket 和 本地数据报 socket 在 bind 的时候，不像 TCP 和 UDP 要绑定 IP 地址和端口，而是**绑定一个本地文件**，这也就是它们之间的最大区别。
