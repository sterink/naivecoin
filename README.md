# 交易
### 概览
本章我们将引入加密货币中的**交易**这个概念。有了交易这个机制之后，我们的区块链将会从一个基本区块链华丽翻身成一个加密货币系统。 最终我们就能通过指定目标用户的地址，和对应的用户进行加密货币交易。 当然，交易之前你必须能证明你是该货币的持有者。

为了达到这个目标，我们还需要了解不少的一些概念。 比如非对称加密，签名，交易发起者(input)和交易接收者(output)等。

### 非对称加密和签名

在非对称加密中，你会有一个公钥-私钥对，其中公钥是通过私钥演绎生成，但私钥不能通过公钥演绎出来。公钥可以安全的分享给其他人，但是私钥只能自己保存。

任何消息都能通过私钥进行加密而生成一个签名。任何拥有对应公钥的人，都能通过该公钥将信息进行解密，进而验证该签名的有效性。

![](https://lhartikk.github.io/assets/Digital_signatures.png)

这里我们将会用到一个叫做[elliptic](https://github.com/indutny/elliptic)的库来实现非对称加密，该库用的是椭圆曲线加密算法(ECDSA)。

总的来说，我们的加密货币系统中会用到两套不同的加密算法来实现不同的目的:

- SHA256: 在工作量证明中对区块进行哈希计算，保证数据的完整性和不可篡改性。
- 非对称加密: 用户身份验证。

### 私钥和公钥

一个有效的私钥是一个32字节的字符串，示例如下：
```
19f128debc1b9122da0635954488b208b829879cf13b3d6cac5d1260c0fd967c
```

一个有效的公钥是由‘04’开头，紧接着64个字节的自负换，示例如下：
```
04bfcab8722991ae774db48f934ca79cfb7dd991229153b9f732ba5334aafcd8e7266e47076996b55a14bf9913ee3145ce0cfc1372ada8ada74bd287450313534a
```
公钥从私钥演绎生成。公钥将会被用作交易的接收者(即接收者地址)。

### 交易概览

在我们编写任何代码之前，我们先对交易的结构进行大致的了解。交易将会有两个部分组成：inputs和outputs。 outputs指定了加密货币交易的接收者，inputs则是用来证明用来交易的货币是确实存在且是被交易发送者所持有的。每个交易中的inputs都会指向一个已经存在的(未花费的)output。

![](https://lhartikk.github.io/assets/transactions.png)

### 交易output
交易接收者(output/TxOut)结构由地址和数量两个成员变量组成。 数量代表了交易的数量。地址就是一个ECDSA的公钥，代表了接收者。这就意味着拥有对应地址的私钥的用户才能访问对应的加密货币。

``` typesript

class TxOut {
    public address: string;
    public amount: number;

    constructor(address: string, amount: number) {
        this.address = address;
        this.amount = amount;
    }
}
```

### 交易input
交易发起者(input/TxIn)结构提供了加密货币的来源信息。每一个txIn都会通过txOutId指向此前的一次交易，本次交易的加密货币就是从此前的该次交易解锁出来的，解锁后的货币就能在txOut中被发送给接收者了。结构中的signature这个属性则是发送者用自己私钥进行加密的签名(签名数据为本次交易的id)，通过用指向的前一次交易中txOut中的地址(即公钥)对该签名进行验证，即能证明该发送者是该交易及所指向的前一次交易的拥有者。

``` typescript
class TxIn {
    public txOutId: string;
    public txOutIndex: number;
    public signature: string;
}
```

需要注意的是这里保存的只是通过私钥进行的签名，而不是私钥本身。在区块链的整个系统中，仅仅存在他的公钥和签名，而不会出现他的私钥。

总的来说，我们可以认为txIns解锁了对应的加密货币，而txtOuts重新锁定这些货币并让接收者成为新的持有者。

![](https://lhartikk.github.io/assets/transactions2.png)

### 交易数据结构

当定义好上面的交易发送者和接收者之后，交易本身的数据结构就变得异常简单了。

``` typescript
class Transaction {
    public id: string;
    public txIns: TxIn[];
    public txOuts: TxOut[];
}
```
### 交易id
交易id代表了一次交易的唯一性，是通过对交易数据结构中的内容做哈希计算出来的。这里要注意的是我们并没有包含发起者的签名，这个会在之后添加。

``` typescript
const getTransactionId = (transaction: Transaction): string => {
    const txInContent: string = transaction.txIns
        .map((txIn: TxIn) => txIn.txOutId + txIn.txOutIndex)
        .reduce((a, b) => a + b, '');

    const txOutContent: string = transaction.txOuts
        .map((txOut: TxOut) => txOut.address + txOut.amount)
        .reduce((a, b) => a + b, '');

    return CryptoJS.SHA256(txInContent + txOutContent).toString();
};
```

### 交易签名
通过签名来保证交易内容不被修改是非常重要的。因为交易都是被公开的，任何人都能够访问所有的交易，就算这些交易还没有来得及加入到区块链当中去。

当对交易进行签名时，事实上我们只会对交易id进行签名。也就是说，参考前面的交易id的生成，只要交易中任何的一项内容发生变化，交易id都会发生变化， 然后相应的对交易id的签名也就会改变，导致整个交易无效。

``` typescript
const signTxIn = (transaction: Transaction, txInIndex: number,
                  privateKey: string, aUnspentTxOuts: UnspentTxOut[]): string => {
    const txIn: TxIn = transaction.txIns[txInIndex];
    const dataToSign = transaction.id;
    const referencedUnspentTxOut: UnspentTxOut = findUnspentTxOut(txIn.txOutId, txIn.txOutIndex, aUnspentTxOuts);
    const referencedAddress = referencedUnspentTxOut.address;
    const key = ec.keyFromPrivate(privateKey, 'hex');
    const signature: string = toHexString(key.sign(dataToSign).toDER());
    return signature;
};
```
这里我们看下如果发生攻击时，交易签名是如何对我们的系统进行保护的：

- 攻击者节点收到一个交易广播：从AAA地址中发送10个币到BBB,交易id为0x555...
- 攻击者随后将接收者地址修改成CCC, 然后把这个交易继续发送到网络中。现在这个广播就内容就会变成：从AAA地址中发送10个币到CCC
- 但是，因为交易数据中的接收者地址被修改了，对应的交易id就不再有效。新的交易id应该会改变，比如变成0x567...。 当其他用户收到该交易时，首先会验证交易id，立即就会发现这个数据被篡改了。
- 如果攻击者修改接收地址的同时，把交易id也修改了呢？因为AAA只是对交易原始id 0x555...进行签名，并没有用私钥对0x567...进行签名， 其他节点验证时即能识破
- 所以最终该篡改的交易都不会被其他节点接受，因为无论怎么修改，它都是无效的。

### 未消费的交易outputs
一笔交易中，发起者必须在input中指定还没有被消费的交易outut。 你在区块链中拥有的加密货币， 指的其实就是在**未消费交易outputs**中，接受者地址为自己的公钥的一系列outputs。

当对交易进行有效性验证时，我们只需要关注**未消费交易outputs**这份清单。当然，这份独立的清单也可以从我们的区块链中演绎出来。这样子实现的话，当我们把交易纳入到区块链中时，我们将同时也会更新 未消费交易outputs这份清单。

**未消费交易output**的数据结构大致如下所示:

``` typescript
class UnspentTxOut {
    public readonly txOutId: string;
    public readonly txOutIndex: number;
    public readonly address: string;
    public readonly amount: number;

    constructor(txOutId: string, txOutIndex: number, address: string, amount: number) {
        this.txOutId = txOutId;
        this.txOutIndex = txOutIndex;
        this.address = address;
        this.amount = amount;
    }
}
```

整一个清单其实就是一个列表:

``` typescript
let unspentTxOuts: UnspentTxOut[] = [];
```

### 未消费交易outputs更新
每当一个新的区块加入到区块链中，我们都必须对我们的未消费outputs进行更新。 因为新的区块将可能会消费掉未消费outputs中的一些outputs，并肯定会引入新的outputs。

为了对此进行处理，我们需要加新区块加入时，将区块中的未消费交易outputs给解析出来：

``` typescript
 const newUnspentTxOuts: UnspentTxOut[] = newTransactions
        .map((t) => {
            return t.txOuts.map((txOut, index) => new UnspentTxOut(t.id, index, txOut.address, txOut.amount));
        })
        .reduce((a, b) => a.concat(b), []);
```

同时，我们还要找出这个新增区块将会消耗掉哪些交易outputs。我们通过检验交易数据的inputs下的项，即可以将这些数据找出来:

``` typescript
const consumedTxOuts: UnspentTxOut[] = newTransactions
        .map((t) => t.txIns)
        .reduce((a, b) => a.concat(b), [])
        .map((txIn) => new UnspentTxOut(txIn.txOutId, txIn.txOutIndex, '', 0));

```

最终我们通过删除已经消费的并且加上新的未消费的，从而产生了新的未消费交易outputs，具体代码如下：

``` typescript
const resultingUnspentTxOuts = aUnspentTxOuts
        .filter(((uTxO) => !findUnspentTxOut(uTxO.txOutId, uTxO.txOutIndex, consumedTxOuts)))
        .concat(newUnspentTxOuts);
```

以上代码片段都是在updateUnspentTxOuts这个方法中实现的。 需要注意的是，这个方法只有在区块含有的交易数据(以及这个块本身)都被验证没有问题的时候才会被调用。


### 交易有效性验证



















