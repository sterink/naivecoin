# 第五章 未决交易

- [概览](#概览)
- [交易池](#交易池)
- [广播](#广播)
- [未决交易有效性验证](#未决交易有效性验证)
- [记账](#记账)
- [更新交易池](#更新交易池)
- [体验](#体验)
- [小结](#小结)
    
### 概览

上一章节中，我们要给一笔交易记账的话，必须自己手动进行一次挖矿，才会把交易记录加到一个区块里面去。 这一章节中，我们将会引入未决交易中继的机制。有了这个机制之后，我们要进行一笔交易的时候，就不需要自己动手挖矿，而是将自己的交易发送到我们的区块链网络中去(即中继传递的概念)，由其他节点在挖矿之后，将我们的交易记录加到他们挖出的新的区块中去。其中这些交易就被称之为「未决交易」。一个典型的例子就是，当一个用户想要发起一笔交易(把一定数量的币发送到指定的地址)，他会把这笔交易广播到整个网络，并希望其他矿工把该笔交易放到区块中去。

对于一个加密货币系统来说，这个功能异常的重要。因为这将意味着我们再也不需要因为要进行一笔交易而自己进行挖矿以对交易记录进行保存。这将大大提高效率。 毕竟，如比特币一样，随时着时间转移，挖一个矿是越来越难。 如果我们这些家里没矿(矿机)的用户想跟别人交易一些比特币，还要自己挖个矿，那这个交易就不知道何年何月才能达成了。

为了达到广播到其他节点并进行同步的目的，和第一章节进行区块广播和同步一样，我们需要对「未决交易」也要进行广播。 也就是说，我们的区块链系统现在会包含以下的广播和同步：

- 区块链 (即区块链及包含在区块中的「已决交易」记录)
- 未决交易 (还没有包含在任何区块中的交易)

本章节的完整代码请查看[这里](https://github.com/zhubaitian/naivecoin/tree/chapter5)

### 交易池

既然我们的交易现在不是在创建后立即由自己进行挖矿记录的，那么我们就需要将这些交易记录下来，才能广播给其他矿工。我们将保存未决交易的地方叫做「交易池」，在比特币系统中也叫做mempool。 

交易池其实就是我们节点中的一个存储了所有「未决交易」的数据结构，为了实现简单，我们这里将其设计成一个由「交易」数据组成的一个列表(如果不记得交易的结构长什么样的，请翻看上一章节记录):

``` typescript
let transactionPool: Transaction[] = [];
```

当然，为了和此前交易必须自己动手挖矿的那个/mindTransaction入口区分开，我们提供了另外一个接口仅作交易用 **POST /sendTransaction** 。 调用这个方法之后，我们的节点将基于此前的钱包机制在我们的本地交易池中创建一笔新的交易。此后我们都会将这个接口作为进行一笔交易优先使用的交易接口。

``` typescript
app.post('/sendTransaction', (req, res) => {
        ...
    })
```

创建一笔交易的流程和[第四章](https://github.com/zhubaitian/naivecoin/tree/chapter4)描述的流程类似，稍微有点区别的是，当交易创建后，我们会将该交易纳入到我们的交易池，而不再是立刻进行挖矿记录。

### 广播

整个「未决交易】的想法就是将这些交易传播到整个区块链网上，让一些节点最终将其“挖”到区块链中去。为了达成这个目标，我们首先要为这些未决交易的传播建立一些简单的网络规则：

- 当一个节点接收到别人广播出来的交易池时，发现里面有自己此前没有见过的「未决交易】记录时，该节点会将这些没见过的记录加入到自己的交易池中，然后将自己的交易池给广播出去
- 当一个节点和其他节点建立点对点连接时，会向目标节点请求对方保存的完整交易池记录

类似区块链的广播和同步，我们需要建立两个新的消息类型来为我们未决交易池的广播同步进行服务：**QUERY_TRANSACTION_POOL** 和 **RESPONSE_TRANSACTION_POOL** 。最终我们的消息类型数据结构将如下所示:

``` typescript
enum MessageType {
    QUERY_LATEST = 0,
    QUERY_ALL = 1,
    RESPONSE_BLOCKCHAIN = 2,
    QUERY_TRANSACTION_POOL = 3,
    RESPONSE_TRANSACTION_POOL = 4
}
```

交易池广播请求数据最终会通过以下方式进行创建:

``` typescript
const responseTransactionPoolMsg = (): Message => ({
    'type': MessageType.RESPONSE_TRANSACTION_POOL,
    'data': JSON.stringify(getTransactionPool())
}); 

const queryTransactionPoolMsg = (): Message => ({
    'type': MessageType.QUERY_TRANSACTION_POOL,
    'data': null
});
```

当节点收到**RESPONSE_TRANSACTION_POOL**这个广播后，我们需要有相应的代码逻辑来因应请求。无论何时我们收到广播过来的未决交易池，我们都会尝试将其加入到我们的本地交易池中去。当然，我们需要对里面的交易做相应的校验。 只有那些我们此前没有见过的交易(不在我们本地保存的那份交易池清单中)，以及有效的交易，我们才会将其纳入到我们的交易池中。紧跟着，如上所述，我们就会将我们的交易池给广播出去，以达到全网同步的效果:

``` typescript
case MessageType.RESPONSE_TRANSACTION_POOL:
    const receivedTransactions: Transaction[] = JSONToObject<Transaction[]>(message.data);
    receivedTransactions.forEach((transaction: Transaction) => {
        try {
            handleReceivedTransaction(transaction);
            //if no error is thrown, transaction was indeed added to the pool
            //let's broadcast transaction pool
            broadCastTransactionPool();
        } catch (e) {
            //unconfirmed transaction not valid (we probably already have it in our pool)
        }
    });
```

### 未决交易有效性验证

因为广播过来的未决交易数据是不可预知的，我们必须对这些号称是未决交易的数据进行有效性验证。我们此前实现的对交易的有效性检查依然会用上， 比如检查数据格式必须正确，交易的inputs，outputs和签名必须一致(即此前章节描述的validateTxIn函数，简单来说就是对每个交易中的每个input，用交易指向的来源output中的地址作为公钥，对input的签名进行解密，并检查和交易id是否一样，以验证input中引用的交易来源确实是属于用户所有。因为只有该公钥是该用户的才能正确解密本次交易的签名， 这就证明了来源交易的拥有者确实就是本次交易的发起者了)。

``` typescript
const validateTxIn = (txIn: TxIn, transaction: Transaction, aUnspentTxOuts: UnspentTxOut[]): boolean => {
    const referencedUTxOut: UnspentTxOut =
        aUnspentTxOuts.find((uTxO) => uTxO.txOutId === txIn.txOutId && uTxO.txOutIndex === txIn.txOutIndex);
    if (referencedUTxOut == null) {
        console.log('referenced txOut not found: ' + JSON.stringify(txIn));
        return false;
    }
    const address = referencedUTxOut.address;

    const key = ec.keyFromPublic(address, 'hex');
    const validSignature: boolean = key.verify(transaction.id, txIn.signature);
    if (!validSignature) {
        console.log('invalid txIn signature: %s txId: %s address: %s', txIn.signature, transaction.id, referencedUTxOut.address);
        return false;
    }
    return true;
};

```

除了以上已有规则外，我们还需要增加一条新规：如果一笔交易中的任何一个用来引用交易货币来源的input在当前的未决交易池中已经存在了，那么该交易将会被视为无效交易，不会纳入到交易池中去。

``` typescript
const isValidTxForPool = (tx: Transaction, aTtransactionPool: Transaction[]): boolean => {
    const txPoolIns: TxIn[] = getTxPoolIns(aTtransactionPool);

    const containsTxIn = (txIns: TxIn[], txIn: TxIn) => {
        return _.find(txPoolIns, (txPoolIn => {
            return txIn.txOutIndex === txPoolIn.txOutIndex && txIn.txOutId === txPoolIn.txOutId;
        }))
    };

    for (const txIn of tx.txIns) {
        if (containsTxIn(txPoolIns, txIn)) {
            console.log('txIn already found in the txPool');
            return false;
        }
    }
    return true;
};
```
这里没有显式定义将未决交易从交易池中移除的操作，当前做法是，每次网络中产生一个新区块时，将会同时去更新交易池。

### 记账

接下来我们要实现相应的逻辑，让一个节点将未决交易池记录到其新挖到的一个区块中去(正常情况下，帮忙记账的节点应该会收到奖励的，但是我们这里并没这样实现)。整个逻辑很简单：当一个节点挖到一个区块时，随即会将挖矿得到的原始交易和我们的交易池一并打包放到新挖出来的区块中去。

``` typescript
const generateNextBlock = () => {
    const coinbaseTx: Transaction = getCoinbaseTransaction(getPublicFromWallet(), getLatestBlock().index + 1);
    const blockData: Transaction[] = [coinbaseTx].concat(getTransactionPool());
    return generateRawNextBlock(blockData);
};
```

因为该交易池在此前接受到RESPONSE_TRANSACTION_POOL广播时已经做了验证，所以我们这里不需要进行任何有效性验证的工作。

### 更新交易池

一旦挖到一个新区块，我们的未决交易就有可能会随新区块一起被记账到区块链中，该未决交易也就变成已决交易。也就是说，每个新区块携带的交易都可能会导致我们的未决交易池不再有效。 比如以下情况:

- 交易池中的某笔未决交易被记账到新区块中(可能是发起交易的节点自己挖矿时记的，也有可能是其他节点记的)
- 未决交易中input所指向的交易货币来源，被发起者的其他交易给消费掉了

这些都会导致我们的交易池失效。交易池是失效了，但是，如我们前两章节阐述的，我们还有一份一直保持更新的「未消费交易outpus」清单，所以我们只需用该清单为基石，将交易池中所有的input不存在于「未消费交易outupts」中的项移除掉就行了。代码逻辑来在如下:

``` typescript
const updateTransactionPool = (unspentTxOuts: UnspentTxOut[]) => {
    const invalidTxs = [];
    for (const tx of transactionPool) {
        for (const txIn of tx.txIns) {
            if (!hasTxIn(txIn, unspentTxOuts)) {
                invalidTxs.push(tx);
                break;
            }
        }
    }
    if (invalidTxs.length > 0) {
        console.log('removing the following transactions from txPool: %s', JSON.stringify(invalidTxs));
        transactionPool = _.without(transactionPool, ...invalidTxs)
    }
};
```

### 体验

#### 启动两个节点(建议在两个命令行终端下)

``` shell
npm run node1
npm run node2
```

#### 查看节点钱包地址

通过postman分别发送请求到3001和3002两个端口的/address接入点去获取两个钱包的地址， 比如节点1: GET http://localhost:3001/address

``` json
{
    "address": "04d4d57026bd7b0d951b8d6c72ed9118004cd0929d10f94d7c41b24dbe9d84fa3bb389f2525c05a46bd8d1203b4b3c0e3499f30e5a55f84c573fcccd94c83bc13a"
}
```
#### 挖矿

节点2进行一次挖矿，获得50个币。 POST以下数据到http://localhost:3002/mineBlock 接口:

``` json
{
	"data": "Some data from node2"
}
```

返回如下:

``` json
{
    "index": 1,
    "previousHash": "91a73664bc84c0baa1fc75ea6e4aa6d1d20c5df664c724e3159aefc2e1186627",
    "timestamp": 1561004915,
    "data": [
        {
            "txIns": [
                {
                    "signature": "",
                    "txOutId": "",
                    "txOutIndex": 1
                }
            ],
            "txOuts": [
                {
                    "address": "04bdc45ca144a5e5c8d0b03443f9aedfc8260d4665ac3fd41bd9eb2f06e4dc8228be1e14a4938837cada904cc81fc5747930f480d4d7888b5e82aac6f0581be0df",
                    "amount": 50
                }
            ],
            "id": "46dc195f7d44c2c1687fbd67a9a40263fd508067f89d8c07ee0e14826394b1d5"
        }
    ],
    "hash": "ca7eff56caf46b90cd7230a7c5684e1c201da48b7293aa282efebca2fabf0792",
    "difficulty": 0,
    "nonce": 0
}
```

#### 查看未消费交易outputs

因为全局未消费交易outputs是全网同步的，所以无论哪个节点去查看，结果都是一样的。 这里以节点1为例, Get以下接口: http://localhost:3001/unspentTransactionOutputs

返回如下:

``` json
[
    {
        "txOutId": "e655f6a5f26dc9b4cac6e46f52336428287759cf81ef5ff10854f69d68f43fa3",
        "txOutIndex": 0,
        "address": "04bfcab8722991ae774db48f934ca79cfb7dd991229153b9f732ba5334aafcd8e7266e47076996b55a14bf9913ee3145ce0cfc1372ada8ada74bd287450313534a",
        "amount": 50
    },
    {
        "txOutId": "46dc195f7d44c2c1687fbd67a9a40263fd508067f89d8c07ee0e14826394b1d5",
        "txOutIndex": 0,
        "address": "04bdc45ca144a5e5c8d0b03443f9aedfc8260d4665ac3fd41bd9eb2f06e4dc8228be1e14a4938837cada904cc81fc5747930f480d4d7888b5e82aac6f0581be0df",
        "amount": 50
    }
]
```

可以看到节点2挖矿激励所得的id为“46dc195f7d44c2c1687fbd67a9a40263fd508067f89d8c07ee0e14826394b1d5”已经被放到未消费交易outputs中，另外一条交易是系统启动时自动创建原始交易产生的，不用管。


#### 查看节点自己拥有的未消费outputs

另外为了方便我们体验，系统还一共了另外一个接口来让我们查看对应节点所拥有的未消费outputs。比如我们可以发送 Get 到接口http://localhost:3002/myUnspentTransactionOutputs 去获取节点2的未消费outputs列表。 返回如下

``` json
[
    {
        "txOutId": "46dc195f7d44c2c1687fbd67a9a40263fd508067f89d8c07ee0e14826394b1d5",
        "txOutIndex": 0,
        "address": "04bdc45ca144a5e5c8d0b03443f9aedfc8260d4665ac3fd41bd9eb2f06e4dc8228be1e14a4938837cada904cc81fc5747930f480d4d7888b5e82aac6f0581be0df",
        "amount": 50
    }
]
```

#### 发起交易

节点2给节点1的钱包发送30个币。 POST以下交易数据到http://localhost:3002/sendTransaction 接口.

``` json
{
	"address": "04d4d57026bd7b0d951b8d6c72ed9118004cd0929d10f94d7c41b24dbe9d84fa3bb389f2525c05a46bd8d1203b4b3c0e3499f30e5a55f84c573fcccd94c83bc13a",
	"amount": 30
}
```

返回如下:

``` json
{
    "txIns": [
        {
            "txOutId": "46dc195f7d44c2c1687fbd67a9a40263fd508067f89d8c07ee0e14826394b1d5",
            "txOutIndex": 0,
            "signature": "30450220130a2d3283337bb3721b992538ba19bbe6561376d0ddc9572595c0af8fb487d502210095d7cfe26717204329e22a386c274ec45d09190f1089fc354ad726ba375d917b"
        }
    ],
    "txOuts": [
        {
            "address": "04d4d57026bd7b0d951b8d6c72ed9118004cd0929d10f94d7c41b24dbe9d84fa3bb389f2525c05a46bd8d1203b4b3c0e3499f30e5a55f84c573fcccd94c83bc13a",
            "amount": 30
        },
        {
            "address": "04bdc45ca144a5e5c8d0b03443f9aedfc8260d4665ac3fd41bd9eb2f06e4dc8228be1e14a4938837cada904cc81fc5747930f480d4d7888b5e82aac6f0581be0df",
            "amount": 20
        }
    ],
    "id": "a4089fb337943eb80f381f107bdc1583c9b282e6cf735751d976ebd2125c5183"
}
```

此时查看区块链和未消费outputs，我们可以看到这笔交易其实还没有被加入到区块链中，也没有被消费掉的。

#### 节点1挖矿并记账

这时如果任何一个节点进行挖矿，就会把节点2发起的交易给记录到新增的区块中去。 比如节点1进行挖矿， POST 以下数据到 http://localhost:3001/mineBlock

``` json
{
	"data": "Some data from node1"
}
```

从返回记录中可以看到，新增的区块交易记录中，除了有挖矿激励得到的50个币的记录，上面节点2发起的交易记录也被记账到区块中了

``` json
{
    "index": 2,
    "previousHash": "ca7eff56caf46b90cd7230a7c5684e1c201da48b7293aa282efebca2fabf0792",
    "timestamp": 1561007564,
    "data": [
        {
            "txIns": [
                {
                    "signature": "",
                    "txOutId": "",
                    "txOutIndex": 2
                }
            ],
            "txOuts": [
                {
                    "address": "04d4d57026bd7b0d951b8d6c72ed9118004cd0929d10f94d7c41b24dbe9d84fa3bb389f2525c05a46bd8d1203b4b3c0e3499f30e5a55f84c573fcccd94c83bc13a",
                    "amount": 50
                }
            ],
            "id": "2e668dcd87fb900f346bab921a8349bc0e39d560c5a07a727cc9484aff471303"
        },
        {
            "txIns": [
                {
                    "txOutId": "46dc195f7d44c2c1687fbd67a9a40263fd508067f89d8c07ee0e14826394b1d5",
                    "txOutIndex": 0,
                    "signature": "30450220130a2d3283337bb3721b992538ba19bbe6561376d0ddc9572595c0af8fb487d502210095d7cfe26717204329e22a386c274ec45d09190f1089fc354ad726ba375d917b"
                }
            ],
            "txOuts": [
                {
                    "address": "04d4d57026bd7b0d951b8d6c72ed9118004cd0929d10f94d7c41b24dbe9d84fa3bb389f2525c05a46bd8d1203b4b3c0e3499f30e5a55f84c573fcccd94c83bc13a",
                    "amount": 30
                },
                {
                    "address": "04bdc45ca144a5e5c8d0b03443f9aedfc8260d4665ac3fd41bd9eb2f06e4dc8228be1e14a4938837cada904cc81fc5747930f480d4d7888b5e82aac6f0581be0df",
                    "amount": 20
                }
            ],
            "id": "a4089fb337943eb80f381f107bdc1583c9b282e6cf735751d976ebd2125c5183"
        }
    ],
    "hash": "c245d2d2f9f1ec050b085fd2be39b3cd51e5af6bb53158297276d9b0b17c2b61",
    "difficulty": 0,
    "nonce": 0
}
```

#### 查看未决交易池

这时我们再查看未决交易池的话，就会发现该交易已经被更新移除掉了

``` json
[]
```


#### 查看节点2的未决交易outputs

这时如果再查看节点2的未决交易outputs，会发现自己就剩余20个币。

```json
[
    {
        "txOutId": "a4089fb337943eb80f381f107bdc1583c9b282e6cf735751d976ebd2125c5183",
        "txOutIndex": 1,
        "address": "04bdc45ca144a5e5c8d0b03443f9aedfc8260d4665ac3fd41bd9eb2f06e4dc8228be1e14a4938837cada904cc81fc5747930f480d4d7888b5e82aac6f0581be0df",
        "amount": 20
    }
]
```

### 小结

通过「未决交易池」机制的引入，我们现在不再需要自己挖矿才能进行一笔交易，其他节点也能帮我们进行记账。但，如前所述，对于帮助我们记账的节点并不会获得任何激励，因为我们的系统中没有去实现交易手续费的功能。

下一个章节我们将会实现一个UI界面来方便大家使用钱包和对区块链进行操作。

[第六章](https://github.com/zhubaitian/naivecoin/tree/chapter6)








