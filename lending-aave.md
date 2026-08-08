# 借贷协议（Lending）— ERC/EIP 标准应用指南

## 领域概述

去中心化借贷协议（以 Aave 为代表）允许用户存入资产获得利息，或通过超额抵押借出资产。核心业务包括存款（deposit）、借款（borrow）、还款（repay）、清算（liquidation），以及利率的动态调整。借贷协议需要处理生息代币、质押凭证、金库统一接口和签名授权等操作。

## 标准总览

| 标准       | 在借贷中的角色                                       | 详细文档 |
| ---------- | ---------------------------------------------------- | -------- |
| **ERC-20** | 存款资产和生息凭证（aToken/cToken）的基础标准          | [ERC-20.md](ERC-20.md) |
| **ERC-2612** | 无 Gas 存款——用户无需 ETH 即可存入资产               | [ERC-2612.md](ERC-2612.md) |
| **ERC-4626** | 统一生息金库接口——任何协议可无信任接入 Aave           | [ERC-4626.md](ERC-4626.md) |
| **EIP-712** | 治理投票和授权操作的链下安全签名                      | [EIP-712.md](EIP-712.md) |

## 标准详解

### ERC-20：借贷的双重角色

在借贷协议中，ERC-20 扮演了两个关键角色：

**1. 底层资产（存款/借款的计价单位）**

```solidity
// Aave 存款流程
// 用户 approve Aave 协议使用 USDC
USDC.approve(aavePool, 1000e6);
// 协议拉取代币
aavePool.deposit(USDC, 1000e6, user, 0);
```

**2. 生息凭证（aToken）**

```solidity
// aToken 是特殊的 ERC-20：
// - 余额随时间自动增长（利息累积）
// - 可以像普通 ERC-20 一样转账
// - 1 aUSDC 对应 1 USDC 的债权（汇率 1:1）

// 用户存入 1000 USDC → 获得 1000 aUSDC
// 一年后（假设 5% APY）→ 余额自动增长到 1050 aUSDC → 可赎回 1050 USDC
// 注：汇率始终 1:1，利息通过增加 aToken 余额来体现
```

### ERC-2612：新用户的第一道门

借贷协议的用户体验痛点：新用户只有 USDC，没有 ETH 用于 Gas。ERC-2612 解决了这个"鸡蛋问题"。

```
痛点场景：新用户 Alice
  - 持有 1000 USDC（在某个 EOA 地址）
  - 持有 0 ETH
  - 想要存入 Aave 赚利息

传统方案：
  ① 需要先获取 ETH（买/转账/用 CEX 提现）
  ② approve(aave, 1000 USDC)  → 需要 ETH 作为 Gas
  ③ aave.deposit(1000 USDC)     → 又需要 ETH 作为 Gas

ERC-2612 + Relayer 方案：
  ① Alice 链下对 Aave 签名（免费，不需要 ETH）
     - permit(USDC, aave, 1000)
     - deposit(Aave, 1000 USDC)
  ② Relayer 打包发送交易（Relayer 付 Gas）
     - USDC.permit(signature)
     - aave.deposit(USDC, 1000, alice)
  ③ Alice 持有 ETH 0 → 1000 aUSDC 在钱包！
```

Aave 和许多 DeFi 协议通过这种方式大幅降低了新用户的门槛。

### ERC-4626：金库的可组合性革命

Aave V3 的 aToken 适配了 ERC-4626，这意味着任何支持 ERC-4626 的协议都可以**零成本接入** Aave 的借贷池。

```solidity
// 一个通用的收益聚合器
contract YieldAggregator {
    // 它可以接入任何 ERC-4626 金库！
    // 无需为 Aave、Yearn、Compound 分别写代码
    function depositToBestVault(
        IERC4626 vault,
        uint256 assets
    ) external {
        // 1. 拉取用户资产
        IERC20(vault.asset()).transferFrom(msg.sender, address(this), assets);
        // 2. 存入金库
        IERC20(vault.asset()).approve(address(vault), assets);
        uint256 shares = vault.deposit(assets, msg.sender);
        // 3. 用户获得 shares（金库份额）
    }
}
```

这解锁了强大的可组合性：
- 钱包可以直接显示所有 ERC-4626 金库的 APY 对比
- 收益聚合器可以无信任地切换最优金库
- 结构化产品可以自由组合"存款+借款+再存款"

### EIP-712：治理的安全基础

Aave 的治理系统依赖 EIP-712 签名：

- **治理提案投票**：代币持有者对提案签名表决
- **委托投票权**：将投票权委托给其他地址
- **安全模块质押**：质押 AAVE 代币的授权签名

EIP-712 确保投票者清楚地看到他们投票的内容，而非对未知哈希签名。

## 标准间的协作关系

```
一次"存入 Aave 赚利息"的标准协作：

  EIP-712             ERC-2612
  (签名格式)  ←────  (permit 签名)
                        │
                        │ 1. 用户签名授权 + 存款指令
                        ▼
                   ┌─────────────┐
                   │   Relayer   │
                   │  (代付 Gas)  │
                   └──────┬──────┘
                          │ 2. 提交交易
                          ▼
  ERC-20           ┌──────────────┐        ERC-4626
  (底层资产) ─────>│  Aave Pool   │<───── (aToken 金库)
                   │              │
                   │  存款 → 铸造 aToken
                   │  借款 → 铸造债务
                   │  利率模型自动调整
                   └──────────────┘
```

## 学习路径建议

1. **先学 ERC-20** — 理解 approve/transferFrom 模型
2. **再学 ERC-4626** — 理解 shares/assets 的转换关系
3. **然后学 ERC-2612 + EIP-712** — 理解无 Gas 授权的签名流程
4. **最后研究 Aave 源码** — 看这些标准如何在实际协议中落地

## 官方资源

- [Aave V3 文档](https://docs.aave.com/developers/)
- [Aave 协议概述](https://aave.com/)
- [Yearn V3 Vaults (ERC-4626 实现)](https://docs.yearn.fi/)
