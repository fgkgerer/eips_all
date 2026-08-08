# ERC-20: Token Standard（同质化代币标准）

## 基本信息

| 属性       | 值                                                            |
| ---------- | ------------------------------------------------------------- |
| 编号       | EIP-20 / ERC-20                                               |
| 标题       | Token Standard                                                |
| 状态       | Final                                                         |
| 类型       | Standards Track                                               |
| 分类       | ERC                                                           |
| 作者       | Fabian Vogelsteller, Vitalik Buterin                          |
| 创建日期   | 2015-11-19                                                    |
| 依赖标准   | 无                                                            |
| 被依赖标准 | ERC-2612, ERC-4626, ERC-777, ERC-1363, ERC-721 (参考) 等     |

## 概述

ERC-20 是以太坊上最基础、最广泛使用的**同质化代币（Fungible Token）**标准接口。它定义了一套统一的 API，使得所有符合该标准的代币可以无缝地在钱包、交易所、DApp 之间互操作。同质化意味着每个代币单位完全相同且可互换（就像法定货币一样）。

## 核心动机

在 ERC-20 出现之前，每个区块链项目都自定义代币合约的接口——有的叫 `send()`，有的叫 `transfer()`，参数顺序和返回值也各不相同。这导致：

- **钱包**无法通用地显示代币余额
- **交易所**需要为每个代币编写定制的集成代码
- **DApp** 无法组合使用不同代币

ERC-20 通过标准化解决了这个问题，成为以太坊生态互操作性的基石。

## 技术规范

### 接口定义

```solidity
// EIP-20: Token Standard
interface IERC20 {
    // ========== 必须实现 ==========

    /// @notice 返回代币总供应量
    function totalSupply() external view returns (uint256);

    /// @notice 返回地址的余额
    function balanceOf(address account) external view returns (uint256);

    /// @notice 从调用者向 to 转账 value 数量的代币
    /// @return 成功返回 true
    function transfer(address to, uint256 value) external returns (bool);

    /// @notice 返回 owner 授权给 spender 的额度
    function allowance(address owner, address spender) external view returns (uint256);

    /// @notice 授权 spender 可动用调用者最多 value 数量的代币
    /// @return 成功返回 true
    function approve(address spender, uint256 value) external returns (bool);

    /// @notice 从 from 向 to 转账 value 数量的代币（需要事先授权）
    /// @return 成功返回 true
    function transferFrom(address from, address to, uint256 value) external returns (bool);

    // ========== 事件 ==========
    event Transfer(address indexed from, address indexed to, uint256 value);
    event Approval(address indexed owner, address indexed spender, uint256 value);
}

// ========== 可选实现 ==========
interface IERC20Metadata {
    function name() external view returns (string memory);
    function symbol() external view returns (string memory);
    function decimals() external view returns (uint8);
}
```

### 核心机制

ERC-20 的核心是 **"授权-转账"（approve-transferFrom）两步模型**：

```
┌──────────┐                    ┌──────────┐
│  用户 A   │                    │  用户 B   │
│ (owner)  │                    │(spender) │
└────┬─────┘                    └────┬─────┘
     │                               │
     │  1. approve(B, 100)          │
     │─────────────────────────────>│  链上记录: allowance[A][B] = 100
     │                               │
     │                               │  2. transferFrom(A, C, 50)
     │                               │─────────────────────────────>
     │                               │  检查 allowance[A][B] >= 50
     │                               │  扣除 allowance, 执行转账
     │                               │
     │  3. approve(B, 0)            │  安全实践: 先清零再设新值
     │─────────────────────────────>│  （防止前置攻击）
```

这种机制支撑了 DEX 的核心流程：用户先 `approve` 授权 Uniswap 路由器，路由器再通过 `transferFrom` 拉取代币完成 swap。

## 应用领域

### 在 DEX（Uniswap）中的应用

ERC-20 在 DEX 中有三层用途：

1. **交易资产**：所有交易对的基础代币（USDC、WETH、DAI 等）
2. **LP Token**：流动性提供者凭证本身就是 ERC-20 代币
3. **路由机制**：Uniswap 路由器通过 `transferFrom` 拉取用户代币完成 swap

```solidity
// Uniswap V2 路由器的典型流程
contract UniswapV2Router02 {
    // 用户已 approve 路由器足够的代币额度
    function swapExactTokensForTokens(
        uint amountIn,
        uint amountOutMin,
        address[] calldata path,
        address to,
        uint deadline
    ) external returns (uint[] memory amounts) {
        // 1. 从用户拉取输入代币
        TransferHelper.safeTransferFrom(
            path[0], msg.sender, pair, amountIn
        );
        // 2. 执行 swap
        // 3. 将输出代币发送给用户
    }
}
```

### 在借贷（Aave）中的应用

- **aToken**（如 aUSDC）是 ERC-20 代币，代表用户的存款凭证，余额随时间增长（利息累积）
- 存款时用户 `approve` Aave 协议，协议通过 `transferFrom` 拉取代币
- 借款也是以 ERC-20 代币形式发放

## 与其他标准的关系

| 相关标准    | 关系类型   | 说明                                                |
| ----------- | ---------- | --------------------------------------------------- |
| **ERC-2612** | 扩展       | 用 EIP-712 链下签名替代 `approve` 交易，解决两步授权痛点 |
| **ERC-777**  | 扩展/替代 | 增加钩子（hooks）机制，在转账前后通知接收方          |
| **ERC-1363** | 扩展       | 支持 `transferAndCall`，在转账同时附带数据调用       |
| **ERC-4626** | 依赖       | 金库的 `asset()` 返回的是 ERC-20 代币地址            |

## 安全考虑

### 1. approve 前置攻击（Front-running）

这是 ERC-20 最著名的安全问题。当用户想把授权额度从 100 改到 200 时：

- 攻击者监控内存池（mempool），在用户新 `approve` 确认前抢跑 `transferFrom` 消耗旧的 100 额度
- 用户的 `approve(B, 200)` 确认后，攻击者再消耗 200 额度
- 总计损失 300

**缓解方案**：
- 使用 `increaseAllowance` / `decreaseAllowance` 而非 `approve`
- 使用 ERC-2612 Permit 机制（链下签名，一次性使用）
- 先 `approve(B, 0)` 再 `approve(B, 200)`（需两笔交易，但攻击者无法在中间插足）

### 2. 无限授权（Infinite Approval）

许多用户为方便对 DEX 授予 `type(uint256).max` 额度。如果合约有漏洞，攻击者可耗尽用户所有代币余额。

### 3. 返回值处理

部分代币（如 USDT）的 `transfer` 不返回 `bool`，需要特殊处理。OpenZeppelin 的 `SafeERC20` 库可统一处理此类兼容性问题。

## 官方资源

- [EIP-20: Token Standard](https://eips.ethereum.org/EIPS/eip-20)
- [GitHub ERCs](https://github.com/ethereum/ERCs/blob/master/ERCS/erc-20.md)
- [OpenZeppelin ERC-20 实现](https://github.com/OpenZeppelin/openzeppelin-contracts/blob/master/contracts/token/ERC20/ERC20.sol)
- [ethereum.org ERC-20 指南](https://ethereum.org/zh/developers/docs/standards/tokens/erc-20/)
