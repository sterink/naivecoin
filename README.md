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
