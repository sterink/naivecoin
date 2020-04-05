>200行代码实现一个最小化可工作区块链，1500行代码实现一个加密货币网络系统。如果这次你还不能理解区块链是怎么回事的话，你打我！如果理解了，你打赏我，右上方给我打个星，Star一下以示鼓励!

作者针对原开源实现 https://github.com/lhartikk/naivecoin 做了一些补充，方便理解。
# 简介

本教程将带领大家从零开始开发一套可行的加密货币系统。开发的基本原则就是尽量的简单易懂。

我们打造的这个项目的名称叫做Naivecoin。 用的开发语言是Typescript。总共分为六个章节。大家可以选择相应的分支进去查看相应的代码。

如果你只是对区块链的实现原理感兴趣，那么你只需要看第一章就足够了，代码相当的简单，只用200行的代码就能让你一窥区块链的全貌。

### [第一章:最小可行区块链](https://github.com/zhubaitian/naivecoin/blob/chapter1/README.md)
这一章节中，我们会用200行左右的代码实现一个简单但五脏俱全的区块链，并引领大家理解区块链的基本原理。

### [第二章 工作量证明和挖矿](https://github.com/zhubaitian/naivecoin/blob/chapter2/README.md)
本章节我们将会在我们的玩具版区块链的基础上加入工作量证明(POW)的支持。在第一章节的版本中， 任何人都都可以在没有任何工作量证明的情况下添加一个区块到区块链中。 当我们引入工作量证明机制之后，一个节点必须要解开一个有相当计算量的拼图(POW Puzzle)之后，才能往区块链上添加一个新的区块。而去解开该拼图，通常就被称为挖矿。

### [第三章 交易](https://github.com/zhubaitian/naivecoin/blob/chapter3/README.md)
本章我们将引入加密货币中的交易机制。有了交易这个机制之后，我们的区块链将会从一个只有基本功能的区块链华丽转身成一个加密货币系统。 最终我们就能通过指定目标用户的地址，和对应的用户进行加密货币交易。

### [第四章 钱包](https://github.com/zhubaitian/naivecoin/blob/chapter4/README.md)
本章节我们将实现一个未加密的钱包功能以进行简单的交易。

### [第五章 交易中继](https://github.com/zhubaitian/naivecoin/blob/chapter5/README.md)
上一章节中，我们要给一笔交易记账的话，必须自己手动进行一次挖矿，才会把交易记录加到一个区块里面去。 这一章节中，我们将会引入未决交易中继的机制。有了这个机制之后，我们要进行一笔交易的时候，就不需要自己动手挖矿，而是将自己的交易发送到我们的区块链网络中去(即中继传递的概念)，由其他节点在挖矿之后，将我们的交易记录加到他们挖出的新的区块中去。其中这些交易就被称之为「未决交易」。一个典型的例子就是，当一个用户想要发起一笔交易(把一定数量的币发送到指定的地址)，他会把这笔交易广播到整个网络，并希望其他矿工把该笔交易放到区块中去。

### [第六章 钱包管理界面和区块链浏览器](https://github.com/zhubaitian/naivecoin/blob/chapter6/README.md)
本章节我们将为我们的区块链实现一个钱包管理界面和一个区块链浏览器。



##### Get blockchain
```
curl http://localhost:3001/blocks
```

##### Mine a block
```
curl -X POST http://localhost:3001/mineBlock
``` 

##### Send transaction
```
curl -H "Content-type: application/json" --data '{"address": "04bfcab8722991ae774db48f934ca79cfb7dd991229153b9f732ba5334aafcd8e7266e47076996b55a14bf9913ee3145ce0cfc1372ada8ada74bd287450313534b", "amount" : 35}' http://localhost:3001/sendTransaction
```

##### Query transaction pool
```
curl http://localhost:3001/transactionPool
```

##### Mine transaction
```
curl -H "Content-type: application/json" --data '{"address": "04bfcab8722991ae774db48f934ca79cfb7dd991229153b9f732ba5334aafcd8e7266e47076996b55a14bf9913ee3145ce0cfc1372ada8ada74bd287450313534b", "amount" : 35}' http://localhost:3001/mineTransaction
```

##### Get balance
```
curl http://localhost:3001/balance
```

#### Query information about a specific address
```
curl http://localhost:3001/address/04f72a4541275aeb4344a8b049bfe2734b49fe25c08d56918f033507b96a61f9e3c330c4fcd46d0854a712dc878b9c280abe90c788c47497e06df78b25bf60ae64
```

##### Add peer
```
curl -H "Content-type:application/json" --data '{"peer" : "ws://localhost:6001"}' http://localhost:3001/addPeer
```
#### Query connected peers
```
curl http://localhost:3001/peers
```