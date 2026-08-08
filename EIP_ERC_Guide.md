# 以太坊改进提案（EIP）完全指南

> 本文档系统梳理 EIP 与 ERC 的概念、分类、仓库结构及常用编号对照，适合开发者快速查阅。

---

## 一、EIP 是什么？

**EIP（Ethereum Improvement Proposal，以太坊改进提案）** 是以太坊生态所有改进提案的**总称和统一入口**。

- 它定义了以太坊协议、客户端 API、合约接口标准、流程规范等一切"该怎么做事"的规则
- 每份 EIP 有一个唯一编号（如 EIP-1559、EIP-2612）
- 状态分为：`Draft` → `Review` → `Last Call` → `Final` / `Stagnant` / `Withdrawn`
- 核心流程宪法：**EIP-1**

> 📌 **所有 ERC 都是 EIP，但不是所有 EIP 都是 ERC。**

---

## 二、EIP 分类体系（EIP-1 规定）

EIP 分为 **三大类**，其中 Standards Track 再拆 **四个子类**：

```
EIP（总集）
├── Standards Track（标准追踪类）
│   ├── Core（核心协议层）
│   ├── Networking（网络层/P2P）
│   ├── Interface（接口层）
│   │   └── ERC（应用层标准，挂在 Interface 下）
│   └── ERC（以太坊请求评议）
├── Meta（元流程类）
└── Informational（信息类）
```

### 1. Standards Track（标准追踪类）

| 子类 | 作用 | 典型例子 |
|---|---|---|
| **Core** | 共识层/执行层协议改动 | EIP-1559（Gas 改革）、EIP-3675（PoS 合并）、EIP-4844（Blob 数据） |
| **Networking** | devp2p、Light ETH 子协议 | EIP-8（RLPx 协议） |
| **Interface** | 客户端 RPC、ABI、语言级约定 | EIP-1474（JSON-RPC 规范） |
| **ERC** | 应用层/合约接口标准 | ERC-20、ERC-721、ERC-2612、ERC-4337 |

### 2. Meta（元流程类）
- 修改 EIP 流程本身的提案
- 如 EIP-1（流程宪法）、EIP-5069（编辑手册）、EIP-7329（EIP/ERC 仓库拆分）

### 3. Informational（信息类）
- 提供设计指南、安全建议，不强制
- 如 EIP-1470（Solidity 安全建议）

---

## 三、ERC 是什么？

**ERC（Ethereum Request for Comments，以太坊请求评议）** 是 EIP 体系中**专门管应用层/合约接口标准**的子集。

### 核心特征
- 挂在 `Standards Track → Interface` 子类下
- 编号与 EIP 共享序列（ERC-20 = EIP-20）
- 定义**链上应用互操作规则**：合约该有哪些 function/event、钱包该怎么解析
- 社区口语习惯说 "ERC-20"，正式文档头部写 `EIP: 20` 也对

### ERC 常见领域
| 领域 | 典型 ERC |
|---|---|
| 同质化代币 | ERC-20 |
| 非同质化代币 | ERC-721 |
| 多代币标准 | ERC-1155 |
| 签名授权 | ERC-2612（Permit）、ERC-3009 |
| 账户抽象 | ERC-4337 |
| 代币化金库 | ERC-4626 |
| 可升级代理 | ERC-1967、ERC-1822 |

---

## 四、EIP vs ERC 对比表

| 维度 | EIP | ERC |
|---|---|---|
| 定位 | 所有以太坊改进提案的总集 | EIP 的子集，专管应用层标准 |
| 范围 | 协议、网络、接口、流程 | 仅合约接口/应用互操作 |
| 编号 | EIP-1、EIP-1559、EIP-2612 | ERC-20（=EIP-20）、ERC-721（=EIP-721） |
| 举例 | EIP-1559（Gas）、EIP-4844（Blob） | ERC-20（代币）、ERC-4337（AA） |
| 存放位置 | `ethereum/EIPs`（Core/Networking/Meta） | `ethereum/ERCs`（2023 年拆分后） |

---

## 五、仓库拆分历史（EIP-7329）

### 背景
早期所有 EIP 都堆在 `ethereum/EIPs` 一个仓库，编号连续。后来生态变大，核心开发者和应用开发者诉求分化。

### EIP-7329 拆分规则（2023 年）

| 仓库 | 存放内容 | 编号规则 |
|---|---|---|
| `ethereum/EIPs` | Core、Networking、Meta、Informational | 偶数编号 |
| `ethereum/ERCs` | 所有 ERC 和 Interface 类标准 | 奇数编号 |

### 拆分后行为
- 历史 PR 保留在 `ethereum/EIPs`
- 老链接 `eips.ethereum.org/EIPS/eip-20` **自动重定向**到 ERC 站点
- ERC 新家：`ercs.ethereum.org`

---

## 六、官方查询链接

