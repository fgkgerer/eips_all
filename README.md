# ERC/EIP 标准学习文档

> 以太坊改进提案（EIP）/ 以太坊请求评论（ERC）标准的中文学习指南，按应用领域分类整理。
>
> 官方查询：https://eips.ethereum.org/all | https://github.com/ethereum/ERCs | https://github.com/ethereum/EIPs

---

## 标准速查表

| 编号       | 标题                                               | 状态     | 类型     | 所属领域                 | 详细文档 |
| ---------- | -------------------------------------------------- | -------- | -------- | ------------------------ | -------- |
| ERC-20     | Token Standard                                     | Final    | ERC      | DEX, 借贷                | [ERC-20.md](ERC-20.md) |
| ERC-165    | Standard Interface Detection                       | Final    | ERC      | NFT                      | [ERC-165.md](ERC-165.md) |
| EIP-712    | Typed Structured Data Hashing and Signing          | Final    | Interface| DEX, 借贷, NFT, 钱包     | [EIP-712.md](EIP-712.md) |
| ERC-721    | Non-Fungible Token Standard                        | Final    | ERC      | DEX, NFT                 | [ERC-721.md](ERC-721.md) |
| ERC-1155   | Multi Token Standard                               | Final    | ERC      | NFT                      | [ERC-1155.md](ERC-1155.md) |
| ERC-1271   | Standard Signature Validation for Contracts        | Final    | ERC      | NFT, 钱包                | [ERC-1271.md](ERC-1271.md) |
| ERC-2612   | Permit Extension for ERC-20                        | Final    | ERC      | DEX, 借贷                | [ERC-2612.md](ERC-2612.md) |
| ERC-2981   | NFT Royalty Standard                               | Final    | ERC      | NFT                      | [ERC-2981.md](ERC-2981.md) |
| ERC-4337   | Account Abstraction via Entry Point                | Final    | ERC      | 钱包                     | [ERC-4337.md](ERC-4337.md) |
| ERC-4626   | Tokenized Vault Standard                           | Final    | ERC      | 借贷                     | [ERC-4626.md](ERC-4626.md) |
| EIP-7702   | Set EOA Account Code                               | Final    | Core     | 钱包                     | [EIP-7702.md](EIP-7702.md) |
| ERC-8004   | Trustless Agents                                   | Draft    | ERC      | AI+Web3                  | [ERC-8004.md](ERC-8004.md) |
| X402       | Internet-Native Payment Protocol                   | 非官方   | 协议     | AI+Web3                  | [X402.md](X402.md) |
| ERC-8183   | Agentic Commerce Standard                          | Draft    | ERC      | AI+Web3                  | [ERC-8183.md](ERC-8183.md) |

---

## 按领域浏览

| 领域               | 核心标准                                                    | 入口文档 |
| ------------------ | ----------------------------------------------------------- | -------- |
| 🏦 **DEX (Uniswap)** | ERC-20, ERC-2612, ERC-721, EIP-712                           | [dex-uniswap.md](dex-uniswap.md) |
| 💰 **借贷 (Aave)**   | ERC-20, ERC-2612, ERC-4626, EIP-712                          | [lending-aave.md](lending-aave.md) |
| 🎨 **NFT (OpenSea)** | ERC-721, ERC-1155, ERC-165, ERC-2981, ERC-1271, EIP-712      | [nft-opensea.md](nft-opensea.md) |
| 👛 **钱包 (MetaMask)**| ERC-4337, EIP-7702                                            | [wallet-metamask.md](wallet-metamask.md) |
| 🤖 **AI+Web3**       | ERC-8004, X402, ERC-8183                                      | [ai-web3.md](ai-web3.md) |

---

## 标准之间的依赖关系

```
                      ┌─────────────────┐
                      │    ERC-20       │ ← 最基础：所有 Token 的根基
                      │  (Token 标准)    │
                      └───────┬─────────┘
                              │
          ┌───────────────────┼───────────────────┐
          │                   │                   │
          ▼                   ▼                   ▼
   ┌─────────────┐    ┌─────────────┐    ┌─────────────┐
   │  ERC-2612   │    │  ERC-4626   │    │  ERC-721     │
   │  (Permit)   │    │  (金库)      │    │  (NFT)       │
   └──────┬──────┘    └─────────────┘    └──────┬──────┘
          │                                     │
          │ 依赖 EIP-712                         │ 强制实现 ERC-165
          │                                     │
          ▼                                     ▼
   ┌──────────────────────────────────────────────────┐
   │                  EIP-712                         │ ← 所有链下签名的基石
   │              (类型化数据签名)                      │
   └──────────┬───────────────────────────────────────┘
              │
     ┌────────┼────────┬──────────────┐
     ▼        ▼        ▼              ▼
  ERC-2612  ERC-1271  ERC-4337    ERC-8004
  (Permit)  (合约签名) (账户抽象)   (Agent 信任)

                      ERC-4337 ──── 互补 ──── EIP-7702
                      (基础设施)              (EOA 升级)

  ERC-8004 ─── 配合 ─── X402 ─── 配合 ─── ERC-8183
  (信任层)            (支付层)            (结算层)
```

