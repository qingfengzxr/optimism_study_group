按照先看整体，再究细节的思路，这篇文章将从上层的视角向大家介绍一条由OP Stack创建的Layer2 chain的架构。方便大家对OP Chain的整体工作流程建立一个直观的印象，为后续深入源码打下基础。

要理解OP Chain的工作流程并不复杂，只需要弄明白几张图即可。这几张图合在一起，共同解释了以下几个关键问题：

1. 网络的参与者有哪些？
2. 用户如何参与到网络中？
3. L1的ETH如何到L2上去？
4. OP Chain有哪些模块？这些模块是怎么合作的？分别负责什么事情？
5. L2 区块是以什么规则来构建链的？

对这些问题有了一定的了解后，再看源码，才不会云里雾里。

## 网络参与者有哪些？
那么，让我们先从网络的参与者有哪些开始。
![](/imgs/OP_Network_Participants_Overview.png.png)

上图展示了4个参与者，分别是**Users**、**Sequencers**、**Verifiers**、**ETH L1 Chain**。

1）Users

- 在OP Chains中，用户可以通过将交易发送到以太坊主网上的合约，在 L2 上存入或提取任意交易（对应Bridges的功能）。
- 向使用ETH一样，使用部署在L2上的智能合约，唯一的差别是，交易是发送到**Sequencers**上，而不是ETH节点。
- 可以查询交易、区块等信息。

2）Sequencers
- 接收来自用户的链下交易（相对于L1而言）。
- 监控和收集链上（L1）的交易信息，主要是质押事件信息。
- 按特定顺序将上述两种交易合并到 L2 区块中。
- 向L1提交批次信息和断言信息。

**Sequencers**是主要的L2区块生产者。

3）Verifiers

验证者有两个目的：
- 为用户提供Rollup数据；
- 验证Rollup的完整性并质疑无效的断言。 

为了网络保持安全，必须至少有一个诚实的验证者能够验证汇总链的完整性并为用户提供区块链数据。

4）ETH L1 Chain
OP Stack目前只支持EVM链。L1链是L2数据有效性和可信任的基石。

从这张图上，我们可以了解到对于OP Chain的网络来说，都包含了哪些参与者。用户是网络的交互核心，用户可以向网络发送交易，也可以查询信息。

## L1的ETH是如何到L2上去的？
![](/imgs/sequencer_sync_deposit.png)

上图展示了OP Chain Sequencer模块同步用户Deposits和接受用户交易的流程。图中为每个行为都标了序号，这些序号基本上代表了逻辑顺序，除了**4.Send Transaction**有一点要注意的外，根据序号观看就好。当然，本文的目的就是为了让大家更轻松的建立直观印象，因此下面会介绍整个流程，也会说明为什么**流程4**需要注意。

一步步来，首先是用户发起了 **1.Submit Deposit** 的操作。Deposit操作的一种方式是，用户在L1发起一笔转账交易，转账给标准桥合约地址，转账金额是你想跨链到L2的ETH的数额。

流程**2. Get Blocks(including deposit event)**, Rollup Node(op-node承担)会按照一定的规则(后面会讲到)，获取L1区块的信息，并作为L2区块的输入信息提供给L2区块使用。其中用户在操作1发起的Deposit交易，会在此时被解析出来。包含用户Deposit信息的L2 Block,称为Deposit Block。

流程**3.Insert deposit block**,Rollup Node通过Engine API调用，将Deposit Block提交给Execution Engine(op-geth承担)。此时，用户在L1的Deposit信息，就已经被L2接收到了，但还没有最终确认。

流程**4.Send Transaction**, 如果仅从Deposit L1 ETH到L2的操作的角度来看，流程4是不必要的，用户在L1发起了Deposit交易后，OP Chain会自动完成剩下的动作，用户不需要额外的再对L2发起交易。但它这个图还包含了接收交易的演示，因此他将这个行为作为流程4表现了出来。不要把**Submit Deposit**和**Send Transaction**关联起来（当然，必须通过Deposit在L2有了ETH，才能在L2发起交易，但他们在OP Chain系统里不是线性关系），这就是上面说的需要注意的地方。

流程**5.Get latest sequencer blocks**，获取最近的排序好的区块，并通过Engine API推送到Execution Engine中。

流程**6.Submit sequencer blocks(ie. a batch) to the batch inbox**, 将进行批处理后的sequencer blocks提交的Batch Inbox上。关于批处理，我们暂时理解为压缩，后面再探讨具体细节。记住它有可以减少sequencer blocks的体积(压缩)，L2可以从L1上获取之前提交给它的信息，根据该信息可以解析出原来的sequencer blocks(解压缩)，这样的特性即可。

