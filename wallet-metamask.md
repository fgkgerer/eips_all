# 钱包（MetaMask）— ERC/EIP 标准应用指南

## 领域概述

钱包（以 MetaMask 为代表）是以太坊用户与 DApp 交互的入口。从最初的简单 EOA（私钥 + 地址）管理，到现在的智能账户、Passkey 签名、代付 Gas、批量交易——钱包正在经历从"钥匙串"到"智能银行"的范式转变。ERC-4337 和 EIP-7702 是这一转变的核心推动力。

## 标准总览

| 标准       | 在钱包中的角色                                         | 详细文档 |
| ---------- | ------------------------------------------------------ | -------- |
| **ERC-4337** | 账户抽象基础设施——智能账户、Bundler、Paymaster          | [ERC-4337.md](ERC-4337.md) |
| **EIP-7702** | EOA 原地升级为智能账户——保留地址、加入 4337 生态        | [EIP-7702.md](EIP-7702.md) |

## 标准详解

### ERC-4337：智能账户的操作系统

#### 为什么需要账户抽象？

传统 EOA 钱包的局限：

```
你的 EOA 地址: 0x1234...abcd

能做什么？                      不能做什么？
✅ 用 ETH 付 Gas               ❌ 用 USDC 付 Gas
✅ ECDSA 签名                   ❌ 手机指纹/面容解锁
✅ 单笔交易                     ❌ approve + swap 一笔完成
✅ 查看私钥（可导出）            ❌ 丢失私钥后恢复
✅ 无限制转账                   ❌ 设置每日支出上限
```

ERC-4337 改变了这一切：

```solidity
// 智能账户可以做的事（通过 validateUserOp 自定义规则）

function validateUserOp(UserOperation calldata op, ...) external returns (uint256) {
    // 注：实际需从 op.callData 中解析目标合约地址和转账金额
    // (address opTarget, uint256 opValue) = abi.decode(op.callData[4:], (...));

    // 规则 1: Passkey 验证（苹果/安卓生物识别）
    require(verifyWebAuthn(op.signature), "Invalid passkey");

    // 规则 2: 日支出限额
    require(dailySpent + opValue <= dailyLimit, "Daily limit exceeded");

    // 规则 3: 仅允许白名单 DApp
    require(whitelisted[opTarget], "Untrusted contract");

    // 规则 4: 检查监护人签名（社交恢复）
    if (opValue > largeAmount) {
        require(verifyGuardian(op.signature), "Need guardian approval");
    }
    return 0;
}
```

#### 钱包如何集成 ERC-4337？

```
MetaMask 智能账户架构:

  ┌─────────────────────────────────────┐
  │         MetaMask 钱包界面            │
  │  - 管理多个智能账户                   │
  │  - Passkey / 指纹签名                 │
  │  - 批量交易确认（approve + swap）     │
  │  - Gas 估算 & Paymaster 配置         │
  └──────────────┬──────────────────────┘
                 │ 签名 UserOperation
                 ▼
  ┌──────────────────────────────────────┐
  │         Bundler (Alchemy/Pimlico)    │
  │  - 收集 UserOperation                │
  │  - 模拟验证                          │
  │  - 打包成交易                         │
  │  - 调用 EntryPoint.handleOps()       │
  └──────────────┬───────────────────────┘
                 │
                 ▼
  ┌──────────────────────────────────────┐
  │         EntryPoint (链上合约)         │
  │  - 验证每个 UserOperation            │
  │  - 调用各账户的 validateUserOp       │
  │  - 执行 callData                     │
  │  - 与 Paymaster 结算 Gas             │
  └──────────────────────────────────────┘
```

#### Paymaster：Gas 不再只是 ETH

```solidity
// ERC-20 Paymaster: 用户用 USDC 支付 Gas
contract ERC20Paymaster is IPaymaster {
    function validatePaymasterUserOp(UserOperation calldata op, ...)
        external returns (bytes memory context, uint256 validationData) {
        // 1. 解析 UserOp 中的 USDC 支付信息
        // 2. 如果同意代付 → 返回验证通过
        // 3. EntryPoint 会从 Paymaster 的存款中扣除 Gas
        return (context, 0);
    }
    // Paymaster 定期结算 USDC 收入
}
```

