# NFT 市场（OpenSea）— ERC/EIP 标准应用指南

## 领域概述

NFT 市场（以 OpenSea 为代表）是数字艺术品、收藏品、虚拟地产、游戏道具等非同质化资产的交易平台。核心流程包括：NFT 铸造（mint）、市场挂单（listing）、交易撮合（matching）、版税分配（royalty）。这是涉及 ERC 标准最多的单一领域，共 6 个标准协同工作。

## 标准总览

| 标准       | 在 NFT 市场中的角色                                  | 详细文档 |
| ---------- | ---------------------------------------------------- | -------- |
| **ERC-721** | NFT 的核心标准——所有权、转移、元数据                  | [ERC-721.md](ERC-721.md) |
| **ERC-1155** | 多代币标准——游戏道具、会员卡、批量资产                | [ERC-1155.md](ERC-1155.md) |
| **ERC-165** | 检测合约支持的接口——市场交互前的安全检查              | [ERC-165.md](ERC-165.md) |
| **ERC-2981** | 版税标准——创作者在二级市场转售中持续获益              | [ERC-2981.md](ERC-2981.md) |
| **ERC-1271** | 合约签名验证——Gnosis Safe 等智能钱包的挂单签名       | [ERC-1271.md](ERC-1271.md) |
| **EIP-712** | 安全的链下挂单签名——防钓鱼、展示可读信息              | [EIP-712.md](EIP-712.md) |

## 标准详解

### ERC-721：NFT 的基石

NFT 市场的一切围绕 ERC-721 展开：

```solidity
// OpenSea Seaport 协议中完成一笔 NFT 交易
function executeOrder(Order calldata order, bytes calldata signature) external {
    // 1. 验证卖家签名（EIP-712）
    // 2. 检查版税（ERC-2981）
    // 3. 验证接口支持（ERC-165：确认是 ERC-721）
    // 4. 转移 NFT
    IERC721(order.token).safeTransferFrom(order.maker, order.taker, order.tokenId);
    // 5. 分配支付（卖家金额 + 版税 + 市场费）
}
```

**ERC-721 的核心操作**：

| 操作             | 用途                                      |
| ---------------- | ----------------------------------------- |
| `tokenURI(id)`   | 获取 NFT 的元数据（图片、属性等）          |
| `ownerOf(id)`    | 查询 NFT 的当前持有者                      |
| `safeTransferFrom`| 转移 NFT（市场交易的核心执行函数）         |
| `setApprovalForAll`| 授权市场合约转移用户的所有 NFT            |

### ERC-1155：多资产的效率之选

当项目涉及多种代币类型时（如游戏），ERC-1155 比 ERC-721 更高效：

```
游戏道具系统示例：
  ERC-721 方案：1000 种道具 → 1000 个合约 → 巨大的部署 Gas
  ERC-1155 方案：1 个合约 → 1000 个 token ID → 经济高效

批量转移优势：
  ERC-721: 转 5 个 NFT → 5 笔交易
  ERC-1155: 转 5 个 NFT → 1 笔 safeBatchTransferFrom → 节省 ~60% Gas
```

### ERC-165：安全检测第一关

在调用任何 NFT 合约之前，市场必须先确认它支持什么接口：

```solidity
// 市场与 NFT 合约交互前的安全检测
function safeInteractWithNFT(address token) internal view {
    // 先确认是否支持 ERC-165 本身
    require(
        IERC165(token).supportsInterface(0x01ffc9a7),
        "Not ERC-165 compliant"
    );
    // 再确认是 ERC-721 还是 ERC-1155
    bool isERC721 = IERC165(token).supportsInterface(0x80ac58cd);
    bool isERC1155 = IERC165(token).supportsInterface(0xd9b67a26);
    require(isERC721 || isERC1155, "Unknown token type");
}
```

如果跳过 ERC-165 检测，市场可能尝试对非 NFT 合约调用 `safeTransferFrom`，导致意外行为或用户资产损失。

### ERC-2981：创作者经济的支柱

ERC-2981 让版税从"市场功能"变成"NFT 属性"：

```solidity
// 创作者部署 NFT 时设置版税
contract ArtistNFT is ERC721, ERC2981 {
    constructor() ERC721("ArtistNFT", "ART") {
        _setDefaultRoyalty(msg.sender, 500); // 5% 版税
    }
}

// 任意市场交易时查询并支付版税
function payRoyalty(address nft, uint256 tokenId, uint256 price) internal {
    (address receiver, uint256 amount) = IERC2981(nft).royaltyInfo(tokenId, price);
    // 版税随 NFT 转移，不依赖单个市场！
}
```

