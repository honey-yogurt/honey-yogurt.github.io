+++
title = 'Chainlink'
date = 2024-03-25T22:58:13+08:00
+++

## chainlink 做了什么？

Chainlink 预言机把**中心化预言机节点替换为去中心化网络**，在网络中有很多预言机节点，每个预言机节点都可以通过自己的渠道去获取数据，然后在去中心化网络中对获得的数据进行共识。**这里的共识方式并不是 BFT，POS，POW 这个意义的共识，而是为了获取可靠的数据**，比如说取平均数；或者类似于体育比赛里面，去掉一个最高分，去掉一个最低分，剩下的取平均数或者中位数，现在 Chainlink 采取的是中位数的共识方式。

在chainlink网络中，对同一个数据，不同预言机在现实世界中拿到的结果不一定相同。

![img.png](/images/blockchain/web3/chainlink-1.png)


如何对不同结果的数据进行聚合呢？

**链上聚合**

![img.png](/images/blockchain/web3/chainlink-2.png)

优点：简单，可信度高，活性高

缺点：费用高，还会存在吃空饷问题。
>吃空饷：预言机节点不向数据提供商获取数据，而是在链上查询其他预言机节点提供的数据。


**链下聚合**

![img.png](/images/blockchain/web3/chainlink-3.png)

1. 定义预言机之间的通信协议
2. 使用门限算法在一定程度上解决吃空饷问题

## chainlink 的解决方案
### Chainlink Data Feeds（链上智能合约喂价）
在 Chainlink Data Feeds 中，不同的预言机节点，通过自己的数据提供商获取价格数据，然后通过预言机网络对多个数据聚合后。比如一个 token 价格，A 节点报价是
400， B节点报价是399，C 节点报价是401 ，预言机节点将所有的报价进行共识后，会把价格**中位数**400 传给链上的智能合约，完成此次喂价。

![img.png](/images/blockchain/web3/chainlink-4.png)


Data feeds 的业务流程涉及两个参与方，第一个是数据提供商，它们会使用自己的数据，或通过第三方获取相应的数据，之后输入到 Chainlink 预言机网络中的一个节点。另外一个参与方就是预言机节点，每个节点可以有一个或者多个数据提供商，每个提供商输入的数据都会在预言机网络里中进行共识，然后网络中会随机选取一个节点，由该节点提交数据到链上。

![img.png](/images/blockchain/web3/chainlink-7.png)

Data feeds 在**链上会部署一个叫做 Aggregator 的合约**，内部使用 Mapping **存储节点网络聚合后的中位数价格**。对于用户合约，可以调用 Aggregator 函数，获得相应的价格数据。


![img.png](/images/blockchain/web3/chainlink-6.png)

Chainlink Data Feeds 主要包含了三个合约，第一个是 Consumer 合约 ( **用户合约** )，第二个是 Proxy 合约 ( **代理合约** )，第三个就是刚才提到的 Aggregator 合约 ( **聚合合约** )。Proxy 合约作为接口连接 Consumer 合约和 Aggregator 合约，屏蔽了 Aggregator 的复杂性，对外向用户提供一个统一的接口（latestRounddata），返回预言机网络最近一次的价格数据。

Chainlink Data Feeds 支持多个不同的区块链网络，如BNB，Solana，Polygon等 ，可以在 Data Feeds 页面找到目前支持的网络和价格数据。同时对于每个价格数据，**都有相应的参数控制数据的更新频率**，以保证数据的实时性和有效性。

![img.png](/images/blockchain/web3/chainlink-5.png)

拿 ETH对USD 价格举例，这里有两个参数，第一个参数叫 Deviation threshold ( **波动阈值** )。如果当前聚合价格对比最近一次更新的价格，**波动率超过0.5%**，预言机需要立即更新 ETH 的当前价格。另一个是 heartbeat ( **心跳计时** )，距离上一次价格更新时间间隔超过 1 个小时的话，预言机网络就会进行新一轮的更新。需要说明的是，当 Deviation threshold 触发更新后，Heartbeat 会重置为 1 小时，重新开始计时。

从上面还可以看到提供价格数据的节点信息，包括节点的数量，节点提供的价格数据，以及节点运营商。可以看到很多大型机构参与了 Chainlink 预言机网络，包括 T-Mobile、SNZ、SyncNode 等，后续 Chainlink 还会进一步扩大合作机构，提供更加稳定和安全的去中心化的喂价服务。

