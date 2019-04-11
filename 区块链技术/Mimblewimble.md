### Mimblewimble & Grin

### 1. 标题和作者简介

​      MimbleWimble 出自《哈利波特》中的一句咒语，这个标题的涵义应该是希望所有读到这篇介绍的人都可以来为这个开放社区做点贡献；

​      作者在 Tor 网络里放了一个 Dot Onion 的 Link，这个 Link 指向一个 TXT 文件，就是 MimbleWimble 协议。作者通过这样的方式隐藏了自己的身份，从此之后便消失匿迹了；

​       作者的协议并没有写完，后来经过 Blockstream 的 Andrew Poelstra 深入的研究，证明了 MimbleWimble 系统的安全性，才算把这个协议真正的完成。

### 2. 技术详解

分别介绍 Mimblewimble 的三个基本组件：Confidential Transaction, Coin Join,OWAS(One Way Aggregate Signatures)

#### 2.1 Confidential Transaction

CT 这个 idea 最早由 Blockstream 的 CEO Adam Back 提出。

备注：Adam Back 被称为比特币的教父，因为他是挖矿算法 Hashcash 的发明人，他是被中本聪在论文里引用了的人

基本的思想是：把交易金额变成密文，同时保证交易的有效性不受影响。
使用 Pedersen Commitment来实现上述目的：

$C=vG+rH$  其中 $v$ 是要隐藏的金额，$r$ 是为隐藏金额选择的随机数，称为"blind factor"

Pedersen Commitment 满足**加法同态性**： $（v_1G + r_1H）+（v_2G + r_2H）=（v_1 + v_2）G+（r_1 + r_2)H$

假设某交易的输入金额为 $a$ ,输出金额为 $b$ ,交易费为 $f$ ，要进行的检查为 $a=b+f$ ：

- Prover 随机选 $\alpha _a\in Z_q, \alpha _b\in Z_q$ ,让 $\alpha_f=\alpha _a-\alpha _b$
- Prover 分别对 $a,b,f$ 做commitment，即$P=\alpha _aG+aH,Q=\alpha _bG+bH,R=\alpha _fG+fH$
- Verifier进行如下验证：$P-Q-R=(\alpha _a-\alpha _b-\alpha_f)G+(a-b-f)H=0G+0H$ 

因所有的运算是在群内，且所有的运算需要模 $p$ ，若 $p=13$， $a-b-f=1-9-5=-13=0(mod\ 13)$，也可通过上述检验，所以需与range proof 结合使用，确保金额是正的且在某个区间内，使计算不会溢出

#### 2.2 CoinJoin

CoinJoin 最早是由 Gregory Maxwell 提出的，主要为保证比特币的隐私，想要达到的效果是打破输入和输出的对应关系来实现隐私保护。

基本的思想是：把两个交易的左侧加在一起，右侧加在一起，还是一个合法的交易。

如下图，将两个交易合并成一个交易，但是需要注意的是，此时CoinJoin交易需要两个签名，Bob 和 Alice 都对此交易单签名。

此方式并没有很好的保护隐私，比如下图中，很容易分析出输入和输出的对应关系。

![coinjoin](images\coinjoin.png)

#### 2.3 OWAS

关于算法的介绍参照链接:<https://download.wpsoftware.net/bitcoin/wizardry/horasyuanmouton-owas.pdf>

本文档目前只介绍其在mimblewimble中的使用方式。

场景假设：Alice的输入为8btc，其中转账给Bob 6btc，找零自己1btc，交易费 1btc

+ input： 
  + input1:  5 btc
  + input2:  3 btc
+ output：
  + out_Bob：6 btc
  + out_change: 1btc
  + out_fee: 1btc 【公开】

第一步:

 Alice和Bob协商转账金额 ：6btc

第二步：

 Alice 对所有的input 以及找零金额进行隐藏，具体的操作为：

+ 选随机值 $r_1$ 对input1 进行隐藏

  $C_{in1}=5*G+r_1H$

+ 选随机值 $r_2$ 对input2 进行隐藏

  $C_{in2}=3*G+r_2H$

+ 选随机值 $r_3$ 对change进行隐藏

  $C_{change}=1*G+r_3H$

第三步：

 Alice 将 $r_{Alice}=r_1+r_2-r_3$ 以及$C_{in1},C_{in2},C_{change}$ 发送给Bob

第四步：

 Bob首先进行验证：

+ $C_{in1}+C_{in2}-C_{change} = (6+fee)G+rH$

Bob 选随机值 $r_4$ 对 out_Bob 进行隐藏:

+ $C_{Bob}=6*G+r_4H$

