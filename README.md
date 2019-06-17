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
- **DIFFICULTY_ADJUSTMENT_INTERVAL**: 定义 修改难易度以适应网络不断增加或者降低的网络算力(hashRate, 即每秒能计算的哈希的数量)的 频率(比特币中是设置成2016个区块， 即每2016个区块，然后根据这些区块生成的耗时，做一次难易度调整)

这里我们将会把区块产出间隔设置成10秒，难易度调整间隔设置成10个区块。这些常量是不会随着时间而改变的，所以我们将其硬编码如下：

``` typescript
// in seconds
const BLOCK_GENERATION_INTERVAL: number = 10;

// in blocks
const DIFFICULTY_ADJUSTMENT_INTERVAL: number = 10;
```

有了这些规则，我们就可以在我们的网络上达成难易度的一致性了。每产出10个区块之后，我们就去检查生成所有这10个区块所消耗的时间，然后和**预期时间**进行对比，然后对难易度进行动态的调整。

这里的**预期时间**是这样指定的：BLOCK_GENERATION_INTERVAL * DIFFICULTY_ADJUSTMENT_INTERVAL。 该预期时间代表了当前网络的算力刚刚好和当前的难易度吻合。

如果耗时超过或者不足**预期时间**的两倍，那么我们就会将difficulty进行对应的减1或者加1。该算法如下：

``` typescript
const getDifficulty = (aBlockchain: Block[]): number => {
    const latestBlock: Block = aBlockchain[blockchain.length - 1];
    if (latestBlock.index % DIFFICULTY_ADJUSTMENT_INTERVAL === 0 && latestBlock.index !== 0) {
        return getAdjustedDifficulty(latestBlock, aBlockchain);
    } else {
        return latestBlock.difficulty;
    }
};

const getAdjustedDifficulty = (latestBlock: Block, aBlockchain: Block[]) => {
    const prevAdjustmentBlock: Block = aBlockchain[blockchain.length - DIFFICULTY_ADJUSTMENT_INTERVAL];
    const timeExpected: number = BLOCK_GENERATION_INTERVAL * DIFFICULTY_ADJUSTMENT_INTERVAL;
    const timeTaken: number = latestBlock.timestamp - prevAdjustmentBlock.timestamp;
    if (timeTaken < timeExpected / 2) {
        return prevAdjustmentBlock.difficulty + 1;
    } else if (timeTaken > timeExpected * 2) {
        return prevAdjustmentBlock.difficulty - 1;
    } else {
        return prevAdjustmentBlock.difficulty;
    }
};
```

### 时间戳校验

第一章节中的区块链版本中，区块结构中的时间戳属性是没有任何意义的，因为我们不会对其作任何校验的工作，也就是说，我们其实可以给该时间戳赋以任何内容。 但现在情况变了，因为我们这里引入了难易度的动态调整，而该调整是基于前10个区块产出的耗时总长度的。所以时间戳我们就不能像之前一样随意赋值了。

既然时间戳的大小会影响区块产出难易度的调整，所以有怀心思的人就会考虑去恶意设置一个错误的时间戳来尝试操纵我们的难易度，以实现对我们的区块链网络进行攻击。为了规避这种风险，我们需要引入以下的规则:

- 新区块时间戳比当前时间最晚不晚过1分钟，那么我们认为该区块有效。
- 新区块时间戳比前一区块的时间戳最早不早过1分钟，那么我们任务该区块在区块链中是有效的。


``` typescript
const isValidTimestamp = (newBlock: Block, previousBlock: Block): boolean => {
    return ( newBlock.timestamp - 60 < getCurrentTimestamp() )
        && ( previousBlock.timestamp - 60 < newBlock.timestamp );
};
```

> 天地会珠海分舵注：这里为什么要有一分钟的缓冲呢？估计是为了既考虑一定程度的容错，也减缓了时间戳恶意修改的攻击。如果还有其他原因的，请指教: zhubaitian@163.com

### 累积难易度

还记得第一章节的区块链的版本中，一旦发生冲突，我们总是选择最长的区块链作为有效的链进行更新。因为我们现在引入了挖矿的难易度，所以我们不能再这样子做了。也就是说，现在正确的链，不是最长的那条链，而是**累积难易度**最大的那条链。什么意思呢？换个说法就是，正确的链，应该是那条消耗了最多计算资源(网络算力*耗时)而产生出来的链。

那么怎么算出一条链的累积难易度呢？我们首先对区块链中每个区块的difficulty进行2^difficulty计算，然后将每个区块的计算结果加起来，最终就是这条链的累积难易度。

这里为什么我们用2^difficulty来计算呢？因为difficulty代表了哈希的二进制表示中前面的0的位数。试想下，一个difficulty是11的块和一个difficulty是5的块相比，总的来说，我们需要多计算2^(11-5) = 2^6 次哈希才能获得想要的结果。因为每一位在二进制中都有0和1两种变化，他们之间相差6个位，固有2^6个变化。

下图中，虽然链A比链B长，但因为链B的累积难易度比链A大，所以链B才是正确的链.

![](https://lhartikk.github.io/assets/Cumulative_difficulties.png)

同时我们还要注意的是，我们累积难易度只和区块的difficulty属性有关系，和区块的真实哈希即其前面的位数是没有任何关系的。 拿一个difficulty为4的区块来说，假如它的哈希是000000a34c...(该哈希同时也满足difficulty为5和6的情况)，我们计算累积难易度是还是会以2^4来算，而不是2^5或者2^6， 即使它前面有6个0.

这种根据难易度选择正确的链的策略也叫做中本聪共识(Nakamoto consensus)， 这也是中本聪的比特币中最重要的一个发明之一。一旦账本出现分叉，矿工们就必须选择一条链来投入资源继续进行挖矿，因为这个关系到矿工挖矿后的激励，所以选择正确的链必须全网达成共识。


### 小结

工作量证明拼图的一个重要特点就是难以解答但易于验证。所以不断调整nonce以算出一个满足一定难度的SHA256哈希，然后简单的验证前面几位是否是0，往往就是这种问题最简单的解决方案。

有了工作量证明机制后，节点现在就需要挖矿，也就是解决工作量证明拼图，才能往区块链中新增加一个区块了。在下一章节中，我们将会为我们的区块链引入交易(Transaction)功能。