**版税 = 创作者的被动收入**：每笔二级交易中创作者自动获得版税（通常 2.5%-10%），无论 NFT 在哪个市场成交。

> 注意：2023 年以来版税执行力引发争议，部分市场（Blur、Sudoswap）采取了更灵活的版税策略。ERC-2981 是查询接口而非强制执行机制，市场可以选择性遵守。

### ERC-1271：智能钱包也能挂单

当 Gnosis Safe 等多签钱包持有 NFT 并想挂单时：

```solidity
// 市场验证挂单签名
function verifyListingSignature(
    address maker,      // Gnosis Safe 地址（合约地址）
    bytes32 orderHash,  // EIP-712 订单哈希
    bytes memory signature
) internal view returns (bool) {
    uint256 size;
    assembly { size := extcodesize(maker) }
    if (size > 0) {
        // 合约地址 → ERC-1271 验证
        return IERC1271(maker).isValidSignature(orderHash, signature) == 0x1626ba7e;
    } else {
        // EOA 地址 → 从 signature 中提取 v, r, s 后 ecrecover
        bytes32 r;
        bytes32 s;
        uint8 v;
        assembly {
            r := mload(add(signature, 0x20))
            s := mload(add(signature, 0x40))
            v := byte(0, mload(add(signature, 0x60)))
        }
        return ecrecover(orderHash, v, r, s) == maker;
    }
}
```

没有 ERC-1271，Gnosis Safe 用户无法在 OpenSea 等市场上挂单——这直接影响大量机构和高净值用户。

### EIP-712：告别盲签

NFT 市场是钓鱼攻击的重灾区。EIP-712 让用户清楚地看到他们正在签名什么：

```
传统盲签（危险）:
  "请签名以下数据: 0xa3f21b..." → 用户不知道在签什么

EIP-712（安全）:
  "您正在对以下订单签名:
   收藏品: Bored Ape Yacht Club #1234
   价格: 65 ETH
   过期时间: 2026-08-02 15:00
   市场: OpenSea
  "
```

## 标准间的协作关系

```
一次 NFT 交易的完整标准调用链：

  卖家挂单:
  ┌──────────────────────────────────────────┐
  │ EIP-712: 结构化订单签名                   │
  │   - 收藏品名称、Token ID、价格、过期时间    │
  │                                          │
  │ ERC-1271: 如果卖家是合约钱包               │
  │   - Gnosis Safe 多签验证                  │
  └──────────────────────────────────────────┘
                    ↓
  买家成交:
  ┌──────────────────────────────────────────┐
  │ ERC-165: 检测 NFT 合约支持的接口           │
  │   - 确认是 ERC-721                        │
  │   - 确认是否支持 ERC-2981（版税）          │
  │   - 确认是否支持元数据                     │
  │                                          │
  │ ERC-2981: 查询版税信息                     │
  │   - 版税接收方、版税金额                   │
  │                                          │
  │ ERC-721: 转移 NFT                         │
  │   - safeTransferFrom(seller, buyer, id)  │
  │                                          │
  │ ERC-20: 支付（如果是 ERC-20 计价）         │
  └──────────────────────────────────────────┘
```

## ERC-721 vs ERC-1155 选择指南

| 场景                         | 推荐      | 原因                              |
| ---------------------------- | --------- | --------------------------------- |
| 数字艺术品（每件唯一）        | ERC-721   | 生态成熟，市场支持最完善           |
| PFP 系列（10K 收藏品）        | ERC-721   | 每个 token 有独立元数据            |
| 游戏道具（武器/药水/金币）    | ERC-1155  | 单合约多资产，批量转移节省 Gas     |
| 限量会员卡（1000 张相同）     | ERC-1155  | 半同质化—每张卡相同，但限量        |
| 活动门票（同一票价/座位）     | ERC-1155  | 批量分发，一次性扫码核销           |

## 学习路径建议

1. **先学 ERC-165** — 理解接口检测是安全交互的前提
2. **再学 ERC-721** — 掌握 NFT 的核心接口和所有权模型
3. **再学 EIP-712** — 理解安全的链下签名
4. **然后学 ERC-1271** — 理解合约钱包如何参与 NFT 市场
5. **然后学 ERC-2981** — 理解版税的标准化
6. **最后学 ERC-1155** — 可选，游戏和多资产场景

## 官方资源

- [OpenSea 开发者文档](https://docs.opensea.io/)
- [Seaport 协议](https://github.com/ProjectOpenSea/seaport)
- [Blur 市场](https://blur.io/)
- [EIP-721 官方规范](https://eips.ethereum.org/EIPS/eip-721)
