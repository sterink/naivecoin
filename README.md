# #1: 最小可行区块链

## 概览
区块链的基础概念非常简单, 说白了就是一个维护着一个持续增长的有序数据记录列表的这么一个分布式数据库。在此章节中我们将实现一个简单的玩具版的区块链。此章节结束时，我们的区块链将实现以下功能：
- 区块和区块链结构定义
- 将包含任意数据的新区块写入到区块链的方法
- 可以与其他运行节点沟通和同步区块链数据的运行节点
- 操作单个运行节点的简单HTTP API

## 区块数据结构
我们首先会从区块结构的定义开始。在当前阶段，我们只给每个区块定义最关键的属性。

- **index**: 区块在区块链中的高度(即序号)
- **data**: 任何需要包括在此区块中的数据
- **timestamp**: 时间戳
- **hash**: 根据 block 内容计算的 sha256 哈希值
- **previousHash**: 前一个区块的哈希值，此属性将显式指定上一个区块是谁

![](https://lhartikk.github.io/assets/blockchain.png)

相应代码大致如下:
```typescript

class Block {

    public index: number;
    public hash: string;
    public previousHash: string;
    public timestamp: number;
    public data: string;

    constructor(index: number, hash: string, previousHash: string, timestamp: number, data: string) {
        this.index = index;
        this.previousHash = previousHash;
        this.timestamp = timestamp;
        this.data = data;
        this.hash = hash;
    }
}
```

## 区块哈希
区块哈希值是区块中最重要的属性之一。哈希值根据区块中的所有数据计算而得，这意味着如果区块中任何属性发生变化，原有的哈希值就不再有效。区块哈希值也能被看成区块的唯一性标识。举例来说，有可能出现两个 index 一致的区块，但是他们总会有不一样的哈希值。
根据以下的代码来计算哈希值：

``` typescript
const calculateHash = (index: number, previousHash: string, timestamp: number, data: string): string =>
    CryptoJS.SHA256(index + previousHash + timestamp + data).toString();
```

需要注意的是，在这个阶段，区块的哈希值与挖矿没有任何关系，因为还未有 POW（工作量证明） 问题需要解决。我们使用区块哈希值来保证区块的完整性，同时也使用它显式的引用上一个区块。

由以上对 hash和 previousHash 属性的处理，可推导出区块链重要的特性，即区块的内容不能被修改，除非同时修改它后续的所有区块内容。
以下的例子描述了这个特性。如果将第44区块的数据从“DESERT”修改成“STREET”，所有后续区块的哈希值也必须被修改。这是由于区块的哈希值取决于其previousHash 的值（以及其它属性）

![](https://lhartikk.github.io/assets/Blockchain_integrity.png)

这个特性在工作量证明被引入时尤其重要。一个区块在区块链中的位置越深（越靠前），要修改它的难度就越大，因为需要同时修改它本身以及它后续的所有区块。

## 创世块
创世块是区块链中的第一个区块。它是唯一一个没有 previousHash 的区块，我们在代码里将创世区块硬编码：

``` typescript

const genesisBlock: Block = new Block(
    0, '816534932c2b7154836da6afc367695e6337db8a921823784c14378abed4f7d7', null, 1465154705, 'my genesis block!!'
);

```

## 创建区块
创建一个新的区块，需要获得上一个区块的哈希值，并创建其他必须的内容（ index, hash, data 和 timestamp)。区块的数据（data字段）由用户提供，其他的参数使用以下代码生成：

``` typescript

const generateNextBlock = (blockData: string) => {
    const previousBlock: Block = getLatestBlock();
    const nextIndex: number = previousBlock.index + 1;
    const nextTimestamp: number = new Date().getTime() / 1000;
    const nextHash: string = calculateHash(nextIndex, previousBlock.hash, nextTimestamp, blockData);
    const newBlock: Block = new Block(nextIndex, nextHash, previousBlock.hash, nextTimestamp, blockData);
    return newBlock;
};
```

## 保存区块链
目前我们使用 JavaScript 的数组，将区块链保存在程序的运行内存中。这意味着当一个运行节点停止时，该节点上的区块链数据不会被持久化。

``` typescript
const blockchain: Block[] = [genesisBlock];
```

## 验证区块完整性
为确保数据完整性，我们应有办法课随时可对一个区块，或者一条区块链上的区块进行有效性验证。特别是当我们的节点从其他运行节点中接收新的区块时，我们就需要验证区块的有效性，以便决定是否接受这些区块。

验证区块的有效性，需要满足以下所有条件：

- 区块的 index 需要比上一个区块大1；
- 区块的 previousHash 属性需要与上一个区块的 hash 属性一致；
- 区块自身的 hash 值需要有效。

以下代码描述了整个验证过程：

``` typescript

const isValidNewBlock = (newBlock: Block, previousBlock: Block) => {
    if (previousBlock.index + 1 !== newBlock.index) {
        console.log('invalid index');
        return false;
    } else if (previousBlock.hash !== newBlock.previousHash) {
        console.log('invalid previoushash');
        return false;
    } else if (calculateHashForBlock(newBlock) !== newBlock.hash) {
        console.log(typeof (newBlock.hash) + ' ' 
          + typeof calculateHashForBlock(newBlock));
        console.log('invalid hash: ' 
          + calculateHashForBlock(newBlock) + ' ' 
          + newBlock.hash);
        return false;
    }
    return true;
};

```

同时我们还必须验证该区块的结构是否正确，以避免其他节点发来的未正确格式的数据造成程序崩溃。

``` typescript
const isValidBlockStructure = (block: Block): boolean => {
    return typeof block.index === 'number'
        && typeof block.hash === 'string'
        && typeof block.previousHash === 'string'
        && typeof block.timestamp === 'number'
        && typeof block.data === 'string';
};
```

现在我们可以验证单个区块是否有效，让我们进一步对一条链上的区块做验证。首先验证链中的第一个区块为创世区块。然后，我们使用以上的方式来依次校验链中的下一个区块，以下为实现代码:

``` typescript
const isValidChain = (blockchainToValidate: Block[]): boolean => {
    const isValidGenesis = (block: Block): boolean => {
        return JSON.stringify(block) === JSON.stringify(genesisBlock);
    };

    if (!isValidGenesis(blockchainToValidate[0])) {
        return false;
    }

    for (let i = 1; i < blockchainToValidate.length; i++) {
        if (!isValidNewBlock(
            blockchainToValidate[i], blockchainToValidate[i - 1])) {
            return false;
        }
    }
    return true;
};

```

## 选择最长链
在任何时候，在区块链中都应该只存在一组区块。在冲突发生的情况下（如：两个运行节点都生成了第72区块， 但某个节点还同时广播生成了第73块），则从中选择包含更长区块的链。在以下的例子中，由于被更长的区块链复写，第72区块: a350235b00 中的数据将不会被包括在区块链中。

![](https://lhartikk.github.io/assets/conflict_resolving.png)

代码实现如下：

``` typescript
const replaceChain = (newBlocks: Block[]) => {
    if (isValidChain(newBlocks)
        && newBlocks.length > getBlockchain().length) {
        console.log('Received blockchain is valid. Replacing current blockchain with received blockchain');
        blockchain = newBlocks;
        broadcastLatest();
    } else {
        console.log('Received blockchain invalid');
    }
};
```

## 节点间通信
每个运行节点都必须能和其他节点分享和同步区块链数据。我们通过以下规则开保证节点间的同步：

- 当一个节点生成新的区块时，该节点会将此区块广播至网络中；
- 当一个节点连接到另一个节点时，该节点将会向另一个节点请求最新的区块信息；
- 当一个节点接收到新区块的广播，而该区块的 index 比节点中保留的最后一个区块大，则该节点要不就是将此区块加到自身的区块链中(index只大1的情况)，或者查询以获得整条链的数据(index比当前节点最后一个区块的index大超过1的情况)。

![](https://lhartikk.github.io/assets/p2p_communication.png)

在此项目中我们使用 WebSocket 技术来实现各个节点的点对点的通信。各个节点活跃的 socket 列表保存在 const sockets: WebSocket[] 变量中。项目没有实现节点发现机制，所以新增加一个节点后，需要手动添加需要连接到的目标节点的地址(WebSocket URLs)。

## 操作节点
用户需要能以某种方式来操作节点。我们将通过实现相应的http服务端来提供相应功能。

``` typescript

 const initHttpServer = ( myHttpPort: number ) => {
    const app = express();
    app.use(bodyParser.json());

    app.get('/blocks', (req, res) => {
        res.send(getBlockchain());
    });
    app.post('/mineBlock', (req, res) => {
        const newBlock: Block = generateNextBlock(req.body.data);
        res.send(newBlock);
    });
    app.get('/peers', (req, res) => {
        res.send(getSockets().map(( s: any ) => 
            s._socket.remoteAddress + ':' + s._socket.remotePort));
    });
    app.post('/addPeer', (req, res) => {
        connectToPeers(req.body.peer);
        res.send();
    });

    app.listen(myHttpPort, () => {
        console.log('Listening http on port: ' + myHttpPort);
    });
};

```

根据以上代码，用户可以通过以下方式操作节点：

- 列举所有区块
- 创建一个由用户指定内容的新区块
- 列举和新增节点

可以通过Curl工具来对节点进行操作，当然你也可以通过postman等工具来实现：

``` shell

#get all blocks from the node
> curl http://localhost:3001/blocks
```

## 架构
每个节点都对外暴露两个web 服务: 一个是用户来给用户对节点进行操作的(HTTP Server)，一个使用来实现节点间的通信的(Websocket HTTP server)。

![](https://lhartikk.github.io/assets/naivechain_architecture.png)

## 小结
到现在为止，我们实现了一个简单的玩具版的区块链。此外，本章节还为我们展示了如何用简单扼要的方法来实现区块链的一些基本原理。下一章节中我们将为naivecoin 中加入工作量证明机制。


# #2 运行测试

## 安装
```
npm install
npm start
```

## 获取区块链
```
curl http://localhost:3001/blocks
```

## 生成一个区块
```
curl -H "Content-type:application/json" --data '{"data" : "Some data to the first block"}' http://localhost:3001/mineBlock
``` 

## 连接到一个节点
```
curl -H "Content-type:application/json" --data '{"peer" : "ws://localhost:6001"}' http://localhost:3001/addPeer
```
## 查询连接的节点列表
```
curl http://localhost:3001/peers
```