## Chainlink VRF（随机数）
在VRF出现之前，主流的生成随机数的方法是根据当前区块中的交易生成 Hash，使用这个 Hash 作为种子生成随机数，由此引出了 MEV 问题。矿工可以选择性打包交易，得到他们想要的随机数，虽然成本比较高，但当利润回报很高时，矿工作恶的可能性也会相应提高很多。
>MEV 是 "Miner Extractable Value"（矿工可提取价值）的缩写，这是一个在加密货币和区块链领域中广泛讨论的概念，尤其是在以太坊网络中。它指的是区块生产者（在以太坊中是矿工，在一些其他区块链中可能是验证者或打包者）通过选择、排列、包括或排除某些交易来从区块中提取价值的能力。 
> 这种价值提取发生在矿工或区块生产者利用他们对交易包含顺序的控制权，从而获得额外的利润。例如，矿工可以通过先后顺序操纵两个密切相关的交易来赚取利润，或者通过预测和利用网络中的特定交易来实现利润最大化。这种行为可能包括但不限于前置交易（front running，即基于未公开信息先行交易）、交易顺序优化以获取最佳的交易费用、以及通过影响随机数生成过程来获利。 
> MEV 的存在在很多情况下对网络的公平性和效率构成挑战。为了解决或减少MEV带来的问题，社区和开发者探索了各种机制，包括引入更公平的交易排序算法、使用隐私技术减少可利用信息、以及开发新的区块链架构设计，例如引入VRF（可验证随机函数）以提供更公平和不可预测的随机数生成方法。

在 Chainlink VRF 中，**随机数是由预言机网络生成**。用户传入一个种子给 VRF 合约，预言机 VRF 节点会使用**节点（预言机节点）私钥和种子**生成一个随机数和 Proof ( 证明 ) 返回给 VRF 合约，VRF 合约 Proof 验证随机数的合法性，如果通过验证，就会把随机数返回给用户。跟单纯链下生成随机数不同的是，Chainlink VRF 生成的随机数可以通过 Proof 证明它是根据特定椭圆曲线算法算出来的，具有可验证性、独特性。

![img.png](/images/blockchain/web3/chainlink-9.png)

![img.png](/images/blockchain/web3/chainlink-8.png)

VRF 本身是一个随机算法的名字，在 1999 年由Micali、Rabin 和 Vadhan 首次提出 VRF这个概念。

相比其他算法，VRF 有三个主要特点：
+ 第一点是可证明性 ( Provability )。它在输出随机数的时候，同时要输出一个 Proof，这个Proof 可以证明这个随机数不是生成者直接生成的，而是**根据输入的种子与自己的私钥去生成，用户可以根据它的公钥和 Proof 去验证**。 
+ 第二个点是独特性 ( Uniqueness )。**每个种子生成的随机数都是唯一的**，一个种子不能生成多个随机数。这样可以保证预言机无法生成多个随机数，从中挑选一个对自身有利的随机数返回给链上智能合约。 
+ 第三个是伪随机性 ( Pseudorandomness )。所有根据算法产生的随机数都是**伪随机数**，没有加入现实世界中的温度、风速等变量作为种子的一部分去生成随机数，因为如果采用这些变量，用户在验证时无法复制这些变量，从而无法进行验证。


VRF算法由三个函数构成：
+ 第一个是公钥私钥对生成函数，用于生成签名用的公私钥对。 
+ 第二个是随机数生成函数。接受用户传入的种子，加上节点自身的私钥，生成一个Randomness 随机数。除了随机数外，还会生成一个 Proof，证明这个随机数是节点根据种子加私钥，然后依据固定算法生成，不是乱生成的。 
+ 第三个是验证函数。节点把随机数和 Proof 返回给链上合约时，链上合约根据节点公钥、种子、生成的随机数、Proof，验证随机数是不是使用随机数生成函数生成的。验证成功返回True，验证失败返回false。

VRF 这三个函数就是使用 Chainlink VRF 的三个步骤。预言机节点生成密钥对时，会把**公钥公布并且保存在链上验证合约中，私钥保存在节点内部**。之后用户发起一笔交易到 **VRF 链上合约去请求随机数，合约在区块中生成 Event Log，并在其中记录用户传入的种子**。预言机节点监听到这个 Event Log 后，生成随机数和 Proof 发回给 VRF 链上合约，合约验证通过则**写入随机数到用户合约**，否则不写入，这就是整个 VRF 的实现过程。


