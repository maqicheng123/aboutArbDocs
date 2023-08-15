
# Arbitrum Rollup Protocol

### 目标 

解决以太坊网络上的拥堵和高费用问题

### 实现
通过将交易批量处理并将其提交到以太坊主链之前进行验证，从而大大提高了交易吞吐量，并降低了交易费用（在layer2上运行所有交易，改变layer2状态，并将layer2全局的Merkle根更新至layer1合约中）

### 应用
Arbitrum One

## 成员角色
+ Sequencer(layer2):  是一个专门指定的全节点，被赋予有限的权力来控制交易的排序（offchain运行）
+ Validator(layer2):
    active validator: 提出新的 RBlock 来推进链的状态,并且活跃的验证者总是需要质押（需白名单）
    defensive validator: 监视rollup协议的运行情况。提出正确的RBlocks，则此不会触发防御。但是，如果提出了不正确的RBlock，则此类节点会通过发布正确的RBlock或押注另一方发布的正确RBlock来进行干预。（需白名单）
    watchtower validator: 不进行质押，它只是监视汇总协议，如果提出了不正确的RBlock，它会发出警报，以便其他人可以干预
+ Inbox (layer1)：
    + sequencer inbox: 接受sequencer节点发送的批次交易数据
    + delayed inbox: 接收User提供的layer2交易数据，等待sequencer节点将其推送至sequencer inbox
+ Outbox (layer1)：接收User layer2->layer1交易数据    
+ Rollup & Challenge Contract(layer1)：用于处理layer2状态根的提交和挑战
+ Bridge Contract(layer1): 传入和传出消息的暂存地

## Layer2 交易打包流程
    
描述：

    1.用户将 L2 交易发送到Sequencer节点。
    2.一旦排序器收到足够的交易，它将它们作为批处理发布到 L1 智能合约中。(brotli压缩)
    3.validator将从 L1 智能合约读取这些交易，并在 L2 状态的本地副本上处理它们。
    4.处理后，将在validator本地生成新的 L2 状态，validator会将此新状态根发布到 L1 智能合约中。
    5.然后，所有其他validator将在其 L2 状态的本地副本上处理相同的交易。
    6.他们将结果的 L2 状态根与发布到 L1 智能合约的原始状态根进行比较。
    7.如果其中一个验证者获得的状态根与发布到 L1 的状态根不同，他们将自己的状态根提交并在 L1 上开始挑战。
    8.挑战将要求挑战者和发布原始状态根的validator轮流证明正确的状态根应该是什么。
    9.无论哪个用户输掉挑战，他们的初始存款（本金）都会被削减。如果发布的原始 L2 状态根无效，它将被未来的验证者销毁，并且不会包含在 L2 链中

### 示意图
![](/image/0.png)





## 防止Sequencer节点作恶：DelayInbox
行为良好的 Sequencer 将接受来自所有请求者的交易并公平对待它们，从而尽快为每个请求者提供承诺的交易结果。

恶意序列器可能会造成问题。恶意 Sequencer 具有强大的能力对交易进行排序，来提前运行每个人的交易，因此它可以为指定用户为代价获得巨大利润。或者，可以一直不排序某些用户的交易，从而使他们的交易永远不会被处理。

如果它拒绝处理您的交易，可以通过延迟的收件箱来打包交易。延迟收件箱队列中的消息将等待Sequencer将它们“释放”到主收件箱中，在那里它们将被添加到收件箱的末尾。行为良好的 Sequencer 通常会在大约 10 分钟后发布延迟消息，但是如果 Sequencer 作恶，则还是可能永远不会发布延迟消息。

此时消息如果在延迟收件箱队列中的时间超过最大延迟间隔（当前为 24 小时），则任何人都可以强制将其提升到主收件箱中。（这可确保排序器只能延迟，而不能彻底拒绝）

