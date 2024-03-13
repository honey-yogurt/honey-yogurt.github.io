+++
title = 'TCP三次握手和四次挥手'
date = 2024-03-13T16:45:10+08:00
+++
## TCP 头格式
{{% details title="展开图片" closed="true" %}}
![TCP 头格式](/images/cs/network/format,png-20230309230534096.png)
{{% /details %}}

**序列号**：在建立连接时由计算机生成的随机数作为其初始值，通过 SYN 包传给接收端主机，每发送一次数据，就「累加」一次该「数据字节数」的大小。**用来解决网络包乱序问题。**

**确认应答号**：指**下一次「期望」**收到的数据的序列号，发送端收到这个确认应答以后可以认为在这个序号以前的数据都已经被正常接收。**用来解决丢包的问题。**

**控制位：**

- *ACK* (Acknowledgment)：该位为 `1` 时，「确认应答号」的字段变为有效，TCP 规定除了最初建立连接时的 `SYN` 包之外该位必须设置为 `1` 。
- *RST* (Reset)：该位为 `1` 时，表示 TCP 连接中出现异常必须**强制断开连接**。
- *SYN* (Synchronize)：该位为 `1` 时，表示**希望建立连接**，并在其「序列号」的字段进行序列号初始值的设定。
- *FIN* (Finish)：该位为 `1` 时，表示今后不会再有数据发送，**希望断开连接**。当通信结束希望断开连接时，通信双方的主机之间就可以相互交换 `FIN` 位为 1 的 TCP 段。
- PSH：该位为 `1` 时，推送函数，请求立即传送数据。
- URG：紧急指针标志，该位为 `1` 时，表明数据包中的数据是紧急的。

**紧急指针**：与URG标志一起使用，指明紧急数据的结束位置。

> 以下为 GPT-4 回答
>
> Q：紧急指针有什么用？可以优先传输吗？
> 
> A：TCP中的URG位指出数据包含紧急数据，但这并不意味着这个数据包会在网络中得到优先传输。网络设备如路由器和交换机主要根据队列管理和服务质量（QoS）策略来处理流量，它们并不解析TCP头部中的URG标志。
> 紧急指针是TCP头部的一个字段，仅当URG位设置为1时才有效。它指示紧急数据在数据流中的位置，允许接收方知道从哪里开始的数据是紧急的，从而可以优先处理这部分数据。紧急指针实际上是一个偏移量，它告诉接收端紧急数据在数据段内的结束位置。
> 使用URG位，接收方可以理解并优先处理标记为紧急的数据。

**校验和**：用于检测数据在传输过程中的任何变化，确保数据的完整性。

**窗口大小**：接收窗口大小，由接收方指定。

**首部长度**：TCP头大小。

**保留**：这些位必须是0。为了将来定义新的用途所保留。

**选项**：长度不定，但长度必须以是32bits的整数倍。常见的选项包括MSS、SACK、Timestamp等等，后续的内容会分别介绍相关选项。

> 不同版本的 TCP 实现标志位不一致，比如有的还有 **CWR**、 **ECE**、 **NC**
> 比如：其中RFC3540将Reserved字段中的最后一位定义为Nonce标志。用于拥塞控制。
> RFC 3168 引入了 CWR 、ECE

## 为什么需要 TCP 协议？ TCP 工作在哪一层？

`IP` 层是「不可靠」的，它不保证网络包的交付、不保证网络包的按序交付、也不保证网络包中的数据的完整性。
{{% details title="展开图片" closed="true" %}}
![OSI 参考模型与 TCP/IP 的关系](/images/cs/network/format,png-20230309230419839.png)
{{% /details %}}

如果需要**保障网络数据包的可靠性**，那么就需要由上层（传输层）的 `TCP` 协议来负责。

因为 TCP 是一个工作在**传输层**的**可靠**数据传输的服务，它能确保接收端接收的网络包是**无损坏、无间隔、非冗余和按序的。**

## 什么是 TCP

TCP 是**面向连接的、可靠的、基于字节流**的传输层通信协议。
{{% details title="展开图片" closed="true" %}}
![img](/images/cs/network/format,png-20230309230424714.png)
{{% /details %}}

- **面向连接**：一定是「**一对一**」才能连接，不能像 UDP 协议可以一个主机同时向多个主机发送消息，也就是一对多是无法做到的；
- **可靠的**：无论的网络链路中出现了怎样的链路变化，TCP 都可以保证一个报文**一定能够到达接收端**；
- **字节流**：用户消息通过 TCP 协议传输时，消息可能会被操作系统「分组」成多个的 TCP 报文，如果接收方的程序如果不知道「消息的边界」，是无法读出一个有效的用户消息的。并且 TCP 报文是「**有序的**」，当「前一个」TCP 报文没有收到的时候，即使它先收到了后面的 TCP 报文，那么也不能扔给应用层去处理，同时对「重复」的 TCP 报文会自动丢弃。

## 什么是 TCP 连接

我们来看看 RFC 793 是如何定义「连接」的：

> *Connections: The reliability and flow control mechanisms described above require that TCPs initialize and maintain certain status information for each data stream. The combination of this information, including sockets, sequence numbers, and window sizes, is called a connection.*

**用于保证可靠性和流量控制维护的某些状态信息的组合，包括 Socket、序列号和窗口大小称为连接。**
{{% details title="展开图片" closed="true" %}}
![img](/images/cs/network/format,png-20230309230428466.png)
{{% /details %}}

所以我们可以知道，建立一个 TCP 连接是需要客户端与服务端达成上述三个信息的共识。

- **Socket**：**由 IP 地址和端口号组成**
- **序列号**：**用来解决乱序问题等**
- **窗口大小**：**用来做流量控制**

> Q：TCB（传输控制块） 是什么？
>
> A：它是TCP协议内核中维护每个连接状态的一种数据结构。
>
> 一个TCB包含了一个TCP连接所需的全部信息,典型的TCB包括:
>
> 1. 本地IP地址和端口号
> 2. 远程IP地址和端口号
> 3. 连接状态(如LISTEN、SYN-SENT、ESTABLISHED等)
> 4. 发送和接收的最后序列号
> 5. 发送和接收窗口大小
> 6. 重传计时器
> 7. 指向发送缓冲区和接收缓冲区的指针
> 8. 各种连接统计数据(如重传次数等)
>
> TCP内核为每一个TCP连接创建一个独立的TCB实例,并根据连接的动作(如收发报文等)来更新TCB中的状态信息。TCB保存了TCP连接所需的所有参数,内核通过查询和维护TCB就能管理和控制连接的传输。
> 当一个TCP连接建立时,内核会创建并初始化一个TCB结构;当连接终止时,内核会释放这个TCB以回收资源。所以TCB是管理TCP连接状态的关键数据结构。
>
> Q：客户端和服务端建立了TCP连接，就是指客户端和服务端内核都维持了一个有效的对应的PCB实例？TCB就是TCP的一个具体的可视化的表现？
>
> A：当客户端和服务端建立一个TCP连接时,双方内核中都会创建并维护一个对应的TCB(传输控制块)实例。TCB保存了该TCP连接的全部状态信息。
>
> 具体来说:
>
> 1. 客户端内核为这个连接创建一个TCB实例,服务端内核也为同一个连接创建一个TCB实例。
> 2. 双方的TCB实例保存着连接的本地/远程IP地址、端口号、序列号、窗口大小等关键信息,但视角不同。
> 3. 双方内核通过读写自己这一端的TCB,来控制发送数据、重传、确认等动作,从而维持住整个TCP连接。
> 4. 当连接终止时,双方内核释放自己这一端的TCB实例。
>
> 所以您说的"TCP连接就是指客户端和服务端内核都维持了一个有效的对应的TCB实例"的表述是准确的。
>
> TCB可以看作是TCP连接的一种具体的数据结构实现和表现形式,是TCP连接在内核中的"影子"。TCP协议的很多机制都是通过操作TCB这个数据结构来完成的。

