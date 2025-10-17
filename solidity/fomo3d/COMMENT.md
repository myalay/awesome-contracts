# FoMo3D 合约附加说明

## now 获取区块时间的漏洞

在 Solidity 中，**`now`** 是一个**全局变量**，它是 `block.timestamp` 的别名，表示**当前区块的时间戳**（以秒为单位的 Unix 时间戳）。

### 在这段代码中的作用

```solidity
uint256 _now = now;  // 获取当前区块时间戳
```

在 Fomo3D 的 `updateTimer` 函数中，`now` 被用来：
1. **获取当前时间**：记录当前区块的时间戳
2. **计算倒计时结束时间**：根据购买的 keys 数量延长游戏时间
3. **判断游戏是否结束**：检查当前时间是否超过回合结束时间

### ⚠️ 重要的安全问题

`now`（即 `block.timestamp`）存在**严重的安全隐患**：

#### 1. **可被矿工操纵**
```solidity
// ❌ 危险！矿工可以在一定范围内操纵时间戳
if (now > round_[_rID].end) {
    // 矿工可以调整 15-30 秒的时间差
}
```

矿工可以在合理范围内（约 15-30 秒）操纵区块时间戳，这在 Fomo3D 这种对时间敏感的游戏中可能导致不公平。

#### 2. **Solidity 版本变化**

| Solidity 版本 | 状态 | 说明 |
|--------------|------|------|
| 0.4.x - 0.6.x | ✅ 支持 | `now` 可以使用 |
| 0.7.0+ | ⚠️ 废弃 | `now` 被废弃，建议用 `block.timestamp` |
| 0.8.0+ | ❌ 移除 | 必须使用 `block.timestamp` |

**现代写法**：
```solidity
// ✅ 推荐写法（Solidity 0.8.x）
uint256 _now = block.timestamp;
```

#### 3. **在 Fomo3D 中的风险**

这正是 README 中提到的安全问题之一：

```markdown
❌ 需要避免的反模式

- 区块时间依赖
  // 可被矿工操纵！
  if (now > endTime) { ... }
```

在 Fomo3D 游戏中，矿工理论上可以：
- 略微调整时间戳，影响倒计时
- 配合"区块阻塞攻击"，操纵游戏结果

### 总结

- **`now` = `block.timestamp`**：当前区块的 Unix 时间戳
- **用途**：时间相关的逻辑判断
- **风险**：可被矿工在小范围内操纵
- **建议**：现代合约应使用 `block.timestamp` 并注意时间依赖的安全问题

这也是为什么 Fomo3D 被列为**极度危险**合约的原因之一！🔴

## extcodesize漏洞

**`extcodesize`** 是 Solidity 的一个**底层汇编指令**（Assembly opcode），用于获取指定地址处的**合约代码大小**（字节数）。

### 在这段代码中的用法

```solidity
modifier isHuman() {
    address _addr = msg.sender;
    uint256 _codeLength;
    
    assembly {_codeLength := extcodesize(_addr)}  // 获取地址的代码长度
    require(_codeLength == 0, "sorry humans only");  // 如果代码长度为0，则是EOA
    _;
}
```

### 工作原理

| 地址类型 | extcodesize 返回值 | 含义 |
|---------|-------------------|------|
| **EOA（外部账户）** | `0` | 普通用户钱包，没有合约代码 |
| **智能合约** | `> 0` | 合约地址，返回合约字节码长度 |

**Fomo3D 的意图**：只允许人类用户（EOA）参与，禁止智能合约调用，防止机器人和自动化攻击。

### ⚠️ 严重的安全漏洞！

这个检测方法**可以被轻易绕过**！这正是 README 中提到的反模式：

```markdown
❌ 需要避免的反模式

- isHuman() 伪检测
  // 可被绕过！
  require(extcodesize(addr) == 0, "humans only");
```

### 🔓 绕过方法

