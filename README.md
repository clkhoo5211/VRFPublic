# 链上可验证随机数（VRF）接入指南

BSC 主网上的可验证随机数服务。你的合约发起请求，预言机用 BLS12-381 唯一签名生成随机数并在链上验证后回调你的合约。随机数种子包含请求区块的哈希，请求发出时没有人能预知结果；任何人可以事后复验。

- 网络：**BNB Smart Chain 主网（chainId 56）**
- 支付：原生 **BNB**
- 接入方式：**订阅制**（预付、按实际成本扣费）或 **付钱即用**（每次请求附带 BNB）

## 1. 合约地址（BSC 主网，chainId 56）

| 合约 | 地址 | 用途 |
|---|---|---|
| Coordinator（协调器） | `0x481044B907C40A8eF406691168064BCe8B3fb836` | 订阅、请求、履约、验证 |
| PayPerRequest | `0x72AA07A2c1852F6EdB1Ee068ca1Af74444A8A5Ac` | 付钱即用入口 |
| Reader | `0x2F7d603EA5Fac589701177f2D570F0417F37bf30` | 查询余额 / 报价 / 复验随机数 |
| HeaderArchive | `0x3366b0cBBf31DBCf4dC1EC4dc87F844a4490A4d6` | 请求区块哈希存档（用于事后复验） |
| OperatorRegistry | `0x01DA9bfe6b614f9497c97efE3C90Dfb172fB45Ca` | 运营方登记 |
| Verifier | `0x46F8c7579B6A299750341237bAaeA6fFbf7A2334` | BLS 签名验证 |

**官方 lane：** `0xfe6f6c68fa6c50e1569c6fe61d9637af93d04632c000c835d41464b41efef2d0`

lane 是一个预言机密钥的标识（公钥的 `keccak256`）。发请求时必须指定 lane。协调器与 PayPerRequest 均为可升级代理合约，**请始终使用上表的代理地址**。

## 2. 参数与限制

| 项目 | 值 |
|---|---|
| 最少确认数 | 3（可填更大，最大 200） |
| 单次请求随机数个数 | 最多 500 |
| 每个订阅的消费合约数 | 最多 100 |
| 回调 gas 上限 | 2,500,000 |
| 请求超时（未履约则取消） | 100,000 块 |
| 履约批量 | 单笔最多 25 个请求 |

费率见第 4 节。

## 3. 快速接入

### 3.1 先定义接口（无需任何额外依赖）

```solidity
interface IRandomnessCoordinator {
    function createSubscription() external returns (uint256 subId);
    function fundSubscription(uint256 subId) external payable;
    function addConsumer(uint256 subId, address consumer) external;
    function cancelSubscription(uint256 subId, address payable to) external;
    function request(bytes32 lane, uint256 subId, uint16 confirmations, uint32 callbackGas, uint32 numWords)
        external returns (uint256 requestId);
    function quote(bytes32 lane, uint32 callbackGas, uint32 numWords) external view returns (uint256);
}

interface IPayPerRequest {
    function quote(uint32 callbackGas, uint32 numWords) external view returns (uint256);
    function request(uint32 callbackGas, uint32 numWords) external payable returns (uint256 requestId);
}
```

你的合约需要实现两个回调（只允许协调器 / PayPerRequest 调用）：

```solidity
function onRandomness(uint256 requestId, uint256[] calldata words) external;
function onRequestCancelled(uint256 requestId) external; // 可选；请求超时被取消时收到通知
```

### 3.2 方式一：订阅制（请求多、常用）

