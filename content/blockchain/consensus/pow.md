+++
title = 'PoW'
date = 2024-03-21T15:32:53+08:00
+++

所谓的共识机制，是保证区块信息在全网节点中保持一致共识的机制。共识机制可分为两个阶段：1）提出共识内容；2）对内容达成共识。

工作量证明PoW（Proof of Work），通过算力的比拼来选取一个节点，由**该节点决定下一轮共识的区块内容（记账权）**。PoW要求节点消耗自身算力尝试不同的随机数（nonce），从而寻找符合算力难度要求的哈希值，不断重复尝试不同随机数直到找到符合要求为止，此过程称为“挖矿”。 具体的流程如下图：

![img.png](/images/blockchain/consensus/pow-1.png)

即我们需要构造一个计算不对称的难题，这个难题在比特币中被选定为以 SHA256 算法计算一个目标哈希，使得这个哈希值符合前 N 位全是 0。通过不停地变更区块头中的随机数即 nonce 的数值，也就是暴力搜索，并对每次变更后的的区块头做双重 SHA256 运算，即 SHA256(SHA256(Block_Header))），将结果值与当前网络的目标值（target全网统一，同时为了保证比特币每10分钟左右出一次块的速度，target可相应调整）做对比，如果小于目标值，则解题成功，工作量证明完成。



第一个找到合适的nonce的节点获得记账权。节点生成新区块后广播给其他节点，其他节点对此区块进行验证，若通过验证则接受该区块，完成本轮共识，否则拒绝该区块，继续寻找合适的nonce。

至今为止，寻找一个满足要求的nonce是非常困难的，要求节点消耗大量的算力。随着有效区块的不断积累，恶意节点需要消耗极大的算力才有可能推翻以前的区块，完成「双花」攻击。