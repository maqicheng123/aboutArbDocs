
# Arbitrum Rollup Protocol

## 目标 

解决以太坊网络上的拥堵和高费用问题

## 实现
通过将交易批量处理并将其提交到以太坊主链之前进行验证，从而大大提高了交易吞吐量，并降低了交易费用

### 成员角色
+ Sequencer(layer2):  是一个专门指定的全节点，被赋予有限的权力来控制交易的排序
+ Validator(layer2): 验证器，负责改变layer2状态，并将状态根提交到layer1
+ Inbox (layer1)：
    + sequencer inbox: 接受sequencer节点发送的批次交易数据
    + delayed inbox: 接受任意User提供的layer2交易数据
+ Outbox (layer1)：接受User layer2->layer1交易数据    
+ Rollup & Challenge Contract(layer1)：用于处理layer2状态根的提交和挑战
+ Bridge Contract(layer1): 用来管理其他合约状态并作为router合约将业务分发到其他合约

## Layer2 交易确认流程
    
描述：

    1.用户将 L2 交易发送到Sequencer节点。
    2.一旦排序器收到足够的交易，它将将它们作为批处理发布到 L1 智能合约中。(brotli压缩)
    3.validator将从 L1 智能合约读取这些交易，并在 L2 状态的本地副本上处理它们。
    4.处理后，将在validator本地生成新的 L2 状态，验证器会将此新状态根发布到 L1 智能合约中。
    5.然后，所有其他validator将在其 L2 状态的本地副本上处理相同的交易。
    6.他们将结果的 L2 状态根与发布到 L1 智能合约的原始状态根进行比较。
    7.如果其中一个验证者获得的状态根与发布到 L1 的状态根不同，他们将在 L1 上开始挑战。
    8.挑战将要求挑战者和发布原始状态根的validator轮流证明正确的状态根应该是什么。
    9.无论哪个用户输掉挑战，他们的初始存款（本金）都会被削减。如果发布的原始 L2 状态根无效，它将被未来的验证者销毁，并且不会包含在 L2 链中

![](/image/0.png)

## 防止Sequencer节点作恶：DelayInbox
行为良好的 Sequencer 将接受来自所有请求者的事务并公平对待它们，从而尽快为每个请求者提供承诺的交易结果

恶意序列器可能会造成一些痛苦。如果它拒绝处理您的交易，您将被迫通过延迟的收件箱，延迟时间更长。恶意 Sequencer 具有强大的能力来提前运行每个人的交易，因此它可以以用户为代价获得巨大利润。

其他任何人都可以提交消息，但非 Sequencer 节点提交的消息将被放入“延迟收件箱”队列中，该队列由 L1 以太坊合约管理。

延迟收件箱队列中的邮件将在那里等待，直到 Sequencer 选择将它们“释放”到主收件箱中，在那里它们将被添加到收件箱的末尾。行为良好的 Sequencer 通常会在大约 10 分钟后发布延迟消息，原因如下所述。

或者，如果邮件在延迟收件箱队列中的时间超过最大延迟间隔（当前为 24 小时），则任何人都可以强制将其提升到主收件箱中。（这可确保排序器只能延迟消息，但不能审查它们。

```solidity
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
1.延迟
2.可重试

### L2->L1
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



### Staking
质押者存入由 Arbitrum 第 1 层合约持有的资金，如果质押者输掉挑战，将被没收。

单个质押可以覆盖一系列RBlocks。每个质押者都押在最新确认的RBlock上;如果您押注在 RBlock 上，您也可以质押该 RBlock 的一个继任者。因此，您可能会被押注在一系列 RBlock 上，这些 RBlock 代表关于链正确历史的单个连贯声明。一个质押就足以让你承诺遵守这一系列的RBlocks。
![](/image/4.png)
### 挑战流程

    



# Anytrust protocol 

todo
---


# Orbit Chain
### 介绍
+ Arbitrum Orbit允许您创建自己的专用链，该链连接到Arbitrum的第2层（L2）链之一：Arbitrum One，Arbitrum Nova或Arbitrum Goerli。

+ 您拥有自己的 Orbit 链，可以自定义其隐私、权限、费用代币、治理等。

+ 允许您不断集成对 Arbitrum Nitro （为 Arbitrum 的 L2 和 Orbit 链的节点提供支持的代码）所做的改进。

### 架构图
![](/image/6.jpg)




## note    
1.RBlock确认后如何将RBlock stateRoot更新到L1 

rollup contract stakeNewNode function
2.交互式证明中如何使用
 挑战者（validator）使用本地数据初始化WASM虚拟机，然后与Rollup合约进行交互。
 
 然后使用WASM虚拟机执行交互式证明，证明结果与挑战者本地数据进行比较，如果不一致则挑战成功，否则挑战失败

    WASM将RBlock操作拆分至一个op code,并在L1 rollup中完成证明操作