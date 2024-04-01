+++
title = 'WebSocket'
date = 2024-04-01T09:37:54+08:00
+++

看起来**服务器主动发送消息给客户端**的场景是怎么做到的？
## 使用 HTTP 不断轮询
其实问题的痛点在于，**怎么样才能在用户不做任何操作的情况下，网页能收到消息并发生变更**。

最常见的解决方案是，**网页的前端代码里不断定时发 HTTP 请求到服务器，服务器收到请求后给客户端响应消息**。

这其实时一种「**伪**」服务器推的形式。它其实并不是服务器主动发消息到客户端，而是客户端自己不断偷偷请求服务器，只是用户无感知而已。

用这种方式的场景也有很多，最常见的就是扫码登录。

比如，某信公众号平台，登录页面二维码出现之后，前端网页根本不知道用户扫没扫，于是不断去向后端服务器询问，看有没有人扫过这个码。而且是以大概 1 到 2 秒的间隔去不断发出请求，这样可以保证用户在扫码后能在 1 到 2 秒内得到及时的反馈，不至于等太久。

使用HTTP定时轮询会有两个比较明显的问题：
+ 当你打开 F12 页面时，你会发现满屏的 HTTP 请求。虽然很小，但这其实也消耗带宽，同时也会增加下游服务器的负担。
+ 最坏情况下，用户在扫码后，需要等个 1~2 秒，正好才触发下一次 HTTP 请求，然后才跳转页面，用户会感到明显的卡顿。

## 长轮询
我们知道，HTTP 请求发出后，一般会给服务器留一定的时间做响应，比如 3 秒，规定时间内没返回，就认为是超时。

如果我们的 HTTP 请求将**超时设置的很大**，比如 30 秒，**在这 30 秒内只要服务器收到了扫码请求，就立马返回给客户端网页。如果超时，那就立马发起下一次请求**。

这样就减少了 HTTP 请求的个数，并且由于大部分情况下，用户都会在某个 30 秒的区间内做扫码操作，所以响应也是及时的。

{{% details title="展开图片" closed="true" %}}
![img.png](/images/cs/network/websocket-1.png)
{{% /details %}}

比如，某度云网盘就是这么干的。所以你会发现一扫码，手机上点个确认，电脑端网页就秒跳转，体验很好。

**像这种发起一个请求，在较长时间内等待服务器响应的机制，就是所谓的长轮询机制**。我们常用的消息队列 RocketMQ 中，消费者去取数据时，也用到了这种方式。
{{% details title="展开图片" closed="true" %}}
![img.png](/images/cs/network/websocket-2.png)
{{% /details %}}

像这种，在用户不感知的情况下，服务器将数据推送给浏览器的技术，就是所谓的**服务器推送**技术，它还有个毫不沾边的英文名，comet 技术，大家听过就好。

上面提到的两种解决方案（不断轮询和长轮询），**本质上，其实还是客户端主动去取数据**。

对于像扫码登录这样的**简单场景**还能用用。但如果是网页游戏呢，游戏一般会有大量的数据需要从服务器主动推送到客户端。

## WebSocket是什么
我们知道 TCP 连接的两端，**同一时间里**，**双方**都可以**主动**向对方发送数据。这就是所谓的**全双工**。

而现在使用最广泛的 **HTTP/1.1**，也是基于TCP协议的，**同一时间里，客户端和服务器只能有一方主动发数据，这就是所谓的半双工**。

也就是说，好好的全双工 TCP，被 HTTP/1.1 用成了半双工。为什么？

这是由于 HTTP 协议设计之初，考虑的是看看网页文本的场景，能做到客户端发起请求再由服务器响应，就够了，**根本就没考虑网页游戏这种**，客户端和服务器之间都要互相主动发大量数据的场景。

所以，为了更好的支持这样的场景，我们需要另外**一个基于TCP的新协议**。

于是新的应用层协议WebSocket就被设计出来了。

大家别被这个名字给带偏了。虽然名字带了个socket，但其实 socket 和 WebSocket 之间，就跟雷峰和雷峰塔一样，**二者接近毫无关系**。
{{% details title="展开图片" closed="true" %}}
![img.png](/images/cs/network/websocket-3.png)
{{% /details %}}