Batch Inbox是一个L1上的常规的EOA账户。

流程**7.Get latest assertable block hash**, 获取最新的区块哈希断言。

流程**8.Verify block hash in the fault proof VM to detect on chain<->off chain divergence**, 验证故障证明虚拟机中的块哈希以检测链上<->链下分歧。

流程**9.Submit validated block hash assertion**, 提交经过验证的区块哈希断言。

流程7，8，9涉及到故障证明相关的内容，这一部分较为复杂，暂时不在这里展开。我们先记住它是用来检测和处理L1和L2链状态分歧的即可，不影响我们理解OP Chain的工作流程。然后，**L2 Output Oracle**是一个合约，该合约主要用来存储**L2 output roots**（一个 32 字节值，用作对 L2 链当前状态的承诺。由sequencer生成和提交）。


## OP Chain模块组成
![](/imgs/op-stack-componenets-001.png)

上图展示了OP Chain的核心组件。从L1和L2两个层级来看，L1层包含以下组件：
- OptimismPortal
一个部署在L1上的合约。会将Deposit交易存在这个合约中，而不是token。会抛出TransactionDeposited事件，Rollup Driver会读取事件信息，然后处理质押汇款。
- BatchInbox
如前文所述，这是一个常规的EOA账户。**Batch Submitter**将batch信息提交到这个地址上。这种方式，可以通过不执行任何 EVM 代码来节省 Gas 成本。
- L2OutputOracle
如前文所述，这是一个存储 L2 output roots的智能合约，用于提款和故障证明。

L2层包含以下组件：
- Rollup Node
1. 独立、无状态的二进制文件。
2. 接收用户的L2交易。
3. 同步并验证 L1 上的rollup数据。
4. 应用特定的rollup块生成规则来从 L1合成块。
5. 使用Engine API 将块追加到 L2 链。
6. 处理 L1 的重组。
7. 将未提交的块分发给其他Rollup Node。

- L2 Execution Engine
1. 一个普通的 Geth 节点，经过少量修改以支持 Optimism。
2. 维持L2状态。
3. 将状态同步到其他 L2 节点以实现快速加入。
4. 向Rollup Node提供Engine API。

- Batch Submitter
一个专门将transaction bathes提交到BatchInbox的后台程序。

- Output Submitter
将L2 output commitments提交到L2OutputOracle的后台进程。

此外，上图还可以将Rollup Node和L2 Execution Engine的关系再展开一下，得到下图：
![](/imgs/op-stack-components-002.png)
这个图展示了以下几个关键信息：
1. 执行引擎(EE)之间使用独立于Rollup Node的对等网络进行通信。可以同步交易，区块状态等。
2. Rollup Node之间也存在一个特有的P2P网络，使用这个网络进行通信，而不是EE之间的P2P网络。
3. Rollup Node通过Engine API和执行引擎交互。

##  L2区块是以什么规则来构建链的？

![](/imgs/workflow-l2-devition-from-l1-001.png)

给定一条L1链，Rollup Chain可以基于L1的区块信息，将L2链完全恢复出来。上图展示了OP Chain从L1恢复L2链的基础逻辑。

让我们先了解一下图中各个元素的含义：
- Epoch
每个L1区块，都会在L2对应一个Epoch。每个Epoch可以包含多个L2区块，起始的L2区块必须是包含L1 Deposit信息的L2区块(Deposit Block)。
- D1... D3
表示Epoch 1中所有的Deposit信息，这些信息是从`OptimismPortal`合约上获得的。
- B1... B7
Transaction Batches。交易Batch不仅可以包含一个L2区块的交易，还可以包含多个L2区块排序好后的交易。
- 一个L2区块中既可以包喊Deposit交易，也可以包含Batch交易。

D1是在L1上发现的第一个Deposit信息，以包含该信息的L1块为基础，作为Epoch1。推导出L2的第一个区块。B1作为同一个L1区块上的batch，和D1放在同一个L2区块中。

以ETH作为L1和上一个教程中部署的L2参数为例，ETH出块的平均时间为12s, L2的平均出块时间为2s。这些值意味着一个Epoch的跨度为12s, 在这12s内，可以包含6个区块。这也是上图中，会有B2, B4, B5, B6, B7的原因。注意，如果从L2的视角来看，B2不代表只有一个区块，它可能是多个区块，因为一个Batch可以从多个排好序的区块中整合交易。

按照同样的逻辑，我们根据在L1的第二个区块中的D2信息，推导出L2上的第二个Deposit Block，也是Epoch2的起始区块。后面的推导都是这样的一个逻辑，一直循环直到达到L1的最新的的区块为止。