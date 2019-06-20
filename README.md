# 第四章 钱包

- [概览](#概览)
- [生成钱包](#生成钱包)
- [钱包余额](#钱包余额)
- [生成交易](#生成交易)
- [使用钱包](#使用钱包)
- [测试体验](#测试体验)
- [小结](#小结)

###  概览
钱包的目的是为了给用户创建更高层的抽象接口来对交易进行管理。

我们最终的目的是让用户可以方便的：
- 创建一个新钱包
- 查看钱包的余额
- 在钱包之间进行交易

以上这些生效后，用户就不需要知道上一章节中描述的inputs和outpus这些交易的细节，就能对交易进行管理了。就好比在比特币网络中，你只需要把比特币打入对应地址就能给别人打入比特币， 同时，你只需要将你自己的地址发布出去，别人就能给你打比特币了。

本章节完整的代码请看[这里](https://github.com/zhubaitian/naivecoin/tree/chapter4)

### 生成钱包

本教程中我们将会用最简单的方法对钱包进行初始化和存储：将未加密的私钥保存在node/wallet/private_key这个文件中。

``` typescript
const privateKeyLocation = 'node/wallet/private_key';

const generatePrivatekey = (): string => {
    const keyPair = EC.genKeyPair();
    const privateKey = keyPair.getPrivate();
    return privateKey.toString(16);
};

const initWallet = () => {
    //let's not override existing private keys
    if (existsSync(privateKeyLocation)) {
        return;
    }
    const newPrivateKey = generatePrivatekey();

    writeFileSync(privateKeyLocation, newPrivateKey);
    console.log('new wallet with private key created');
};
```

如之前所言，我们的公钥(钱包地址)是通过私钥演绎出来的。

``` typescript
const getPublicFromWallet = (): string => {
    const privateKey = getPrivateFromWallet();
    const key = EC.keyFromPrivate(privateKey, 'hex');
    return key.getPublic().encode('hex');
};
```

需要提一提的是，把私钥明文保存在文件中是一个非常不安全的做法。我们这样子做只是为了演示的简单起见而已。同时，一个钱包当前只支持一个私钥，所以，如果你需要一个新的公钥来作为地址的话，必须要创建一个新的钱包。

### 钱包余额

复习下上一章节提到的一个说法：你在区块链中拥有的加密货币， 指的其实就是在「未消费交易outputs」中，接收者地址为自己的公钥的一系列outputs。

这意味着，我们如果要查看钱包余额的话，事情就变得非常简单了：你只需要将该地址下的所有「未消费交易output」记录的货币数加起来就完了。

``` typescript
const getBalance = (address: string, unspentTxOuts: UnspentTxOut[]): number => {
    return _(unspentTxOuts)
        .filter((uTxO: UnspentTxOut) => uTxO.address === address)
        .map((uTxO: UnspentTxOut) => uTxO.amount)
        .sum();
};
```

为了让演示更简单，我们在查询一个钱包地址的余额时并不需要提供任何私钥信息。也就是说，任何人都可以查看别人的账户余额。

### 生成交易

进行加密货币交易时，用户不应该需要关心交易中的inputs和outputs这些细枝末节。但是，当用户A的账户有50个币时，如果他要给用户B发送10个币时，交易后面究竟发生了什么事情呢？

这种情况下，系统会将10个币发送到B的公钥地址，同时会将剩余的40个币还给用户A。也就是说，来源的50个币必须消费完，所以在将来源的币赋给交易的outputs时必须进行拆分。交易完后必须将来源50币的output在「未消费交易outputs」中删除，将新产生的两个outputs加上去。 也就是说「未消费交易outputs」上的货币总量不会变，只是有些币被交易了，地址属于不同的用户了而已。

下图演示了上面所提及交易：

![](https://lhartikk.github.io/assets/tx_generation.png)

下面我们来看一个更复杂点的场景：

- 用户C最开始拥有0个币
- 之后的三个交易让C分别获得了10，20，30个币
- C想要给D转发55个币。

这种情况下，用户C的所有3个outputs(在「未消费交易outputs」中address为C公钥的那些outputs)都为被用上才能凑够55个币给D,剩余的5个币则会还给C。

![](https://lhartikk.github.io/assets/tx_generation2.png)

那么如何将以上的描述用代码逻辑来表示呢？ 首先，我们会为交易创建相应的inputs。怎么创建呢？我们会遍历「未消费交易outputs」中地址为发送者公钥的项，直到找到能够凑够足够的币数(outputs的币数加起来大于等于目标币数)的outputs。

``` typescript
const findTxOutsForAmount = (amount: number, myUnspentTxOuts: UnspentTxOut[]) => {
    let currentAmount = 0;
    const includedUnspentTxOuts = [];
    for (const myUnspentTxOut of myUnspentTxOuts) {
        includedUnspentTxOuts.push(myUnspentTxOut);
        currentAmount = currentAmount + myUnspentTxOut.amount;
        if (currentAmount >= amount) {
            const leftOverAmount = currentAmount - amount;
            return {includedUnspentTxOuts, leftOverAmount}
        }
    }
    throw Error('not enough coins to send transaction');
};
```
如代码所示，我们除了找到满足条件的那些未消费的交易outputs，还会记录下交易后剩余的需要还给发送者的币数leftOverAmount。

找到这些未消费的交易outputs之后，我们就可以为当前交易创建对应的inputs了:

``` typescript
const toUnsignedTxIn = (unspentTxOut: UnspentTxOut) => {
    const txIn: TxIn = new TxIn();
    txIn.txOutId = unspentTxOut.txOutId;
    txIn.txOutIndex = unspentTxOut.txOutIndex;
    return txIn;
};
const {includedUnspentTxOuts, leftOverAmount} = findTxOutsForAmount(amount, myUnspentTxouts);
const unsignedTxIns: TxIn[] = includedUnspentTxOuts.map(toUnsignedTxIn);

```
做法很简单，主要就是将每个input中的txOutId指向上面找到的对应的未消费交易output项目。

紧跟着就需要创建示例中的2个outputs了：一个output是给接收者的，另外一个output是发送者自己的，因为需要把剩余的币数还回来。当然，如果inputs中指向的币数总和刚好等于交易量，即leftOverAmount为0， 我们就只需要创建一个发送给目标用户的output就够了。

``` typescript
const createTxOuts = (receiverAddress:string, myAddress:string, amount, leftOverAmount: number) => {
    const txOut1: TxOut = new TxOut(receiverAddress, amount);
    if (leftOverAmount === 0) {
        return [txOut1]
    } else {
        const leftOverTx = new TxOut(myAddress, leftOverAmount);
        return [txOut1, leftOverTx];
    }
};
```
最后，我们会进行交易id计算(对交易内容做哈希)，并对交易进行签名，最终将交易签名赋予给每个input：

``` typescript
const tx: Transaction = new Transaction();
    tx.txIns = unsignedTxIns;
    tx.txOuts = createTxOuts(receiverAddress, myAddress, amount, leftOverAmount);
    tx.id = getTransactionId(tx);

    tx.txIns = tx.txIns.map((txIn: TxIn, index: number) => {
        txIn.signature = signTxIn(tx, index, privateKey, unspentTxOuts);
        return txIn;
    });
```

### 使用钱包

我们会提供'/mineTransaction‘这个api来让用户方便的使用钱包这个功能：

``` typescript
app.post('/mineTransaction', (req, res) => {
        const address = req.body.address;
        const amount = req.body.amount;
        const resp = generatenextBlockWithTransaction(address, amount);
        res.send(resp);
    });
```

如代码所示，使用者只需要提供接收方的地址和交易数量就能通过该api来实现交易。该api调用后为首先进行一次挖矿，获得一笔50个币的原始交易，然后根据接收方地址和交易数量完成指定交易，最终将这些交易记录到新增加的区块中，同时更新我们的「未消费交易outputs」。

### 测试体验

- 启动

为了方便测试，本人对package.json中的启动脚本做了些修改， 让我们可以快速的启动两个节点进行测试，而不需要每次启动都输入一堆的参数。

``` shell
npm run node1
npm run node 2
```

同时, 在启动第二个节点时，加入PEER参数，让第二个节点自动和节点1建立P2P连接。
``` json
"scripts": {
    "prestart": "npm run compile",
    "node1": "HTTP_PORT=3001 P2P_PORT=6001 WALLET=1 npm start ",
    "node2": "HTTP_PORT=3002 P2P_PORT=6002 WALLET=2 PEER=ws://localhost:6001 npm start ",
    "start": "node src/main.js",
    "compile": "tsc"
  },
```

最后，通过新加的WALLET参数的支持，我们会为每个节点自动初始化一个钱包，这样我们就更容易观察钱包和交易的行为。

- 提供额外的查看「未消费交易outputs」接口

因为「未消费交易outputs」这个清单是非常重要的，这时我们进行交易的基础。所以很有必要提供一个额外的接口来查看里面的数据的变化。你可以通过GET的方式发送请求到 "http://localhost:3001/unspentTransactionOutputs" 接口来获得该清单。

注意，如果你起了两个节点，那么端口使用3001和3002都是可以的。 因为该清单是一个分布式清单，和我们的区块链一样，是全网同步的。

有了这些之后，你就可以方便的通过postman调用相应的接口对钱包和交易功能进行体验了。



### 小结

我们刚刚实现了一个未加密的钱包功能以进行简单的交易。虽然这个交易算法(mineTransaction接口相关的逻辑)中最多只能有两个接收者outputs(一个是接收者，一个是自己)， 但是我们底层的区块链接口是能支持任意数量的outputs的。比如你可以创建一个inputs是50个币，outputs分别是5，15和30个币的交易， 但你必须手动填充这些数据并调用/mineRawBlock这个接口来达成。

迄今为止，我们进行一次交易，还是需要先进行一次挖矿，才能将交易信息加入到新的区块里面。我们当前的各个节点不会对未记录在区块中的交易做任何信息交换。 这些问题都将会在下一个章节进行解决。

本章节完整的代码请看[这里](https://github.com/zhubaitian/naivecoin/tree/chapter4)

[第五章](https://github.com/zhubaitian/naivecoin/blob/chapter4/README.md)