## 如何唯一确定一个 TCP 连接

TCP 四元组可以唯一的确定一个连接，四元组包括如下：

+ 源地址
+ 源端口
+ 目的地址
+ 目的端口

{{% details title="展开图片" closed="true" %}}
![TCP 四元组](/images/cs/network/format,png-20230309230433082.png)
{{% /details %}}
源地址和目的地址的字段（32 位）是在 IP 头部中，作用是通过**IP 协议发送报文给对方主机**。

源端口和目的端口的字段（16 位）是在 TCP 头部中，作用是告诉 **TCP 协议应该把报文发给哪个进程**。

> 有一个 IP 的服务端监听了一个端口，它的 TCP 的最大连接数是多少？

服务端通常固定在某个本地端口上监听，等待客户端的连接请求。因此，**客户端 IP 和端口是可变的**，其理论值计算公式如下：

{{% details title="展开图片" closed="true" %}}
![img](/images/cs/network/format,png-20230309230436594.png)
{{% /details %}}
对 IPv4，客户端的 IP 数最多为 `2` 的 `32` 次方，客户端的端口数最多为 `2` 的 `16` 次方，也就是服务端单机最大 TCP 连接数，约为 `2` 的 `48` 次方。

当然，服务端最大并发 TCP 连接数远不能达到理论上限，会受以下因素影响：

- **文件描述符限制**，**每个 TCP 连接都是一个文件**，如果文件描述符被占满了，会发生 Too many open files。Linux 对可打开的文件描述符的数量分别作了三个方面的限制：
    - **系统级**：当前系统可打开的最大数量，通过 `cat /proc/sys/fs/file-max` 查看；
    - **用户级**：指定用户可打开的最大数量，通过 `cat /etc/security/limits.conf` 查看；
    - **进程级**：单个进程可打开的最大数量，通过 `cat /proc/sys/fs/nr_open` 查看；
- **内存限制**，每个 TCP 连接都要占用一定内存，操作系统的内存是有限的，如果内存资源被占满后，会发生 OOM。

> 以下为 GPT-4 回答
>
> Q：每个 TCP 连接都是一个文件？我可以查看这么文件吗？
>
> A：实际上是指在操作系统层面上，**TCP连接被抽象为一个文件描述符**。在Unix和类Unix系统中（如Linux），几乎所有的I/O操作，包括对普通文件、设备文件、管道、套接字（socket）的操作，都是通过文件描述符来实现的。文件描述符是一个**非负整数**，为操作系统管理打开的文件或其他I/O资源提供了一个**唯一的引用**。
>
> 这里的“文件”并不意味着有一个实际的文件存储在磁盘上，而是说操作系统利用文件描述符这种抽象机制来管理TCP连接，包括套接字。套接字（socket）连接的状态信息（如连接状态、序列号、窗口大小等）被操作系统内核管理，而不是存储在可直接访问的文件中。
>
> 因此，你不能像访问一个普通文件那样查看这些状态信息。不过，你可以通过使用一些系统诊断工具和命令来查看系统中当前的TCP连接状态和相关信息。例如：
>
> - `netstat`：这个工具可以显示系统中的网络连接、路由表、接口统计信息等。使用`netstat -tunap`命令可以查看到当前的TCP连接，包括使用中的端口和连接状态。
> - `ss`：这是另一个实用工具，用于查看套接字统计信息。它可以被认为是`netstat`的现代替代品，提供了更多的信息和更快的执行速度。使用`ss -tunap`可以获得关于TCP和UDP连接的详细信息。
> - `/proc`文件系统：Linux的`/proc`文件系统包含了大量的系统信息，其中`/proc/net/tcp`和`/proc/net/udp`文件包含了TCP和UDP连接的详细状态信息。通过查看这些文件，你可以获取到系统当前所有TCP和UDP连接的详细状态，但这些信息是以十六进制和低级格式呈现的，解读起来可能较为复杂。

## UDP 和 TCP 有什么区别呢？分别的应用场景是？

UDP 不提供复杂的控制机制，利用 IP 提供面向「**无连接**」的通信服务。

UDP 协议非常简单，头部只有 `8` 个字节（64 位），UDP 的头部格式如下：

{{% details title="展开图片" closed="true" %}}
![UDP 头部格式](/images/cs/network/format,png-20230309230439961.png)
{{% /details %}}
- 目标和源端口：主要是告诉 UDP 协议应该把报文发给哪个进程。
- 包长度：该字段保存了 UDP **首部的长度跟数据的长度之和**。
- 校验和：校验和是为了提供**可靠**的 UDP 首部和数据而设计，**防止**收到在网络传输中**受损**的 UDP 包。

**TCP 和 UDP 区别：**

*1. 连接*

- TCP 是面向连接的传输层协议，传输数据前**先要建立连接**。
- UDP 是不需要连接，**即刻传输**数据。

*2. 服务对象*

- TCP 是**一对一**的两点服务，即一条连接只有两个端点。
- UDP 支持一对一、一对多、多对多的交互通信

*3. 可靠性*