![img.png](/images/blockchain/web3/chainlink-10.png)

![img.png](/images/blockchain/web3/chainlink-11.png)

用户 Consumer 合约不直接跟预言机节点交互，而是通过 Coordinator 合约协助。**用户需要在 Consumer 合约中实现特定的函数，以便接受 Coordinator 合约回传的随机数**。这里需要实现的回调接口为 FulfillRandWords，具体调用流程如下：
+ 用户在 Consumer 合约中调用 Coordinator 合约的 RequestRandomWords 接口请求 VRF 随机数。Coordinator 合约收到请求后，生成 Event Log，并在其中写入 Meta data 等配置信息。链下预言机监控到出现 Event Log 后，提取里面的配置信息，结合当前的区块 Hash 算出一个种子，具体算法为用户 Seed 加区块哈希再生成一个新的哈希作为最终 Seed。 
+ 预言机节点根据种子和自己的私钥计算得到随机数 （ randomness ) 和 Proof ，然后调用 Coordinator 合约的 FullfillRandomWords 接口进行回传。Coordinator 合约对收到的随机数和 Proof 进行验证，验证成功以后通过 FulfillRandomWords 接口把随机数返回给 Consumer 合约，最终用户就收到这个随机数了。 
+ 至于随机数的使用，用户可以在 FulfillRandomWords 接口中实现，也可以编写另外的函数去实现。Coordinator 合约调用 Consumer 的 FulfillRandomWords 接口时有 callBack gas limit 限制，当 FulfillRandomWords 接口中消耗的 gas 超过这个 limit 时，Coordinator 会调用失败，导致无法传递随机数给 Consumer 合约，所以 FulfillRandomWords 接口中避免实现复杂的逻辑，最好只做一个随机数的存储，然后通过另外的接口去使用。


![img.png](/images/blockchain/web3/chainlink-17.png)

+ Chainlink VRF 目前最大的应用场景是 NFT Mint。NFT 需要创建、分发，创建的时候，每一个 NFT 的稀有度不同，分配的用户不同。根据业务需要，有可能还要设置白名单，白名单里面的 holder 空投不同的 NFT。比如无聊猿猴项目，空投 serum 血清给 Holder ，血清的随机空投就是用的 VRF 随机数。 
+ 另一个应用是抽奖。比如一些 IDO 平台，用户购买通证，然后去质押通证，平台会赠送白名单参与抽奖，奖励的大小就可以使用 VRF 进行选取。

## ChainLink Keepers（合约自动化执行）

![img.png](/images/blockchain/web3/chainlink-12.png)

正如智能合约无法主动获取外界数据一样，也无法自己触发自己。在没有自动化工具之前，智能合约都是由**开发者手动触发**的。对于小型项目方，需要自己在服务器上写一段脚本，每次通过私钥去触发合约交易，这样不仅存在曾经提到的单节点风险，并且会占用团队的时间和资源。

除了手动触发以外，还可以通过 **Bounty（赏金） 模式**实现智能合约的自动执行。挖过 YFI 的人都知道，曾经一段时间，YFI 需要用户手工点一下 Claim，才能把收益发到各个用户的钱包里，每次点击的人会发一些额外的奖励，这就是 Bounty 模式。有一个著名的项目叫 ETH Alarm Clock，可以给某个时间点去触发某个交易的人汇一些赏金。它的问题在于只有一个人能成功触发交易，而成功触发的人会获得所有奖励，这会导致 Gas 竞赛。为了成为成功触发的那个人，大家不断增加 Gas Fee ，导致了链的拥堵，就像是抢 NFT 一样。但是实际上，在这个场景中，只要一个人去调用函数就可以了，很多人抢会造成了不必要的浪费。

Chainlink Keepers 是的一个**去中心化合约执行服务**，可以实现链上合约的自动化执行。

开发团队可以注册一个 UpKeep，**每个区块中都会去检测一下监控的的合约状态**，如果符合预设条件，就调用函数，当然也可以不设置预设条件（相当于条件判断结果恒等于True），根据时间来调用特定合约中的特定函数。

Chainlink Keepers 可以在**不需要输入的情况下，不断地根据预设的逻辑去检测智能合约的状态**，如果满足的话就执行，不满足的话，等待下一次检测。

