# #1 工作量证明

### 概览
本章节我们将会在我们的玩具版区块链的基础上加入工作量证明(POW)的支持。在第一章节的版本中， 任何人都都可以在没有任何工作量证明的情况下添加一个区块到区块链中。 当我们引入工作量证明机制之后，一个节点必须要完成一个有相当计算量的拼图任务(POW Puzzle)之后，才能往区块链上添加一个新的区块。而去解决该拼图任务，通常就被称为挖矿。

引入工作量证明机制之后，我们还可以对一个新区块的产出时间作出大致的控制。大概的做法就是动态的改变拼图任务的难易程度来达到控制的效果：如果最近的区块产生的太快了，那么就将拼图任务的难度提升，反之，则将拼图任务的难度降低。

需要点出来的时，本章节中我们还没有引入 **交易(Transaction)** 这个概念。这就意味着矿工挖出一个区块后，并不会获得相应的奖励。 一般来说，在加密货币中，如果矿工挖到了一个区块，是应该获得相应的激励的。

### 难易程度(difficulty)， 随机数(nonce) 及 工作量证明拼图任务(POW puzzle)
在上一个章节的区块链的基础上，我们将会为区块结构加入两个新的属性：difficulty和nonce。要弄清楚这俩货是干嘛用的，我们必须先对工作量证明拼图任务作一些必要的阐述。

工作量证明拼图任务是一个什么样的任务呢？其实就是去计算出一个满足条件的区块哈希。怎么才算满足条件呢？如果这个计算出来的哈希的前面的0的数目满足指定的个数。那么这个个数又是谁指定的呢？对，就是上面的这个difficulty指定的。如果difficulty指定说哈希前面要有4个0，但你计算出来的哈希前面只有3个0，那么这个哈希就不是个有效的哈希。

下图展示的就是不同难易程度的情况下，哈希是否有效：

![](https://lhartikk.github.io/assets/difficulty_examples.png)

以下代码用来检查指定difficulty下哈希是否有效:

``` typescript
const hashMatchesDifficulty = (hash: string, difficulty: number): boolean => {
    const hashInBinary: string = hexToBinary(hash);
    const requiredPrefix: string = '0'.repeat(difficulty);
    return hashInBinary.startsWith(requiredPrefix);
};
```

区块的哈希是通过对区块的内容算sha256来获得的，通过相同的区块内容做哈希，我们是无法算出出符合指定difficulty的哈希， 因为内容一直没有边，哈希也就一直不变， 这就是为什么我们引入了nonce这个属性到区块中，我们可以控制nonce的改变来获得不同的哈希值。只要我们的内容有任何一点点的改变，算出来的哈希就肯定是不一样的。一旦相应的nonce修改后让我们获得到指定difficulty的哈希，我们挖矿也就成功了！

既然现在我们加入了difficulty和nonce到区块中，我们还是先看看区块结构现在长什么样吧：

``` typescript
class Block {

    public index: number;
    public hash: string;
    public previousHash: string;
    public timestamp: number;
    public data: string;
    public difficulty: number;
    public nonce: number;

    constructor(index: number, hash: string, previousHash: string,
                timestamp: number, data: string, difficulty: number, nonce: number) {
        this.index = index;
        this.previousHash = previousHash;
        this.timestamp = timestamp;
        this.data = data;
        this.hash = hash;
        this.difficulty = difficulty;
        this.nonce = nonce;
    }
}
```
当然，我们的创世区块是硬编码的，记得把它也给更新成相应的结构哦。

### 挖矿

如上所述， 为了解开我们的工作量证明拼图这个任务，我们需要不停的修改区块中的nonce然后计算修改后的哈希，直到算出满足条件的哈希。这个满足条件的哈希什么时候才会跑出来则完全是个随机的事情。所以我们要做的就是给nonce一个初始值，然后不停的循环，修改，计算哈希，直到满足条件的心仪的它的出现。

``` typescript
const findBlock = (index: number, previousHash: string, timestamp: number, data: string, difficulty: number): Block => {
    let nonce = 0;
    while (true) {
        const hash: string = calculateHash(index, previousHash, timestamp, data, difficulty, nonce);
        if (hashMatchesDifficulty(hash, difficulty)) {
            return new Block(index, hash, previousHash, timestamp, data, difficulty, nonce);
        }
        nonce++;
    }
};
```

一旦满足条件的区块的哈希给找到了，挖矿成功！然后我们就需要将该区块广播到网络上，让其他节点接受我们的区块，并更新最新的账本。这个和第一章节的情况并无二致。

### 难易度共识

我们现在已经拥有找出和验证满足指定难易度的区块的哈希的手段了，但是，这个难易程度是如果决定的呢？全网也需要一致。不然我的难易度是要挖半天才能出来，别人的是几毫秒就出来，那这个区块链网络就有问题。

所以，这里必须要有一个方法来让所有的节点都一致认同当前挖矿的难易度。为了做到这一点，我们首先引入一些用于计算当前难易度的规则。

先定义以下的一些常量：

- **BLOCK_GENERATION_INTERVAL**： 定义 一个区块产出的 频率(比特币中该值是设置成10分钟的， 即每10分钟产出一个区块)
- **DIFFICULTY_ADJUSTMENT_INTERVAL**: 定义 修改难易度以适应网络不断增加或者降低的网络算力的 频率(比特币中是设置成2016个区块， 即每2016个区块，然后根据这些区块生成的耗时，做一次难易度调整)

