支付 Gas 的方式：
- **ETH**：最传统的方式
- **USDC**：通过 ERC-20 Paymaster
- **DApp 赞助**：DApp 为新用户支付 Gas（无 Gas onboarding）
- **订阅模式**：钱包服务商提供月费 Gas 套餐

### EIP-7702：EOA 的升级之路

#### 核心问题：EOA 如何获得智能能力？

```
现状：数亿个 EOA 地址存在，持有大量资产
目标：让这些 EOA 获得 ERC-4337 智能账户的全部能力
难点：不能换地址（ENS、授权、历史记录都绑定在原地址上）
```

EIP-7702 是答案——**原地址升级**：

```solidity
// 用户: 发送一笔 EIP-7702 交易
//   授权: EOA.privateKey 签名 → delegation to SmartWalletImpl
//   结果: 用户的 0x1234... 地址现在可以执行 SmartWalletImpl 的代码！

// 升级后的 EOA 同时拥有:
//   - 私钥签名（传统方式，向后兼容）
//   - 智能合约代码（批量交易、Passkey、代付 Gas...）
```

#### MetaMask 的 Delegation Toolkit

MetaMask 基于 EIP-7702 构建了用户友好的升级流程：

```
1. MetaMask 弹出"升级"提示
   → 清晰列出新增能力 + 保留的一切

2. 用户签名授权（一笔交易）
   → EIP-7702 SetCodeTransaction

3. 升级完成
   → 同一地址，同样的 ETH/代币/NFT/ENS
   → 新功能：批量交易、Passkey、Paymaster...

4. 如果用户需要更高安全性
   → 升级到 EIP-7702 后自动兼容 ERC-4337
   → 使用 Bundler + EntryPoint 的完整基础设施
```

### ERC-4337 + EIP-7702：完整的升级路径

```
普通 EOA 用户
    │
    │ EIP-7702 授权
    ▼
智能 EOA (同一地址 + 合约代码)
    │
    │ 自动兼容
    ▼
ERC-4337 智能账户 (Bundler + Paymaster + EntryPoint)

最终效果：
  - 保留原地址、原资产、原 ENS
  - 获得智能账户全部能力
  - 零迁移成本
```

## 标准间的协作关系

```
MetaMask 交易的完整流程：

  用户操作
    │
    ├── 简单交易（EOA 模式）:
    │   EIP-7702 委托代码 → 扩展 EOA 能力
    │
    └── 复杂交易（智能账户模式）:
        ERC-4337 UserOperation
          ├── EIP-712 签名格式（安全、可读）
          ├── Paymaster 代付 Gas（USDC / DApp 赞助）
          ├── Bundler 打包上链
          └── EntryPoint 统一执行入口
```

## 学习路径建议

1. **先理解 EOA 的局限** — 为什么需要改变
2. **深入 ERC-4337** — 理解 UserOperation、Bundler、Paymaster、EntryPoint 四个核心组件
3. **再学 EIP-7702** — 理解它如何与 ERC-4337 互补，如何让 EOA 原地升级
4. **实际体验** — 在 MetaMask 中创建智能账户，体验批量交易和 Passkey 签名

## 官方资源

- [MetaMask 智能账户](https://metamask.io/)
- [ERC-4337 官方实现](https://github.com/eth-infinitism/account-abstraction)
- [EIP-7702 解析（Web3Auth）](https://blog.web3auth.io/eip-7702-explained-how-it-works-and-everything-you-need-to-know/)
- [EIP-7702 攻击面分析（Nethermind）](https://www.nethermind.io/blog/eip-7702-attack-surfaces-what-developers-should-know)
- [Account Abstraction in 2026（Thirdweb）](https://blog.thirdweb.com/account-abstraction-in-2026-how-eip-7702-and-erc-4337-are-transforming-ethereum-wallets-for-developers/)