WebSocket 是一种在单个TCP连接上进行全双工通信的协议。WebSocket 使得客户端和服务器之间的数据交换变得更加简单，允许服务端主动向客户端推送数据。

## 怎么建立WebSocket连接
我们平时刷网页，一般都是在浏览器上刷的，一会刷刷图文，这时候用的是 HTTP 协议，一会打开网页游戏，这时候就得切换成我们新介绍的 WebSocket 协议。

为了兼容这些使用场景。浏览器在 **TCP三次握手**建立连接之后，都**统一使用 HTTP 协议先进行一次通信**。

+ 如果此时是普通的 HTTP 请求，那后续双方就还是老样子继续用普通 HTTP 协议进行交互，这点没啥疑问。
+ 如果这时候是想建立 WebSocket 连接，就会在 HTTP 请求里带上一些**特殊的header头**，如下：
```text
Connection: Upgrade
Upgrade: WebSocket
Sec-WebSocket-Key: T2a6wZlAwhgQNqruZ2YUyg==\r\n
```

这些 header 头的意思是，浏览器想**升级协议**（Connection: Upgrade），并且想升级成 WebSocket 协议（Upgrade: WebSocket）。同时带上一段**随机生成的 base64 码**（Sec-WebSocket-Key），发给服务器。

如果服务器正好支持升级成 WebSocket 协议。就会走 WebSocket 握手流程，同时根据客户端生成的 base64 码，用某个**公开的算法**变成另一段字符串，放在 HTTP 响应的 Sec-WebSocket-Accept 头里，同时带上**101状态码**，发回给浏览器。HTTP 的响应如下：
```text
HTTP/1.1 101 Switching Protocols\r\n
Sec-WebSocket-Accept: iBJKv/ALIW2DobfoA4dmr3JHBCY=\r\n
Upgrade: WebSocket\r\n
Connection: Upgrade\r\n
```
HTTP 状态码=200（正常响应）的情况，大家见得多了。101 确实不常见，它其实是指**协议切换**。
{{% details title="展开图片" closed="true" %}}
![img.png](/images/cs/network/websocket-4.png)
{{% /details %}}
之后，浏览器也用同样的公开算法将base64码转成另一段字符串，如果这段字符串跟服务器传回来的**字符串一致**，那验证通过。
{{% details title="展开图片" closed="true" %}}
![img.png](/images/cs/network/websocket-5.png)
{{% /details %}}
就这样经历了一来一回**两次 HTTP 握手**，WebSocket就建立完成了，后续双方就可以使用 webscoket 的数据格式进行通信了。
{{% details title="展开图片" closed="true" %}}
![img.png](/images/cs/network/websocket-6.png)
{{% /details %}}


## WebSocket抓包
我们可以用wireshark抓个包，实际看下数据包的情况。
{{% details title="展开图片" closed="true" %}}
![img.png](/images/cs/network/websocket-7.png)
{{% /details %}}
上面这张图，注意画了红框的第2445行报文，是**WebSocket的第一次握手**，意思是发起了一次带有特殊Header的HTTP请求。
{{% details title="展开图片" closed="true" %}}
![img.png](/images/cs/network/websocket-8.png)
{{% /details %}}
上面这个图里画了红框的4714行报文，就是服务器在得到第一次握手后，响应的**第二次握手**，可以看到这也是个 HTTP 类型的报文，返回的状态码是 101。同时可以看到返回的报文 header 中也带有各种WebSocket相关的信息，比如Sec-WebSocket-Accept。
{{% details title="展开图片" closed="true" %}}
![img.png](/images/cs/network/websocket-9.png)
{{% /details %}}
上面这张图就是全貌了，从截图上的注释可以看出，WebSocket和HTTP一样都是基于TCP的协议。**经历了三次TCP握手之后，利用 HTTP 协议升级为 WebSocket 协议**。

你在网上可能会看到一种说法："WebSocket 是基于HTTP的新协议"，其实这并不对，因为WebSocket只有在建立连接时才用到了HTTP，升级完成之后就跟HTTP没有任何关系了。

