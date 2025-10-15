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