- TCP 是**可靠交付数据**的，数据可以无差错、不丢失、不重复、按序到达。
- UDP 是**尽最大努力交付**，不保证可靠交付数据。但是我们可以基于 UDP 传输协议实现一个可靠的传输协议，比如 QUIC 协议，具体可以参见这篇文章：[如何基于 UDP 协议实现可靠传输？](https://xiaolincoding.com/network/3_tcp/quic.html)

*4. 拥塞控制、流量控制*

- TCP 有**拥塞控制和流量控制**机制，保证数据传输的安全性。
- UDP 则没有，即使网络非常拥堵了，也不会影响 UDP 的发送速率。

*5. 首部开销*

- TCP 首部长度较长，会有一定的开销，首部在没有使用「选项」字段时是 `20` 个字节，如果使用了「选项」字段则会变长的。即**至少20个字节**。
- UDP 首部**只有 8 个字节**，并且是固定不变的，开销较小。

*6. 传输方式*

- TCP 是**流式传输**，没有边界，但保证顺序和可靠。
- UDP 是一个包一个包的发送，是有边界的，但可能会**丢包和乱序**。

*7. 分片不同*

- TCP 的数据大小如果大于 MSS 大小，则会在传输层进行分片，目标主机收到后，也同样在传输层组装 TCP 数据包，如果中途丢失了一个分片，只需要传输丢失的这个分片。
- UDP 的数据大小如果大于 MTU 大小，则会在 IP 层进行分片，目标主机收到后，在 IP 层组装完数据，接着再传给传输层。

> Q：什么是 MSS 和 MTU？
>
> A：`MTU`：一个网络包的**最大长度**，以太网中一般为 `1500` 字节；`MSS`：除去 IP头部 和 TCP 头部之后，一个网络包所能容纳的 TCP 数据的**最大长度**；

**TCP 和 UDP 应用场景：**

由于 TCP 是面向连接，能保证数据的可靠性交付，因此经常用于：

- `FTP` 文件传输；
- HTTP / HTTPS；

由于 UDP 面向无连接，它可以随时发送数据，再加上 UDP 本身的处理既简单又高效，因此经常用于：

- 包总量较少的通信，如 `DNS` 、`SNMP` 等；
- 视频、音频等多媒体通信；
- 广播通信；

> 以下为 GPT-4 回答
>
> Q：为什么视频，音频看起来是连续的数据很符合流式传输，怎么不用 TCP呢？
> 
> A：
>
> + **低延迟**：视频和音频流通常需要实时或接近实时的传输，以保持流畅的播放体验。UDP提供了比TCP更低的延迟，因为它**不需要建立连接、不保证数据包的顺序、也不需要等待丢失的数据包被重新发送**。这意味着即使某些数据包丢失，视频或音频内容也可以继续播放，只是质量可能稍有下降。
> + **多播和广播传输**：UDP支持多播和广播传输，允许数据包同时发送给多个接收者，这对于实时视频会议和在线直播等应用非常有用。相比之下，TCP是基于点对点的通信模型，不支持这种类型的传输。
> + **容忍数据丢失**：对于视频和音频流来说，维持**播放的连贯性和实时性比完全的数据完整性更重要**。丢失的帧可能导致短暂的画面或声音质量下降，但这通常优于等待数据重新传输所引起的延迟或卡顿。UDP允许应用层直接处理数据的任何丢失或重排序问题，而不是依赖于底层协议的重试机制。
> + **节省带宽**：**UDP的头部开销比TCP小**，因为它不包含用于数据可靠性和流控制的复杂机制。这意味着在相同网络条件下，使用UDP可以将更多的带宽用于实际的媒体数据传输，而不是协议开销。
> + **灵活的错误恢复和控制策略**：使用UDP时，应用程序可以根据自己的需求实现定制的错误恢复策略。例如，对于视频播放，一个简单的策略可能是跳过丢失的帧，而对于音频通话，可能会采用某种形式的错误隐藏或数据插补来掩盖丢包带来的影响。
>
> Q：TCP 不是也能实现“多播”吗？只要服务端和多个客户端都建立连接，然后服务端向所有的客户端push数据就行了。
>
> A：
>
> + **一对一的连接**：TCP是一种基于连接的协议，这意味着每个TCP连接只能在一个客户端和一个服务器之间建立。因此，即使服务器向多个客户端发送相同的数据，也需要为每个客户端维护一个独立的连接。这增加了服务器的负载，尤其是在客户端数量众多时。
> + **缺乏效率**：在多播通信中，数据包被发送一次，可以被网络中的多个接收者同时接收。而TCP需要服务器为每个连接单独发送数据，这意味着相同的数据必须被复制和发送多次，这样会增加网络带宽的使用和服务器的工作负载。
> + **资源消耗**：对于每一个独立的TCP连接，服务器都必须分配内存和计算资源来维护连接状态、处理流量控制和进行可靠性传输。当客户端数量大幅增加时，这种资源消耗会显著增长，可能导致服务器资源紧张。
> + **缺乏多播优化**：网络设备（如路由器）和协议（如IGMP）支持IP多播，允许数据包在网络中高效地分发给多个接收者。TCP缺乏这种内建的多播支持，因此无法利用这些网络层面的多播效率优化。



> 为什么 UDP 头部没有「首部长度」字段，而 TCP 头部有「首部长度」字段呢？

原因是 TCP 有**可变长**的「选项」字段，而 UDP 头部长度则是**不会变化**的，无需多一个字段去记录 UDP 的首部长度。

> 为什么 UDP 头部有「包长度」字段，而 TCP 头部则没有「包长度」字段呢？

先说说 TCP 是如何计算负载数据长度：
{{% details title="展开图片" closed="true" %}}
![img](/images/cs/network/format,png-20230309230445811.png)
{{% /details %}}

其中 **IP 总长度 和 IP 首部长度，在 IP 首部格式是已知的**。TCP 首部长度，则是在 TCP 首部格式已知的，所以就可以求得 TCP 数据的长度。

大家这时就奇怪了问：“UDP 也是基于 IP 层的呀，那 UDP 的数据长度也可以通过这个公式计算呀？ 为何还要有「包长度」呢？”

这么一问，确实感觉 UDP 的「包长度」是冗余的。

我查阅了很多资料，我觉得有两个比较靠谱的说法：

- 第一种说法：因为**为了网络设备硬件设计和处理方便，首部长度需要是 `4` 字节的整数倍**。如果去掉 UDP 的「包长度」字段，那 UDP 首部长度就不是 `4` 字节的整数倍了，所以我觉得这可能是为了补全 UDP 首部长度是 `4` 字节的整数倍，才补充了「包长度」字段。
- 第二种说法：**如今的 UDP 协议是基于 IP 协议发展的，而当年可能并非如此**，依赖的可能是别的不提供自身报文长度或首部长度的网络层协议，因此 UDP 报文首部需要有长度字段以供计算。

## TCP 连接建立

### 三次握手

TCP 是面向连接的协议，所以使用 TCP 前必须先建立连接，而**建立连接是通过三次握手来进行的**。三次握手的过程如下图：
{{% details title="展开图片" closed="true" %}}
![TCP 三次握手](/images/cs/network/TCP三次握手.drawio.png)
{{% /details %}}

+ 一开始，客户端和服务端都处于 `CLOSE` 状态。先是服务端主动监听某个端口，处于 `LISTEN` 状态

{{% details title="展开图片" closed="true" %}}
![第一个报文 —— SYN 报文](/images/cs/network/format,png-20230309230500953.png)
{{% /details %}}
+ 客户端会随机初始化序号（`client_isn`），将此序号置于 TCP 首部的「序号」字段中，同时把 `SYN` 标志位置为 `1`，表示 `SYN` 报文。接着把第一个 SYN 报文发送给服务端，表示向服务端发起连接，**该报文不包含应用层数据**，之后客户端处于 `SYN-SENT` 状态。

{{% details title="展开图片" closed="true" %}}
![第二个报文 —— SYN + ACK 报文](/images/cs/network/format,png-20230309230504118.png)
{{% /details %}}
+ 服务端收到客户端的 `SYN` 报文后，首先服务端也随机初始化自己的序号（`server_isn`），将此序号填入 TCP 首部的「序号」字段中，其次把 TCP 首部的「确认应答号」字段填入 `client_isn + 1`, 接着把 `SYN` 和 `ACK` 标志位置为 `1`。最后把该报文发给客户端，该**报文也不包含应用层数据**，之后服务端处于 `SYN-RCVD` 状态。

{{% details title="展开图片" closed="true" %}}
![第三个报文 —— ACK 报文](/images/cs/network/format,png-20230309230508297.png)
{{% /details %}}
+ 客户端收到服务端报文后，还要向服务端回应最后一个应答报文，首先该应答报文 TCP 首部 `ACK` 标志位置为 `1` ，其次「确认应答号」字段填入 `server_isn + 1` ，最后把报文发送给服务端，这次报文**可以携带客户到服务端的数据**，之后客户端处于 `ESTABLISHED` 状态。

+ 服务端收到客户端的应答报文后，也进入 `ESTABLISHED` 状态。

> **第三次握手是可以携带数据的，前两次握手是不可以携带数据的**
>
> 即 同步报文 是不能携带数据的，两个线程连基本状态都没有完成同步，连接都没有建立成功，携带数据也没用。
>
> Q：为什么没用？假如它携带数据了为什么不能处理这个数据？或者直接处理这个数据有什么危险？
> 
> A：假设第一次 SYN客户端就携带数据了 ，服务器也处理数据了，客户端会知道这个服务器回应（假设通过第二次SYN携带回来）是想要的数据吗（万一是上一个相同四元组连接的历史数据呢），有没有丢失数据呢？因为客户端还没有收到服务端的初始化序号，怎么知道接收的数据有没有丢失呢？比如服务端传到数据是从序列号100-300，可是客户端收到的是200-300，客户端并不知道开始序号是多少，丢了数据也不知道。
> 只有在第一次第二次 SYN 包中，协商好 TCP 连接的基本状态信息才能保证后面的数据正常接收。

### 为什么是三次而不是两次或者四次

#### 面试版

- 三次握手才可以**阻止重复历史连接的初始化**（主要原因）
- 三次握手才可以同步双方的初始序列号
- 三次握手才可以避免资源浪费

#### 详解版

在前面我们知道了什么是 **TCP 连接**：

- 用于保证可靠性和流量控制维护的某些状态信息，这些信息的组合，包括 **Socket、序列号和窗口大小**称为连接。

所以，重要的是**为什么三次握手才可以初始化 Socket、序列号和窗口大小并建立 TCP 连接。**

接下来，以三个方面分析三次握手的原因：

- 三次握手才可以**阻止重复历史连接的初始化**（主要原因）
- 三次握手才可以同步双方的初始序列号
- 三次握手才可以避免资源浪费

*原因一：避免历史连接*

我们来看看 RFC 793 指出的 TCP 连接使用三次握手的**首要原因**：

> *The principle reason for the three-way handshake is to prevent old duplicate connection initiations from causing confusion.*

三次握手的**首要原因是为了防止旧的重复连接初始化造成混乱。**

我们考虑一个场景，客户端先发送了 SYN（seq = 90）报文，然后客户端宕机了，而且这个 SYN 报文还被网络阻塞了，服务端并没有收到，接着客户端重启后（相同的TCP四元组，这并不是一个罕见的情况），又重新向服务端建立连接，发送了 SYN（seq = 100）报文（*注意！不是重传 SYN，重传的 SYN 的序列号是一样的*）。

> **简单来说就是旧的 SYN包在新的 SYN 包之前到达服务端** 。

看看三次握手是如何阻止历史连接的：
{{% details title="展开图片" closed="true" %}}
![三次握手避免历史连接](/images/cs/network/format,png-20230309230525514.png)
{{% /details %}}

客户端连续发送多次 SYN（**都是同一个四元组**）建立连接的报文，在**网络拥堵**情况下：

- 一个「旧 SYN 报文」比「最新的 SYN」 报文早到达了服务端，那么此时服务端就会回一个 `SYN + ACK` 报文给客户端，此报文中的确认号是 91（90+1）。
- 客户端收到后，发现自己期望收到的确认号应该是 100 + 1，而不是 90 + 1，于是就会回 RST 报文。
- 服务端收到 RST 报文后，就会释放连接。
- 后续最新的 SYN 抵达了服务端后，客户端与服务端就可以正常的完成三次握手了。

> *以下为 GPT-4 回答*
>
> Q：服务端收到客户端的 RST 报文之后，服务端释放连接，具体是做什么呢？
>
> A： 当服务器端收到客户端发送的RST报文后,它会执行以下操作来释放连接:
>
> 1. **关闭对应的传输控制块(TCB)**   传输控制块(Transmission Control Block,TCB)包含了TCP连接的全部状态信息。服务器端会根据RST报文中的四元组信息找到对应的TCB,并关闭它。
> 2. **删除重传队列和接收队列中的数据**   重传队列和接收队列存有该连接的未传送和已接收但未被上层应用读取的数据。服务器会清空这些队列中的数据。
> 3. **删除socket对应的数据结构**   每个socket在内核中都有一个数据结构表示,服务器会释放这个数据结构所占用的内存空间。
> 4. **从半连接队列中移除对应的socket**   如果RST报文对应的是一个处于半连接状态(SYN-RCVD状态)的连接请求,服务器会从半连接队列中移除该socket结构。
> 5. **向高层应用返回错误码**   如果有应用程序在等待这个连接,内核会向它返回一个错误码(如ECONNRESET),通知它连接已被重置。
> 6. **发送最后的ACK确认**     根据TCP的规范,**服务器在处理完RST报文后,应当发送一个ACK确认报文,通知对方已经终止连接**。 总的来说,释放连接的过程就是根据RST报文找到对应的连接状态数据,并彻底清理该连接所占用的所有内核资源,使得连接完全终止。这个过程确保了TCP连接的正确拆除,避免出现资源泄露等问题。

上述中的「旧 SYN 报文」称为历史连接，TCP 使用三次握手建立连接的**最主要原因就是防止「历史连接」初始化了连接**。

> 如果服务端在收到  RST 报文之前，先收到了「新 SYN 报文」，也就是服务端收到客户端报文的顺序是：「旧 SYN 报文」->「新 SYN 报文」，此时会发生什么?
>
> 当服务端第一次收到 SYN 报文，也就是收到 「旧 SYN 报文」时，就会回复 `SYN + ACK` 报文给客户端，此报文中的确认号是 91（90+1）。
>
> 然后这时再收到「新 SYN 报文」时，就会回 [Challenge Ack (opens new window)](https://xiaolincoding.com/network/3_tcp/challenge_ack.html)报文给客户端，**这个 ack 报文并不是确认收到「新 SYN 报文」的，而是上一次的 ack 确认号**，也就是91（90+1）。所以客户端收到此 ACK 报文时，发现自己期望收到的确认号应该是 101，而不是 91，于是就会回 RST 报文。

> *以下为 GPT-4 回答*
>
> Q: 为什么 Challenge Ack 是确认旧的报文呢？
>
> A：这实际上是一个确认旧的、已经确认过的序列号的ACK。
>
> + **保护机制**：通过确认已经确认过的序列号，挑战应答作为一种保护机制，以防止潜在的攻击者利用伪造的序列号来破坏或劫持TCP连接。这种方法确保了连接的一致性和安全性，防止了攻击者的干扰。
> + **误差检测**：在网络通信中，数据包可能因为各种原因（如网络拥塞、路径变化等）丢失或延迟。挑战应答通过确认旧的报文来帮助检测这些异常情况，使得TCP协议能够正确处理这些情况，而不是错误地将其视为新的或未经确认的数据。
> + **避免意外的连接重置**：如果挑战应答确认了一个不期望的序列号（例如，一个未来的序列号），它可能导致连接被意外重置。通过确认一个已知的、安全的序列号，TCP避免了因序列号异常而导致的连接不稳定或中断。
> + **简化错误恢复**：确认旧的报文简化了错误恢复过程。它允许发送方和接收方在已经建立的连接基础上，利用现有的机制（如重传）来处理可能的错误或丢包，而无需引入额外的复杂性来处理异常情况。
>
> 我的理解：
>
> 至少在这个场景中，服务端收到第一个 SYN （即旧的SYN），服务端是不可能知道客户端已经宕机了，所以服务端理所应当的认为第一个 SYN 是正确的，出于保护机制（不能来一个就确认下一个是正确的，这样很容易被伪造以及破坏TCP连接）依然重复确认旧的 SYN。

**如果是两次握手连接，就无法阻止历史连接**，那为什么 TCP 两次握手为什么无法阻止历史连接呢？

我先直接说结论，主要是因为**在两次握手的情况下，服务端没有中间状态给客户端来阻止历史连接，导致服务端可能建立一个历史连接，造成资源浪费**。

在两次握手的情况下，服务端在收到 SYN 报文后，就进入 ESTABLISHED 状态，意味着这时可以给对方发送数据，但是客户端此时还没有进入 ESTABLISHED 状态，假设这次是历史连接，**客户端判断到此次连接为历史连接，那么就会回 RST 报文来断开连接，而服务端在第一次握手的时候就进入 ESTABLISHED 状态**，**所以它可以发送数据的（比如一些服务是服务端 PUSH 数据的）**，但是它并不知道这个是历史连接，它只有在收到 RST 报文后，才会断开连接。
{{% details title="展开图片" closed="true" %}}
![两次握手无法阻止历史连接](/images/cs/network/fe898053d2e93abac950b1637645943f.png)
{{% /details %}}

可以看到，如果采用两次握手建立 TCP 连接的场景下，服务端在向客户端发送数据前，并没有阻止掉历史连接，导致服务端建立了一个历史连接，又白白发送了数据，妥妥地浪费了服务端的资源。

> Q：建立了历史连接，浪费了服务端什么资源？即建立了一个 TCP 连接，客户端和服务端都需要付出什么资源？
>
> A：
>
> + 至少有**内存和处理器资源**：服务端需要分配内存来创建并维护TCB（传输控制块），用于存储TCP连接的状态信息。同时，处理器资源也将被用于处理过时的SYN报文，以及后续的握手过程和该连接的任何数据处理。
> + **带宽和网络资源**：在建立连接的过程中，服务端需要发送SYN-ACK响应，并处理客户端的ACK响应。这些额外的网络通信会占用网络带宽和资源。

因此，**要解决这种现象，最好就是在服务端发送数据前，也就是建立连接之前，要阻止掉历史连接，这样就不会造成资源浪费，而要实现这个功能，就需要三次握手**。

所以，**TCP 使用三次握手建立连接的最主要原因是防止「历史连接」初始化了连接。**

*原因二：同步双方初始序列号*

TCP 协议的通信双方， 都必须维护一个「序列号」， 序列号是可靠传输的一个关键因素，它的作用：

- 接收方可以去除重复的数据；
- 接收方可以根据数据包的序列号按序接收；
- 可以标识发送出去的数据包中， 哪些是已经被对方收到的（通过 ACK 报文中的序列号知道）；

可见，序列号在 TCP 连接中占据着非常重要的作用，所以当客户端发送携带「初始序列号」的 `SYN` 报文的时候，需要服务端回一个 `ACK` 应答报文，表示客户端的 SYN 报文已被服务端成功接收，那当服务端发送「初始序列号」给客户端的时候，依然也要得到客户端的应答回应，**这样一来一回，才能确保双方的初始序列号能被可靠的同步。**
{{% details title="展开图片" closed="true" %}}
![四次握手与三次握手](/images/cs/network/format,png-20230309230639121.png)
{{% /details %}}

四次握手其实也能够可靠的同步双方的初始化序号，但由于**第二步和第三步可以优化成一步**，所以就成了「三次握手」。

**而两次握手只保证了一方的初始序列号能被对方成功接收，没办法保证双方的初始序列号都能被确认接收。**

*原因三：避免资源浪费*

如果只有「两次握手」，当客户端发生的 `SYN` 报文在网络中阻塞，客户端没有接收到 `ACK` 报文，就会重新发送 `SYN` ，**由于没有第三次握手，服务端不清楚客户端是否收到了自己回复的 `ACK` 报文，所以服务端每收到一个 `SYN` 就只能先主动建立一个连接**，这会造成什么情况呢？

如果客户端发送的 `SYN` 报文在网络中阻塞了，重复发送多次 `SYN` 报文，那么服务端在收到请求后就会**建立多个冗余的无效链接，造成不必要的资源浪费。**
{{% details title="展开图片" closed="true" %}}
![两次握手会造成资源浪费](/images/cs/network/format,png-20230309230636571.png)
{{% /details %}}

即两次握手会造成消息滞留情况下，服务端重复接受无用的连接请求 `SYN` 报文，而造成重复分配资源。

不使用「两次握手」和「四次握手」的原因：

- 「两次握手」：无法防止历史连接的建立，会造成双方资源的浪费，也无法可靠的同步双方序列号；
- 「四次握手」：三次握手就已经理论上最少可靠连接建立，所以不需要使用更多的通信次数。

## 如何在 Linux 系统中查看 TCP 状态？

TCP 的连接状态查看，在 Linux 可以通过 `netstat -napt` 命令查看。

> `netstat` 是一个常用于检查网络和网络连接统计的命令行工具。`-napt` 参数通常与 Linux 系统中的 `netstat` 命令一起使用，但要注意的是，这个组合实际上是将多个参数合并在一起使用的，而不是代表某个具体技术或协议的简写。
>
> - `-n` 选项让 `netstat` 显示网络地址和端口号而不是尝试将它们解析成名字；即使用数字形式显示地址和端口。
> - `-a` 选项表示显示所有连接和监听端口。
> - `-p` 选项显示哪个进程在使用指定的套接字/端口（需要有足够的权限，通常是 root 权限）。
> - `-t` 选项指定显示 TCP 连接信息。
>
> 因此，`-napt` 实际上是 `-n`, `-a`, `-p`, `-t` 这几个选项的组合，用于显示数字地址的所有 TCP 连接及其相关进程信息，而不是某个特定术语的简写。

{{% details title="展开图片" closed="true" %}}
![TCP 连接状态查看](/images/cs/network/format,png-20230309230520683.png)
{{% /details %}}


## 既然 IP 层会分片，为什么 TCP 层还需要 MSS 呢？
{{% details title="展开图片" closed="true" %}}
![MTU 与 MSS](/images/cs/network/format,png-20230309230633447.png)
{{% /details %}}

- `MTU`：一个网络包的**最大长度**，以太网中一般为 `1500` 字节；
- `MSS`：除去 IP 和 TCP 头部之后，一个网络包所能容纳的 TCP 数据的**最大长度**；

如果在 TCP 的整个报文（头部 + 数据）交给 IP 层进行分片，会有什么异常呢？

**以下是假设**：

当 IP 层有一个超过 `MTU` 大小的数据（TCP 头部 + TCP 数据）要发送，那么 IP 层就要进行分片，把数据分片成若干片，保证每一个分片都小于 MTU。把一份 IP 数据报进行分片以后，由目标主机的 IP 层来进行重新组装后，再交给上一层 TCP 传输层。

这看起来井然有序，但这存在隐患的，**那么当如果一个 IP 分片丢失，整个 IP 报文的所有分片都得重传**。

因为 IP 层本身没有超时重传机制，它由传输层的 TCP 来负责超时和重传。

当某一个 IP 分片丢失后，接收方的 IP 层就无法组装成一个完整的 TCP 报文（头部 + 数据），也就无法将数据报文送到 TCP 层，所以接收方不会响应 ACK 给发送方，因为发送方迟迟收不到 ACK 确认报文，所以会触发超时重传，就会**重发「整个 TCP 报文（头部 + 数据）」**。

所以，为了达到最佳的传输效能 TCP 协议在**建立连接的时候通常要协商双方的 MSS 值**，当 TCP 层发现数据超过 MSS 时，则就先会进行分片，当然由它形成的 IP 包的长度也就不会大于 MTU ，自然也就不用 IP 分片了。

> IP 不用分片了，丢失这个 IP 包之后，只需要传这个 IP 包，这个 IP包可能就是 TCP 报文的一部分，只需要传一部分，上一中方式要传全部。假设 MSS = 100字节

{{% details title="展开图片" closed="true" %}}
![握手阶段协商 MSS](/images/cs/network/format,png-20230309230628926.png)
{{% /details %}}
经过 TCP 层分片后，如果一个 TCP 分片丢失后，**进行重发时也是以 MSS 为单位**，而不用重传所有的分片，大大增加了重传的效率。

## 第一次握手丢失了会怎么样

### 面试版

客户端触发超时重传（指数退避策略，1s,2s,4s,8s... ），重传的 SYN 报文的序列号都是一样的。重传次数超过操作系统规定的值（一般默认为5）后继续等待上次 RTO*2，断开TCP连接。

> RTO：超时重传（等待）时长

> Q：断开TCP连接会先发一个 RST 吗？
>
> A：一般来说不需要。
> RST报文**主要用于在已建立的连接中出现错误时，或者需要强制关闭一个连接时使用**。如果连接尚未建立，就不存在需要显式关闭的已建立连接，因此发送RST报文并非必需。
> 然而，不同的操作系统和TCP栈实现可能有不同的行为，一些系统可能在放弃连接尝试时选择发送RST报文，尽管这在技术上不是必须的。这样做可能是为了通知远端主机客户端已经停止了连接尝试，特别是在远端主机可能已经收到了某些重传的SYN报文但还未来得及响应时。但这种行为更多取决于具体实现，而**不是TCP协议的明确要求**。

### 详解版

当客户端想和服务端建立 TCP 连接的时候，首先第一个发的就是 SYN 报文，然后进入到 `SYN_SENT` 状态。

在这之后，如果客户端迟迟收不到服务端的 SYN-ACK 报文（第二次握手），就会触发「超时重传」机制，重传 SYN 报文，而且**重传的 SYN 报文的序列号都是一样的**。

> 在没有建立连接时候，只有超时重传机制。快重传等机制都是在建立连接之后（协商好状态数据）才能生效

不同版本的操作系统可能超时时间不同，有的 1 秒的（后文以这个为标准），也有 3 秒的，这个超时时间是写死在内核里的，如果想要更改则需要重新编译内核，比较麻烦。

当客户端在 1 秒后没收到服务端的 SYN-ACK 报文后，客户端就会重发 SYN 报文，那到底重发几次呢？

在 Linux 里，客户端的 SYN 报文最大重传次数由 `tcp_syn_retries`内核参数控制，这个参数是可以自定义的，默认值一般是 5。

```sh
cat /proc/sys/net/ipv4/tcp_syn_retries
5
```

通常，第一次超时重传是在 1 秒后，第二次超时重传是在 2 秒，第三次超时重传是在 4 秒后，第四次超时重传是在 8 秒后，第五次是在超时重传 16 秒后。没错，**每次超时的时间是上一次的 2 倍**。

**当第五次超时重传后，会继续等待 32 （RTO*2）秒**，如果服务端仍然没有回应 ACK，客户端就不再发送 SYN 包，然后**断开 TCP 连接**。

> 继续等待不是固定32秒，还是上一次超时时间的两倍。

所以，总耗时是 1+2+4+8+16+32=63 秒，大约 1 分钟左右。

举个例子，假设 tcp_syn_retries 参数值为 3，那么当客户端的 SYN 报文一直在网络中丢失时，会发生下图的过程：

{{% details title="展开图片" closed="true" %}}
![img](/images/cs/network/第1次握手丢失.png)
{{% /details %}}

具体过程：

- 当客户端超时重传 3 次 SYN 报文后，由于 tcp_syn_retries 为 3，已达到最大重传次数，于是再等待一段时间（时间为上一次超时时间的 2 倍），如果还是没能收到服务端的第二次握手（SYN-ACK 报文），那么客户端就会断开连接。

## 第二次握手丢失了，会发生什么？

### 面试版

+ 客户端以为第一次握手丢失了（因为第二次握手是对第一次握手的ACK包），客户端超时重发（见《第一次握手丢失》一节）。
+ 服务端收不到第二次握手的ACK（也就是第三次握手），服务端也会触发超时重发（指数退避策略），超过最大次数之后继续等待一个指数退避间隔，最后断开连接。

> 服务端在第二次握手丢失之后断开TCP连接是否会发RST同样取决于具体的TCP实现和操作系统行为。
> 一些TCP栈可能选择发送RST报文，从而允许双方清理相关资源，而其他实现可能认为既然连接未完全建立，就没有必要发送RST。

### 详解版

当服务端收到客户端的第一次握手后，就会回 SYN-ACK 报文给客户端，这个就是第二次握手，此时服务端会进入 `SYN_RCVD` 状态。

第二次握手的 `SYN-ACK` 报文其实有两个目的 ：

- 第二次握手里的 ACK， 是对第一次握手的确认报文；
- 第二次握手里的 SYN，是服务端发起建立 TCP 连接的报文；

所以，如果第二次握手丢了，就会发生比较有意思的事情，具体会怎么样呢？

因为第二次握手报文里是包含对客户端的第一次握手的 ACK 确认报文，所以，如果客户端迟迟没有收到第二次握手，那么**客户端就觉得可能自己的 SYN 报文（第一次握手）丢失了**，于是**客户端就会触发超时重传机制，重传 SYN 报文**。

然后，因为第二次握手中包含服务端的 SYN 报文，所以当客户端收到后，需要给服务端发送 ACK 确认报文（第三次握手），服务端才会认为该 SYN 报文被客户端收到了。

那么，如果**第二次握手丢失了，服务端就收不到第三次握手**，于是**服务端这边会触发超时重传机制，重传 SYN-ACK 报文**。

在 Linux 下，SYN-ACK 报文的最大重传次数由 `tcp_synack_retries`内核参数决定，默认值是 5。

```sh
cat /proc/sys/net/ipv4/tcp_synack_retries
5
```

因此，当第二次握手丢失了，客户端和服务端都会重传：

- 客户端会重传 SYN 报文，也就是第一次握手，最大重传次数由 `tcp_syn_retries`内核参数决定；
- 服务端会重传 SYN-ACK 报文，也就是第二次握手，最大重传次数由 `tcp_synack_retries` 内核参数决定。

举个例子，假设 tcp_syn_retries 参数值为 1，tcp_synack_retries 参数值为 2，那么当第二次握手一直丢失时，发生的过程如下图：
{{% details title="展开图片" closed="true" %}}
![img](/images/cs/network/第2次握手丢失.png)
{{% /details %}}

具体过程：

- 当客户端超时重传 1 次 SYN 报文后，由于 tcp_syn_retries 为 1，已达到最大重传次数，于是再等待一段时间（时间为上一次超时时间的 2 倍），如果还是没能收到服务端的第二次握手（SYN-ACK 报文），那么客户端就会断开连接。
- 当服务端超时重传 2 次 SYN-ACK 报文后，由于 tcp_synack_retries 为 2，已达到最大重传次数，于是再等待一段时间（时间为上一次超时时间的 2 倍），如果还是没能收到客户端的第三次握手（ACK 报文），那么服务端就会断开连接。

## 第三次握手丢失了，会发生什么？

### 面试版

服务端以为第二次握手丢失了，服务端超时重传第二次握手报文，直到收到第三次握手，或者达到最大重传次数之后继续等待一个指数退避间隔，然后断开连接。

### 详解版

客户端收到服务端的 SYN-ACK 报文后，就会给服务端回一个 ACK 报文，也就是第三次握手，此时客户端状态进入到 `ESTABLISH` 状态。

因为这个第三次握手的 ACK 是对第二次握手的 SYN 的确认报文，所以当第三次握手丢失了，如果**服务端**那一方迟迟收不到这个确认报文，就会**触发超时重传机制**，重传 SYN-ACK 报文，直到收到第三次握手，或者达到最大重传次数。

> Q：客户端发送三次握手（ack 报文）后就可以发送数据了，而被动方此时还是 syn_received 状态，如果 ack 丢了，那客户端发的数据是不是也白白浪费了？
>
> A：不是的，即使服务端还是在 syn_received 状态，收到了客户端发送的数据，还是可以建立连接的，并且还可以正常收到这个数据包。这是因为数据报文中是有 ack 标识位，也有确认号，这个确认号就是确认收到了第二次握手。如下图：
> ![img](/images/cs/network/相同ack.png) 
>
> 所以，服务端收到这个数据报文，是可以正常建立连接的，然后就可以正常接收这个数据包了。
>
> Q：为什么 ack 是一样的？
>
> A：数据报文紧接着 第三次握手ACK报文发出去，服务端没有发送任何数据过来，所以客户端数据报文确认服务端的上一个报文的序列号不会改变。所以是一样的。
>
> Q：如果第三次握手携带了数据然后丢失了，客户端紧接着发了另外一个携带数据的包，会是什么样的情景？
>
> A：由上面可知，连续发的数据的ACK是确定的，服务端收到了后面的包，确认客户端收到了第二次握手，所以 TCP 连接可以照常建立。服务端发现收到的包的 seq 不是自己在第二次握手中的ACK值（也就是不是自己期待的下一个序列号），服务端就知道了包延时或者丢失了。接下来就是 TCP 的重传机制发挥作用了，比如：超时重传，快速重传等，详见《TCP重传&流量控制&拥塞控制》一文。

注意，**ACK 报文是不会有重传的，当 ACK 丢失了，就由对方重传对应的报文**。

> 仅是 ACK 报文丢失不会重传的，第二次握手是 SYN+ACK，丢失了会重传。携带数据的ACK（PSH+ACK）也会重传

举个例子，假设 tcp_synack_retries 参数值为 2，那么当第三次握手一直丢失时，发生的过程如下图：

{{% details title="展开图片" closed="true" %}}
![img](/images/cs/network/第三次握手丢失.drawio.png)
{{% /details %}}

具体过程：

- 当服务端超时重传 2 次 SYN-ACK 报文后，由于 tcp_synack_retries 为 2，已达到最大重传次数，于是再等待一段时间（时间为上一次超时时间的 2 倍），如果还是没能收到客户端的第三次握手（ACK 报文），那么服务端就会断开连接。

## TCP 连接断开

### 四次挥手

**双方都可以主动断开连接**，断开连接后主机中的「资源」将被释放，四次挥手的过程如下图：
{{% details title="展开图片" closed="true" %}}
![客户端主动关闭连接 —— TCP 四次挥手](/images/cs/network/format,png-20230309230614791.png)
{{% /details %}}

- 客户端打算关闭连接，此时会发送一个 TCP 首部 `FIN` 标志位被置为 `1` 的报文，也即 `FIN` 报文，之后客户端进入 `FIN_WAIT_1` 状态。
- 服务端收到该报文后，就向客户端发送 `ACK` 应答报文，接着服务端进入 `CLOSE_WAIT` 状态。
- 客户端收到服务端的 `ACK` 应答报文后，之后进入 `FIN_WAIT_2` 状态。
- **等待服务端处理完数据**后，也向客户端发送 `FIN` 报文，之后服务端进入 `LAST_ACK` 状态。
- 客户端收到服务端的 `FIN` 报文后，回一个 `ACK` 应答报文，之后进入 `TIME_WAIT` 状态
- 服务端收到了 `ACK` 应答报文后，就进入了 `CLOSE` 状态，至此服务端已经完成连接的关闭。
- 客户端在经过 `2MSL` 一段时间后，自动进入 `CLOSE` 状态，至此客户端也完成连接的关闭。

你可以看到，每个方向都需要**一个 FIN 和一个 ACK**，因此通常被称为**四次挥手**。

这里一点需要注意是：**主动关闭连接的，才有 TIME_WAIT 状态。**

> 服务端主动关闭流程和上面一模一样，就是把上图中两边的图调换以下，服务器端主动关闭同样也有 TIME_WAIT。

### 为什么挥手需要四次？

再来回顾下四次挥手双方发 `FIN` 包的过程，就能理解为什么需要四次了。

- 关闭连接时，客户端向服务端发送 `FIN` 时，仅仅表示客户端不再发送数据了但是还能接收数据。
- 服务端收到客户端的 `FIN` 报文时，先回一个 `ACK` 应答报文，而服务端可能还有数据需要处理和发送，等服务端不再发送数据时，才发送 `FIN` 报文给客户端来表示同意现在关闭连接。

从上面过程可知，服务端通常需要等待完成数据的发送和处理，所以服务端的 `ACK` 和 `FIN` 一般都会分开发送，因此是需要四次挥手。

> 但是**在特定情况下，四次挥手是可以变成三次挥手的**，详情参考《TCP四次挥手，可以变成三次吗》

## 第一次挥手丢失了，会发生什么？

### 面试版

超时重传（指数退避策略）FIN报文，达到操作系统规定的次数之后继续等待一个指数退避间隔，直接进入close状态。

> Q：异常进入close状态会发送 RST 报文吗？
> 
> A：无论是主动端还是被动端，异常进入 close 状态之前都会发送一个 RST 报文。发送RST是应对异常情况而非正常关闭流程的做法。

### 详情版

当客户端（主动关闭方）调用 close 函数后，就会向服务端发送 FIN 报文，试图与服务端断开连接，此时客户端的连接进入到 `FIN_WAIT_1` 状态。

正常情况下，如果能及时收到服务端（被动关闭方）的 ACK，则会很快变为 `FIN_WAIT2`状态。

如果第一次挥手丢失了，那么客户端迟迟收不到被动方的 ACK 的话，也就会**触发超时重传机制，重传 FIN 报文**，重发次数由 `tcp_orphan_retries` 参数控制。

当客户端重传 FIN 报文的次数超过 `tcp_orphan_retries` 后，就不再发送 FIN 报文，则会在等待一段时间（时间为上一次超时时间的 2 倍），如果还是没能收到第二次挥手，那么**直接进入到 `close` 状态**。

举个例子，假设 tcp_orphan_retries 参数值为 3，当第一次挥手一直丢失时，发生的过程如下图：

{{% details title="展开图片" closed="true" %}}
![第一次挥手丢失](/images/cs/network/第一次挥手丢失.png)
{{% /details %}}
具体过程：

- 当客户端超时重传 3 次 FIN 报文后，由于 tcp_orphan_retries 为 3，已达到最大重传次数，于是再等待一段时间（时间为上一次超时时间的 2 倍），如果还是没能收到服务端的第二次挥手（ACK报文），那么客户端就会断开连接。

## 第二次挥手丢失了，会发生什么？

### 面试版

ACK 报文是不会重传的，第二次挥手丢失后，主动方会触发超时重传机制重传第一次挥手FIN报文（等同于第一次挥手丢失）。

至少需要阅读详解版 Q&A

### 详解版

当服务端收到客户端的第一次挥手后，就会先回一个 ACK 确认报文，此时服务端的连接进入到 `CLOSE_WAIT` 状态。

在前面我们也提了，**ACK 报文是不会重传的**，所以如果服务端的第二次挥手丢失了，**客户端就会触发超时重传机制**，重传 FIN 报文，直到收到服务端的第二次挥手，或者达到最大的重传次数。

举个例子，假设 tcp_orphan_retries 参数值为 2，当第二次挥手一直丢失时，发生的过程如下图：

{{% details title="展开图片" closed="true" %}}
![第二次挥手丢失](/images/cs/network/第二次挥手丢失.png)
{{% /details %}}
具体过程：

- 当客户端超时重传 2 次 FIN 报文后，由于 tcp_orphan_retries 为 2，已达到最大重传次数，于是再等待一段时间（时间为上一次超时时间的 2 倍），如果还是没能收到服务端的第二次挥手（ACK 报文），那么客户端就会断开连接。

这里提一下，当客户端收到第二次挥手，也就是收到服务端发送的 ACK 报文后，客户端就会处于 `FIN_WAIT2` 状态，在这个状态需要等服务端发送第三次挥手，也就是服务端的 FIN 报文。

对于 close 函数关闭的连接，不会再发送新的数据，只能接收数据，所以`FIN_WAIT2` 状态不可以持续太久，而 `tcp_fin_timeout` 控制了这个状态下连接的持续时长，默认值是 60 秒。

> Q：关于 close 函数关闭的数据接收情况
> 
> A：当一端调用close()主动关闭连接时,实际上分两个阶段:
>
> 1. FIN_WAIT_1和FIN_WAIT_2状态
>
> 在这两个状态中,主动关闭端虽然已经不能再发送新的数据,但是kernel的**TCP接收缓冲区**仍然是可以接收来自对端的数据的。
>
> kernel会继续接收并交付所有按序到达的数据给应用进程,直到对端也发送FIN结束数据传输为止。
>
> 1. TIME_WAIT状态
>
> 一旦进入TIME_WAIT状态,表示双方数据传输都已结束,这时kernel的TCP接收缓冲区也不再接收任何新的数据包。
>
> Q："对于 close 函数关闭的连接"这句话意味着关闭方式有多种了，那么有哪几种呢？
> 
> A：
>
> + **标准的close调用**：这是最常见的关闭TCP连接的方式。当一个应用程序完成了数据的发送和接收后，它会调用`close`函数来关闭socket。这会导致操作系统发送一个FIN报文给对端，开始四次挥手过程。
> + **shutdown函数**：除了`close`调用外，`shutdown`函数也可以用来关闭一个TCP连接的一部分（发送或接收）。`shutdown`允许更精细的控制，例如，可以只关闭连接的发送部分，这会导致发送FIN报文，但仍然可以接收数据直到对端也关闭连接。
> + **设置SO_LINGER选项**：在某些情况下，应用程序可能需要在关闭连接时有更多的控制。通过设置socket选项`SO_LINGER`，可以指定在关闭连接时等待数据发送和确认的时间。这种方式可以用来确保所有待发送的数据都被发送到网络中或者确保所有已发送的数据都被确认，之后才真正关闭连接。
> + **意外关闭**：如果一个应用程序崩溃或被强制终止，操作系统会自动关闭该应用程序打开的所有socket，这通常会导致TCP连接被意外关闭。在这种情况下，操作系统可能会直接发送RST报文来重置连接，而不是正常的四次挥手过程。
>
> Q：那假如在使用 close 函数关闭， tcp_fin_timeout 时间之后  TCP 被动关闭端有源源不断的数据发过来什么办？
> 
> A：
>
> 1. 主动关闭端不会立即强制关闭连接，而是可能会重传FIN包，尝试再次通知被动关闭端准备关闭连接。
> 2. 对于继续接收到的在窗口内的数据包，主动关闭端将继续发送ACK确认。
> 3. 如果收到的数据包序列号超出了接收窗口，主动关闭端会发送重复的ACK，确认最后一个正确接收的序列号。
> 4. 在一定时间后，如果被动关闭端仍未发送FIN包来回应主动关闭端的FIN包，并且持续发送新数据，主动关闭端可能会发送RST包来强制关闭连接。发送RST通常是在尝试了一系列重传FIN包和发送ACK确认后，作为最后手段执行的。
>
> **不同的操作系统的细微之处可能有所不同**，有的可能直接发送 RST 包，有的可能会尝试重传FIN包，但是核心还是**最终会选择发送RST包强制终止那一端的连接**。

这意味着对于调用 close 关闭的连接，如果在 60 秒后还没有收到 FIN 报文，客户端（主动关闭方）的连接就会直接关闭，如下图：

{{% details title="展开图片" closed="true" %}}
![img](/images/cs/network/fin_wait_2.drawio.png)
{{% /details %}}
但是注意，如果主动关闭方使用 shutdown 函数关闭连接，指定了只关闭发送方向，而接收方向并没有关闭，那么意味着主动关闭方还是可以接收数据的。

> Q：shutdown 和 close 的区别
> 
> A：shutdown和close都可以用于关闭TCP连接,但是两者有一些重要区别:
>
> 1. shutdown是部分关闭，close是完全关闭。 shutdown可以选择只关闭读或写或同时关闭,相当于半关闭连接。 而close则是完全关闭整个连接的读写数据通道。
> 2. shutdown在关闭写通道后,还能接收数据。 如果使用shutdown(sockfd, SHUT_WR);关闭写通道,将无法继续发送数据,但还可以接收对方发来的数据。 而close在任何一端发起关闭后,双方很快就无法继续通信了。
> 3. close的行为更加彻底。 close不仅关闭连接,还释放所有相关内核资源,而shutdown只中断数据通信通道。
> 4. 对端接收方面 ，对端在收到shutdown的通知时,会得到相应的EOF信号,了解这一侧已经关闭了,需要有处理逻辑。 而收到close的通知后,对端连接也会被系统自动关闭。
>
> 因此,使用shutdown可以更好地控制连接的半关闭状态和通信模式,并让对端作出及时响应。 而close则是一招彻底了结当前连接的方式,不需要对端特别处理。 总的来说,如果需要更精细的通信控制,可以选择先shutdown,如果只是想快速终止连接,那直接close更方便。

此时，如果**主动关闭方一直没收到第三次挥手，那么主动关闭方的连接将会一直处于 `FIN_WAIT2` 状态**（**`tcp_fin_timeout` 无法控制 shutdown 关闭的连接**）。如下图：

{{% details title="展开图片" closed="true" %}}
![img](/images/cs/network/fin_wait_2死等.drawio.png)
{{% /details %}}
## 第三次挥手丢失了，会发生什么？

### 面试版

第三次挥手丢失了，被动关闭端就收不到主动端的ACK包，就会触发超时重发机制，超过最大次数之后继续等待一个指数退避策略间隔后直接进入close状态

主动关闭方需要看调用什么函数关闭的，调用close函数关闭，则会等待超过 tcp_fin_timeout断开连接直接进入close状态；调用shutdown函数关闭后，则会死等，一直处于 TIME_WAIT_2。

### 详解版

当服务端（被动关闭方）收到客户端（主动关闭方）的 FIN 报文后，内核会自动回复 ACK，同时连接处于 `CLOSE_WAIT` 状态，顾名思义，它表示等待应用进程调用 close 函数关闭连接。

此时，**内核是没有权利替代进程关闭连接**，必须由**进程主动调用 close 函数**来触发服务端发送 FIN 报文。

服务端处于 CLOSE_WAIT 状态时，调用了 close 函数，内核就会发出 FIN 报文，同时连接进入 LAST_ACK 状态，等待客户端返回 ACK 来确认连接关闭。

如果迟迟收不到这个 ACK，服务端就会重发 FIN 报文，重发次数仍然由 `tcp_orphan_retrie`s 参数控制，这与客户端重发 FIN 报文的重传次数控制方式是一样的。

举个例子，假设 `tcp_orphan_retrie`s = 3，当第三次挥手一直丢失时，发生的过程如下图：

{{% details title="展开图片" closed="true" %}}
![img](/images/cs/network/第三次挥手丢失.drawio.png)
{{% /details %}}
具体过程：

- 当服务端重传第三次挥手报文的次数达到了 3 次后，由于 tcp_orphan_retries 为 3，达到了重传最大次数，于是再等待一段时间（时间为上一次超时时间的 2 倍），如果还是没能收到客户端的第四次挥手（ACK报文），那么服务端就会断开连接。
- 客户端因为是通过 close 函数关闭连接的，处于 FIN_WAIT_2 状态是有时长限制的，如果 tcp_fin_timeout 时间内还是没能收到服务端的第三次挥手（FIN 报文），那么客户端就会断开连接。

## 第四次挥手丢失了，会发生什么？

### 面试版

被动端等不到 第三次挥手的ACK 报文（也就是第四次挥手），被动端会超时重传（指数退避策略），达到最大次数后等待一个指数退避间隔后直接进入close状态。

客户端在收到第三次挥手后，就会进入 TIME_WAIT 状态，开启时长为 2MSL 的定时器，如果途中再次收到第三次挥手（FIN 报文）后，就会**重置定时器**，当等待 2MSL 时长后，客户端就会断开连接正常进入close状态。

### 详解版

当客户端收到服务端的第三次挥手的 FIN 报文后，就会回 ACK 报文，也就是第四次挥手，此时客户端连接进入 `TIME_WAIT` 状态。

在 Linux 系统，TIME_WAIT 状态会持续 2MSL 后才会进入关闭状态。

然后，服务端（被动关闭方）没有收到 ACK 报文前，还是处于 LAST_ACK 状态。

如果第四次挥手的 ACK 报文没有到达服务端，服务端就会重发 FIN 报文，重发次数仍然由前面介绍过的 `tcp_orphan_retries` 参数控制。

举个例子，假设 tcp_orphan_retries 为 2，当第四次挥手一直丢失时，发生的过程如下：

{{% details title="展开图片" closed="true" %}}
![img](/images/cs/network/第四次挥手丢失drawio.drawio.png)
{{% /details %}}
具体过程：

- 当服务端重传第三次挥手报文达到 2 时，由于 tcp_orphan_retries 为 2， 达到了最大重传次数，于是再等待一段时间（时间为上一次超时时间的 2 倍），如果还是没能收到客户端的第四次挥手（ACK 报文），那么服务端就会断开连接。
- 客户端在收到第三次挥手后，就会进入 TIME_WAIT 状态，开启时长为 2MSL 的定时器，如果途中再次收到第三次挥手（FIN 报文）后，就会**重置定时器**，当等待 2MSL 时长后，客户端就会断开连接。