Bob 计算 $K=(r_{Alice}-r_4)H$ ，把 k 作为公钥公布，$r_{Alice}-r_4$ 作为私钥

矿工验证交易有效性的方式为：

$C_{in1}+C_{in2}-C_{change}-C_{Bob}-(K+fee*H) =0$

以上是基础的minblewimble方案，即最初被提出的txt文件中描述的场景

接下来描述 Andrew Poelstra 完善后的版本。

### 3.更完整的Mimblewimble协议

MimbleWimble的交易确认依赖于两个基本属性:

- **0和验证。** 输出总和减去输入总是等于零，证明交易没有凭空创造新的资金，而且**不会显示实际金额**。
- **拥有私钥即拥有交易输出的所有权。** 像大多数其他加密货币一样，交易输出通过拥有ECC私钥来保证其所有权。 然而，在MimbleWimble中，证明一个所有者拥有这些私钥并不是通过直接签署交易来实现的。

#### 3.1实例描述

一个交易核，包含：

- kernel excess，用来确保等式平衡
- 交易签名（采用kernel excess作为签名公钥）

假设

+  $C_{out1}=42*G + 1*H$  
+  $C_{out2}=99*G + 2*H$ 
+  $C_{in1}=113*G + 3*H$

```
(42*G + 1*H) + (99*G + 2*H) - (113*G + 3*H) = 28*G + 0*H
```

接下来使用的签名公钥是 `28*G`，也称为kernel_excess

任何一笔交易必须满足以下条件： （为了描述简便，这里忽略掉交易费部分）

```
sum(outputs) - sum(inputs) = kernel_excess
```

这个条件同样适用于区块，因为区块只是一系列聚合的交易输入、交易输出和交易核。我们可以把所有的交易输出加起来，减去所有的交易输入，将结果与所有交易核中的kernel excess之和做比较：

```
sum(outputs) - sum(inputs) = sum(kernel_excess)
```

上面描述的MimbleWimble区块和交易设计有一个小问题，有可能从一个区块中的数据来重建交易（即找出一笔或几笔完整的交易，分辨哪一笔交易输入对应哪一笔交易输出）。这个对于隐私而言当然是不好的事情。这个问题也被称为子集问题（"subset" problem） - 给定一系列交易输入、交易输出和交易核，有可能能够从中分辨出一个子集来重新拼出对应的完整的交易（很像拼图游戏）。

例如，假如有下面的两笔交易：

```
(in1, in2) -> (out1), (kern1)
(in3) -> (out2), (kern2)

```

我们能够聚合它们并构建下面的区块（或一笔聚合交易（*aggregate transaction*））：

```
(in1, in2, in3) -> (out1, out2), (kern1, kern2)

```

很容易利用等式平衡关系用穷举法试验所有可能的组合，从而找出原始的交易关系：

```
(in1, in2) -> (out1), (kern1)

```

只要找出了一笔交易，那么剩下的当然也是符合等式平衡关系的，于是很容易就拼凑出另一笔交易：

```
(in3) -> (out2), (kern2)

```

为了大幅降低这个拼凑的可能性，从而缓解这个问题的不利影响，我们设计一个交易核偏移因子（*kernel offset*）给每一个交易核。 这也是一个致盲因子（或者说一个私钥），它需要加到kernel excess当中用于验证等式平衡关系：

```
sum(outputs) - sum(inputs) = kernel_excess + kernel_offset

```

当我们聚合这些交易到区块的时候，我们在区块头中存储一个（且仅一个）聚合偏移因子（aggregate offset）（即所有交易核偏移因子的总和）。这样一来，因为我们一个区块只有一个偏移因子，再也不可能将其分拆对应到每一笔交易的交易核偏移因子了，从而也就不可能再从区块中拼凑出任何一笔交易了。

```
sum(outputs) - sum(inputs) = sum(kernel_excess) + kernel_offset

```

具体的实现方法就是，在创建交易时将 `k` 分割成 `k1+k2`。 对于交易核 `(k1+k2)*G`，我们在交易核中发布出去的是 `k1*G`（称之为：the excess），以及 `k2`（称为：the offset），并跟以前一样使用 `k1*G` 作为公钥来对交易进行签名。 在矿工构建区块的时候，我们对打包的所有交易的`k2`（the offset）求和，以生成一个单个的聚合值（aggregate `k2` offset）用于该区块所打包的所有交易。一旦区块打包完成并发布和被链所接受，其原始的对应每笔交易的`k2` （the offset）即成为不可恢复的。

参考文献：

<https://github.com/mimblewimble/docs/wiki/MimbleWimble-Origin>

<https://github.com/mimblewimble/grin/blob/master/doc/intro_ZH-CN.md>