#### 方法 1：在构造函数中调用
```solidity
contract Attacker {
    constructor() {
        // ✅ 在构造函数中，extcodesize(address(this)) 返回 0！
        // 因为合约代码还未部署完成
        FoMo3D game = FoMo3D(targetAddress);
        game.buyXid{value: 1 ether}(1);  // 可以成功调用！
    }
}
```

**原理**：在合约的构造函数执行期间，合约代码还没有被完全部署到区块链上，所以 `extcodesize` 返回 0。

#### 方法 2：使用 CREATE2 预计算地址

```solidity
// 高级攻击：利用 CREATE2 预测地址并提前调用
```

### 为什么这样设计？

Fomo3D 试图防止：
- 🤖 **机器人批量购买**：阻止自动化脚本
- ⚡ **闪电交易**：防止在同一交易中进行复杂操作
- 📊 **套利合约**：阻止智能合约进行套利

但这个防护**形同虚设**！

### 正确的做法

**重要认知：无法真正阻止合约调用，也不应该阻止**

```solidity
// ❌ 不要这样做 - 试图阻止合约调用
modifier isHuman() {
    assembly {_codeLength := extcodesize(msg.sender)}
    require(_codeLength == 0, "humans only");
    _;
}
// 问题：可在构造函数中绕过，且破坏可组合性

// ✅ 正确做法 - 防止真正的漏洞（重入攻击）
modifier nonReentrant() {
    require(!locked, "No reentrancy");
    locked = true;
    _;
    locked = false;
}
// 注意：这只防止重入，不防止合约调用（也不应该防止）

// ✅ 现代最佳实践
// 1. 使用 OpenZeppelin 的 ReentrancyGuard 防止重入
// 2. 接受可组合性，允许合约交互
// 3. 使用其他安全机制：CEI模式、时间锁、紧急暂停等
```

### 关键区别

| 防护机制 | 目的 | 是否可行 | 是否应该做 |
|---------|------|---------|-----------|
| `extcodesize` | 阻止合约调用 | ❌ 无法实现 | ❌ 破坏可组合性 |
| `nonReentrant` | 防止重入攻击 | ✅ 可以实现 | ✅ 必要的安全措施 |

**现代 DeFi 哲学**：拥抱可组合性，防护真实漏洞，而非盲目排斥合约交互。

```solidity
// 现代 DeFi 的做法
contract ModernDeFi {
    // 1. 防止重入攻击
    bool private locked;
    modifier nonReentrant() {
        require(!locked);
        locked = true;
        _;
        locked = false;
    }
    
    // 2. 接受合约调用，设计可组合性
    function stake(uint256 amount) external nonReentrant {
        // 允许任何地址（EOA或合约）调用
        // 但防止重入攻击
    }
    
    // 3. 使用其他安全机制
    // - 检查效应交互模式（CEI Pattern）
    // - 紧急暂停开关
    // - 时间锁
    // - 多签治理
}
```

- ❌ 不要试图阻止合约调用（做不到，也不应该）
- ✅ 应该防止具体的攻击向量（重入、溢出、权限等）
- ✅ 设计可组合性（让合约之间能安全交互）

### Assembly 语法说明

```solidity
assembly {
    _codeLength := extcodesize(_addr)
}
```

- `assembly { }` - 内联汇编块
- `:=` - 赋值操作符（在汇编中）
- `extcodesize(addr)` - 获取地址的代码大小（以字节为单位）

### 总结

- **`extcodesize`**：EVM 操作码，返回地址的合约代码字节数
- **返回 0**：EOA（外部账户/用户钱包）
- **返回 > 0**：智能合约地址
- **致命缺陷**：可在构造函数中绕过，**不能**真正阻止合约调用
- **教训**：不要依赖 `extcodesize` 来区分用户和合约

这也是为什么 Fomo3D 被列为研究案例的原因之一 —— 它展示了一个看似聪明但实际无效的安全机制！🔴

---

## Fomo3D 的分红机制 - Mask算法

Fomo3D 使用了一个非常巧妙的 **Mask 算法**来实现 **O(1) 复杂度**的多人分红，无需遍历所有玩家。

### 核心问题

如何在 Gas 费用有限的情况下，高效地给成千上万个持有 Keys 的玩家分配分红？

