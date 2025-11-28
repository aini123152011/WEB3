# Week 10 Day 4: 区块链节点服务与基础设施监控

**预计学习时间**: 6-8小时  
**难度**: ⭐⭐⭐⭐

## 学习目标

通过本节课程,你将能够:
- 理解并选择合适的 RPC 节点服务商
- 掌握自建以太坊节点的搭建与维护
- 使用 The Graph 构建去中心化数据索引
- 搭建 Prometheus + Grafana 监控区块链基础设施
- 使用 Tenderly 和 Defender 进行交易调试与监控

## 课程内容

### 1. 节点服务提供商 vs 自建节点

#### 1.1 RPC 服务商对比

在生产环境中，高可用的 RPC 节点至关重要。

| 特性 | Infura | Alchemy | QuickNode | Moralis |
| :--- | :--- | :--- | :--- | :--- |
| **支持链** | ETH, Polygon, Optimism, Arbitrum, Near, Aurora | ETH, Polygon, Optimism, Arbitrum, Solana, Astar | 15+ 链支持 (包括 BSC, Ava) | ETH, BSC, Polygon, Solana, etc. |
| **特色功能** | 行业标准, IPFS 支持, Archive 数据 | Supernode (高可靠), Notify (Webhooks), Transact (防掉单) | 低延迟, 专用节点, 强大的 NFT API | 跨链 API, 实时数据流, Auth SDK |
| **免费层** | 100k 请求/天 | 300M CU/月 (约 10-15M 请求) | 限制较多 | 慷慨的免费层 |
| **适用场景** | 基础连接, IPFS | 高并发 DApp, NFT 平台 | 多链项目, 高频交易 | 快速开发, 数据聚合 |

**配置多重 RPC 提供商 (高可用)**:
```javascript
// ethers.js FallbackProvider
const { ethers } = require("ethers");

const infuraProvider = new ethers.providers.InfuraProvider("homestead", "YOUR_KEY");
const alchemyProvider = new ethers.providers.AlchemyProvider("homestead", "YOUR_KEY");

const fallbackProvider = new ethers.providers.FallbackProvider([
  { provider: infuraProvider, priority: 1, weight: 1 },
  { provider: alchemyProvider, priority: 1, weight: 1 },
]);

// 使用 fallbackProvider 就像使用普通 provider 一样
fallbackProvider.getBlockNumber().then(console.log);
```

#### 1.2 自建节点 (Geth/Erigon)

**硬件要求 (全节点)**:
- **CPU**: 4+ 核快速 CPU
- **RAM**: 16GB+ (建议 32GB)
- **SSD**: 2TB+ NVMe SSD (必须是 SSD，HDD 无法同步)
- **网络**: 25MBit/s+ 带宽

**使用 Docker 运行 Geth**:
```yaml
# docker-compose.yml
version: "3"
services:
  geth:
    image: ethereum/client-go:stable
    container_name: geth-node
    restart: unless-stopped
    ports:
      - "30303:30303"
      - "30303:30303/udp"
      - "8545:8545"
      - "8546:8546"
    volumes:
      - ./data:/root/.ethereum
    command:
      - --http
      - --http.addr=0.0.0.0
      - --http.vhosts=*
      - --http.corsdomain=*
      - --http.api=web3,eth,net,txpool
      - --ws
      - --ws.addr=0.0.0.0
      - --ws.origins=*
      - --ws.api=web3,eth,net,txpool
      - --syncmode=snap
```

**Erigon (高性能归档节点)**:
Erigon 存储效率更高，适合需要历史状态查询的场景。

### 2. 数据索引服务 (The Graph)

#### 2.1 为什么需要索引

直接从节点查询复杂数据（如"查询某用户拥有的所有 NFT"或"查询某交易对的历史成交量"）非常慢且低效。The Graph 通过监听链上事件，将数据索引到 PostgreSQL 数据库，提供 GraphQL 查询接口。

