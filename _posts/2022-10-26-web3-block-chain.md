---
layout: post
title: Web3.0-区块链
categories: [web3.0]
description: Web3.0-区块链
keywords: web3.0, block chain, 区块链
---

# Web3.0 - 区块链

最近在学习web3.0相关知识，权且做些笔记。



## 1. 相关概念

什么是区块链呢？

简单地回答 --- 区块链就是一个又一个区块组成的链条。



专业一点回答这个问题：

-   狭义上讲：区块链是一种将数据区块按时间顺序相连的链式、不可篡改和不可伪造的分布式账本。
-   广义上讲：区块链是一种按照时间顺序将若干数据区块相连的链式、无中心、不可篡改、不可伪造、集体维护、全程留下痕迹、交易可以追溯的分布式账本（数据库）



区块链是一个建立在P2P网络上的分布式数据库，区块链使用密码技术链接，由共识确认的，按时间顺序追加的 分布式账本。



## 2. 区块链特点

去中心化、自治、安全（数据不可篡改）、匿名性





## 3. 密码学相关原理

>   比特币叫做加密货币，但是实际上是”不加密“的，包括转账等操作都是公开的。

区块链的密码技术有**数字签名算法**和**哈希算法**。

数字签名算法是数字签名标准的一个子集，表示了只用作数字签名的一个特定的公钥算法；

而哈希算法是将任意长度的二进制明文映射为较短的二进制串的算法，并且不同的明文很难映射为相同的Hash值。



### 3.1 hash function

密码学中的hash算法/函数，被称为 cryptographic hash function。它有3个重要性质：

1.  哈希碰撞（**collision resistance**）。哈希碰撞不可避免，但是很难认为制造出碰撞来。

    比如说，有2个不相同的x和y，Hash(x) = Hash(y)，这就是哈希碰撞。但是很难反过来根据Hash(x) 去求出y值来。

    利用这个性质，可以用来计算一个message的digest，比如hash(m)=d，如果m被人篡改了，那么hash(m)肯定发生变化。

2.  hiding。

    hiding是说哈希函数的计算过程是单向的，是不可逆的，给定一个输入x，可以算出他的哈希值H（x）。但是从哈希值没法反推出原来的输入x。也就是说这个哈希值没有透露有关这个x的任何信息。当然暴力求解还是有可能反推出来的。所以hiding这个性质的前提是输入空间足够大，大道暴力无法求出来。

3.  puzzle friendly

    他的意思是说哈希值的计算事先是不可预测的，如果你预测它落在那个范围之内，没有好办法，只能一个一个去试一下。

    >   比特币的挖矿，实际上就是找一个nonce，nonce和区块头的其他信息一个座位输入，算出一个hash值，这个hash值要小于等于目标阈值。puzzle friendly就是说这个挖矿的过程没有捷径，只能不停的试大量的nonce才能找到符合要求的解，所以这个过程才可以被用来做工作量证明，叫做proof of work（POW）。你挖到矿了找到符合要求的nonce一定是因为你做大量的工作，其他没有捷径。
    >
    >   但是有人找到这个nonce，发布出去，其他人验证是不是符合要求很容易，只有算一次哈希值就行了。
    >
    >   挖矿很难，验证很容易。(difficult to solve ,but easy to verify)

比特币中用的哈希函数叫作SHA-256(secure hash algorithm )以上三个性质它都是满足的。



### 3.2 签名

 在比特币系统中开账户：在本地创立一个公私钥匙对(public key ,private key)，这就是一个账户。开户过程不需要任何人审批！

公私钥匙对是来自于非对称的加密技术(asymmetric encryption algorithm)。公钥相当于银行卡账号，私钥相当于密码。向别人转账，需要知道别人的公钥。

理论上来说，有可能产生相同的公私钥对，但这也仅仅是理论。实际上这种概率微乎其微。产生公私钥的过程是随机的，需要选择好的随机源。

>   假如A想向B转10个比特币，A把交易放在区块链上，别人怎么知道这笔交易是A发起的呢?这就需要A要用自己的私钥给交易签名，其他人收到这笔交易后，要用A的公钥去验证签名正确性。**签名用私钥，验证用公钥**，**用的仍然是同一个人的**。





## 4. 区块链数据结构

区块链数据以区块的形式存在，区块又分为区块头和区块体。

**区块头**占80个字节，包含6个字段：

| 字段                    | 大小（字节） | 含义                                                   |
| ----------------------- | ------------ | ------------------------------------------------------ |
| 父区块哈希值            | 32           | 记录该区块的上一个区块的Hash值，以此维持一个链         |
| 版本号                  | 4            | 记录了区块头的版本号，用于跟踪软件/协议的更新          |
| 时间戳                  | 4            | 记录了该区块的创建时间戳                               |
| 难度系数                | 4            | 记录了该区块链工作量证明的难度目标                     |
| 随机数(nonce)           | 4            | 记录用于证明工作量的计算参数                           |
| 默克尔根（merkle root） | 32           | 记录该区块中交易的merkle树根，以此归纳区块中的交易信息 |

区块体 主要包含大量交易信息，结构为merkle tree。

其结构为：

![](/images/posts/web3/block-head-body.png)

比特币中的节点分为两类：

-   **全节点** (保存整个区块的内容，即块头块身都有，有交易的具体信息)
-   **轻节点** (例如手机上的比特币钱包)(只有块头)



## 5.  merkle proof

如何向一个轻节点证明某个交易是写入区块链的?

这时需要用到merkle proof ：
> 找到交易所在的位置(图中最底行的其中一个区块)，这时该区块一直往上到根节点的路径就叫merkle proof

![](/images/posts/web3/merkleProof.png)

merkle proof 可以证明merkle tree里面包含了某个交易，所以这种证明又叫proof of membership或 proof of inclusion。


对于一个轻节点来说，验证一个merkle proof复杂度是多少? 假设最底层有n个交易，则merkle proof 复杂程度是θ(log(n))。



如何证明merkle tree里面没有包含某个交易？ 即proof of non-membership。 

<比特币中不需要做这种不存在证明> 


可以把整棵树传给轻节点，轻节点收到后验证树的构造都是对的，每一层用到的哈希值都是正确的，说明树里只有这些叶节点，要找的交易不在里面，就证明了proof of non-membership。 

问题在于，它的复杂度是线性的θ(n)，是比较笨的方法。 

如果对叶节点的排列顺序做一些要求，比如按照交易的哈希值排序（比如说从小到大）。 

要查的交易先算出一个哈希值，看看如果它在里面该是哪个位置。 

比如说在第三个第四个之间，这时提供的proof是第三个第四个叶节点都要往上到根节点。 

如果其中哈希值都是正确的，最后根节点算出的哈希值也是没有被改过的，说明第三、四个节点在原来的merkle tree里面，确实是相邻的点。 

要找的交易如果存在的话，应该在这两个节点中间。但是它没有出现，所以就不存在。 




