❌ **低效方法**（不可行）：
```solidity
// 遍历所有玩家 - Gas 费用极高！
for (uint i = 0; i < allPlayers.length; i++) {
    dividends[players[i]] += earnings * keys[players[i]] / totalKeys;
}
```

✅ **高效方法**（Mask 算法）：
```solidity
// O(1) 复杂度 - 只更新全局变量！
round_[_rID].mask += profitPerKey;
```

### Mask 算法原理

#### 核心数据结构

```solidity
// 全局：回合级别
struct Round {
    uint256 mask;      // 累积的每 Key 收益（放大 1e18 倍）
    uint256 keys;      // 总 Keys 数量
    uint256 pot;       // 奖池
    // ... 其他字段
}

// 个人：玩家在某回合的数据
struct PlayerRounds {
    uint256 keys;      // 玩家持有的 Keys 数量
    uint256 mask;      // 玩家的收益掩码（已结算的收益）
}
```

#### 算法步骤

**1️⃣ 当有收益需要分配时（有人购买 Keys）**

```solidity
function updateMasks(uint256 _rID, uint256 _pID, uint256 _gen, uint256 _keys)
    private
    returns(uint256)
{
    // 计算每个 Key 的收益（Profit Per Token）
    // _gen 是要分配的总收益，keys 是总 Key 数
    uint256 _ppt = (_gen * 1e18) / round_[_rID].keys;
    
    // 🔑 关键：更新全局 mask（累加）
    round_[_rID].mask = _ppt + round_[_rID].mask;
    
    // 更新玩家的 mask（防止自己给自己分红）
    uint256 _pearn = (_ppt * _keys) / 1e18;
    plyrRnds_[_pID][_rID].mask = 
        (((round_[_rID].mask * _keys) / 1e18) - _pearn) + plyrRnds_[_pID][_rID].mask;
    
    // 返回精度损失的尘埃（dust）
    return(_gen - (_ppt * round_[_rID].keys) / 1e18);
}
```

**2️⃣ 计算玩家未领取的收益**

```solidity
function calcUnMaskedEarnings(uint256 _pID, uint256 _rIDlast)
    private
    view
    returns(uint256)
{
    // 公式：(全局 mask × 我的 Keys) - 我的 mask
    return (
        ((round_[_rIDlast].mask * plyrRnds_[_pID][_rIDlast].keys) / 1e18) 
        - plyrRnds_[_pID][_rIDlast].mask
    );
}
```

### 工作流程示例

假设有 3 个玩家：

```
初始状态：
- Alice: 100 Keys
- Bob:   200 Keys  
- Carol: 200 Keys
- 总计:  500 Keys
- round.mask = 0
- 所有玩家的 mask = 0
```

**第 1 轮分红：分配 10 ETH**

```solidity
// 每 Key 收益 = 10 ETH / 500 Keys = 0.02 ETH
ppt = (10 * 1e18) / 500 = 0.02 * 1e18

// 更新全局 mask
round.mask = 0 + 0.02e18 = 0.02e18

// 各玩家应得收益：
Alice: 100 * 0.02 = 2 ETH  ✅
Bob:   200 * 0.02 = 4 ETH  ✅
Carol: 200 * 0.02 = 4 ETH  ✅
```

**第 2 轮分红：再分配 5 ETH**

```solidity
// 每 Key 收益 = 5 ETH / 500 Keys = 0.01 ETH
ppt = (5 * 1e18) / 500 = 0.01 * 1e18

// 更新全局 mask（累加）
round.mask = 0.02e18 + 0.01e18 = 0.03e18

// 各玩家累计收益：
Alice: 100 * 0.03 = 3 ETH  ✅ (第1轮 2 + 第2轮 1)
Bob:   200 * 0.03 = 6 ETH  ✅ (第1轮 4 + 第2轮 2)
Carol: 200 * 0.03 = 6 ETH  ✅ (第1轮 4 + 第2轮 2)
```

**Alice 提取收益后**

