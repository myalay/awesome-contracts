# 🔐 Awesome Contract - 危险智能合约安全研究项目

<div align="center">

![License](https://img.shields.io/badge/license-MIT-blue.svg)
![Solidity](https://img.shields.io/badge/Solidity-0.4.x--0.8.x-green.svg)
![Status](https://img.shields.io/badge/status-教育研究-orange.svg)

**⚠️ 警告：本项目收录的所有合约均具有极高风险，仅供学习研究，切勿部署或使用！**

</div>

---

## 📖 项目简介

**Awesome Contract** 是一个专注于区块链智能合约安全研究的教育项目。本项目收录了以太坊历史上一些**极具争议性和危险性**的智能合约，对其进行深度技术分析、安全漏洞剖析和风险评估，旨在：

- 🎓 **教育目的**：帮助开发者理解智能合约的安全风险
- 🔍 **安全研究**：分析真实案例中的漏洞和攻击手法
- ⚠️ **警示作用**：揭示区块链生态中的欺诈模式
- 💡 **技术学习**：从反面案例学习合约设计的陷阱

> "学习历史是为了避免重蹈覆辙。" 
> 
> 这些合约曾造成数千万美元的损失和无数用户的财产损失。通过研究它们，我们可以更好地理解什么是**不应该**做的。

---

## 🎯 项目目标

### 核心使命

1. **安全教育**
   - 提供完整的合约源码和详细注释
   - 深入分析每个合约的工作机制
   - 揭示潜在的安全漏洞和攻击向量

2. **风险警示**
   - 明确标注每个合约的危险等级
   - 详细说明可能造成的经济损失
   - 提供真实的历史损失数据

3. **技术研究**
   - 分析独特的技术实现
   - 研究经济模型和博弈论
   - 探讨智能合约设计的边界

4. **行业规范**
   - 推动智能合约安全标准
   - 促进开发者安全意识
   - 为监管和审计提供参考

---

## 📂 项目结构

```
awesome-contract/
├── README.md                    # 项目总览（本文件）
├── LICENSE                      # 开源许可证
└── solidity/                    # Solidity 合约目录
    ├── fomo3d/                  # FoMo3D 博彩游戏合约
    │   ├── Fomo3D.sol          # 完整合约源码
    │   └── README.md           # 详细分析文档
    └── powh3d/                  # PoWH3D 庞氏分红合约
        ├── PowH3D.sol          # 完整合约源码
        └── README.md           # 详细分析文档
```

---

## 🗂️ 已收录合约

### 危险等级说明

- 🔴 **极度危险**：庞氏骗局、高概率资金损失
- 🟠 **高度危险**：严重漏洞、可被攻击操纵
- 🟡 **中度危险**：设计缺陷、经济模型问题
- 🔵 **低度危险**：轻微问题、教育意义

---

### 1. 🔴 FoMo3D - 倒计时博彩游戏

<table>
<tr>
<td><strong>合约名称</strong></td>
<td>Fear of Missing Out 3D</td>
</tr>
<tr>
<td><strong>危险等级</strong></td>
<td>🔴 极度危险</td>
</tr>
<tr>
<td><strong>合约类型</strong></td>
<td>博彩游戏 + 庞氏骗局</td>
</tr>
<tr>
<td><strong>发布时间</strong></td>
<td>2018年7月</td>
</tr>
<tr>
<td><strong>历史损失</strong></td>
<td>约 30,000 ETH（数千万美元）</td>
</tr>
<tr>
<td><strong>主要风险</strong></td>
<td>
• 博彩性质，大概率亏损<br>
• 可被"区块阻塞攻击"操纵<br>
• 手续费高达20-60%<br>
• 奖池可能被鲸鱼垄断
</td>
</tr>
<tr>
<td><strong>技术特点</strong></td>
<td>
• 倒计时机制<br>
• 复杂的资金分配系统<br>
• 空投奖励机制<br>
• 团队选择策略
</td>
</tr>
<tr>
<td><strong>已知漏洞</strong></td>
<td>
• isHuman() 防护可被绕过<br>
• 易受区块阻塞攻击<br>
• 博弈论导致后期停滞
</td>
</tr>
<tr>
<td><strong>学习价值</strong></td>
<td>
✅ 复杂状态管理<br>
✅ 事件驱动架构<br>
✅ 博弈论实践<br>
❌ 反面教材：如何不设计游戏
</td>
</tr>
</table>

📁 **详细分析**：[solidity/fomo3d/README.md](./solidity/fomo3d/README.md)

**核心机制**：
```
玩家用 ETH 购买 Keys → 成为"最后买家"
↓
每次购买延长倒计时 30 秒
↓
倒计时结束时，最后买家赢得奖池 48%
```

**真实案例**：
- 第一轮被黑客通过"区块阻塞攻击"赢得 10,469 ETH
- 攻击者填满区块，阻止其他人交易
- 官网域名就叫 `exitscam.me`（退出骗局）

---

### 2. 🔴 PoWH3D - 庞氏分红系统

<table>
<tr>
<td><strong>合约名称</strong></td>
<td>Proof of Weak Hands 3D</td>
</tr>
<tr>
<td><strong>危险等级</strong></td>
<td>🔴 极度危险</td>
</tr>
<tr>
<td><strong>合约类型</strong></td>
<td>庞氏骗局 + 分红系统</td>
</tr>
<tr>
<td><strong>发布时间</strong></td>
<td>2018年2月</td>
</tr>
<tr>
<td><strong>历史损失</strong></td>
<td>数百万美元（多次版本迭代）</td>
</tr>
<tr>
<td><strong>主要风险</strong></td>
<td>
• 典型的庞氏骗局结构<br>
• 买卖手续费高达20%<br>
• 必然崩盘（无新买家时）<br>
• 代币无任何内在价值
</td>
</tr>
<tr>
<td><strong>技术特点</strong></td>
<td>
• 联合曲线定价（Bonding Curve）<br>
• 自动分红机制（profitPerShare）<br>
• Masternode推荐系统<br>
• 复利再投资功能
</td>
</tr>
<tr>
<td><strong>数学模型</strong></td>
<td>
• 线性价格增长：P(n) = P₀ + (n-1) × Δ<br>
• 精密的分红计算系统<br>
• Gas优化的分红分配
</td>
</tr>
<tr>
<td><strong>学习价值</strong></td>
<td>
✅ 联合曲线（Bonding Curve）设计<br>
✅ 高效分红分配算法<br>
✅ 精度处理（magnitude 放大）<br>
❌ 反面教材：庞氏骗局的典范
</td>
</tr>
</table>

📁 **详细分析**：[solidity/powh3d/README.md](./solidity/powh3d/README.md)

**核心机制**：
```
买入代币（扣10%手续费）→ 手续费分配给所有持币人
↓
持币自动获得分红
↓
卖出代币（再扣10%手续费）→ 手续费再次分配
```

**巧妙设计**：
- 使用 `profitPerShare_` 实现O(1)复杂度的分红分配
- 联合曲线确保价格可预测
- 但本质仍是零和游戏

---

## 🎓 如何使用本项目

### 适用人群

- ✅ 智能合约开发者（学习安全陷阱）
- ✅ 安全审计人员（了解攻击模式）
- ✅ 区块链研究者（研究经济模型）
- ✅ 学生和教育工作者（教学案例）
- ❌ **普通投资者**（这不是投资指南！）

### 学习路径

#### 1. 初级学习者

```
开始 → 阅读主 README（本文件）
  ↓
  了解每个合约的基本机制
  ↓
  查看风险警示部分
  ↓
  理解为什么不应该使用这些合约
```

#### 2. 中级开发者

```
开始 → 阅读合约的详细 README
  ↓
  研究合约源码
  ↓
  理解技术实现细节
  ↓
  分析已知漏洞和攻击向量
  ↓
  思考如何避免类似问题
```

#### 3. 高级研究者

```
开始 → 深入研究合约代码
  ↓
  尝试在测试网部署和测试
  ↓
  编写安全分析报告
  ↓
  寻找未被发现的漏洞
  ↓
  贡献改进的分析文档
```

### 阅读每个合约分析文档

每个合约的 README.md 包含：

1. **📋 合约概述**
   - 基本信息（部署时间、版本、链接）
   - 设计目的和背景

2. **⚙️ 技术实现**
   - 代码架构分析
   - 关键函数解析
   - 数据结构说明

3. **💰 经济模型**
   - 资金流动图
   - 激励机制
   - 博弈论分析

4. **🐛 安全漏洞**
   - 已知漏洞列表
   - 攻击案例复现
   - 修复建议

5. **⚠️ 风险评估**
   - 危险等级
   - 可能造成的损失
   - 真实历史数据

6. **🎓 学习价值**
   - 值得学习的技术点
   - 需要避免的反模式
   - 相关资源链接

---

## ⚠️ 严重警告

<div align="center" style="background: #ff0000; color: white; padding: 10px; margin: 10px 0;">

### 🚨 请勿将这些合约用于生产环境！🚨

</div>

### 法律风险

- 🚫 **在大多数国家，部署这类合约可能违法**
  - 可能构成非法集资
  - 可能构成诈骗罪
  - 可能构成赌博罪

- ⚖️ **法律后果**
  - 刑事责任
  - 民事赔偿
  - 资产冻结

### 经济风险

- 💸 **参与者必然损失**
  - 庞氏骗局注定崩盘
  - 博彩游戏大概率亏损
  - 高额手续费侵蚀本金

- 📉 **历史数据**
  - FoMo3D：大多数玩家亏损 >50%
  - PoWH3D：仅早期少数人获利
  - 无数个类似项目归零

### 技术风险

- 🐛 **智能合约漏洞**
  - 可能存在未知bug
  - 可能被黑客攻击
  - 无法回滚或修复

- 🔓 **权限问题**
  - 可能有后门
  - 管理员可能跑路
  - 资金可能被盗

### 道德风险

- 🤔 **参与即为共犯**
  - 推广这类项目害人害己
  - 破坏区块链行业声誉
  - 造成社会危害

---

## 📚 教育意义

尽管这些合约极其危险，但它们具有重要的**教育和研究价值**：

### 1. 技术层面

#### ✅ 值得学习的技术

- **联合曲线（Bonding Curve）**
  ```solidity
  // PoWH3D 的定价算法
  P(n) = P₀ + (n - 1) × Δ
  ```
  可用于：代币发售、自动做市、预测市场

- **高效分红系统**
  ```solidity
  // 使用 profitPerShare 实现 O(1) 分红
  dividends = (profitPerShare × tokens - paidOut) / magnitude
  ```
  可用于：DAO分红、质押奖励、利润共享

- **Gas 优化技巧**
  - 避免遍历数组
  - 使用事件而非存储
  - 批量操作合并

- **模块化架构**
  - 合约分层
  - 库的使用
  - 接口设计

#### ❌ 需要避免的反模式

- **isHuman() 伪检测**
  ```solidity
  // 可被绕过！
  require(extcodesize(addr) == 0, "humans only");
  ```
  
- **区块时间依赖**
  ```solidity
  // 可被矿工操纵！
  if (now > endTime) { ... }
  ```

- **缺乏紧急暂停机制**
  ```solidity
  // 应该有：
  modifier whenNotPaused() { ... }
  ```

### 2. 经济学层面

#### 博弈论分析

- **纳什均衡**：理解多人博弈
- **激励机制**：如何设计代币经济
- **零和游戏**：认识庞氏骗局本质

#### 机制设计

- **拍卖理论**：FoMo3D的倒计时机制
- **流动性曲线**：PoWH3D的价格发现
- **推荐系统**：多级分销的陷阱

### 3. 安全审计

#### 常见漏洞类型

| 漏洞类型 | 示例合约 | 影响 |
|---------|---------|------|
| 重入攻击 | DAO | 资金被盗 |
| 整数溢出 | BeautyChain | 代币增发 |
| 权限控制 | Parity | 合约自毁 |
| 区块阻塞 | FoMo3D | 游戏操纵 |
| 前端依赖 | PoWH3D | 访问风险 |

#### 审计清单

```markdown
- [ ] 重入攻击防护（ReentrancyGuard）
- [ ] 整数溢出检查（SafeMath）
- [ ] 权限控制完善（Ownable, RBAC）
- [ ] 时间戳依赖审查
- [ ] 外部调用风险评估
- [ ] Gas限制考虑
- [ ] 前端安全（防钓鱼）
```

---

## 🔬 研究方法

如果你想深入研究这些合约（仅限学术目的），建议以下流程：

### 1. 环境准备

```bash
# 克隆本项目
git clone https://github.com/myalay/awesome-contract.git
cd awesome-contract

# 安装 Hardhat 和依赖
npm install --save-dev hardhat
npm install

# 初始化 Hardhat 项目（如需要）
npx hardhat init

# 启动本地测试网络
npx hardhat node
```

### 2. 静态分析

```bash
# 使用 Slither 进行静态分析
pip install slither-analyzer
slither solidity/fomo3d/Fomo3D.sol

# 使用 Mythril 检测漏洞
myth analyze solidity/powh3d/PowH3D.sol
```

### 3. 测试网部署

```javascript
// 仅在测试网（如 Goerli、Sepolia）部署！
// 切勿在主网部署！

const contract = await deployer.deploy(
  Fomo3D,
  { network: 'goerli' }  // 必须是测试网
);
```

### 4. 漏洞复现

```javascript
// 复现区块阻塞攻击（在测试环境）
async function blockStuffing() {
  // 发送大量高 Gas 交易
  for (let i = 0; i < 100; i++) {
    await sendHighGasTx();
  }
  // 然后发送关键交易
  await buyKeys();
}
```

### 5. 撰写报告

```markdown
# 安全分析报告模板

## 1. 合约概述
- 合约名称
- 功能描述
- 部署信息

## 2. 代码审计
- 架构分析
- 关键函数
- 依赖关系

## 3. 漏洞发现
- 漏洞类型
- 严重程度
- 复现步骤
- 影响范围

## 4. 修复建议
- 短期方案
- 长期方案
- 最佳实践

## 5. 总结
- 风险评级
- 使用建议
```

---

## 🤝 贡献指南

我们欢迎安全研究者和开发者贡献更多案例和分析！

### 贡献内容

#### 1. 新增合约案例

如果你发现值得收录的危险合约，请提供：

- ✅ 完整的合约源码
- ✅ 详细的技术分析文档
- ✅ 安全风险评估
- ✅ 真实的历史数据（如有）
- ✅ 相关的新闻报道链接

#### 2. 改进现有分析

- 补充漏洞细节
- 更新历史数据
- 添加更多案例
- 修正错误信息

#### 3. 翻译文档

帮助将文档翻译成其他语言（英文、日文、韩文等）

### 贡献流程

```bash
# 1. Fork 本项目
# 2. 创建特性分支
git checkout -b feature/add-new-contract

# 3. 添加你的内容
# 遵循现有的目录结构和文档格式

# 4. 提交更改
git commit -m "feat: 添加 XXX 合约分析"

# 5. 推送到你的仓库
git push origin feature/add-new-contract

# 6. 创建 Pull Request
```

### 提交规范

遵循 [Conventional Commits](https://www.conventionalcommits.org/) 规范：

```
feat: 添加新功能
fix: 修复bug
docs: 文档更新
style: 代码格式调整
refactor: 代码重构
test: 测试相关
chore: 构建/工具链更新
```

### 审核标准

我们会审查：

- ✅ 技术分析的准确性
- ✅ 风险警示的充分性
- ✅ 文档的完整性
- ✅ 代码的可读性
- ❌ 是否包含恶意代码
- ❌ 是否有误导性内容

---

## 📋 项目路线图

### 已完成 ✅

- [x] 项目初始化
- [x] FoMo3D 完整分析
- [x] PoWH3D 完整分析
- [x] 主 README 编写

### 进行中 🚧

- [ ] 添加更多经典案例
- [ ] 建立分类体系
- [ ] 开发分析工具

### 计划中 📅

#### 第一阶段：经典庞氏合约

- [ ] PlusToken 合约分析
- [ ] MMM 金字塔合约
- [ ] 各类资金盘合约

#### 第二阶段：DeFi 漏洞合约

- [ ] The DAO 合约（重入攻击）
- [ ] Parity 钱包（权限漏洞）
- [ ] BeautyChain（整数溢出）
- [ ] bZx 闪电贷攻击案例

#### 第三阶段：NFT 和 GameFi

- [ ] 有漏洞的 NFT 合约
- [ ] 链游经济崩盘案例
- [ ] Rug Pull（跑路）案例

#### 第四阶段：工具和自动化

- [ ] 合约安全扫描工具
- [ ] 自动化漏洞检测
- [ ] 可视化分析工具
- [ ] 交互式教学平台

---

## 🔗 相关资源

### 官方文档

- [Solidity 官方文档](https://docs.soliditylang.org/)
- [Ethereum 开发者资源](https://ethereum.org/en/developers/)
- [OpenZeppelin 合约库](https://docs.openzeppelin.com/contracts/)

### 安全工具

- **[Slither](https://github.com/crytic/slither)** - 静态分析工具
- **[Mythril](https://github.com/ConsenSys/mythril)** - 安全分析框架
- **[Echidna](https://github.com/crytic/echidna)** - 模糊测试工具
- **[MythX](https://mythx.io/)** - 商业化安全服务

### 学习资源

- **[CryptoZombies](https://cryptozombies.io/)** - 智能合约教程
- **[Ethernaut](https://ethernaut.openzeppelin.com/)** - 安全挑战
- **[Damn Vulnerable DeFi](https://www.damnvulnerabledefi.xyz/)** - DeFi 安全挑战
- **[Smart Contract Security Best Practices](https://consensys.github.io/smart-contract-best-practices/)**

### 审计报告

- **[OpenZeppelin Audits](https://blog.openzeppelin.com/security-audits/)**
- **[Trail of Bits Audits](https://github.com/trailofbits/publications)**
- **[Consensys Diligence](https://consensys.net/diligence/)**

### 事故分析

- **[Rekt News](https://rekt.news/)** - DeFi 黑客事件追踪
- **[Blockchain Threat Intelligence](https://newsletter.blockthreat.io/)**
- **[DeFi Hacks Analysis](https://github.com/SunWeb3Sec/DeFiHackLabs)**

---

## 📊 统计数据

### 历史损失统计

根据公开数据，本项目收录的合约造成的损失：

| 合约 | 时间 | 总投入 | 估计损失 | 受害者数量 |
|------|------|--------|---------|-----------|
| FoMo3D | 2018年 | ~30,000 ETH | ~$10M+ | 数千人 |
| PoWH3D | 2018年 | 未知 | 数百万美元 | 数千人 |
| **总计** | - | - | **>$10M** | **数千人** |

*注：以上数据基于公开信息估算，实际损失可能更高*

### 常见风险分类

```
庞氏骗局型: 60%
├─ PoWH3D 类分红系统
├─ 多级分销模式
└─ 资金池模式

博彩游戏型: 30%
├─ FoMo3D 类倒计时游戏
├─ 概率类游戏
└─ 竞猜类游戏

其他类型: 10%
├─ 假币发行
├─ 虚假项目
└─ 纯粹骗局
```

---

## 🙏 致谢

感谢以下项目和个人为区块链安全做出的贡献：

- **安全研究机构**
  - OpenZeppelin
  - Trail of Bits
  - Consensys Diligence
  - PeckShield
  - SlowMist

- **开源社区**
  - Ethereum Foundation
  - Solidity Team
  - All security researchers

- **教育资源**
  - Smart Contract Security Best Practices
  - SWC Registry
  - Ethereum Security Tooling

---

### ⚠️ 免责声明

```
本项目仅供教育和研究目的使用。

1. 本项目收录的所有合约均为真实历史上的危险合约
2. 这些合约已造成实际的经济损失和社会危害
3. 我们强烈反对部署、使用或推广这类合约
4. 参与此类项目可能违反您所在地区的法律
5. 因使用本项目中的代码造成的任何损失，作者不承担责任
6. 请仅将本项目用于学习、研究和教育目的

如果您发现任何人使用本项目中的代码进行非法活动，
请立即向相关执法部门报告。
```

---

### 📞 联系方式

- **问题反馈**：[GitHub Issues](https://github.com/myalay/awesome-contracts/issues)

---

## 🌟 Star History

如果这个项目对你的学习和研究有帮助，请给我们一个 Star ⭐

<!-- [![Star History Chart](https://api.star-history.com/svg?repos=myalay/awesome-contract&type=Date)](https://star-history.com/#myalay/awesome-contract&Date) -->

---

## 📝 更新日志

### v1.0.0 (2024-xx-xx)

- ✨ 初始版本发布
- ✅ 添加 FoMo3D 完整分析
- ✅ 添加 PoWH3D 完整分析
- 📚 完善项目文档
- 🎨 优化文档结构

---

**最后更新**: 2024年10月

**维护状态**: 🟢 活跃维护中