#### 2.2 开发 Subgraph

**安装工具**:
```bash
npm install -g @graphprotocol/graph-cli
```

**初始化项目**:
```bash
graph init --studio my-nft-subgraph
# 选择协议: ethereum
# 选择网络: mainnet
# 输入合约地址: 0x...
# 输入 ABI 路径
```

**定义 Schema (`schema.graphql`)**:
```graphql
type Token @entity {
  id: ID!
  tokenId: BigInt!
  owner: User!
  uri: String
  transfers: [Transfer!]! @derivedFrom(field: "token")
}

type User @entity {
  id: ID!
  tokens: [Token!]! @derivedFrom(field: "owner")
  balance: BigInt!
}

type Transfer @entity {
  id: ID!
  token: Token!
  from: User!
  to: User!
  timestamp: BigInt!
  blockNumber: BigInt!
  txHash: Bytes!
}
```

**编写映射 (`src/mapping.ts`)**:
```typescript
import { Transfer as TransferEvent } from "../generated/Contract/Contract"
import { Token, User, Transfer } from "../generated/schema"

export function handleTransfer(event: TransferEvent): void {
  // 处理 User
  let fromUser = User.load(event.params.from.toHex())
  if (!fromUser) {
    fromUser = new User(event.params.from.toHex())
    fromUser.balance = BigInt.fromI32(0)
    fromUser.save()
  }

  let toUser = User.load(event.params.to.toHex())
  if (!toUser) {
    toUser = new User(event.params.to.toHex())
    toUser.balance = BigInt.fromI32(0)
    toUser.save()
  }

  // 处理 Token
  let token = Token.load(event.params.tokenId.toString())
  if (!token) {
    token = new Token(event.params.tokenId.toString())
    token.tokenId = event.params.tokenId
    token.uri = "" // 可以通过 IPFS 获取
  }
  token.owner = toUser.id
  token.save()

  // 记录 Transfer
  let transfer = new Transfer(event.transaction.hash.toHex() + "-" + event.logIndex.toString())
  transfer.token = token.id
  transfer.from = fromUser.id
  transfer.to = toUser.id
  transfer.timestamp = event.block.timestamp
  transfer.blockNumber = event.block.number
  transfer.txHash = event.transaction.hash
  transfer.save()
}
```

#### 2.3 部署 Subgraph

1.  **Codegen**: `graph codegen`
2.  **Build**: `graph build`
3.  **Deploy**: `graph deploy --studio my-nft-subgraph`

### 3. 基础设施监控 (Prometheus + Grafana)

#### 3.1 监控指标

- **节点健康**: 块高度增长、Peers 数量、同步状态
- **RPC 性能**: 请求延迟、错误率、吞吐量 (RPS)
- **服务器资源**: CPU, RAM, Disk I/O, Network
- **业务指标**: 待处理交易数、Gas 价格波动

#### 3.2 配置 Prometheus

```yaml
# prometheus.yml
global:
  scrape_interval: 15s

scrape_configs:
  - job_name: 'geth'
    metrics_path: /debug/metrics/prometheus
    static_configs:
      - targets: ['geth-node:6060']
  
  - job_name: 'node_exporter'
    static_configs:
      - targets: ['node-exporter:9100']
```

#### 3.3 Grafana 面板

- **导入 Dashboard**: 使用 ID `13877` (Geth Prometheus Dashboard)
- **设置报警**: 当块高度停止增长超过 1 分钟时发送 Slack/Telegram 通知

### 4. 交易调试与智能监控

#### 4.1 Tenderly

Tenderly 是 Web3 开发者的全栈监控平台。

- **Debugger**: 可视化查看交易执行堆栈，状态变化。
- **Simulator**: 模拟交易执行，用于测试网/主网部署前的最终验证。
- **Alerts**: 监听合约事件。

