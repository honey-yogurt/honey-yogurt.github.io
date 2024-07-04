+++
title = 'PoS'
date = 2024-03-21T15:22:57+08:00
+++
## 基本介绍
PoW要求所有的节点消耗大量的算力来争夺记账权，但在每一轮共识中，只有一个节点的工作量有效，意味着有大量的资源被浪费。因此，权益证明机制Proof of Stake（PoS）在2013年被提出并首次应用在PeerCoin系统中，目的是解决资源浪费的问题。

在探讨 POS 之前，我们可以先把它与我们熟悉的投票进行类比，从宏观的角度理解其运行原理：

![img.png](/images/blockchain/consensus/pos-1.png)

在这整个validator pool中：
1. **提议**：所有的validator都会参与投票，有一个validator比较特别，他被选举出来对区块进行提议（提出共识内容），也因此被称为leader，例如在交易池中选择了100条交易后，打包成一个区块，交由其他validators验证及投票；
2. **投票**：其他validator对于上述区块中的所有交易验证完签名且无误后（对内容达成共识），正常情况下投“赞成”票，反之则是“反对”；
3. **出结果**：当投票完成后，得到多数“赞成”的区块成功上链，重新启动新的轮次。


而PoS与之的关联在于，必须**通过质押资产（staking）来成为validator！**（例如在以太坊中需要质押32个eth）

以太坊的PoS实现主要通过其“以太坊2.0”升级引入，特别是通过引入了名为“Beacon Chain”的新链来实现。以下是以太坊PoS共识机制的具体实现流程，以及它是如何工作的详细说明：

**初始化和加入**
+ 质押（Staking）：想要成为验证者（validator）的参与者必须在Beacon Chain上质押32个ETH。通过这种方式，验证者承诺遵守网络规则。 
+ 验证者注册：一旦质押完成，该参与者的信息将被注册为一个潜在的验证者，并且会在之后被随机选择来提议或者验证区块。

**共识机制的运作**
+ 周期性的权益证明：Beacon Chain通过一系列时代（epoch）来组织时间，每个时代大约持续6.4分钟，包含了32个槽（slot），每个槽约持续12秒。每个槽有一个被选定的验证者负责提议一个新区块，而其他验证者则负责对这个区块进行验证和投票。 
+ 区块提议：在每个槽位，被选定的验证者将收集交易，构造一个新的区块，并将其提议给网络。 
+ 委员会选择和投票：每个槽位还会随机选择一个委员会，由一组验证者组成，这些验证者的任务是对提议的区块进行验证并投票。如果区块收到足够多的票数（超过委员会成员的2/3），该区块则被认为是有效的并被添加到区块链上。 

**奖励和惩罚**
+ 验证者通过正确地提议和验证区块来获得奖励，包括新创建的ETH和网络交易费用。相反，如果验证者的行为不符合规则（如提议或验证无效的区块），他们会受到惩罚，即失去一部分质押的ETH。 



## 抽签算法：VRF(Verifiable Random Function)
事实上在POS的整个过程中，首先要确定的是选民的范围，确定选民后才进入到提议、投票等步骤。

POS中的选民分为两类：leader&committee。

![img.png](/images/blockchain/consensus/pos-1.png)

选民的选择对于整个系统的重要性来说尤其重要，可以想象如果选择算法不够充分随机，恶意节点通过贿赂等手段不断拉拢、扩大恶意节点范围，或者预先计算出出块节点进行针对性攻击，整个系统的安全性就非常岌岌可危了。

因此引入一种安全的随机算法至关重要：在整个validator pool足够大的基础上，通过足够随机的抽签算法，任何人都无法提前知道谁将负责下一次的出块，以此进一步提升系统的安全性。

这里我们将要了解的算法叫做VRF，承担了上述随机抽签的职责。

VRF的大体思路如下，每轮区块出块前：
1. 每个节点依据相同的规则计算出一个随机数R；
2. 全网有一个统一的阈值D，所有R < D的节点视为当选；
3. 同时，节点还需要提供一个证明P，来让他人验证R确实是按照既定规则出的。

> + 要按照一个既定的规则，生成一个随机数，我们熟悉的哈希函数就能起到这个效果；
> + 哈希函数的参数选择什么？每个节点的私钥地址SK是个好的选择，每个节点均拥有私钥但又各不相同，最终生成的R就具备了不确定性；
> + 但是单一的SK并不足够，因为Hash（SK）的结果是确定的，抽签只能抽一次。因此哈希函数的参数还需一个足够实效性的数据，使得所有节点只有在抽签时才能真正知道参数是什么，最新的区块数据就是一个选择（当然可以有各类变体，下文暂以最新区块数据为例）。

在此基础上我们可以来看VRF重要的4个函数了：

![img.png](/images/blockchain/consensus/pos-2.png)