---

## 按层次分类

### 🔵 代币层（Token Layer）— 资产如何表示
- **ERC-20** — 同质化代币（货币、治理代币、LP Token）
- **ERC-721** — 非同质化代币（艺术品、域名、Uniswap V3 LP 仓位）
- **ERC-1155** — 多代币（游戏道具、会员卡、门票）

### 🟢 交互层（Interaction Layer）— 如何安全操作
- **ERC-165** — 接口检测（调用前确认合约支持什么功能）
- **EIP-712** — 类型化签名（让人看懂签名内容，防钓鱼）
- **ERC-1271** — 合约签名验证（Gnosis Safe 多签也能签名挂单）
- **ERC-2612** — 无 Gas 授权（签名替代 approve 交易）

### 🟡 应用层（Application Layer）— 特定领域的标准
- **ERC-4626** — 收益金库统一接口（Yearn/Aave/Compound 都用一个 API）
- **ERC-2981** — NFT 版税（创作者持续获得二次交易分成）

### 🟠 账户层（Account Layer）— 钱包的进化
- **ERC-4337** — 账户抽象基础设施（智能账户、代付 Gas、Passkey）
- **EIP-7702** — EOA 原地升级（保留原地址，获得智能能力）

### 🔴 AI+Web3 层（AI Agent Economy）— 未来的前沿
- **ERC-8004** — AI Agent 信任基础设施（身份、声誉、验证）
- **X402** — AI Agent 支付协议（HTTP 402，按需付费）
- **ERC-8183** — AI Agent 结算标准（托管式雇佣→交付→结算）

---

## 学习路线推荐

### 路线 A：DApp 开发者（DeFi / NFT）
```
ERC-20 → EIP-712 → ERC-2612 → ERC-721 → ERC-1155 → ERC-4626
```

### 路线 B：基础设施开发者（钱包 / 工具）
```
ERC-20 → ERC-165 → EIP-712 → ERC-1271 → ERC-4337 → EIP-7702
```

### 路线 C：AI + Web3 开发者
```
ERC-20 → ERC-721 → EIP-712 → ERC-8004 → X402 → ERC-8183
```

---

## 文件索引

```
ERC(EIP)标准/
├── README.md              ← 你在这里
│
├── 独立标准文档 (14 个)
├── ERC-20.md              # Token Standard
├── ERC-165.md             # Standard Interface Detection
├── EIP-712.md             # Typed Structured Data Signing
├── ERC-721.md             # Non-Fungible Token
├── ERC-1155.md            # Multi Token Standard
├── ERC-1271.md            # Contract Signature Validation
├── ERC-2612.md            # Permit Extension
├── ERC-2981.md            # NFT Royalty
├── ERC-4337.md            # Account Abstraction
├── ERC-4626.md            # Tokenized Vault
├── EIP-7702.md            # Set EOA Account Code
├── ERC-8004.md            # Trustless Agents
├── X402.md                # Internet-Native Payment
├── ERC-8183.md            # Agentic Commerce
│
├── 领域汇总文档 (5 个)
├── dex-uniswap.md         # DEX 领域
├── lending-aave.md        # 借贷领域
├── nft-opensea.md         # NFT 领域
├── wallet-metamask.md     # 钱包领域
└── ai-web3.md             # AI+Web3 领域
```

---

## 外部资源

- [EIPs 官方网站](https://eips.ethereum.org/all) — 所有 EIP 的官方列表
- [GitHub ERCs](https://github.com/ethereum/ERCs) — ERC 标准仓库
- [GitHub EIPs](https://github.com/ethereum/EIPs) — EIP 标准仓库
- [ethereum.org 开发者文档](https://ethereum.org/zh/developers/docs/) — 以太坊官方中文开发者指南
- [OpenZeppelin Contracts](https://github.com/OpenZeppelin/openzeppelin-contracts) — 最广泛使用的标准实现
