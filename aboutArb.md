
# Arbitrum Rollup Protocol

### 目标

解决以太坊网络上的拥堵和高费用问题

### 实现

通过将交易批量处理并将其提交到以太坊主链之前进行验证，从而大大提高了交易吞吐量，并降低了交易费用

### 应用

Arbitrum One

### 成员角色

+ Sequencer(layer2):  是一个专门指定的全节点，被赋予有限的权力来控制交易的排序，并将交易数据批量提交至layer1（offchain运行）
+ Validator(layer2):
   + RBlock : 验证者在本地执行sequencer提交批次数据后，将state Data交到layer2的Rollup合约，合约根据状态数据生成RBlock。
   + active validator: 提出新的 RBlock 来推进链的状态,并且活跃的验证者总是需要质押（需白名单）
   + defensive validator: 监视rollup协议的运行情况。提出正确的RBlocks，则此不会触发防御。但是，如果提出了不正确的RBlock，则此类节点会通过发布正确的RBlock或押注另一方发布的正确RBlock来进行干预。（需白名单）
   + watchtower validator: 不进行质押，它只是监视汇总协议，如果提出了不正确的RBlock，它会发出警报，以便其他人可以干预
+ Inbox (layer1)：
  + sequencer inbox: 接受sequencer节点发送的批次交易数据
  + delayed inbox: 接收任意User提供的layer2交易数据，等待sequencer节点将其推送至sequencer inbox
+ Outbox (layer1)：接收User layer2->layer1交易数据
+ Rollup & Challenge Contract(layer1)：接收validator提交的layer2状态数据，并提供挑战入口
+ Bridge Contract(layer1): 传入和传出消息的暂存地（包含延迟消息队列 delayedInboxAccs，排序器消息队列sequencerInboxAccs）

RBlock 示意图

![](/image/9.webp)

### Layer2 交易打包流程

描述：

    1.用户将 L2 交易发送到Sequencer节点。
    2.一旦排序器收到足够的交易，它将它们作为批处理发布到 L1 智能合约中。(brotli压缩)
    3.validator将从 L1 智能合约读取这些交易，并在 L2 状态的本地副本上处理它们。
    4.处理后，将在validator本地生成新的 L2 状态，validator会将此新状态根发布到 L1 智能合约中。
    5.然后，其他validator将在其 L2 状态的本地副本上处理相同的交易。
    6.他们将结果的 L2 状态根与发布到 L1 智能合约的原始状态根进行比较。
    7.如果其中一个验证者获得的状态根与发布到 L1 的状态根不同，他们将自己的状态根提交并在 L1 上开始挑战。
    8.挑战将要求挑战者和发布原始状态根的validator轮流证明正确的状态根应该是什么。
    9.无论哪个用户输掉挑战，他们的初始存款（本金）都会被削减。如果发布的原始 L2 状态根无效，它将被未来的验证者销毁，并且不会包含在 L2 链中

### 示意图

![](/image/0.png)

### DelayInbox : layer2数据打包的另一个方案

行为良好的 Sequencer 将接受来自所有请求者的交易并公平对待它们，从而尽快为每个请求者提供承诺的交易结果。

恶意序列器可能会造成问题。 Sequencer有对交易进行排序的权利，可以通过提前运行某些交易，来为指定用户获得巨大利润。或者，可以一直不排序某些用户的交易，从而使他们的交易永远不会被处理。

如果它拒绝处理您的交易，可以通过DelayInbox来打包交易。延迟收件箱队列中的消息将等待Sequencer将它们“释放”到主收件箱中，在那里它们将被添加到收件箱的末尾。行为良好的 Sequencer 通常会在大约 10 分钟后发布延迟消息，但是如果 Sequencer 作恶，则还是可能不会发布消息。

此时消息如果在延迟收件箱队列中的时间超过最大延迟间隔（当前为 24 小时），则任何人都可以强制将其提升到主收件箱中。（这可确保排序器只能延迟，而不能审查拒绝）