- **生成部分**
  1. **R = VRF_Hash（SK，M）**，每个节点基于自己的私钥信息以及最新的区块数据，通过VRF_Hash函数计算出随机数R 
  2. **P = VRF_Proof（SK，M）**，同样的参数也用来生成证明P

- **验证部分**，如果某节点当选，全网可通过下列的两个证明函数来验证R的生成是否符合规范

  3. **R = ？VRF_P2H（P）**
  4. **VRF_Verify(PK,M,P) =? True**

至此一次可验证的抽签得以实现，这也是VRF命名的由来。

VRF 的简要版实现
+ **生成部分**
  + P = VRF_Proof（SK，M）= Signature（SK，Hash（M））
  + R = VRF_Hash（SK，M）= Hash（P）
+ **验证部分**，如果某节点当选，全网拥有的可供验证的数据为R、P、PK、M
  + R = ？VRF_P2H（P）———R  =？Hash（P）
  + VRF_Verify(PK,M,P) =? True  ——— 用PK将P解出 =？ Hash（M）


## Finalization：Gasper的Accountable Safety
在POS投票这一大语境下，我们有了**抽取选民、提议区块**及**投票**等工具。

我们通过下面的场景考虑接下来的问题，如果有人试图从某些“上古”的区块进行分叉，会发生什么：

![img.png](/images/blockchain/consensus/pos-3.png)

**PoW**:

情况一：其算力远低于全网的51%

攻击极大概率无法实现。在攻击者算力远低于主网其他算力的情况下，主要算力依旧会依照最长链原则进行挖矿，攻击者落后且（极大概率）挖的慢，故很难实现。

情况二：其算力达到甚至超过全网的51%

可以实现但有很大经济代价。攻击者落后但挖的快，有望赶上。但计算nonce的巨大算力是其要付出的代价。

**PoS**:

情况一：其staking远低于全网的1/3

攻击无法实现。正常validator的票数超过2/3，链依旧会按正常情况进行增长。

情况二：其staking达到甚至超过全网的2/3

攻击似乎即可以实现，也没有挖矿的算力代价？

这里我们研究的本质是finalization问题，如果攻击者可以轻易地从某些远古时期进行分叉，那么原先记录在链中的大量交易全都失效，其代价可想而知。我们希望在PoS中，对于被finalized的区块同样能形成一种保护，要推翻他们需要付出巨大的代价（如PoW中的算力代价），以此成为系统安全的保障。

我们将借助以太坊的Gasper来看这个问题，我们将看到Gasper是如何通过一套机制，确保分叉链与主链上的区块无法同时被finalized，除非付出全网 1/3 staking总量的代价。（为便于理解，我们会将模型进行一定的简化）

首先是justified、finalized等重要设定：

**投票是可以“跨”区块的**

![img.png](/images/blockchain/consensus/pos-4.png)

所有的 validator 对区块进行投票时候，没有规定必须一个区块一个区块进行投票，完全可以对某个区间的区块（B0-B2）都认同

**得到2/3投票的区块被justified**

![img.png](/images/blockchain/consensus/pos-5.png)


创世区块天然被justified。

finalized的情况（direct child的要求初看下会显得突兀，后续结合slash conditions就会清楚）。

![img.png](/images/blockchain/consensus/pos-6.png)


**本身被 justified , 它的direct child 也被 justified，那么它本身就被finalized。**

其次是Gasper的slashing conditions，违反相关规定的validator将被罚没所有staking资产：

![img.png](/images/blockchain/consensus/pos-7.png)

![img.png](/images/blockchain/consensus/pos-8.png)


再次回顾我们要证明的最终问题，一旦有被finalized的区块产生，后续区块想能被 finalized仅能顺着该区块所在的链。换言之，处于不同branches上的两个区块am、bn不可能被同时finalized，除非付出全网1/3staking总量的代价。

我们来看证明：

情况一：假设 h(am) = h(bn)

![img.png](/images/blockchain/consensus/pos-9.png)


由于am与bn至少各收到了2/3的赞成票，两者重叠部分面积至少为1/3总质押量（红色区域），这部分deposit将因为违背了condition I而被罚没。（若橙框向右移动或者蓝框向左移，重叠部分面积将扩大，意味着更多的质押金将被罚没。）

情况2：假设h(am) ≠ h(bn)，不妨设h(am) < h(bn)

![img.png](/images/blockchain/consensus/pos-10.png)


**由此可知在slashing conditions的保证下，两个conflicting 的区块无法同时被finalized，分叉问题得以解决，这也被称为gasper的重要性质Accountable Safety**。

我们能看到，在上述机制的保证下，要发动我们开篇的攻击，其代价至少为500万个以太坊，系统的安全很大程度便是植根于此。

![img.png](/images/blockchain/consensus/pos-11.png)



