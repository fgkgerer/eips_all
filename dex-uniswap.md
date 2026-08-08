# DEX（去中心化交易所）— ERC/EIP 标准应用指南

## 领域概述

去中心化交易所（DEX，以 Uniswap 为代表）通过智能合约实现无需中介的代币兑换。用户保留资产托管权，通过流动性池（AMM）自动做市。DEX 需要处理代币转账、流动性管理、链下签名授权和订单签名等核心操作，依赖以下标准。

## 标准总览

| 标准       | 在 DEX 中的角色                                | 详细文档 |
| ---------- | ---------------------------------------------- | -------- |
| **ERC-20** | 所有交易资产和 LP Token 的基础标准              | [ERC-20.md](ERC-20.md) |
| **ERC-2612** | 无 Gas 授权，实现"一次签名完成 approve+swap"   | [ERC-2612.md](ERC-2612.md) |
| **ERC-721** | Uniswap V3 流动性仓位的 NFT 化                 | [ERC-721.md](ERC-721.md) |
| **EIP-712** | 订单签名和 Permit 签名的安全可读格式            | [EIP-712.md](EIP-712.md) |

## 标准详解

### ERC-20：DEX 的血液

ERC-20 是 DEX 上所有资产的通用语言。每一笔 swap 都是 ERC-20 代币的转移。

**典型流程（Uniswap V2 swap）**：

```solidity
// 用户已 approve 路由器
// 1. 路由器拉取用户代币
tokenIn.transferFrom(msg.sender, pair, amountIn);
// 2. AMM 计算输出金额
// 3. 路由器向用户发送输出代币
tokenOut.transfer(msg.sender, amountOut);
```

**关键角色**：
- 交易代币（USDC、ETH、DAI 等）
- LP Token（流动性提供凭证，本身也是 ERC-20）
- `approve` + `transferFrom` 是 swap 的前置步骤

### ERC-2612 + Permit2：Gas 优化的授权方案

ERC-2612 是代币层面的标准——让 ERC-20 代币原生支持链下签名授权（`permit()`）。但多数已部署的代币（如 USDC、DAI）并未实现 ERC-2612。

Uniswap 的 Permit2 解决了这个问题：它作为一个**中间合约**，让任何 ERC-20 都能享受无 Gas 授权的便利。用户只需 approve Permit2 一次，后续所有操作都可以通过签名完成。

**Permit2 的创新**：

```
传统流程（2 笔交易）:
  ① USDC.approve(router, max)     → Gas: ~45k
  ② router.swapExactTokensForTokens  → Gas: ~150k
  总计: ~195k Gas, 2 次确认

Permit2 流程（1 笔交易 + 1 次签名）:
  ① 用户链下签名（免费）
  ② router.permit + swap           → Gas: ~150k
  总计: ~150k Gas, 1 次确认
```

Permit2 还支持：
- **可过期额度**：签名设定的额度在一定时间后自动失效
- **额度上限**：不像 `approve(max)` 那样无限制
- **跨 DApp 共享**：一次签名授权，多个 DApp 可用

### ERC-721：Uniswap V3 的流动性 NFT

Uniswap V3 的集中流动性仓位的独特性使 ERC-721 成为最佳表示方式：

```solidity
// 每个 Uniswap V3 仓位 = 一个 NFT
struct Position {
    uint96 nonce;
    address operator;
    address token0;
    address token1;
    uint24 fee;          // 0.05%, 0.30%, 1.00%
    int24 tickLower;     // 价格下限
    int24 tickUpper;     // 价格上限
    uint128 liquidity;
    // ...
}
```

这种设计带来新可能：
- **流动性仓位可转让**：NFT 可以直接交易
- **仓位可抵押**：作为其他借贷协议的抵押品
- **仓位可组合**：不同协议的流动性管理策略

### EIP-712：安全的链下签名

DEX 中所有需要用户签名的操作都基于 EIP-712：

| 操作               | 签名内容                                           |
| ------------------ | -------------------------------------------------- |
| ERC-2612 Permit    | `Permit(owner, spender, value, nonce, deadline)`   |
| Permit2 授权       | `PermitBatch(permit details...)`                   |
| 限价单（Limit Order）| `Order(maker, tokenIn, tokenOut, amount, price)`  |
| UniswapX 订单      | `DutchOrder(...)`                                  |

钱包（MetaMask）解析 EIP-712 类型后，用户看到的是清晰可读的信息："授权 Uniswap 使用最多 1000 USDC"而非一串十六进制。

## 标准间的协作关系

```
一次典型的 Uniswap swap 涉及的标准流水线：

  ERC-2612          EIP-712           ERC-20
  (用户签名)  ←──  (签名格式)    ──→  (代币转账)
      │                                 │
      │ 1. 用户链下                     │ 3. swap 执行
      │    permit 签名                  │    代币转移
      │                                │
      ▼                                ▼
  ┌─────────────────────────────────────────┐
  │         Uniswap 路由器合约               │
  │  2. 验证 permit 签名                    │
  │  3. transferFrom 拉取代币               │
  │  4. AMM 计算兑换率                      │
  │  5. transfer 输出代币给用户              │
  └─────────────────────────────────────────┘
```

## 学习路径建议

1. **先学 ERC-20** — 理解代币的 approve/transferFrom 模型，这是 DEX 的基础
2. **再学 EIP-712** — 理解链下签名的安全格式
3. **然后学 ERC-2612** — 理解 Permit 如何将两步合并为一步
4. **最后学 ERC-721** — 理解 Uniswap V3 的仓位设计

## 官方资源

- [Uniswap V2 文档](https://docs.uniswap.org/contracts/v2/overview)
- [Uniswap V3 文档](https://docs.uniswap.org/contracts/v3/overview)
- [Uniswap Permit2](https://github.com/Uniswap/permit2)