```solidity
contract MyGame {
    address constant COORDINATOR = 0x481044B907C40A8eF406691168064BCe8B3fb836;
    bytes32 constant LANE = 0xfe6f6c68fa6c50e1569c6fe61d9637af93d04632c000c835d41464b41efef2d0;
    uint256 public subId;
    mapping(uint256 => address) public playerOf;

    // ① 一次性设置：开订阅、充值、登记本合约为消费者（也可以用钱包在外部完成）
    function setup() external payable {
        IRandomnessCoordinator c = IRandomnessCoordinator(COORDINATOR);
        subId = c.createSubscription();
        c.fundSubscription{value: msg.value}(subId);
        c.addConsumer(subId, address(this));
    }

    // ② 发请求：确认数 12，回调 gas 200,000，要 1 个随机数
    function play() external {
        uint256 id = IRandomnessCoordinator(COORDINATOR).request(LANE, subId, 12, 200_000, 1);
        playerOf[id] = msg.sender;
    }

    // ③ 回调：只允许协调器调用
    function onRandomness(uint256 requestId, uint256[] calldata words) external {
        require(msg.sender == COORDINATOR, "only coordinator");
        // 使用 words[0]；不要 revert；不要在这里写大量存储
    }

    function onRequestCancelled(uint256 requestId) external {
        require(msg.sender == COORDINATOR, "only coordinator");
        // 请求超时被取消：作废本轮 / 退款。不要在这里重新请求
    }
}
```

示例为说明流程而简化，省略了权限控制；实际使用请自行为 `setup`、提取余额等函数加上访问控制。

订阅可以由任何地址创建和充值，创建者是订阅持有者。不再使用时，持有者调用 `cancelSubscription` 取回余额（有未履约请求时不可取消）。

### 3.3 方式二：付钱即用（偶尔用、不想管订阅）

```solidity
address constant PPR = 0x72AA07A2c1852F6EdB1Ee068ca1Af74444A8A5Ac;

function play() external payable {
    uint256 price = IPayPerRequest(PPR).quote(200_000, 1);   // 回调 gas, 随机数个数
    uint256 id = IPayPerRequest(PPR).request{value: price}(200_000, 1); // 多付的当场退回
}
// 随机数会转发到调用者合约的 onRandomness(requestId, words)
```

回调失败时随机数会被保存，可调用 PayPerRequest 的 `retry(requestId)` 再次投递，不会丢失。

## 4. 费用

- **订阅制**：发请求时按该 lane 的最高 gas 价**预留**最坏成本（预留不扣钱）；履约时**按实际成本**扣费，不超过预留额，多余的留在订阅里。
  实收 = 实际 gas 成本 × (1 + 溢价) + 固定费
- **付钱即用**：价格 = 协调器报价 × (1 + 加价)；多付的部分当场退回。
- **费率**

| 项目 | 数值 | 说明 |
|---|---|---|
| 溢价 | **10%** | 在实际 gas 成本上加 10% |
| 固定费 | **0.001 BNB** | 每次履约收取一次 |
| 付钱即用加价 | **10%** | 仅付钱即用模式，在协调器报价上再加 10% |

- **链上读取（以链上为准）**：
  - 溢价 `premiumBps`、固定费 `flatFee`：`Coordinator.settings()`
  - 加价 `markupBps`：`PayPerRequest.markupBps()`
- **报价与余额查询**：
  - `Coordinator.quote(lane, callbackGas, numWords)`：发一个请求要预留多少
  - `PayPerRequest.quote(callbackGas, numWords)`：付钱即用的价格（当前 gas 价下）
  - `Reader.affordable(subId, lane, callbackGas, numWords)`：订阅还能发几个
  - `Reader.shortfall(subId, lane, callbackGas, numWords, n)`：发 n 个还差多少 wei

回调 gas 越大、随机数个数越多、当时 gas 价越高，费用越高。

## 5. 回调规则与安全用法

1. 回调里**不要 revert**，也不要超过请求时填写的 `callbackGas`，否则会记为回调失败，需要重新投递。
2. 回调里**不要写大量存储**：只存种子或结果摘要，其余在 view 函数里重算。
3. 请求发出后到回调之间，**锁定相关状态**，不要让用户在看到结果之后再决定是否参与。
4. 价值大的抽奖，确认数建议 **≥ 12**（抗链重组）。
5. 请求超时会被取消并通知 `onRequestCancelled`：在里面作废本轮或退款，**不要重新请求**（重新请求等于换种子重摇）。
6. 回调里允许再发新请求（连续抽奖）。