```solidity
// Alice 提取 3 ETH
alice.mask = 0.03e18 * 100 = 3e18  // 标记已提取

// 再次计算 Alice 的收益
earnings = (round.mask * alice.keys) / 1e18 - alice.mask
         = (0.03e18 * 100) / 1e18 - 3
         = 3 - 3 = 0 ✅ 正确！
```

### 资金分配流程

每次有人购买 Keys 时，ETH 被这样分配：

```solidity
function distributeInternal(uint256 _rID, uint256 _pID, uint256 _eth, ...)
{
    // 假设收到 100 ETH
    
    // 1. 计算分红份额（假设 40%）
    uint256 _gen = (_eth * 40) / 100;  // 40 ETH 给所有 Key 持有者
    
    // 2. 空投池（1%）
    uint256 _air = _eth / 100;         // 1 ETH
    airDropPot_ += _air;
    
    // 3. 扣除外部分配（14% + P3D份额）
    _eth = _eth.sub(固定费用);
    
    // 4. 剩余的进入奖池
    uint256 _pot = _eth.sub(_gen);     // 剩余进入奖池
    
    // 5. 🔑 更新 mask 分配收益（O(1) 操作！）
    uint256 _dust = updateMasks(_rID, _pID, _gen, _keys);
    
    // 6. 尘埃归入奖池
    round_[_rID].pot = _pot + _dust + round_[_rID].pot;
}
```

### 收益类型

玩家有 3 种收益来源：

```solidity
struct Player {
    uint256 win;    // 赢得大奖（48%奖池）
    uint256 gen;    // 分红收益（从 mask 计算）
    uint256 aff;    // 推荐收益（10%）
}

// 提取时计算总收益
function withdrawEarnings(uint256 _pID) private returns(uint256) {
    // 1. 更新分红收益
    updateGenVault(_pID, plyr_[_pID].lrnd);
    
    // 2. 总收益 = 大奖 + 分红 + 推荐
    uint256 _earnings = plyr_[_pID].win + plyr_[_pID].gen + plyr_[_pID].aff;
    
    // 3. 清零
    if (_earnings > 0) {
        plyr_[_pID].win = 0;
        plyr_[_pID].gen = 0;
        plyr_[_pID].aff = 0;
    }
    
    return _earnings;
}
```

### 技术亮点

✅ **优点**：

1. **O(1) 时间复杂度**：无论有多少玩家，分红只需一次操作
2. **Gas 高效**：不需要遍历数组，极大节省 Gas
3. **实时计算**：随时可以查询未领取的收益
4. **精度处理**：乘以 1e18 避免小数，尘埃归入奖池

❌ **缺点**：

1. **复杂难懂**：新手很难理解 mask 的工作原理
2. **精度损失**：整数除法会产生"尘埃"（dust）
3. **依赖全局状态**：round.mask 必须正确维护

### 数学公式

```
玩家未领取收益 = (全局 mask × 玩家 Keys) - 玩家 mask

其中：
- 全局 mask = Σ(每次分红金额 / 总Keys)
- 玩家 mask = 已经结算给该玩家的收益

精度放大系数 = 1e18（避免小数）
```

### 对比 PoWH3D

Fomo3D 的 Mask 算法和 PoWH3D 的 `profitPerShare` 机制**本质相同**：

| 项目 | 变量名 | 用途 |
|------|--------|------|
| Fomo3D | `round.mask` | 累积的每 Key 收益 |
| PoWH3D | `profitPerShare_` | 累积的每代币收益 |
| 共同点 | O(1) 分红 | 高效的多人分红算法 |

### 总结

Fomo3D 的分红机制是其**技术上最精彩的部分**之一：

- ✅ **算法巧妙**：Mask 算法实现 O(1) 分红
- ✅ **Gas 优化**：即使有 10000 个玩家也不会卡住
- ✅ **数学精准**：精度处理得当（除了微小尘埃）
- ⚠️ **但依然危险**：技术优秀不等于项目安全

这是一个**值得学习的技术模式**，广泛应用于现代 DeFi 项目中的质押、分红、流动性挖矿等场景。