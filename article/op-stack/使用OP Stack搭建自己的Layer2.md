本文将以教程的形式展开，向大家介绍如何使用OP Stack搭建一条自己的基于Sepolia测试链(L1)的Layer2 (OP Stack Rollup链)。我们将尽量做到完全可复刻，跟着文章的步骤来，都可以成功搭建。
## 一、概览
OP Stack Rollup 旨在使用 EVM 等效区块链（例如以太坊、OP 主网或标准以太坊测试网）作为其 L1 链。 本教程使用 Sepolia 测试网作为 L1 链。 我们建议您也使用 Sepolia。 您还可以使用其他与 EVM 兼容的区块链，但可能会遇到意外错误。 如果您想使用备用网络，请务必仔细检查每个命令，并将任何特定于 Sepolia 的值替换为您的网络的值。 由于您要将 OP Stack 链部署到 Sepolia，因此您需要有权访问 Sepolia 节点。 您可以使用像 Alchemy 这样的节点提供商（更简单），也可以运行您自己的 Sepolia 节点。

使用OP Stack构建Layer2是一件简单但又没那么简单的事情，简单在于他真的可以在有限的步骤内就构建起一条我们自己的Layer2，不简单在于其系统毕竟还有一定的复杂度，如果直接一股脑的冲进去编译、部署，容易感到有点混乱。因此，让我们先通过思维导图的形式来直观的了解一下需要构建的内容。

1）部署组件
![](/imgs/deploy-components.png)
部署组件包含Smart Contracts、Sequencer Node、Batcher、Proposer四个部分。 其中需要启动4个可执行程序：

op-geth：执行客户端

op-node：共识客户端

op-batcher：将交易从 Sequencer 发布到 L1 区块链的服务。

op-proposer：负责将交易结果（以 L2 状态根的形式）发布到 L1 区块链。 这允许 L1 上的智能合约读取 L2 的状态。

2）源代码模块分布
![](/imgs/source_code_components.png)


## 二、构建基础环境
OP Stack的构建依赖许多的基础组件，如node, pnpm，direnv等。在进行部署前，我们需要将基础环境配置好。教程中，我们会简要的列出配置各个组件需要注意的地方，部分常用组件，我们会直接链接其安装文档，参考对应的文档安装即可，本文中不展开具体细节。

