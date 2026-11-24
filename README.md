# Anubis Chain — 攻击面分析

Fork 自 `github.com/anubis-chain`（7 个官方仓库）。

## 组件（fork 保留在独立仓库）

| 仓库 | 内容 |
|---|---|
| [AEPs](https://github.com/futuredao0001-pixel/AEPs) | AEP-1: DAI 作为 gas 代币 |
| [anubis-faucet](https://github.com/futuredao0001-pixel/anubis-faucet) | 测试网水龙头 |
| [safe-smart-account](https://github.com/futuredao0001-pixel/safe-smart-account) | Safe v1.5.0 |
| [subgraph-starter](https://github.com/futuredao0001-pixel/subgraph-starter) | The Graph 子图 |

## 链参数

| 项 | Testnet | Mainnet |
|---|---|---|
| chainId | **202601** | **6714** |
| network id | 202601 | anubis |
| RPC | https://cheras-rpc.anubispace.org/rpc | — |
| Explorer | https://cheras-rpc.anubispace.org | — |
| Client | Geth/v1.0.0-10f78136-20261117/linux-amd64/go1.26.1 | — |
| 高度 | ~18,991,000 | — |

## AEP-1 核心机制（DAI 作为 gas）

- 创世即部署 DAI 系统合约 `0x83fd06F0846d9D90B3016bF670Efe2E0B11cDe14`
- `balanceOf` mapping 在 **storage slot 0**
- 协议**直接 GetState/SetState** 操作 balanceOf，**不走 EVM，不发 Transfer 事件**
- Purchase: `gasLimit × gasPrice` 从发送者 DAI 余额扣
- Refund: `remainingGas × gasPrice` 返还
- `msg.value` 从**原生余额**单独验证
- 费用进系统地址 `0xffffFFFfFFffffffffffffffFfFFFfffFFFfFFfE`
- Aria 硬分叉后切回标准原生代币记账

## 攻击面

### 1. DAI 合约权限

```
owner()   = 0x110667fc23d315c09ec2a0259848bd1133304f11
totalSupply = 76,225 DAI
写入函数   = 只有 transferOwnership + renounceOwnership（OZ v5 Ownable）
无 mint / burn / pause / blacklist
```
非 owner 调用全部 revert `0x118cdaa7` = `OwnableUnauthorizedAccount(address)`

### 2. 资产分布（5,942 个 Transfer，282 个持有者）

| 地址 | ERC20-DAI | native | nonce | 备注 |
|---|---|---|---|---|
| `0x850b102ca66f3f0711dc2a61b1474d895ed151dd` | 40,017 | 40,016.99 | 109 | **faucet 热钱包** |
| `0x6e4ece960d0eaf9755d96cc3bd20bcf93e14a154` | 30,000 | 30,000.00 | 0 | 冷钱包 |
| `0x9f7b1763f0dc4df2062c67e1fbfc9b55c9fea8ac` | 1,418 | 1,417.99 | 18 | |
| `0xc13b65f7c53cd6db2ea205a4b574b4a0858720a6` | 1,000 | 999.86 | 114 | |
| `0xbaeb92889696217a3a6be2175e5a95dc4cffc9f7` | 1,000 | 1,000.00 | 0 | 冷钱包 |

零地址转出 5,075 次共 76,125 DAI（创世 mint）

### 3. RPC 暴露面

```
eth_accounts      → 5 个地址（全部余额 0）
txpool_content    → ★ 可读，泄露 pending 交易完整内容
eth_getProof      → 可读
personal_* / admin_* / debug_* / miner_* → 不可用
```

### 4. Faucet 线上架构

```
anubisfaucets.com → AWS CloudFront → ALB / S3 双源站
X-Amzn-Trace-Id / x-amzn-RequestId 暴露 ALB
GET  /api/health     → 200  泄露 chainId/tokenAddress/senderAddress
GET  /api/distribute → 405  到达应用层
POST /api/distribute → 403  AWS SigV4 签名错误（错误路由）
/.  //  /assets/     → 403  S3 AccessDenied XML
```

`GET /api/health` 返回：
```json
{"ok":true,"enabled":true,"chainId":202601,
 "tokenAddress":"0x83fd06f0846d9d90b3016bf670efe2e0b11cde14",
 "senderAddress":"0x850b102ca66f3f0711dc2a61b1474d895ed151dd"}
```

### 5. Faucet 代码审计要点

- `store.reserve` 用单条条件 INSERT 原子占位（`one_active_claim` 唯一索引限制全局只有 1 个活跃 claim）
- `token_hash` UNIQUE，Turnstile token 不可重放
- `turnstile.js` 标准验证（`success` + `hostname` + `action==='faucet_claim'`）
- `http.js` 强制 `CF-Connecting-IP`，注释明确禁止 `X-Forwarded-For`
- `sender.prepare` 先 simulate 再签名
- `sender.receipt` 校验 Transfer 事件（from/to/value 三项匹配）
- **`settle()` 若 claim 卡在 `signed` 且 broadcast 持续失败，会永久占用唯一活跃槽**

## 已排除

- DAI 合约直接接管（OZ Ownable 保护）
- 9,726 个弱私钥测试（0x01-0xFF + 0x0001-0xFFFF 抽样 + Hardhat 默认 + 常见助记词 + 字符串 hash + 整数派生）→ 未命中
- source map（全部 403）
- S3 bucket 名（13 个变体全部 404）
- workers.dev 直连（11 个变体不可达）
- JS bundle 密钥（只有 DAI 合约地址）

## 待验证方向

1. AWS Lambda / Secrets Manager 里的 `FAUCET_PRIVATE_KEY`
2. ALB origin 直连（绕过 CloudFront 的 POST 路由错误）
3. txpool 抢跑 / MEV
4. `settle()` 的 signed 卡槽 DoS
5. 桥合约 `app.anubisbridge.com`
6. Safe v1.5.0 合约审计