![img.png](/images/blockchain/web3/chainlink-13.png)

![img.png](/images/blockchain/web3/chainlink-14.png)

Chainlink Keepers 首先调用 **UpKeep 合约里面的检测函数**，执行函数中的条件判断，判断条件结果为 True 或者 False，如果是结果为 False 就跳过，在下一个区块重新检查。如果为True，会通过 UpKeep Registry 合约调用 UpKeep 合约的 PerformUpKeep 函数，用户可以在 PerformUpKeep 中定义具体的执行逻辑。

![img.png](/images/blockchain/web3/chainlink-15.png)


Chainlink Keepers 由三个合约组成，分别为 KeepersCompatible、KeepersRegistrar、KeepersRegistry。

KeepersCompatible 是用户合约，为了保证 KeepersRegistry 能成功回调，KeepersCompatible 合约需要实现两个函数，一个是 checkUpKeep，另一个是 performUpKeep。**KeepersCompatible 只有在 KeepersRegistar 中注册成功后，预言机节点才能通过 checkUpKeep 判断是否需要执行 performUpKeep** 。

具体注册、调用流程如下：
+ 首先 KeepersCompatible 调用 KeepersRegistrar 合约的 register 方法进行**注册**。 
+ 链下预言机调用 KeepersRegistrar 中的 approve **批准**该注册，将 KeepersCompatible 中的 **checkUpKeep 写入 KeepersRegitstry** 。 
+ 之后链下预言机网络在每个区块中都会调用 KeepersRegistry 中的 checkUpKeep 检测 KeepersCompatible 是否满足触发条件，如果为 True 就会调用 performUpKeep 函数执行后续逻辑。如果检查结果为 False，则在下个区块中再调用 checkUpkeep 进行检测。


![img.png](/images/blockchain/web3/chainlink-16.png)

+ 第一个应用是**自动复利**。很多 DeFi 应用会支付利息给存款用户，用户如果不提款，就相当于是单利，简单的说就是存一年给一年的利息。如果用户每隔一段时间将产生的利息提出来再存入，进行复利投资，就可以将收益最大化。这提取和存入的操作，是一个按照时间而定的固定操作，可以通过 Keepers 去完成操作，比如 Beefy，Alchemix，SNX 等项目都是通过 Keepers完成这种复利投资。 
+ 第二个应用是**借贷平台的清算** ( Liquidation )，如 AAVE 和 B-protocol。当用户在协议中的质押资产价格下跌，跌到警戒线以下，协议就需要把抵押物清算掉以避免进一步的损失。按照这个场景，checkUpKeep 函数里面可以写标的物拍卖的清算价格，performUpKeep 写具体的清算逻辑。checkUpKeep 返回为 Ture 时，Keepers 自动执行 performUpKeep 里面的对质押资产进行清算。 
+ 第三个应用是 **DEX 限价单（Limit order）**。DEX 中使用的自动做市商（Automatic Market Maker：AMM）不同于中心化交易所的订单簿模式，没有办法进行限价单。如果想用 limit order，需要自己编写逻辑，当通证价格低于某一个阈值时，去购买或者卖出，也就是用 Keepers ，在 checkUpkeep 里写判断逻辑，在 performUpkeep 里写执行逻辑。 
+ 第四个应用是**流动性管理和跨链 NFT 铸造**，比如在 Polygon mint 一个 NFT（因为以太坊主网gas比较贵）， 得到 ID 或者 features 属性值 （ 到底是稀有还是不稀有 ），这个数值可以通过 relay 写回到主网，帮助用户实现跨链铸造 NFT。 
+ 第五个应用是**动态 NFT**。它的逻辑是区块链上的 NFT，会随着现实世界中相关属性的变化而变化。比如天气NFT，外界下雨 NFT 就显示下雨，外界很热 NFT 就显示一个太阳。从外部获取数据，根据所获取的数据做出一些改变，这里就会频繁使用 Keepers。

## Chainlink Any API
Any API 可以将任何 web2 的数据作为数据源提供给链上。

![img.png](/images/blockchain/web3/chainlink-18.png)

![img.png](/images/blockchain/web3/chainlink-19.png)

Any API 由用户部署的 Consumer 合约和 Chainlink 部署的 Client LINK 合约 以及节点运营商部署的 Operator 合约组成。

![img.png](/images/blockchain/web3/chainlink-20.png)