```solidity


//将交易提交至延迟收件箱
//bridge.sol
function enqueueDelayedMessage(
        uint8 kind,
        address sender,
        bytes32 messageDataHash
    ) external payable returns (uint256) {}

//sequencer未作恶情况 ：sequencer在将批次交易数据提交时，会从bridge合约中读取delayedInbox队列的数据，并将其排在批次数据队尾
//sequencer.sol
function addSequencerL2Batch(                                       
    uint256 sequenceNumber,
    bytes calldata data,
    uint256 afterDelayedMessagesRead,  
    IGasRefunder gasRefunder,           
    uint256 prevMessageCount,
    uint256 newMessageCount
) external override refundsGas(gasRefunder) {
    ....
    acc = keccak256(abi.encodePacked(beforeAcc, dataHash, delayedAcc)); 
    sequencerInboxAccs.push(acc); //将delayedMessage hash 推到批次数据队尾
    ....
}

//sequencer 作恶的情况 ：sequencer在提交批次数据的时候，不收集delayedInbox队列的数据，24小时后可以调用forceInclusion将delayedMessage强制将数据推到批次数据队尾
//sequencerInbox.sol   
 function forceInclusion(
        uint256 _totalDelayedMessagesRead,
        uint8 kind,
        uint64[2] calldata l1BlockAndTime,
        uint256 baseFeeL1,
        address sender,
        bytes32 messageDataHash
    ) external;
```
![](/image/8.png)


## 跨链消息传递：
 
### L1->L2
eth deposit
1.延迟
2.可重试

### L2->L1
eth withdraw
1.立即
2.可重试

---



## 挑战机制
Arbitrum是乐观的，这意味着Arbitrum通过让任何一方（“验证者”）在第1层发布该方声称正确的RBlock，然后给其他人一个挑战该主张的机会来推进其链的状态。如果质询期（大约一周）过去了，并且没有人质疑声称的RBlock，则 Arbitrum 确认RBlock正确。如果有人在质疑期间对索赔提出质疑，那么Arbitrum会使用有效的争议解决协议（来识别哪一方在撒谎。说谎者将没收押金，说真话的人将收取部分押金作为对他们努力的奖励（一些存款被烧毁，保证即使有一些串通，骗子也会受到惩罚）

### 概念提要

 RBlock : 验证者可以提议 RBlock。新的 RBlock最初将无法解决。最终每个 RBlock 都将通过被确认或被拒绝来解决。已确认的 RBlock 构成了该链已确认的历史。
>每个 RBlock 包含：
>+ R块编号
>+ 前任 RBlock 编号：在此之前（声称）正确的最后一个 RBlock 的 RBlock 编号
>+ 链历史中已创建的 L2 区块数量
>+ 链历史中已消耗的收件箱消息数量
>+ 链历史上产生的输出的哈希值。


Staking: 质押者存入由 Arbitrum 第 1 层合约持有的资金，如果质押者输掉挑战，将被没收。

单个质押可以覆盖一系列RBlocks。每个质押者都押在最新确认的RBlock上;如果您押注在 RBlock 上，您也可以质押该 RBlock 的一个继任者。因此，您可能会被押注在一系列 RBlock 上，这些 RBlock 代表关于链正确历史的单个连贯声明。一个质押就足以让你承诺遵守这一系列的RBlocks。

```solidity
//创建一个新节点并将权益移至其上 （validator提交layer2新状态数据）
function stakeOnNewNode(
    Assertion calldata assertion,
    bytes32 expectedNodeHash,
    uint256 prevNodeInboxMaxCount
) public onlyValidator whenNotPaused {}


//将权益移至现有子节点  (validator同意提交的layer2新状态数据)
function stakeOnExistingNode(uint64 nodeNum, bytes32 nodeHash) public onlyValidator whenNotPaused{}
```

![](/image/5.webp)
### 挑战流程
![](/image/2.webp)



    



# Anytrust protocol 

### 目标
通过接受温和的信任假设来进一步降低成本。