| Dependency                                                                        | Version  | Version Check Command |
| --------------------------------------------------------------------------------- | -------- | --------------------- |
| [git(opens in a new tab)](https://git-scm.com/)                                   | `^2`     | `git --version`       |
| [go(opens in a new tab)](https://go.dev/)                                         | `^1.21`  | `go version`          |
| [node(opens in a new tab)](https://nodejs.org/en/)                                | `^20`    | `node --version`      |
| [pnpm(opens in a new tab)](https://pnpm.io/installation)                          | `^8`     | `pnpm --version`      |
| [foundry(opens in a new tab)](https://github.com/foundry-rs/foundry#installation) | `^0.2.0` | `forge --version`     |
| [make(opens in a new tab)](https://linux.die.net/man/1/make)                      | `^3`     | `make --version`      |
| [jq(opens in a new tab)](https://github.com/jqlang/jq)                            | `^1.6`   | `jq --version`        |
| [direnv(opens in a new tab)](https://direnv.net/)                                 | `^2`     | `direnv --version`    |

### 1. 系统环境说明
由于每个人使用的系统版本不同，环境也不同。因此由于系统环境原因产生的异常，我们无法完全囊括，这需要朋友们自己解决。

本文将以基于源代码构建的方式展开，因此一般来说，你可以使用任何Linux发行版来尝试构建。笔者的系统环境是macOS。
### 2. 安装Go 或者 更新Go
参考：https://go.dev/doc/install

### 3. 安装Rust
由于OP Stack的依赖组件foundry依赖Rust环境，因此也需要进行安装。

参考：https://www.rust-lang.org/tools/install

### 4. 安装node
参考：https://nodejs.org/en/learn/getting-started/how-to-install-nodejs

### 5. 安装direnv
direnv 是 shell 的扩展。 它通过一项新功能增强了现有的 shell，使得可以根据当前目录加载和卸载环境变量，它会检查当前目录或者父目录中是否存在 .envrc 或者 .env 文件，如果存在，就会将其中的环境变量导出到当前的 shell 中。

参考：https://direnv.net/docs/installation.html

1）`ubuntu`可直接运行以下命令进行安装：
```shell
sudo apt install direnv
```

2）在bash,或者zsh的配置文件中添加hook。
```
eval "$(direnv hook zsh)"
```

**注意**：hook是必须添加的，参考：https://direnv.net/docs/hook.html

### 6. 安装make
绝大部分Linux发行版本，都可以使用系统自带的包管理工具进行安装。这里值得一提的是macOS, macOS默认的make是3.8.1版本的，使用brew安装后的make将被安装为gmake。 如果要覆盖成make, 请添加以下内容至bashrc文件。
```bash
PATH="/opt/homebrew/opt/make/libexec/gnubin:$PATH"
```

### 7. 安装pnpm
pnpm 是 npm 的替代品，速度更快，效率更高。
```shell
npm install -g pnpm
```

### 8. 安装foundry
Foundry 是一个使用 Rust 编写的用于以太坊应用程序开发的速度极快、可移植且模块化的工具包。安装依赖Rust开发环境。

参考：https://book.getfoundry.sh/getting-started/installation

#### 方式一：使用cargo进行安装。
```shell
cargo install --git https://github.com/foundry-rs/foundry --profile local --locked forge cast chisel anvil
```

#### 方式二：使用脚本安装
```shell
curl -L https://foundry.paradigm.xyz | bash
```
执行完成后，更新shell环境：
```shell
source ~/.zshenv
```
运行`foundryup`命令进行安装。

### 9. 安装git和jq
这两个使用系统的包管理器下载安装即可。

## 三、OP核心组件构建
### 1. 构建Optimism Monorepo

#### 1）克隆Optimism代码仓库
```shell
cd ~
git clone https://github.com/ethereum-optimism/optimism.git
```

#### 2）切换成教程分支
OP为初学者们提供了专门的教程分支，目前我们使用这个分支来构建即可。注意请不要将该分支使用于生产环境。如果要在生产环境进行部署，请切换最新的release版本进行部署。值得一提的是，最好也不要直接用`main`分支，OP的仓库正处于快速迭代中，直接选用`main`分支，极有可能带入新的问题。
```shell
git checkout tutorials/chain
```

#### 3）检查环境依赖
optimism库中提供了一个脚本工具，来一键检查依赖的基础工具的版本是否符合标准。建议调整所有依赖的基础工具的版本为OP Stack推荐的版本。
```shell
./packages/contracts-bedrock/scripts/getting-started/versions.sh
```
指向该脚本将得到如下输出：
![](/imgs/env-check-example-01.png)

#### 4）安装依赖包
```shell
pnpm install
```

#### 5）编译关键的 可执行程序
```shell
make op-node op-batcher op-proposer
pnpm build
```

### 2. 构建op-geth
#### 1）克隆op-geth代码仓库
```shell
cd ~
git clone https://github.com/ethereum-optimism/op-geth.git
```

#### 2）编译op-geth可执行程序
先进入op-geth代码仓库的一级目录下，和`Makefile`同级。
```shell
make geth
```


## 四、填充环境变量

#### 1）进入optimism目录
cd进入自己本地的optimism仓库目录下。

#### 2）拷贝模板配置
```shell
cp .envrc.example .envrc
```
至本教程写完为止，如果没有大的更新的话，配置文件形如：
![](/imgs/envrc-example.png)

#### 3）填充配置中的L1的环境信息
编辑.envrc文件，修改其中`L1_RPC_URL`和`L1_RPC_KIND`配置项。

`L1_RPC_URL`: L1节点对应的RPC URL。在本教程中，对应的是Sepolia节点。

如果你也使用的教程中的alchemy平台，可以在这里得到：
![](/imgs/alchemy-example-001.png)


`L1_RPC_KIND`: 连接的L1 RPC类型，用于通知最佳交易收据获取。有效选项：alchemy，quicknode，infura，parity，nethermind，debug_geth，erigon，basic，any。在教程中，使用的**alchemy**, 则填写**alchemy**即可。

#### 4）生成测试用的地址
在设置链时，我们需要四个地址及其私钥：

- Admin(管理员)地址具有升级合约的能力
- Batcher(批处理器)地址将顺序器事务数据发布到 L1
- Proposer(提议者)地址将 L2 事务结果（状态根）发布到 L1
- Sequencer(排序器)地址在 p2p 网络上签署区块

OP Stack提供了一键生成的脚本，我们直接使用该脚本生成即可，非常方便。
```shell
# 进入optimism仓库的一级目录下
cd ~/optimism  #记得修改为自己的路径
# 执行wallets脚本
./packages/contracts-bedrock/scripts/getting-started/wallets.sh
```

如果一切正常，你将获得类似于这样的输出：
```shell
Copy the following into your .envrc file:
  
# Admin address
export GS_ADMIN_ADDRESS=0x9625B9aF7C42b4Ab7f2C437dbc4ee749d52E19FC
export GS_ADMIN_PRIVATE_KEY=0xbb93a75f64c57c6f464fd259ea37c2d4694110df57b2e293db8226a502b30a34

# Batcher address
export GS_BATCHER_ADDRESS=0xa1AEF4C07AB21E39c37F05466b872094edcf9cB1
export GS_BATCHER_PRIVATE_KEY=0xe4d9cd91a3e53853b7ea0dad275efdb5173666720b1100866fb2d89757ca9c5a
  
# Proposer address
export GS_PROPOSER_ADDRESS=0x40E805e252D0Ee3D587b68736544dEfB419F351b
export GS_PROPOSER_PRIVATE_KEY=0x2d1f265683ebe37d960c67df03a378f79a7859038c6d634a61e40776d561f8a2
  
# Sequencer address
export GS_SEQUENCER_ADDRESS=0xC06566E8Ec6cF81B4B26376880dB620d83d50Dfb
export GS_SEQUENCER_PRIVATE_KEY=0x2a0290473f3838dbd083a5e17783e3cc33c905539c0121f9c76614dda8a38dca
```

拷贝输出的地址并更新至.envrc文件中。最终你的配置文件，应该长这个样子：
![](/imgs/address-output-example-01.png)

#### 5）给测试地址发送测试代币
我们需要向刚刚生成的地址发送一些测试代币，来支持Layer2的部署及运行。本教程中用的是Sepolia测试网，因此需要发送Sepolia ETH。关于如何获取对应链的测试代币，就需要读者朋友自显神通了。在教程中我使用的是alchemy平台，可以从alchemy平台直接获取，不过每24小时只能获取到0.5个，有一定的限制。

- `Admin` — 0.5 Sepolia ETH
- `Proposer` — 0.2 Sepolia ETH
- `Batcher` — 0.1 Sepolia ETH

Sequencer地址不需要ETH代币，因为它不会发送交易。

值得一提的是，我在部署的过程中，没有用完这些的代币，实际消耗远低于这些数额。这样的配额通常都是足够的，不需要再额外增加，除非你有特定的需求。

#### 6）加载环境变量
现在我们已经把需要的配置都配置好了，部署需要的Gas也准备完毕了。让我们回到部署流程上来。

关于envrc配置，请时刻记得它是optimism仓库下的。现在我们回到optimism仓库目录中，然后执行：
```shell
direnv allow
```
direnv工具将重载当前目录下的.envrc配置文件，如果一切正常，你将看到类似输出：
```shell
direnv: loading ~/optimism/.envrc                                            
direnv: export +DEPLOYMENT_CONTEXT +ETHERSCAN_API_KEY +GS_ADMIN_ADDRESS +GS_ADMIN_PRIVATE_KEY +GS_BATCHER_ADDRESS +GS_BATCHER_PRIVATE_KEY +GS_PROPOSER_ADDRESS +GS_PROPOSER_PRIVATE_KEY +GS_SEQUENCER_ADDRESS +GS_SEQUENCER_PRIVATE_KEY +IMPL_SALT +L1_RPC_KIND +L1_RPC_URL +PRIVATE_KEY +TENDERLY_PROJECT +TENDERLY_USERNAME
```
如果你没有看到相应的输出，那么有两种情形：
1）direnv向你抛出了异常
2）direnv没有任何输出
第一种情形，根据抛出的异常提示执行即可，direnv通常自己就在输出中给出了解决方法。
第二种情形，比较大的可能是direnv构建安装的不对，请参考direnv官网的按照教程重新部署，千万别忘了配置**hock**!

## 五、生成网络配置
1）进入optimism仓库

2）进入packages/contracts-bedrock目录下
```shell
cd packages/contracts-bedrock
```

3）生成配置
运行以下脚本在deploy-config目录中生成getting-started.json配置文件。
```shell
./scripts/getting-started/config.sh
```

## 六、部署工厂合约(可选)
工厂合约用于协助OP Stack部署Layer所需的智能合约。这是一个可选项，因为在教程中使用的sepolia测试网上已经部署了，所以不需要再部署。如果你使用的不是sepolia测试网，那么你需要自行部署这个合约。可以参考教程：
https://docs.optimism.io/builders/chain-operators/tutorials/create-l2-rollup#deploy-the-create2-factory-optional

## 七、部署L1合约
完成上述准备工作后，从这里开始，我们终于可以正式开始部署我们的OP Layer2测试链了。第一步，我们先部署所需的关键的L1智能合约！

#### 1）部署L1合约
注意，现在我们应处于`optimism/packages/contracts-bedrock`目录中。

```shell
forge script scripts/Deploy.s.sol:Deploy --private-key $GS_ADMIN_PRIVATE_KEY --broadcast --rpc-url $L1_RPC_URL --slow
```
如果一切正常，你将看到类似输出：
![](/imgs/deploy-contract-001.png)
![](/imgs/deploy-contract-002.png)
...
![](/imgs/deploy-contract-003.png)

#### 2）生成合约的构建产物
```shell
forge script scripts/Deploy.s.sol:Deploy --sig 'sync()' --rpc-url $L1_RPC_URL
```
如果一切正常，你将看到类似输出：
![](/imgs/deploy-contract-004.png)

## 八、生成L2配置文件
现在我们已经设置了 L1 智能合约，接下来可以创建在共识客户端和执行客户端中使用的多个配置文件了，这些配置文件有专门的工具可以自动生成。

需要生成三个重要文件：

- genesis.json 包含执行客户端链的创世状态
- rollup.json 包含共识客户端的配置信息
- jwt.txt 是一个 JSON Web 令牌，允许共识客户端和执行客户端安全地通信（以太坊客户端中使用相同的机制）
#### 1）进入op-node目录
```shell
# 记得替换为自己本地的路径
cd ~/optimism/op-node
```

#### 2）创建genesis.json
```shell
go run cmd/main.go genesis l2 \
  --deploy-config ../packages/contracts-bedrock/deploy-config/getting-started.json \
  --deployment-dir ../packages/contracts-bedrock/deployments/getting-started/ \
  --outfile.l2 genesis.json \
  --outfile.rollup rollup.json \
  --l1-rpc $L1_RPC_URL
```

#### 3）创建一个验证key
```shell
openssl rand -hex 32 > jwt.txt
```

#### 4）拷贝gensis.json和jwt.txt进op-geth仓库
op-geth初始化时需要用到他们。
```shell
cp genesis.json ~/op-geth
cp jwt.txt ~/op-geth
```

## 九、初始化op-geth
OP Stack 节点与以太坊节点一样，也有一个执行客户端。 执行客户端负责执行交易并存储/更新区块链的状态。 OP Stack 执行客户端有多种实现，包括 op-geth（由 Optimism Foundation 维护）、op-erigon（由 Test in Prod 维护）和 op-nethermind（即将推出）。 在本教程中，您将使用 op-geth 存储库中的 op-geth 实现。

现在我们将启动执行客户端 op-geth。 请注意，在下一步启动共识客户端之前，您不会开始看到任何交易。
#### 1）开启一个新的终端
#### 2）进入op-geth仓库目录下

#### 3）创建datadir目录
```shell
mkdir datadir
```

#### 4）初始化op-geth
```shell
build/bin/geth init --datadir=datadir genesis.json
```
如果一切正常，你将看到类似输出：
![](/imgs/geth-init-001.png)

## 十、启动op-geth
```shell
./build/bin/geth \
  --datadir ./datadir \
  --http \
  --http.corsdomain="*" \
  --http.vhosts="*" \
  --http.addr=0.0.0.0 \
  --http.api=web3,debug,eth,txpool,net,engine \
  --ws \
  --ws.addr=0.0.0.0 \
  --ws.port=8546 \
  --ws.origins="*" \
  --ws.api=debug,eth,txpool,net,engine \
  --syncmode=full \
  --gcmode=archive \
  --nodiscover \
  --maxpeers=0 \
  --networkid=42069 \
  --authrpc.vhosts="*" \
  --authrpc.addr=0.0.0.0 \
  --authrpc.port=8551 \
  --authrpc.jwtsecret=./jwt.txt \
  --rollup.disabletxpoolgossip=true
```

我们在此处使用 --gcmode=archive 运行 op-geth，因为该节点将充当Sequencer。 在归档模式下运行 Sequencer 非常有用，因为 op-proposer 需要访问完整状态。 如果您想节省磁盘空间，请随意以`full`模式运行其他（非 Sequencer）节点。

## 十一、启动op-node
当op-geth运行起来后，我们就可以运行 op-node了。 和以太坊一样，OP Stack 有一个共识客户端（op-node）和一个执行客户端（op-geth）， 共识客户端通过引擎 API“驱动”执行客户端。

共识客户端负责确定属于区块链一部分的区块和交易的列表和排序。 OP Stack 共识客户端有多种实现，包括 op-node（由 Optimism Foundation 维护）、magi（由 a16z 维护）和 hildr（由 OptimismJ 维护）。 在本教程中，您将使用 Optimism Monorepo 中的op-node实现。

#### 1）打开一个新的终端
#### 2）进入op-node目录下
```shell
# 记得替换为自己的本地路径
cd ~/optimism/op-node
```

#### 3）启动op-node
```shell
./bin/op-node \
  --l2=http://localhost:8551 \
  --l2.jwt-secret=./jwt.txt \
  --sequencer.enabled \
  --sequencer.l1-confs=5 \
  --verifier.l1-confs=4 \
  --rollup.config=./rollup.json \
  --rpc.addr=0.0.0.0 \
  --rpc.port=8547 \
  --p2p.disable \
  --rpc.enable-admin \
  --p2p.sequencer.key=$GS_SEQUENCER_PRIVATE_KEY \
  --l1=$L1_RPC_URL \
  --l1.rpckind=$L1_RPC_KIND
```

运行此命令后，您应该可以开始看到操作节点开始从 L1 链同步 L2 块。 一旦op-node赶上 L1 链的顶端，它将开始向op-geth发送块以供执行。 那时，您将开始看到在op-geth内部创建的块。

## 十二、启动op-batcher
op-batcher 从 Sequencer 获取交易并将这些交易发布到 L1。 一旦这些 Sequencer 交易包含在最终确定的 L1 区块中，它们就正式成为规范链的一部分。 op-batcher 很关键！

请密切关注 Batcher 地址的余额，因为如果有大量交易需要发布，它会快速消耗 ETH。
#### 1）打开一个新的终端
#### 2）进入op-batcher目录下
```shell
# 记得替换为自己的本地路径
cd ~/optimism/op-batcher
```
#### 3）启动op-batcher
```shell
./bin/op-batcher \
  --l2-eth-rpc=http://localhost:8545 \
  --rollup-rpc=http://localhost:8547 \
  --poll-interval=1s \
  --sub-safety-margin=6 \
  --num-confirmations=1 \
  --safe-abort-nonce-too-low-count=3 \
  --resubmission-timeout=30s \
  --rpc.addr=0.0.0.0 \
  --rpc.port=8548 \
  --rpc.enable-admin \
  --max-channel-duration=1 \
  --l1-eth-rpc=$L1_RPC_URL \
  --private-key=$GS_BATCHER_PRIVATE_KEY
```

## 十三、启动op-proposer
Proposer 是一项负责将交易结果（以 L2 状态根的形式）发布到 L1 区块链的服务。 这允许 L1 上的智能合约读取 L2 的状态，这对于跨链通信和状态更改之间的协调是必要的。 未来 Proposer 很可能会被删除，但目前它是 OP Stack 的必要组成部分。 您将使用 Optimism Monorepo 中找到的 Proposer 组件的 op-proposer 实现。
#### 1）打开一个新的终端
#### 2）进入op-proposer目录
```shell
# 记得替换为自己的本地路径
cd ~/optimism/op-proposer
```
#### 3）启动op-proposer
```shell
./bin/op-proposer \
  --poll-interval=12s \
  --rpc.port=8560 \
  --rollup-rpc=http://localhost:8547 \
  --l2oo-address=$(cat ../packages/contracts-bedrock/deployments/getting-started/L2OutputOracleProxy.json | jq -r .address) \
  --private-key=$GS_PROPOSER_PRIVATE_KEY \
  --l1-eth-rpc=$L1_RPC_URL
```

## 十四、终端输出检查
完成op-geth、op-node、op-batcher、op-proposer的启动后，如果一切正常，在不多时（需要等待一小会儿，通常不会太长）后，你将在终端中看到类似于这样的输出：
#### 1）op-geth
![](/imgs/op-geth-console-output-001.png)
#### 2）op-node
![](/imgs/op-node-console-output-001.png)

#### 3）op-batcher
![](op-batcher-console-output-001.png)

#### 4）op-proposer
![](/imgs/op-proposer-console-output-001.png)
刚开始可能一直阻塞在这里，没有日志输出。不用紧张，一段时间后，我们就可以看到间断的打印如下输出了：
![](/imgs/op-proposer-consele-output-002.png)

如果你看到的输出不是这样的，或者有异常报错终止了，请根据报错提示，寻找解决方案。
OP文档中整理了部分常见的错误场景，参见：https://docs.optimism.io/builders/chain-operators/management/troubleshooting

如果你的问题在文档中没有列出来，可以到OP论坛、OP中文力量以及OP Discard中寻找帮助。


## 十五、验证：和自己的Layer2交互
现在，我们拥有了一个功能齐全的 OP Stack Rollup。跑起来后，我们当然要验证下自己的Layer2是否正常运行了。测试验证的方式有很多种，可以用web3.js写个demo交互，也可以直接用metamask进行交互。

如果你完全按照本教程来，将在`http://localhost:8545`上有一个运行中的sequencer节点。

这里，我按照自己的喜好，选用web3.py库来进行功能验证。读者朋友完全可以根据自己的喜好来。如果你想完全复刻本文的测试，也不用担心，不会有多复杂，目前任何一个现代化的Linux系统，都预装了Python环境。只需要执行以下命令，即可安装web3.py，然后跟随本文运行测试脚本即可。
```shell
pip install web3
```

### 1. 查询最新的区块信息
#### 1）创建helloop.py文件
```python
import asyncio
from web3 import AsyncWeb3

async def test():
    w3 = AsyncWeb3(AsyncWeb3.AsyncHTTPProvider('http://localhost:8545'))
    result = await w3.is_connected()
    print(result)

    block = await w3.eth.get_block('latest')
    print(block)

# Create an event loop
loop = asyncio.get_event_loop()

# Use the event loop to run the test function
loop.run_until_complete(test())
```

#### 2）运行helloop.py
```shell
python helloop.py
```
如果一切正常，你将看到类似的输出：
![](/imgs/python-example-output-001.png)


### 2. 从L1上跨一笔ETH到L2上
将 Sepolia ETH 存入我们的L2链的最简单的方法是将 ETH 直接发送到L1 标准桥(我们在部署L1合约时部署的)合约。

#### 1）进入contracts-bedrock目录
```shell
# 记得替换为自己本地的路径
cd ~/optimism/packages/contracts-bedrock
```

#### 2）获取L1标准桥合约地址
```shell
cat deployments/getting-started/L1StandardBridgeProxy.json | jq -r .address
```

#### 3）发送一些L1 ETH到标准桥合约地址上
假设我希望账户A在L2上拥有自己的ETH，那么需要操作账户A在对应的L1上向标准桥合约地址发送ETH。验证用，发送一点点就可以了，比如我验证时，发送的0.01 Sepolia ETH。直接用小狐狸钱包操作即可，L1标准桥合约已经部署在对应的L1上了。

注意：转账成功后，仍需要等待一段时间才能在L2看到，差不多5分钟左右。

#### 4）添加查询余额的代码
```python
import asyncio
from web3 import AsyncWeb3

async def test():
    w3 = AsyncWeb3(AsyncWeb3.AsyncHTTPProvider('http://localhost:8545'))
    result = await w3.is_connected()
    print(result)

    balance = await w3.eth.get_balance('0x7EA1Da61d41fC1109972aBF23F9d13A1E6B30123')
    print(balance)

# Create an event loop
loop = asyncio.get_event_loop()

# Use the event loop to run the test function
loop.run_until_complete(test())    
```

#### 5）运行helloop.py
```shell
python helloop.py
```
如果一切正常，你将看到类似输出：
![[python-example-output-002.png]]

## 总结
至此，我们的教程也要结束了。现在我们已经拥有了一个可以正常使用且功能齐全的 OP Stack Rollup。接下来，就可以根据我们的需求进行使用了。

值得一提的是，也许出于某些需求，可能有朋友会对OP Stack的各个组件进行定制和修改，但必须注意，OP Stack不会对这些内容做出兼容的保证。因此，风险需要自行承担。