## 6. 事后复验

任何人可调用 `Reader.verifyFulfilment(lane, requestId, blockNumber, numWords, sig)` 复算并核对随机数。节点履约后会自动把请求区块哈希存入 HeaderArchive，所以超过 256 块的请求也能复验。

## 7. 常见错误

| 现象 | 原因与处理 |
|---|---|
| `Underfunded` | 订阅可用余额小于预留额。充值，或用 `Reader.shortfall` 算差额 |
| `Underpaid` | 付钱即用：`msg.value` 低于报价。先调 `quote` |
| 回调没触发 | 查 `DeliveryFailed` 事件；订阅制可由任何人 `redeliver`，付钱即用调 `retry` |
| 请求迟迟没履约 | 超时（100,000 块）后会被取消，并通知你的 `onRequestCancelled` |
| 回调被拒 | 回调只接受协调器 / PayPerRequest 的调用 |

## 8. 运营方（运行预言机节点）

任何团队都可以申请成为运营方，运行自己的预言机节点并获得服务费分成。

**准备**
- 一台长期在线的 Linux 服务器（Docker 或发布包）。
- RPC：必须支持 `getLogs`（公共节点通常不行），建议至少两个供应商做故障转移。
- 两把**不同**的密钥：BLS 私钥（加密存本机 keystore，**永不上链、永不外传**）和发交易的热钱包（需要 BNB 付 gas）。
- 保证金：申请时随交易存入 OperatorRegistry，**最低 2 BNB**。

**流程**
1. 本机生成密钥，得到公钥与 lane。
2. 生成持有证明（PoP，绑定链、注册表、申请人和 lane，防抢注与重放）。
3. 调用 `OperatorRegistry.applyOperator(pk, pop)`，同时存入保证金。
4. 当前申请需**审批**（`permissionless=false`）；审批通过后，注册表在协调器登记你的 lane。
5. 启动节点，节点监听该 lane 的请求并履约。

**收益**：每次履约的服务费（溢价 + 固定费）按分成比例记给你的收款地址，**默认分成 70%**（上限 90%），其余归协议；履约 gas 补偿记给提交交易的地址。提取调用 `claimCredits(to)`。

**义务与罚没**

| 项目 | 数值 |
|---|---|
| 最低保证金 | 2 BNB |
| 判定未履约 | 请求就绪后超过 8000 块仍未履约，任何人可举报 |
| 每次罚没 | 0.1 BNB |
| 同一运营方两次计罚的最短间隔 | 1 小时（同一事故只罚一次） |
| 罚没累计上限 | 7 天滑动窗口内不超过保证金的 20% |
| 争议期 | 2 天（仲裁方可否决；到期后任何人可执行罚没） |
| 自动挂起 | 7 天内累计 3 次罚没 |
| 解押期 | 14 天（申请退出时起算，之后可 `withdraw`） |
| lane 最高 gas 价 | 范围 0.1 ~ 2 gwei，新 lane 默认 1 gwei |

- 罚没款不归协议方，转入销毁地址。
- 只罚"区块哈希可得"的请求（最近 256 块，或已存档）；节点离线过久导致无法履约的请求不罚运营方。
- 被挂起后补足保证金（`topUp`），由审批方恢复。
- 退出：`requestExit`，解押期内不再接新请求，保证金可能被未决罚没扣减；期满 `withdraw(to)`。
- 以上数值以链上 `OperatorRegistry.params()` 为准。

## 9. 限制

- 仅 BNB Smart Chain 主网；仅用 BNB 支付。
- 接口与 Chainlink 不兼容，已有 Chainlink 合约需按上面的接口改写。
- 单个运营方可以拒绝履约；请求超时会被取消并通知你，不会静默失败。对不能承受随机数延迟的业务，不要只依赖单个 lane。