## WebSocket的消息格式
数据包在WebSocket中被叫做**帧**，我们来看下它的数据格式长什么样子。
{{% details title="展开图片" closed="true" %}}
![img.png](/images/cs/network/websocket-10.png)
{{% /details %}}
这里面字段很多，但我们只需要关注下面这几个。

**opcode**字段：这个是用来标志这是个**什么类型的数据帧**。比如。

+ 等于 1 ，是指text类型（string）的数据包 
+ 等于 2 ，是二进制数据类型（[]byte）的数据包 
+ 等于 8 ，是关闭连接的信号

**payload**字段：存放的是我们**真正想要传输的数据的长度**，单位是**字节**。比如你要发送的数据是字符串"111"，那它的长度就是3。
{{% details title="展开图片" closed="true" %}}
![img.png](/images/cs/network/websocket-11.png)
{{% /details %}}
另外，可以看到，我们存放**payload 长度的字段有好几个**，我们既可以用最前面的7bit, 也可以用后面的7+16bit 或 7+64bit。

那么问题就来了。

我们知道，在数据层面，大家都是 01 二进制流。我怎么知道**什么情况下应该读 7 bit，什么情况下应该读7+16bit呢**？

WebSocket会用最开始的7bit做标志位。不管接下来的数据有多大，都**先读最先的7个bit**，根据它的取值决定还要不要再读个 16bit 或 64bit。
+ 如果最开始的7bit的值是 0~125，那么它就表示了 payload 全部长度，只读最开始的7个bit就完事了。
+ 如果是126（0x7E）。那它表示payload的长度范围在 126~65535 之间，接下来还需要**再读16bit**。这16bit会包含payload的真实长度。
+ 如果是127（0x7F）。那它表示payload的长度范围>=65536，接下来还需要**再读64bit**。这64bit会包含payload的长度。这能放2的64次方byte的数据，换算一下好多个TB，肯定够用了。

**payload data**字段：这里存放的就是**真正要传输的数据**，在知道了上面的payload长度后，就可以根据这个值去**截取对应的数据**。

WebSocket的数据格式也是数据头（内含payload长度） + payload data 的形式。

这是因为 TCP 协议本身就是全双工，但直接使用纯裸TCP去传输数据，会有粘包的"问题"。为了解决这个问题，上层协议一般会用**消息头+消息体的格式去重新包装要发的数据**。

而**消息头里一般含有消息体的长度**，通过这个长度可以去截取真正的消息体。

HTTP 协议和大部分 RPC 协议，以及我们今天介绍的WebSocket协议，都是这样设计的。
{{% details title="展开图片" closed="true" %}}
![img.png](/images/cs/network/websocket-12.png)
{{% /details %}}

## WebSocket 如何限流
实施限流时，应考虑以下因素：
+ 限流粒度：可以在不同的粒度上实施限流，例如全局、按用户、按IP等。 
+ 应对策略：当达到限流条件时，确定是拒绝请求、延迟处理还是其他应对方式。 
+ 监控和调整：监控限流效果并根据实际情况调整策略，以达到最佳效果。

服务端限流：
+ 消息大小限制：初始化 WebSocket 时，可以通过设置 `ReadBufferSize` 和 `WriteBufferSize` 来限制接收和发送消息的大小。
+ 固定窗口计数器、滑动日志窗口、令牌桶、漏桶算法

中间件或代理限流：
+ Nginx：通过配置限流模块，可以控制连接数和请求频率。 
+ HAProxy：可以配置来限制并发连接数和每秒请求数。

客户端限流：
+ 限制请求频率：在客户端实现延时或等待逻辑，确保请求发送频率不会过高。
+ 智能重试机制：对失败的请求进行重试，但是要有退避策略，避免在短时间内大量重试导致服务器压力。

## WebSocket的使用场景
WebSocket完美继承了 TCP 协议的**全双工**能力，并且还贴心的提供了解决粘包的方案。

它适用于需要**服务器和客户端（浏览器）频繁交互**的大部分场景，比如网页/小程序游戏，网页聊天室，以及一些类似飞书这样的网页协同办公软件。

在使用 WebSocket 协议的网页游戏里，怪物移动以及攻击玩家的行为是服务器逻辑产生的，对玩家产生的伤害等数据，都需要由服务器主动发送给客户端，客户端获得数据后展示对应的效果。