| 用途 | 链接 |
|---|---|
| EIP 全量列表（含重定向） | https://eips.ethereum.org/all |
| EIP 主仓库（GitHub） | https://github.com/ethereum/EIPs |
| ERC 专用仓库（GitHub） | https://github.com/ethereum/ERCs |
| ERC 站点 | https://ercs.ethereum.org |
| EIP-1（流程宪法） | https://eips.ethereum.org/EIPS/eip-1 |
| EIP-7329（拆分规则） | https://eips.ethereum.org/EIPS/eip-7329 |

---

## 七、常用 ERC ↔ EIP 编号对照表

| 编号 | 名称 | 领域 | 一句话说明 |
|---|---|---|---|
| ERC-20 / EIP-20 | Token Standard | 同质化代币 | `transfer/approve/transferFrom/allowance` |
| ERC-721 / EIP-721 | Non-Fungible Token | NFT | 每个 tokenId 唯一，`ownerOf/tokenURI` |
| ERC-1155 / EIP-1155 | Multi Token | 多代币 | 单合约管理同质+非同质代币 |
| ERC-2612 / EIP-2612 | Permit Extension | 签名授权 | EIP-712 签名代替 `approve()` 交易 |
| ERC-3009 / EIP-3009 | Transfer Authorization | 签名转账 | 直接签 transfer（非改 allowance） |
| ERC-4337 / EIP-4337 | Account Abstraction | 账户抽象 | 用 UserOperation + Bundler 实现智能钱包 |
| ERC-4626 / EIP-4626 | Tokenized Vault | 金库标准 | DeFi 收益金库统一接口 |
| ERC-1967 / EIP-1967 | Proxy Storage Slots | 可升级代理 | 定义代理合约的存储槽位 |
| ERC-1822 / EIP-1822 | Universal Upgradeable Proxy | 通用代理 | UUPS 可升级代理模式 |
| ERC-3155 / EIP-3155 | Debug Tracing | 调试追踪 | 标准化调试追踪输出格式 |
| ERC-1167 / EIP-1167 | Minimal Proxy | 轻量代理 | 极简 clone 代理（用于工厂合约） |
| ERC-1271 / EIP-1271 | Signature Validation | 合约签名验证 | 合约钱包验证 EIP-712 签名的标准接口 |

---

## 八、常用 Core EIP 速查

| 编号 | 名称 | 一句话说明 |
|---|---|---|
| EIP-155 | Replay Protection | 用 chainId 防止跨链重放交易 |
| EIP-1559 | Fee Market | 基础费 + 小费机制，BASEFEE 自动调节 |
| EIP-3675 | PoS Consensus | 将以太坊从 PoW 转为 PoS |
| EIP-4844 | Proto-Danksharding | Blob 数据，降低 L2 费用 |
| EIP-712 | Typed Structured Data Signing | 带类型的结构化数据签名（Permit 的基础） |
| EIP-1014 | CREATE2 | 确定性合约地址部署 |
| EIP-1820 | Registry Contract | 合约接口注册表（ERC-777 依赖） |

---

## 九、实战判断：什么时候叫 EIP，什么时候叫 ERC？

| 场景 | 叫法 |
|---|---|
| 改以太坊本身（Gas、opcode、区块、共识、P2P） | **EIP**（如 EIP-1559） |
| 定链上应用互操作规则（合约 function/event） | **ERC**（如 ERC-20） |
| 写 Solidity 合约"要实现某个标准" | 引用 **ERC 编号** |
| 做节点/客户端/硬分叉讨论 | 引用 **EIP 编号** |
| 同一篇文档头部 | 写 `EIP: 20` 或 `ERC: 20` 都对 |

---

## 十、EIP-2612 Permit 标准速查（附录）

### Permit 消息结构（EIP-712 强制契约）
```
Permit(
    address owner,
    address spender,
    uint256 value,
    uint256 nonce,
    uint256 deadline
)
```

### 强制实现的 3 个新函数
```solidity
function permit(address owner, address spender, uint256 value,
                uint256 deadline, uint8 v, bytes32 r, bytes32 s) external;
function nonces(address owner) external view returns (uint256);
function DOMAIN_SEPARATOR() external view returns (bytes32);
```

### PERMIT_TYPEHASH（一字不差）
```solidity
keccak256("Permit(address owner,address spender,uint256 value,uint256 nonce,uint256 deadline)")
```

### DOMAIN_SEPARATOR 推荐构造
```solidity
keccak256(abi.encode(
    keccak256("EIP712Domain(string name,string version,uint256 chainId,address verifyingContract)"),
    keccak256(bytes(name)),
    keccak256(bytes(version)),
    block.chainid,
    address(this)
));
```

---

> 📝 **文档维护建议**：以太坊标准迭代很快，查询时始终以官方站点最新状态为准。本文档整理自 2026 年 8 月，部分 ERC 编号和状态后续可能更新。