**设置 Tenderly Alert**:
1.  Target: 合约地址
2.  Trigger Type: Function Call / Event Emitted / State Change
3.  Condition: `failed() == true` (监听失败交易) 或 `amount > 1000 ETH` (监听大额转账)
4.  Destination: Slack / Discord / Email / Webhook

**GitHub Actions 集成 Tenderly**:
```yaml
# 自动将合约推送到 Tenderly
- name: Push to Tenderly
  uses: tenderly/github-action@v1
  with:
    api_key: ${{ secrets.TENDERLY_API_KEY }}
    account: my-org
    project: my-project
    contracts: |
      MyToken: ${token_address}
```

#### 4.2 OpenZeppelin Defender

用于安全的合约运维 (DevSecOps)。

- **Admin**: 管理多签钱包、Timelock 合约。
- **Sentinel**: 监控合约风险，自动暂停合约。
- **Autotask**: 定时执行脚本 (如定期更新预言机价格，自动收割收益)。

**Autotask 示例 (自动维护)**:
```javascript
const { ethers } = require("ethers");
const { DefenderRelaySigner, DefenderRelayProvider } = require('defender-relay-client/lib/ethers');

exports.handler = async function(credentials) {
  const provider = new DefenderRelayProvider(credentials);
  const signer = new DefenderRelaySigner(credentials, provider, { speed: 'fast' });

  const contract = new ethers.Contract(ADDRESS, ABI, signer);
  
  // 检查是否需要维护
  const needsMaintenance = await contract.checkUpkeep("0x");
  
  if (needsMaintenance) {
    const tx = await contract.performUpkeep("0x");
    console.log(`Tx hash: ${tx.hash}`);
  }
}
```

## 实践练习

### 练习1: 构建高可用 RPC 代理

编写一个 Node.js 服务，代理 RPC 请求：
1.  配置 3 个不同的 RPC 端点 (Infura, Alchemy, Public Node)。
2.  实现轮询或基于延迟的负载均衡。
3.  如果某个节点返回错误或超时，自动重试下一个节点。
4.  记录请求日志和各节点健康状态。

### 练习2: 部署 Subgraph

1.  选择一个测试网上的 ERC721 合约。
2.  初始化 Subgraph 项目。
3.  编写 Schema 和 Mapping 索引所有 `Transfer` 事件。
4.  部署到 The Graph Studio (或 Hosted Service)。
5.  在 Playground 中查询 "拥有最多 NFT 的前 10 个用户"。

### 练习3: 设置交易报警

1.  注册 Tenderly 账号。
2.  Import 你的测试网合约。
3.  设置一个报警规则：当发生 `Transfer` 事件且金额 > 100 时，发送邮件通知。
4.  编写脚本触发该条件，验证是否收到报警。

## 学习检查清单

- [ ] 了解主流 RPC 服务商的优缺点
- [ ] 理解 Geth/Erigon 节点的搭建要求
- [ ] 掌握 The Graph 的工作原理和开发流程
- [ ] 能够编写 GraphQL Schema 和 AssemblyScript Mapping
- [ ] 理解 Prometheus + Grafana 监控架构
- [ ] 熟练使用 Tenderly 进行交易调试
- [ ] 了解 OpenZeppelin Defender 的自动化运维功能

## 扩展阅读

- [The Graph Docs](https://thegraph.com/docs/)
- [Ethereum Node Monitoring](https://github.com/eth-educators/eth-node-monitoring)
- [Tenderly Documentation](https://docs.tenderly.co/)
- [OpenZeppelin Defender Docs](https://docs.openzeppelin.com/defender/)
- [Geth Metrics](https://geth.ethereum.org/docs/monitoring/metrics)

## 下节预告

下一节我们将学习**合约升级与维护**,包括:
- 代理合约模式详解 (Transparent, UUPS, Beacon)
- 使用 Hardhat Upgrades 插件部署可升级合约
- 升级流程与治理 (多签控制升级)
- 合约暂停与紧急救援机制
- 数据迁移策略
