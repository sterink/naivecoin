# 钱包
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