```solidity
//将交易提交至延迟收件箱

//使用bridge合约中的enqueueDelayedMessage函数将交易提交至延迟队列
//bridge.sol
function enqueueDelayedMessage(
        uint8 kind,
        address sender,
        bytes32 messageDataHash
    ) external payable returns (uint256) {
        ...
        delayedInboxAccs.push(Messages.accumulateInboxMessage(prevAcc, messageHash));
         ...
    }

//sequencer在将批次交易数据提交时，会从bridge合约中读取delayedInbox队列的数据，并将其排在批次数据队尾
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

//sequencer 下线：24小时后可以调用forceInclusion将delayedMessage强制将数据推到批次数据队尾
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
### 示意图
![](/image/8.png)

## 跨链消息传递

### Layer1 -> Layer2 :
 本质为用户根据需求调用Inbox.sol函数，并通过bridge.sol将消息放入延迟队列中，然后通过sequencer将其放入主队列中，等待validator对交易的处理，使得消息在layer2上生效

消息类型

```solidity
uint8 constant L2_MSG = 3;
uint8 constant L1MessageType_L2FundedByL1 = 7;
uint8 constant L1MessageType_submitRetryableTx = 9;
uint8 constant L1MessageType_ethDeposit = 12;
uint8 constant L1MessageType_batchPostingReport = 13;
uint8 constant L2MessageType_unsignedEOATx = 0;
uint8 constant L2MessageType_unsignedContractTx = 1;
```

核心逻辑

 ```solidity
    // Inbox.sol
    //---------------------core----------------------
     function _deliverMessage(
        uint8 _kind,
        address _sender,
        bytes memory _messageData
    ) internal returns (uint256) {
        if (_messageData.length > MAX_DATA_SIZE)
            revert DataTooLarge(_messageData.length, MAX_DATA_SIZE);
        uint256 msgNum = deliverToBridge(_kind, _sender, keccak256(_messageData));
        emit InboxMessageDelivered(msgNum, _messageData);
        return msgNum;
    }

    function deliverToBridge(
        uint8 kind,
        address sender,
        bytes32 messageDataHash
    ) internal returns (uint256) {
        return
            bridge.enqueueDelayedMessage{value: msg.value}(
                kind,
                AddressAliasHelper.applyL1ToL2Alias(sender),
                messageDataHash
            );
    }
    //---------------------core----------------------


    //具体业务函数
    function depositEth() public payable whenNotPaused onlyAllowed returns (uint256) {
        ...
        return _deliverMessage(
                L1MessageType_ethDeposit,
                msg.sender,
                abi.encodePacked(dest, msg.value)
            );
    }

    function sendWithdrawEthToFork(
        uint256 gasLimit,
        uint256 maxFeePerGas,
        uint256 nonce,
        uint256 value,
        address withdrawTo
    ) external whenNotPaused onlyAllowed returns (uint256) {
        ...
        return
            _deliverMessage(
                L2_MSG,
                ...
            );
    }

    function sendL2Message(bytes calldata messageData)
        external
        whenNotPaused
        onlyAllowed
        returns (uint256)
    {
       ...
       return _deliverMessage(L2_MSG, msg.sender, messageData);
    }

 ```
### Layer2 -> Layer1 :
L2 到 L1 消息以调用 L2 ArbSys 预编译协定的方法 sendTxToL1 开始。一旦消息包含在断言中（通常在 ~1 小时内）并且断言得到确认（通常大约 ~ 1 周），任何客户端都可以执行该消息。为此，客户端首先通过调用 Arbitrum 链的“虚拟”/预编译式 NodeInterface **合约的方法 constructOutboxProof 检索证明数据。然后，可以在 Outbox 的方法 executeTransaction 中使用返回的数据来执行 L1 执行。

```solidity
//layer2预编合约中调用函数来发送layer2->layer1消息
//ArbSys.sol
function sendTxToL1(address destination, bytes calldata data)
        external
        payable
        returns (uint256);

rollup layer2 交易打包流程...

//等待交易确认后，调用layer1的outbox合约的executeTransaction函数在layer1上执行交易
//Outbox.sol
 function executeTransaction(
        bytes32[] calldata proof,
        uint256 index,
        address l2Sender,
        address to,
        uint256 l2Block,
        uint256 l1Block,
        uint256 l2Timestamp,
        uint256 value,
        bytes calldata data
    ) external {
        ....
    }