### 实现
AnyTrust依靠外部数据可用性委员会（以下简称“委员会”）来存储数据并按需提供数据。委员会有N名成员，AnyTrust假设其中至少有两名是诚实的。这意味着，如果N-1委员会成员承诺提供对某些数据的访问，则至少有一方必须诚实，确保数据可用，以便汇总协议可以正常运行。

### 应用
Arbitrum Nova
### 成员角色
Sequencer : 与Arbitrum Rollup中的Sequencer类似相同，负责将交易排序，但是将数据不是直接提交至layer1，而是提交至委员会
Data Availability Committee :数据委员会则用来存储layer2数据的节点组成的委员会，至少有两名成员是诚实的
Data Availability certificates(DACert):批量数据组成的证书, 用来证明数据委员会已经存储了layer2数据
Availability Servers:提供REST API接口，允许通过哈希获取数据块。
Keysets:密钥集指定委员会成员的公钥以及数据可用性证书有效所需的签名数.包含：委员会成员人数、每个委员会成员的BLS 公钥、所需的委员会签名数。L1 密钥集管理器合约维护当前有效的密钥集列表，并由L2 链的所有者可以在此列表中添加或删除密钥集

### Layer2 交易打包流程
1. 用户将 L2 交易发送到Sequencer节点。
2. Sequencer通过 RPC 将该批次的数据以及到期时间（通常在未来三周）并行发送给所有委员会成员。
3. 每个委员会成员将数据存储在其后备存储中，并按数据的哈希进行索引.
4. 每个委员会成员使用其 BLS 密钥对（哈希、过期时间）进行签名，并将带有成功指示器的签名返回给排序器。
5. 排序器收集到足够的签名后，它可以聚合签名并为（哈希、过期时间）对创建有效的 DACert
6. Sequencer 将该 DACert 发布到 L1 收件箱合约，使其可供 L2 的 AnyTrust 链软件使用。

### 示意图
![](/image/10.png)
### 回退至Arbitrum Rollup
Sequencer 未能在几分钟内收集到足够的签名，它将放弃使用委员会的尝试，并将通过将完整数据直接发布到 L1 链来“fall back to rollup”。

### 与Arbitrum Rollup的区别
Arbitrum Rollup通过在 L1 以太坊上将数据（以批处理、压缩形式）作为调用数据发布来提供数据访问。为此支付的以太坊gas是Arbitrum成本的最大组成部分。

AnyTrust依靠外部数据可用性委员会来存储数据并按需提供数，据显着的成本节约。

---

# Orbit Chain
### 介绍
+ Arbitrum Orbit允许您创建自己的专用链，该链连接到Arbitrum的第2层（L2）链之一：Arbitrum One，Arbitrum Nova或Arbitrum Goerli。

+ 您拥有自己的 Orbit 链，可以自定义其隐私、权限、费用代币、治理等。

+ 允许您不断集成对 Arbitrum Nitro （为 Arbitrum 的 L2 和 Orbit 链的节点提供支持的代码）所做的改进。

### 架构图
![](/image/6.jpg)




## note :   
+ RBlock确认后如何将RBlock stateRoot更新到L1 

>>rollup contract stakeNewNode function


+ 交互式证明中如何使用
 >>挑战者（validator）使用本地数据初始化WASM虚拟机，然后与Rollup合约进行交互。然后使用WASM虚拟机执行交互式证明，证明结果与挑战者本地数据进行比较，如果不一致则挑战成功，否则挑战失败,WASM将RBlock操作拆分至一个op code,并在L1 rollup中完成证明操作

Sequencer还在L1以太坊链上发布其序列。Sequencer 会定期（在生产中每隔几分钟）连接提要中的下一组交易，压缩它们以提高效率，并将结果作为调用数据发布到以太坊上。这是交易序列的最终和官方记录。一旦此以太坊交易在以太坊上具有最终性，它记录的第 2 层 Nitro 交易将具有最终性。这些交易是最终的，因为它们在序列中的位置具有最终性，并且交易的结果是确定性的，任何一方都是可知的。这就是“硬结局”。

## 参考资料:
https://docs.arbitrum.io/inside-arbitrum-nitro/
