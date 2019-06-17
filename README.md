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

### 交易接收者
交易接收者(txOut)结构由地址和数量两个成员变量组成。 数量代表了交易的数量。地址就是一个ECDSA的公钥，代表了接收者。这就意味着拥有对应地址的私钥的用户才能访问对应的加密货币。

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

### 交易发起者
交易发起者(txIn)结构提供了加密货币的来源信息。每一个txIn都会通过txOutId指向此前的一次交易，本次交易的加密货币就是从此前的该次交易解锁出来的，解锁后的货币就能在txOut中被发送给接收者了。结构中的signature这个属性则是发送者用自己私钥进行加密的签名，通过用指向的前一次交易中txOut中的地址(即公钥)对该签名进行验证，即能证明该发送者是该交易的拥有者。

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







