```

 

---

## optimistic rollup 挑战机制

Arbitrum是乐观的，这意味着Arbitrum通过让任何一方（“验证者”）在第1层发布该方声称正确的RBlock，然后给其他人一个挑战该主张的机会来推进其链的状态。如果质询期（大约一周）过去了，并且没有人质疑声称的RBlock，则 Arbitrum 确认RBlock正确。如果有人在质疑期间对索赔提出质疑，那么Arbitrum会使用有效的争议解决协议（来识别哪一方在撒谎。说谎者将没收押金，说真话的人将收取部分押金作为对他们努力的奖励（一些存款被烧毁，保证即使有一些串通，骗子也会受到惩罚）


### 交互式欺诈证明

争议起因：

在sequencer将数据提交至layer1后，会触发事件 emit SequencerBatchData(seqMessageIndex, data) , 出块validator通过监听事件将该批次的数据在本地状态机器中执行，并调用rollup合约中的stakeOnNewNode函数，将状态根提交到rollup合约中。其他验证者节点watch到事件后，将相同的批次数据进行本地执行。如果出现与原始提交的状态根不同，也可以调用stakeOnNewNode函数将状态根提交至rollup合约。

如图所示：

![](/image/5.webp)


解决争议：

挑战者使用本地数据初始化WAVM状态(WASM编译的geth)，并调用challenger合约中的createChallenge函数来创建挑战，并提交机器状态根和全局状态根。
challengerManager合约会将挑战者的机器状态二分，并记录当前挑战的分段情况。

被挑战者使用本地数据初始化WAVM状态(WASM编译的geth)，调用bisectExecution函数，选择其中一个挑战者的分段，
并执行该段操作，并将自己的机器状态进行二分，并更新挑战的分段情况。

.....

当分段到只有一个op code时,挑战者使用本地机器生成状态proof，并挑战者调用oneStepProveExecution，在challenger合约中，根据挑战提供的merkle根的proof生成一个机器对象，并使用该对象来执行最后一步有争议段上的操作。根据执行后的mach.hash()，来决定挑战者是否胜利。

```solidity
    bytes32 afterHash = osp.proveOneStep(
            ExecutionContext({maxInboxMessagesRead: challenge.maxInboxMessages, bridge: bridge}),
            challengeStart,
            selection.oldSegments[selection.challengePosition],
            proof
    );

    //若不同，则挑战者胜利，否则被挑战者胜利
    require(
            afterHash != selection.oldSegments[selection.challengePosition + 1],
            "SAME_OSP_END"
    );
```


如图所示：

宏观：

![](/image/7.png)

微观：

![](/image/2.webp)




# Anytrust protocol

### 目标

通过接受温和的信任假设来进一步降低成本。

### 实现

AnyTrust依靠外部数据可用性委员会（以下简称“委员会”）来存储数据并按需提供数据。委员会有N名成员，AnyTrust假设其中至少有两名是诚实的。这意味着，如果N-1委员会成员承诺提供对某些数据的访问，则至少有一方必须诚实，确保数据可用，以便汇总协议可以正常运行。

### 应用

Arbitrum Nova

### 成员角色

+ Sequencer : 与Arbitrum Rollup中的Sequencer类似相同，负责将交易排序，但是将数据不是直接提交至layer1，而是提交至委员会
+ Data Availability Committee :数据委员会则用来存储layer2数据的节点组成的委员会，至少有两名成员是诚实的
+ Data Availability certificates(DACert):批量数据组成的证书, 用来证明数据委员会已经存储了layer2数据
+ Availability Servers:提供REST API接口，允许通过哈希获取数据块。
+ Keysets:密钥集指定委员会成员的公钥以及数据可用性证书有效所需的签名数.包含：委员会成员人数、每个委员会成员的BLS 公钥、所需的委员会签名数。L1 密钥集管理器合约维护当前有效的密钥集列表，并由L2 链的所有者可以在此列表中添加或删除密钥集

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

AnyTrust依靠外部数据可用性委员会来存储数据并按需提供数据，据显着的成本节约。

---

# Orbit Chain

### 介绍

+ 允许您不断集成对 Arbitrum Nitro （为 Arbitrum 的 L2 和 Orbit 链的节点提供支持的代码）所做的改进。

+ Arbitrum Orbit允许您创建自己的专用链，该链连接到Arbitrum的第2层（L2）链之一：Arbitrum One，Arbitrum Nova或Arbitrum Goerli。

+ 您拥有自己的 Orbit 链，可以自定义其隐私、权限、费用代币、治理等。


### 架构图

![](/image/6.jpg)



## 参考资料

<https://docs.arbitrum.io/inside-arbitrum-nitro/>

<https://soliditydeveloper.com/arbitrum-nitro/>

<https://medium.com/privacy-scaling-explorations/a-technical-introduction-to-arbitrums-optimistic-rollup-860955ea5fec/>

<https://hackernoon.com/arbitrum-architecture-inbox/>