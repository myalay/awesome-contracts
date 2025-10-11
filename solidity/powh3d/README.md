# 什么是 PoWH3D？

## 项目概述

[合约地址](https://etherscan.io/address/0xb3775fb83f7d12a36e0475abdd1fca35c091efbe)

**PoWH3D**（Proof of Weak Hands 3D）是一个建立在以太坊上的去中心化智能合约，于2018年2月在Etherscan上验证部署。它是一个自治的"金字塔"系统，结合了代币交易、分红机制和推荐奖励系统。

### 核心特点

- **自动分红系统**：每笔买入、卖出、转账都会收取10%的手续费，这些手续费会自动分配给所有代币持有者
- **动态定价机制**：代币价格随着供应量的增加而上涨，采用数学公式动态计算
- **Masternode（主节点）机制**：持有100个以上代币的用户可以生成推荐链接，获得30%的推荐奖励
- **无管理员提款权限**：管理员只能修改合约名称、代币符号等，无法取走资金或禁用提款功能
- **完全去中心化**：一旦部署，合约将永久运行，无法被关闭

## 合约架构

### 主要组件

```solidity
contract Hourglass {
    // 代币基本信息
    string public name = "PowH3D";
    string public symbol = "P3D";
    uint8 constant public decimals = 18;
    
    // 核心参数
    uint8 constant internal dividendFee_ = 10;  // 10%手续费
    uint256 constant internal magnitude = 2**64; // 精度放大倍数
    uint256 public stakingRequirement = 100e18;  // Masternode要求
    
    // 关键状态变量
    mapping(address => uint256) internal tokenBalanceLedger_;  // 代币余额
    mapping(address => int256) internal payoutsTo_;  // 已支付分红
    uint256 internal profitPerShare_;  // 每代币累积分红
    uint256 internal tokenSupply_;  // 代币总供应量
}
```

### 定价机制：联合曲线（Bonding Curve）

PoWH3D采用**线性联合曲线**定价模型，这是一个数学上确定的定价算法，确保：
- 📈 随着供应量增加，价格自动上涨
- 📉 随着供应量减少，价格自动下跌
- 💰 买卖价格由数学公式保证，无法人为操纵

#### 核心参数

```solidity
uint256 constant internal tokenPriceInitial_ = 0.0000001 ether;      // 初始价格：0.0000001 ETH
uint256 constant internal tokenPriceIncremental_ = 0.00000001 ether; // 价格增量：0.00000001 ETH
```

#### 价格曲线原理

**线性增长模型**：
```
第 n 个代币的价格 = 初始价格 + (n - 1) × 价格增量

P(n) = P₀ + (n - 1) × Δ

其中：
- P₀ = 0.0000001 ETH（初始价格）
- Δ = 0.00000001 ETH（价格增量）
- n = 代币序号（从1开始）
```

**实例**：
```
第 1 个代币：0.0000001 ETH
第 2 个代币：0.0000001 + 0.00000001 = 0.00000011 ETH
第 3 个代币：0.0000001 + 2 × 0.00000001 = 0.00000012 ETH
第 100 个代币：0.0000001 + 99 × 0.00000001 = 0.00000109 ETH
第 1000 个代币：0.0000001 + 999 × 0.00000001 = 0.00001099 ETH
```

#### 买入公式详解

**问题**：投入 `E` 个ETH，能买多少个代币？

**数学推导**：

假设当前供应量为 `S`，买入 `T` 个代币需要支付的总成本为：

```
Cost = ∫[S to S+T] (P₀ + n × Δ) dn
     = [P₀ × n + Δ × n²/2] from S to S+T
     = P₀ × T + Δ × [(S+T)² - S²]/2
     = P₀ × T + Δ × T × (S + T/2)
```

简化后得到关于 `T` 的一元二次方程：
```
Δ/2 × T² + (P₀ + Δ×S) × T - E = 0
```

使用求根公式：
```
T = [-b + √(b² + 4ac)] / 2a

其中：
a = Δ/2
b = P₀ + Δ×S
c = -E
```

**合约代码实现**（第696-724行）：

```solidity
function ethereumToTokens_(uint256 _ethereum) internal view returns(uint256) {
    uint256 _tokenPriceInitial = tokenPriceInitial_ * 1e18;
    uint256 _tokensReceived = 
     (
        (
            // 求根公式
            SafeMath.sub(
                (sqrt
                    (
                        (_tokenPriceInitial**2)  // b²
                        +
                        (2*(tokenPriceIncremental_ * 1e18)*(_ethereum * 1e18))  // 4ac
                        +
                        (((tokenPriceIncremental_)**2)*(tokenSupply_**2))  // 额外项
                        +
                        (2*(tokenPriceIncremental_)*_tokenPriceInitial*tokenSupply_)  // 额外项
                    )
                ), _tokenPriceInitial
            )
        )/(tokenPriceIncremental_)
    )-(tokenSupply_);
  
    return _tokensReceived;
}
```

#### 卖出公式详解

**问题**：卖出 `T` 个代币，能获得多少ETH？

**数学推导**：

假设当前供应量为 `S`，卖出 `T` 个代币（供应量变为 `S-T`）：

```
Revenue = ∫[S-T to S] (P₀ + n × Δ) dn
        = [P₀ × n + Δ × n²/2] from S-T to S
        = P₀ × T + Δ × [S² - (S-T)²]/2
        = P₀ × T + Δ × T × (S - T/2)
```

简化为：
```
E = (P₀ + Δ × S - Δ × T/2) × T
```

**合约代码实现**（第731-753行）：

```solidity
function tokensToEthereum_(uint256 _tokens) internal view returns(uint256) {
    uint256 tokens_ = (_tokens + 1e18);
    uint256 _tokenSupply = (tokenSupply_ + 1e18);
    uint256 _etherReceived =
    (
        SafeMath.sub(
            (
                (
                    (
                        tokenPriceInitial_ + (tokenPriceIncremental_ * (_tokenSupply/1e18))
                    ) - tokenPriceIncremental_
                ) * (tokens_ - 1e18)
            ), (tokenPriceIncremental_ * ((tokens_**2 - tokens_)/1e18)) / 2
        )
    /1e18);
    return _etherReceived;
}
```

#### 价格曲线可视化

```
价格 (ETH)
    ^
    |                                           ╱
    |                                       ╱
    |                                   ╱
0.001|                               ╱
    |                           ╱
    |                       ╱
    |                   ╱
0.0001|           ╱╱╱
    |     ╱╱╱
    |╱╱╱╱
    +-----------------------------------------> 供应量
    0        1000      5000      10000

线性联合曲线特点：
- 斜率恒定（Δ = 0.00000001）
- 价格与供应量成正比
- 买卖价格完全由公式决定
```

#### 买入价格 vs 卖出价格

由于有10%的手续费，实际买入和卖出价格有差异：

```
买入流程：
1. 用户支付：10 ETH
2. 扣除手续费：10 × 10% = 1 ETH
3. 实际购买：9 ETH
4. 获得代币：ethereumToTokens_(9 ETH)

卖出流程：
1. 卖出代币：T tokens
2. 理论价值：tokensToEthereum_(T)
3. 扣除手续费：tokensToEthereum_(T) × 10%
4. 实际获得：tokensToEthereum_(T) × 90%
```

#### 完整计算示例

**场景：第一次买入**

```
初始状态：
- 供应量：0 tokens
- 投入：1 ETH

步骤1：扣除手续费
- 手续费：1 × 10% = 0.1 ETH
- 实际投入：0.9 ETH

步骤2：计算获得代币数量
使用公式：T = [-b + √(b² + 4ac)] / 2a

其中：
- P₀ = 0.0000001 ETH
- Δ = 0.00000001 ETH
- S = 0（当前供应量）
- E = 0.9 ETH

代入计算：
a = 0.00000001 / 2 = 0.000000005
b = 0.0000001 + 0.00000001 × 0 = 0.0000001
c = -0.9

T = [-0.0000001 + √(0.0000001² + 4 × 0.000000005 × 0.9)] / (2 × 0.000000005)
  = [-0.0000001 + √(0.00000000000001 + 0.000000018)] / 0.00000001
  ≈ [-0.0000001 + 0.00000424264] / 0.00000001
  ≈ 0.00000423264 / 0.00000001
  ≈ 423.264 tokens
```

**场景：后续买入**

```
当前状态：
- 供应量：1000 tokens
- 投入：1 ETH

步骤1：计算当前平均价格
当前第1000个token的价格：
P(1000) = 0.0000001 + 999 × 0.00000001 = 0.00001099 ETH

步骤2：扣除手续费后买入
- 实际投入：0.9 ETH
- 基础价格更高，获得的代币更少
- 大约获得：~82 tokens（具体需代入公式计算）
```

#### 价格滑点分析

**大额买入的影响**：

```
买入量（ETH） | 平均价格 | 滑点
------------|---------|-----
0.1         | 低      | ~0%
1.0         | 中      | ~5%
10.0        | 高      | ~20%
100.0       | 很高    | ~50%
```

由于价格随供应量线性增长，大额买入会显著提高平均成本。

#### 数学特性总结

1. **确定性**：价格完全由数学公式决定，无法人为操纵
2. **对称性**：买入和卖出使用相同的曲线（除手续费外）
3. **连续性**：价格平滑变化，无跳跃
4. **单调性**：价格始终随供应量单调递增
5. **流动性保证**：合约本身就是做市商，始终可以买卖

#### 与传统交易所的对比

| 特性 | PoWH3D联合曲线 | 传统交易所 |
|------|---------------|-----------|
| 价格决定 | 数学公式 | 供需匹配 |
| 流动性 | 无限（公式保证） | 取决于订单簿 |
| 滑点 | 可预测（公式计算） | 不可预测 |
| 做市商 | 合约本身 | 第三方 |
| 手续费 | 10%（固定） | 0.1%-1%（变动） |
| 价格操纵 | 几乎不可能 | 可能（通过大单） |

## 简单玩法指南

### 1. 买入代币（Buy）

**操作方式**：
```javascript
// 无推荐人
contract.buy(0x0000000000000000000000000000000000000000, {
    value: ethers.utils.parseEther("1.0")  // 投入1 ETH
});

// 有推荐人
contract.buy("0x推荐人地址", {
    value: ethers.utils.parseEther("1.0")
});
```

**资金分配**：
```
假设买入 10 ETH：
├─ 手续费：1 ETH (10%)
│  ├─ 推荐奖励：0.3 ETH → 推荐人（如果有且符合条件）
│  └─ 持币人分红：0.7 ETH → 所有代币持有者
└─ 实际购买：9 ETH → 兑换成代币
```

**注意事项**：
- ✅ 投入越多，获得的代币越多（但单价也越高）
- ✅ 早期买入者获得更低的价格
- ❌ 自己产生的手续费不会分红给自己
- ❌ 买入即亏损10%（需要价格上涨才能回本）

### 2. 卖出代币（Sell）

**操作方式**：
```javascript
// 卖出部分代币
const tokensToSell = ethers.utils.parseEther("100");
contract.sell(tokensToSell);

// 查看能获得多少ETH
const ethReceived = await contract.calculateEthereumReceived(tokensToSell);
```

**资金流动**：
```
假设卖出价值 10 ETH 的代币：
├─ 手续费：1 ETH (10%) → 分配给剩余持币人
└─ 实际获得：9 ETH → 转到你的钱包
```

**特点**：
- 卖出会降低代币总供应量
- 卖出价格 = 当前价格 × 代币数量 × 90%
- 越晚卖出，如果价格上涨，收益越高

### 3. 提取分红（Withdraw）

**操作方式**：
```javascript
// 查看可领取分红
const dividends = await contract.myDividends(true);
console.log(`可领取：${ethers.utils.formatEther(dividends)} ETH`);

// 提取分红
contract.withdraw();
```

**分红来源**：
- 其他人买入代币产生的手续费
- 其他人卖出代币产生的手续费
- 其他人转账代币产生的手续费
- 推荐奖励（如果你是Masternode）

**计算公式**：
```
你的分红 = (全局每代币累积分红 × 你的代币数量 - 已领取分红) / 放大倍数
```

### 4. 复利再投资（Reinvest）

**操作方式**：
```javascript
contract.reinvest();
```

**工作原理**：
- 将你的分红直接用于购买更多代币
- **不收取额外手续费**（相比提现后再买入更划算）
- 增加你的持币量，未来获得更多分红

**复利效果示例**：
```
初始：100 tokens
每天分红：1%

选项A - 每天提取（Withdraw）：
Day 30: 100 tokens + 30 ETH 现金

选项B - 每天再投资（Reinvest）：
Day 30: ~135 tokens（复利增长）
```

### 5. 完全退出（Exit）

**操作方式**：
```javascript
contract.exit();
```

**执行步骤**：
1. 自动卖出所有代币（扣除10%手续费）
2. 自动提取所有分红
3. 一次性获得所有ETH

**适用场景**：
- 想要完全离场
- 不想分两步操作
- 立即需要全部资金

### 6. Masternode（主节点）推荐

**成为Masternode的条件**：
- 持有 ≥ 100 个代币（可调整）

**推荐奖励**：
```
用户通过你的推荐链接买入 10 ETH：
├─ 总手续费：1 ETH
├─ 推荐奖励：0.3 ETH → 你获得
└─ 剩余分红：0.7 ETH → 所有持币人
```

**生成推荐链接**：
```
https://powh.io/?masternode=你的以太坊地址
```

**注意**：
- 必须持有≥100代币才能获得推荐奖励
- 不能推荐自己（会被合约拒绝）
- 推荐奖励存储在`referralBalance_`中，调用`withdraw()`时一起领取

## 游戏机制总结

### 盈利模式

```
收益来源：
1. 代币价格上涨（供应量增加导致）
2. 持币分红（其他人交易产生的手续费）
3. 推荐奖励（作为Masternode）

风险来源：
1. 买卖手续费共20%（买入10% + 卖出10%）
2. 价格可能下跌（供应量减少时）
3. 流动性风险（可能没人接盘）
```

### 盈亏平衡点

```
假设买入 10 ETH：
├─ 实际获得代币：9 ETH 价值
└─ 盈亏平衡：代币价格需上涨 11.11% + 获得足够分红

计算：9 ETH × 1.1111 = 10 ETH（卖出还要扣10%）
实际需要：9 ETH × 1.1111 × 1.1111 ≈ 11.11 ETH 价值
```

### 策略建议

**保守策略**：
1. 小额投入测试
2. 定期提取分红
3. 达到盈利目标立即退出

**激进策略**（风险极高）：
1. 大额投入成为Masternode
2. 推广获得推荐奖励
3. 分红再投资追求复利

**理性策略**：
1. **不要投入**（这是庞氏骗局）
2. 仅用于学习智能合约机制
3. 理解分布式金融的风险

## 风险警告 ⚠️

### 这是一个庞氏骗局！

1. **零和游戏**：所有收益都来自后来者的损失
2. **必然崩盘**：当没有新买家时，系统崩溃
3. **无内在价值**：代币本身没有任何实际用途
4. **法律风险**：在很多国家可能违法

### 技术风险

1. **智能合约漏洞**：虽然经过审计，但仍可能存在未知bug
2. **Gas费用**：以太坊网络拥堵时操作成本极高
3. **前端风险**：钓鱼网站可能窃取私钥
4. **流动性风险**：大额卖出可能找不到买家

### 历史教训

PoWH3D系列曾发生过：
- 合约被黑客攻击
- 社区分裂和分叉
- 大量用户损失资金

## 仅供学习研究

本文档旨在：
- ✅ 解释智能合约的技术实现
- ✅ 分析去中心化分红机制
- ✅ 理解联合曲线定价模型
- ❌ **绝不推荐实际投资**

**请勿使用真实资金参与此类项目！**


# PoWH3D 分红机制

## 核心机制：profitPerShare_

这是一个**累积分红追踪系统**，巧妙地实现了按比例分配。

## 工作原理

### 关键变量

```solidity
// 每个代币累积获得的总分红（放大了 2^64 倍）
uint256 internal profitPerShare_;

// 用户持有的代币数量
mapping(address => uint256) internal tokenBalanceLedger_;

// 用户已经领取的分红（也放大了 2^64 倍）
mapping(address => int256) internal payoutsTo_;

// 放大倍数，避免 Solidity 中的小数问题
uint256 constant internal magnitude = 2**64;
```

## 完整示例演示

### 初始状态

```
总供应量: 0 tokens
profitPerShare_: 0
```

### 🔹 第一步：用户 A 买入

```
用户 A 买入 10 ETH
├─ 手续费: 1 ETH (10%)
│  ├─ 推荐人: 0 (无推荐人)
│  └─ 持币人分红: 1 ETH
├─ 实际购买: 9 ETH
└─ 获得代币: 1000 tokens

结果:
- 总供应量: 1000 tokens
- 用户 A 持有: 1000 tokens
- 分红池: 1 ETH
- profitPerShare_: 1 ETH * 2^64 / 1000 = 18,446,744,073,709,551 (单位：wei * 2^64 / token)
```

**注意**: 用户 A 不会获得这 1 ETH 分红（因为是他自己买入产生的）

```solidity
// 第 682-683 行：新买家不获得自己产生的分红
int256 _updatedPayouts = (int256) ((profitPerShare_ * _amountOfTokens) - _fee);
payoutsTo_[_customerAddress] += _updatedPayouts;
```

### 🔹 第二步：用户 B 买入

```
用户 B 买入 10 ETH
├─ 手续费: 1 ETH (10%)
│  └─ 持币人分红: 1 ETH
├─ 实际购买: 9 ETH
└─ 获得代币: 900 tokens（价格已上涨）

结果:
- 总供应量: 1900 tokens
- 用户 B 持有: 900 tokens
- 新增分红: 1 ETH
- profitPerShare_ 增加: 1 ETH * 2^64 / 1900

当前 profitPerShare_:
原值 + 新增 = 18,446,744,073,709,551 + (1 ETH * 2^64 / 1900)
```

### 🔹 第三步：计算用户 A 的分红

```solidity
// 用户 A 的可领取分红
dividendsOf(A) = (profitPerShare_ * tokenBalanceLedger_[A] - payoutsTo_[A]) / magnitude

计算:
= (profitPerShare_ * 1000 - 已支付) / 2^64
≈ 1 ETH (来自用户 B 的买入)
```

**用户 A 获得了用户 B 买入产生的分红！**

### 🔹 第四步：用户 C 买入

```
用户 C 买入 10 ETH
├─ 手续费: 1 ETH
│  └─ 持币人分红: 1 ETH
├─ 获得代币: 850 tokens
└─ 总供应: 2750 tokens

profitPerShare_ 再次增加:
增加量 = 1 ETH * 2^64 / 2750
```

### 🔹 第五步：查看所有人的分红

**用户 A（持有 1000 tokens）：**
```
分红 = (profitPerShare_ * 1000 - payoutsTo_[A]) / 2^64
     ≈ 用户 B 的 1 ETH × (1000/1900) + 用户 C 的 1 ETH × (1000/2750)
     ≈ 0.526 ETH + 0.364 ETH
     ≈ 0.89 ETH
```

**用户 B（持有 900 tokens）：**
```
分红 = (profitPerShare_ * 900 - payoutsTo_[B]) / 2^64
     ≈ 用户 C 的 1 ETH × (900/2750)
     ≈ 0.327 ETH
```

**用户 C（持有 850 tokens）：**
```
分红 = 0 ETH（自己的买入不产生分红）
```

## 数学验证

### 验证比例分配

用户 C 买入产生的 1 ETH 分红：

```
用户 A 获得: 1 ETH × (1000/2750) = 0.364 ETH (36.4%)
用户 B 获得: 1 ETH × (900/2750) = 0.327 ETH (32.7%)
用户 C 获得: 0 ETH (0%)
总计: 0.691 ETH

剩余 0.309 ETH？
这部分分配给了用户 C 自己买入的 850 tokens，
但通过 payoutsTo_ 机制被排除了。
```

**关键点**: 每次新买入产生的分红，会按照**当时的持币比例**分配给所有现有持币人。

## 为什么这个设计很巧妙？

### 1️⃣ Gas 效率
```
❌ 糟糕的方法：遍历所有持币人
for(每个持币人) {
    转账分红  // 每次买入都要循环所有用户，Gas 费用爆炸！
}

✅ 巧妙的方法：累积记账
profitPerShare_ += newDividends / totalSupply  // 只需一次计算！
用户领取时才结算
```

### 2️⃣ 延迟结算
- 分红不会立即转账
- 用户可以随时提取（withdraw）
- 或者再投资（reinvest）

### 3️⃣ 精确计算
```solidity
// 使用 magnitude = 2^64 放大，避免精度损失
uint256 constant internal magnitude = 2**64;

// 所有计算都放大 2^64 倍
// 最后除以 magnitude 恢复原始值
```

## 关键代码片段解析

### 买入时更新分红池

```solidity
// 第 667 行：更新每代币累积分红
profitPerShare_ += (_dividends * magnitude / (tokenSupply_));

// 第 682-683 行：排除新买家获得自己产生的分红
int256 _updatedPayouts = (int256) ((profitPerShare_ * _amountOfTokens) - _fee);
payoutsTo_[_customerAddress] += _updatedPayouts;
```

**解释**:
- `profitPerShare_ * _amountOfTokens`: 新买家"应得"的所有历史分红
- `- _fee`: 减去他自己刚产生的分红
- 结果：他只能获得未来的分红，不能获得自己产生的

### 卖出时更新

```solidity
// 第 360-361 行：卖出代币时更新已支付分红
int256 _updatedPayouts = (int256) (profitPerShare_ * _tokens + (_taxedEthereum * magnitude));
payoutsTo_[_customerAddress] -= _updatedPayouts;
```

### 提取分红

```solidity
// 第 326 行：提取时更新已支付记录
payoutsTo_[_customerAddress] += (int256) (_dividends * magnitude);
```

## 总结

### ✅ 是的，合约确实实现了按持币比例分配！

**机制**:
1. 每次买入/卖出收取 10% 手续费
2. 其中 ~6.67% 分配给所有持币人
3. 使用 `profitPerShare_` 累积追踪每代币应得分红
4. 用户可随时领取 `(profitPerShare_ × 持币量 - 已领取) / magnitude`

**特点**:
- ✅ 自动按比例分配
- ✅ Gas 效率高（不需要遍历所有用户）
- ✅ 新买家不能获得自己产生的分红
- ✅ 精确计算（使用 magnitude 避免精度损失）

**数学公式**:
```
用户分红 = (全局每代币累积分红 × 用户持币量 - 用户已领取分红) / 放大倍数
```

这是一个在区块链上实现**高效分红系统**的经典设计模式！

# PoWH3D 如何领取分红？

## 🎯 三种领取方式对比

| 方式 | 函数 | 结果 | 适用场景 |
|-----|------|------|---------|
| **直接提取** | `withdraw()` | ETH 到钱包 | 想拿到现金 |
| **复利再投资** | `reinvest()` | 买入更多代币 | 追求复利增长 |
| **完全退出** | `exit()` | 卖出代币 + 提取分红 | 想完全离场 |

## 📖 详细说明

### 1️⃣ withdraw() - 提取分红

#### 代码逻辑

```solidity
function withdraw() onlyStronghands() public {
    // 1. 获取调用者地址
    address _customerAddress = msg.sender;
    
    // 2. 计算可领取分红（不包括推荐奖励）
    uint256 _dividends = myDividends(false);
    
    // 3. 更新已领取记录（避免重复领取）
    payoutsTo_[_customerAddress] += (int256) (_dividends * magnitude);
    
    // 4. 加上推荐奖励
    _dividends += referralBalance_[_customerAddress];
    referralBalance_[_customerAddress] = 0;  // 清零推荐余额
    
    // 5. 转账到用户钱包
    _customerAddress.transfer(_dividends);
    
    // 6. 触发提取事件
    onWithdraw(_customerAddress, _dividends);
}
```

#### 使用示例

**Web3 调用：**
```javascript
// 查看可领取分红
const dividends = await contract.myDividends(true);
console.log(`可领取: ${ethers.utils.formatEther(dividends)} ETH`);

// 提取分红
const tx = await contract.withdraw();
await tx.wait();
console.log("提取成功！");
```

**Gas 估算：** ~50,000 - 100,000 gas

#### 前置条件

```solidity
modifier onlyStronghands() {
    require(myDividends(true) > 0);  // 必须有分红
    _;
}
```

**注意**：如果没有分红（= 0），调用会失败！

#### 获得的分红包括

- ✅ 持币分红（来自其他人的买入/卖出手续费）
- ✅ 推荐奖励（如果你是 Masternode）
- ✅ 累积的所有历史分红

### 2️⃣ reinvest() - 复利再投资

#### 代码逻辑

```solidity
function reinvest() onlyStronghands() public {
    // 1. 获取分红金额
    uint256 _dividends = myDividends(false);
    address _customerAddress = msg.sender;
    
    // 2. 虚拟"支付"分红（标记为已领取）
    payoutsTo_[_customerAddress] += (int256) (_dividends * magnitude);
    
    // 3. 加上推荐奖励
    _dividends += referralBalance_[_customerAddress];
    referralBalance_[_customerAddress] = 0;
    
    // 4. 用分红购买代币（不产生额外手续费！）
    uint256 _tokens = purchaseTokens(_dividends, 0x0);
    
    // 5. 触发再投资事件
    onReinvestment(_customerAddress, _dividends, _tokens);
}
```

#### 复利效应示例

```
初始持有: 100 tokens
分红: 1 ETH

选项 A - withdraw()
  → 获得 1 ETH 现金
  → 持有 100 tokens

选项 B - reinvest()
  → 获得 0 ETH 现金
  → 持有 100 + X tokens（用 1 ETH 买入的代币）
  → 未来分红基数更大！
```

#### 复利计算

假设每天有 1% 的分红：

```
Day 0:  100 tokens
Day 1:  101 tokens (reinvest 1 token)
Day 2:  102.01 tokens (1% of 101)
Day 30: 134.78 tokens (复利增长)

如果每天 withdraw:
Day 30: 仍是 100 tokens + 30 ETH 现金
```

**复利公式：** `Final = Initial × (1 + rate)^days`

### 3️⃣ exit() - 完全退出

#### 代码逻辑

```solidity
function exit() public {
    address _customerAddress = msg.sender;
    uint256 _tokens = tokenBalanceLedger_[_customerAddress];
    
    // 步骤 1: 卖出所有代币
    if(_tokens > 0) sell(_tokens);
    
    // 步骤 2: 提取所有分红
    withdraw();
}
```

#### 执行流程

```
调用 exit()
  ↓
1. 检查代币余额
  ↓
2. 如果有代币 → 调用 sell(_tokens)
   - 卖出收取 10% 手续费
   - 获得 ETH
   - 手续费分配给剩余持币人
  ↓
3. 调用 withdraw()
   - 提取所有累积分红
   - 提取推荐奖励
  ↓
4. 获得总 ETH = 卖出所得 + 分红
```

#### 退出成本

```
持有 100 tokens，价值 10 ETH
分红余额: 2 ETH

卖出代币:
  10 ETH × 0.9 = 9 ETH (扣 10% 手续费)

提取分红:
  2 ETH (无手续费)

总获得: 9 + 2 = 11 ETH
```

## 🎮 实际操作指南

### 场景 1：日常提取分红

```javascript
// 1. 查看分红
const dividends = await contract.myDividends(true);
console.log(`分红: ${ethers.utils.formatEther(dividends)} ETH`);

// 2. 如果分红 > 0，提取
if (dividends > 0) {
    const tx = await contract.withdraw();
    await tx.wait();
    console.log("提取成功！");
}
```

### 场景 2：复利策略

```javascript
// 自动复利脚本
async function autoCompound() {
    const dividends = await contract.myDividends(true);
    
    // 如果分红超过 0.1 ETH，自动再投资
    if (dividends > ethers.utils.parseEther("0.1")) {
        const tx = await contract.reinvest();
        await tx.wait();
        console.log("复利成功！");
    }
}

// 每天运行一次
setInterval(autoCompound, 24 * 60 * 60 * 1000);
```

### 场景 3：完全退出

```javascript
// 一键退出
const tx = await contract.exit();
await tx.wait();
console.log("已退出，所有 ETH 已到账！");
```

## 📊 分红来源

你的分红来自：

### 1. 持币分红（主要来源）

```
其他用户买入 → 10% 手续费 → 6.67% 分配给所有持币人
其他用户卖出 → 10% 手续费 → 10% 分配给所有持币人
其他用户转账 → 10% 手续费 → 10% 分配给所有持币人
```

**你的份额 = (你的代币 / 总代币供应) × 分红池**

### 2. 推荐奖励（如果是 Masternode）

```
条件: 持有 ≥ 100 tokens
奖励: 通过你的推荐链接买入的用户产生的手续费的 30%
```

**示例：**
```
用户通过你的链接买入 10 ETH
手续费: 1 ETH
你获得: 1 × 0.1 × 0.3 = 0.03 ETH
```

## 💡 最佳实践

### ✅ 推荐策略

1. **小额频繁提取**
   - 定期提取小额分红
   - 降低合约风险暴露

2. **复利增长（风险自负）**
   - 如果看好项目，可以 reinvest
   - 但要注意庞氏风险！

3. **设置止盈点**
   - 达到盈利目标就 exit
   - 不要贪心

### ❌ 避免的错误

1. **不要等分红太多才提取**
   - 合约可能随时崩盘
   - 早提早安全

2. **不要盲目 reinvest**
   - 这是庞氏骗局
   - 后期可能无人接盘

3. **不要忘记 Gas 费**
   - 小额提取可能 Gas > 分红
   - 等积累到一定金额再提取

## 🔍 查询分红的方法

### 查看自己的分红

```solidity
function myDividends(bool _includeReferralBonus) public view returns(uint256)
```

**参数：**
- `true`: 包含推荐奖励
- `false`: 只看持币分红

**Web3 调用：**
```javascript
// 总分红（包含推荐）
const totalDividends = await contract.myDividends(true);

// 只看持币分红
const holdingDividends = await contract.myDividends(false);

// 推荐奖励
const referralBonus = totalDividends - holdingDividends;

console.log(`持币分红: ${holdingDividends}`);
console.log(`推荐奖励: ${referralBonus}`);
console.log(`总计: ${totalDividends}`);
```

## ⚠️ 重要提醒

### 风险警告

1. **这是庞氏骗局** - 分红来自后来者的损失
2. **没有保证** - 价格可能归零
3. **流动性风险** - 可能无法提取
4. **智能合约风险** - 可能有漏洞

### 提取失败的可能原因

```
❌ 分红为 0
❌ 合约 ETH 不足（被挤兑）
❌ Gas 不足
❌ 合约暂停（虽然这个合约没有暂停功能）
```

## 📈 收益计算

### 简化公式

```
总收益 = 代币增值 + 累积分红 - 买入成本

盈亏平衡 = 买入成本 × 1.25
（考虑 20% 的买卖手续费）
```

### 实际例子

```
买入: 10 ETH
实际获得代币: 9 ETH 价值

情况 A - 价格上涨 30%
  代币价值: 9 × 1.3 = 11.7 ETH
  卖出获得: 11.7 × 0.9 = 10.53 ETH
  分红: 1 ETH
  总收益: 10.53 + 1 - 10 = 1.53 ETH ✅ 赚钱

情况 B - 价格不变
  代币价值: 9 ETH
  卖出获得: 9 × 0.9 = 8.1 ETH
  分红: 1 ETH
  总收益: 8.1 + 1 - 10 = -0.9 ETH ❌ 亏钱
```

## 🎓 总结

### 三种方式的选择

- 💸 **需要现金** → `withdraw()`
- 📈 **追求复利**（风险自负）→ `reinvest()`
- 🚪 **想要离场** → `exit()`

### 记住

1. 这是零和游戏
2. 大多数人会亏钱
3. 早提取早安全
4. 不要投入输不起的钱

**仅供学习研究，切勿实际投资！**

