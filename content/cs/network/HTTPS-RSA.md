+++
title = 'HTTPS RSA 握手'
date = 2024-04-02T17:19:20+08:00
+++

传统的 TLS 握手基本都是使用 RSA 算法来实现密钥交换的，在将 **TLS 证书**部署服务端时，证书文件其实就是**服务端的公钥**，会在 TLS 握手阶段传递给客户端，而服务端的私钥则一直留在服务端，一定要确保私钥不能被窃取。

在 RSA 密钥协商算法中，**客户端会生成随机密钥，并使用服务端的公钥加密后再传给服务端**。根据非对称加密算法，公钥加密的消息仅能通过私钥解密，这样服务端解密后，双方就得到了相同的密钥，再用它加密应用消息。

我用 Wireshark 工具抓了用 RSA 密钥交换的 TLS 握手过程，你可以从下面看到，一共经历了四次握手：
{{% details title="展开图片" closed="true" %}}
![img.png](/images/cs/network/HTTPS-RSA-1.png)
{{% /details %}}
对应 Wireshark 的抓包，我也画了一幅图，你可以从下图很清晰地看到该过程：
{{% details title="展开图片" closed="true" %}}
![img.png](/images/cs/network/HTTPS-RSA-2.png)
{{% /details %}}

## TLS 第一次握手
客户端首先会发一个「Client Hello」消息，字面意思我们也能理解到，这是跟服务器「打招呼」。
{{% details title="展开图片" closed="true" %}}
![img.png](/images/cs/network/HTTPS-RSA-3.png)
{{% /details %}}
消息里面有客户端使用的 **TLS 版本号、支持的密码套件列表，以及生成的随机数（Client Random）**，这个随机数会被服务端保留，它是生成对称加密密钥的材料之一。

## TLS 第二次握手
当服务端收到客户端的「Client Hello」消息后，会确认 TLS 版本号是否支持，和从密码套件列表中选择一个密码套件，以及生成随机数（Server Random）。

接着，返回「**Server Hello**」消息，消息里面有**服务器确认的 TLS 版本号，也给出了随机数（Server Random），然后从客户端的密码套件列表选择了一个合适的密码套件**。
{{% details title="展开图片" closed="true" %}}
![img.png](/images/cs/network/HTTPS-RSA-4.png)
{{% /details %}}

这个密码套件看起来真让人头晕，好一大串，但是其实它是有固定格式和规范的。基本的形式是「**密钥交换算法 + 签名算法 + 对称加密算法 + 摘要算法**」， 一般 WITH 单词前面有两个单词，第一个单词是约定密钥交换的算法，第二个单词是约定证书的验证算法。比如刚才的密码套件的意思就是：
+ 由于 WITH 单词只有一个 RSA，则说明握手时密钥交换算法和签名算法都是使用 RSA；
+ 握手后的通信使用 AES 对称算法，密钥长度 128 位，分组模式是 GCM；
+ 摘要算法 SHA256 用于消息认证和产生随机数；

然后，服务端为了证明自己的身份，会发送「**Server Certificate**」给客户端，这个消息里含有**数字证书**。
{{% details title="展开图片" closed="true" %}}
![img.png](/images/cs/network/HTTPS-RSA-5.png)
{{% /details %}}

随后，服务端发了「**Server Hello Done**」消息，目的是告诉客户端，我已经把该给你的东西都给你了，本次打招呼完毕。
{{% details title="展开图片" closed="true" %}}
![img.png](/images/cs/network/HTTPS-RSA-6.png)
{{% /details %}}

## 客户端验证证书
详见 PKI 体系

## TLS 第三次握手
客户端验证完证书后，认为可信则继续往下走。

接着，客户端就会生成一个**新的随机数** (pre-master)，用**服务器的 RSA 公钥加密**该随机数，通过「**Client Key Exchange**」消息传给服务端。
{{% details title="展开图片" closed="true" %}}
![img.png](/images/cs/network/HTTPS-RSA-7.png)
{{% /details %}}

服务端收到后，用 RSA 私钥解密，得到客户端发来的随机数 (pre-master)。

至此，**客户端和服务端双方都共享了三个随机数，分别是 Client Random、Server Random、pre-master。**

于是，双方根据已经得到的**三个随机数，生成会话密钥（Master Secret）**，它是对称密钥，用于对后续的 HTTP 请求/响应的数据加解密。

生成完「会话密钥」后，然后客户端发一个「**Change Cipher Spec**」，告诉**服务端开始使用加密方式发送消息**。
{{% details title="展开图片" closed="true" %}}
![img.png](/images/cs/network/HTTPS-RSA-8.png)
{{% /details %}}
然后，客户端再发一个「**Encrypted Handshake Message（Finished）**」消息，把**之前所有发送的数据做个摘要**，再用**会话密钥（master secret）加密一下**，让服务器做个验证，验证加密通信「是否可用」和「之前握手信息是否有被中途篡改过」。
{{% details title="展开图片" closed="true" %}}
![img.png](/images/cs/network/HTTPS-RSA-9.png)
{{% /details %}}

> Q: 发送的所有数据，是从 client hello 开始的吗？应该是包括客户端和服务端的吧？
> 
> A: 在TLS握手过程中，无论是客户端还是服务端发送的“Encrypted Handshake Message”（通常是“Finished”消息），它们都包含了整个握手过程中双方交换的全部消息的摘要。这意味着每一方计算摘要时，都会考虑自己和对方发送的所有消息，确保整个握手过程的完整性和安全性。
> 
> 这个机制确保了双方对整个握手过程有完全一致的理解和确认。每一方都验证了对方看到的握手消息与自己发送和接收的消息是一致的，这是TLS握手协议的一个关键安全特性。通过这样的设计，TLS能够抵御某些类型的攻击，比如中间人攻击，同时确保通信双方可以安全地协商出加密通信的密钥。

可以发现，**「Change Cipher Spec」及之前传输的 TLS 握手数据都是明文**，之后都是对称密钥加密的密文。

## TLS 第四次握手
服务器也是同样的操作，发「**Change Cipher Spec**」和「**Encrypted Handshake Message**」消息，如果双方都验证加密和解密没问题，那么握手正式完成。

Change Cipher Spec 同样是明文。
>Q: 为什么第四次握手还需要 Encrypted Handshake Message ，第三次握手不是已经将前面数据全摘要了吗？服务端验证没问题的话，为什么还要摘要加密一次？
> 
>A: 服务端验证了摘要，只是证明了客户端能够解密服务端的消息和使用正确的密钥加密消息，服务端发送的加密“Finished”消息让客户端也有机会证明服务端具有相同的能力。服务端不发送这个消息让客户端确认服务端同样可以解密和正确加密的话，客户端发送数据很可能是没有意义的。

最后，就用「会话密钥」加解密 HTTP 请求和响应了。

## RSA 算法的缺陷
**使用 RSA 密钥协商算法的最大问题是不支持前向保密**。

因为客户端传递随机数（用于生成对称加密密钥的条件之一）给服务端时使用的是公钥加密的，服务端收到后，会用私钥解密得到随机数。所以**一旦服务端的私钥泄漏了，过去被第三方截获的所有 TLS 通讯密文都会被破解**。
> 前两个随机数都是明文的，如果服务器私钥泄露了，第三个随机数也就变成了“明文”了，那么对称加密密钥也就是“明文”了，所有的通信数据都可以被解密了。





