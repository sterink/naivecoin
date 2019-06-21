
# 第六章 钱包管理界面和区块链浏览器

- [概览](#概览)
- [增加新接口](#增加新接口)
- [前端技术选型](#前端技术选型)
- [区块链浏览器](#区块链浏览器)
- [钱包管理界面](#钱包管理界面)
- [体验](#体验)
- [小结](#小结)
  
### 概览

本章节我们将为我们的区块链实现一个钱包管理界面和一个区块链浏览器。 我们的后台现在已经提供了不少的一些HTTP接口(Restful API)，我们的UI实现，其实只需要向后台提供的接口发送请求并把结果可视化就行了。

未达成这个，我们还需要额外增加一些接口和逻辑，比如:

- 查询区块和交易信息的接口
- 查询指定钱包地址的信息的接口

### 增加新接口

首先我们增加一个可以根据已知区块哈希值查询指定区块的接口:

``` typescript
app.get('/block/:hash', (req, res) => {
        const block = _.find(getBlockchain(), {'hash' : req.params.hash});
        res.send(block);
    }); 
```

然后增加一个根据交易id查询对应交易的接口：

``` typescript
app.get('/transaction/:id', (req, res) => {
        const tx = _(getBlockchain())
            .map((blocks) => blocks.data)
            .flatten()
            .find({'id': req.params.id});
        res.send(tx);
    });
```

我们也许还想根据某个钱包的地址去查看钱包相应的信息。 我们可以将属于该地址的「未消费交易outputs」给全部列出来，然后UI逻辑就可以根据该结果来计算该钱包余额了。

``` typescript
app.get('/address/:address', (req, res) => {
        const unspentTxOuts: UnspentTxOut[] =
            _.filter(getUnspentTxOuts(), (uTxO) => uTxO.address === req.params.address)
        res.send({'unspentTxOuts': unspentTxOuts});
    });
````

我们也可以根据指定地址来计算其已消费交易outputs来将该地址下的消费历史记录呈现出来。


### 前端技术选型

我们的前端使用vue.js来实现。因为前端实现并不是我们的重点，所以我们这里不会像此前介绍区块链实现一样把前端的实现代码和逻辑都走一遍。这里只做简单的介绍。代码量不多，也很简单，大家可以在项目的ui文件夹下找到。

### 区块链浏览器

我们可以通过区块链浏览器来查看我们的区块链的状态。典型的使用场景可能会是，比如你想快速简便的查看指定钱包地址的余额，或者去验证一笔交易是否在该区块中已经记账。

我们需要做的仅仅是向节点后台(默认3001端口)发送http请求，然后获得对应的返回结果，然后友好的展示呈现出来给大家。我们不会去实现任何会修改区块链状态的功能，因此，打造这样的一个区块链浏览器要做的仅仅是去服务器把数据取出来，然后通过界面友好的呈现出来而已。

下图就是区块链浏览器运行后的一个示例:


![](https://lhartikk.github.io/assets/explorer_ui.png)


### 钱包管理界面

我们也会为钱包管理实现相应的页面功能。用户通过该管理界面可以将加密货币发送给别人，也可以查看对应钱包地址的余额。同时，我们还会把交易池的信息给呈现出来。

下图就是钱包管理界面运行后的一个示例:

![](https://lhartikk.github.io/assets/wallet_ui.png)


### 体验

#### 安装和启动后台节点

首先，和之前一样，我们先按顺序启动节点1和节点2。

打开一个终端，进入到项目目录，运行:

``` shell
npm install
npm run node1
```

然后打开第二个终端，启动第二个节点:

``` shell
npm install
npm run node2
```

#### 安装和启动区块链浏览器
启动完后台节点后，我们开始启动区块链浏览器。 首先打开第三个终端并定位到项目目录，然后进入到ui这个文件夹下的***naivecoin-explorer***这个文件夹，执行以下命令后，区块链浏览器将运行在8081端口:

``` shell
npm install
npm start
```
然后浏览器(chrome)中访问http://localhost:8081 就能正常访问区块链浏览器了。

#### 安装和启动钱包管理界面
和启动区块链浏览器类似， 首先打开第四个终端并进入项目目录下的***ui/naivecoin-wallet***这个目录， 然后执行:

``` shell
npm install
npm start
```
然后通过浏览器输入http://localhost:8082 就能正常访问钱包管理界面了。


### 小结

本来这两个ui的代码是在另外一个github仓库中的，但是为了方便自己，也为了方便大家，所以将其放到我们区块链项目下的ui文件夹去了。一开始两个管理界面使用的端口都是8081，不能同时启动，所以将其中一个改成8082。

本章节的代码实现请看[这里](https://github.com/zhubaitian/naivecoin/tree/chapter6)














