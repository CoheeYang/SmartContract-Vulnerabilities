[TOC]

# 1. Uniswap v4 核心功能与架构解析

## 1.1 概述

Uniswap v4 是基于以太坊虚拟机（EVM）的非托管自动化做市商（AMM）协议，旨在通过高度可定制的池设计和架构优化，解决此前版本的局限性。

其核心改进包括**钩子（Hooks）**、**单例模式（Singleton）**、**闪电记账（Flash Accounting）** 和**原生ETH支持**，同时引入灵活的记账机制和开发者工具。

### 1. 钩子（Hooks）

- **功能扩展**：开发者可在池的生命周期（如初始化、添加/移除流动性、交换前后）插入自定义逻辑。例如：
  - 动态调整手续费（如基于波动性的费用模型）。
  - 实现限价订单、时间加权交易（TWAMM）。
  - 集成预言机或MEV保护机制。
- **灵活性**：钩子通过回调函数（如 `beforeSwap`/`afterSwap`）实现，池创建时可指定钩子合约，且回调权限在池初始化后不可变更。
- **费用分配**：钩子可收取部分手续费，并分配给流动性提供者、开发者或其他方。

### 2. 单例模式（Singleton）
- **统一管理**：所有池状态由单一合约 `PoolManager.sol` 管理，相比此前版本的工厂模式，池部署成本降低约99%。
- **高效路由**：结合闪电记账，多池原子操作（如跨池交易）的Gas成本显著减少。

### 3. 闪电记账（Flash Accounting）
- **临时存储**：利用 EIP-1153 的瞬态存储操作码，仅在交易结束时结算净余额（Delta），避免中间状态写入存储的高成本。
- **原子性保证**：通过 `unlock` 机制锁定池状态，确保操作结束时净余额归零，防止资金风险。

### 4. 原生ETH支持
- **Gas优化**：直接支持ETH/WETH交易对，避免包装/解包ETH的额外Gas消耗（ETH转账成本降低约50%）。

### 5. 自定义记账（Custom Accounting）
- **绕过集中流动性**：钩子可通过返回Delta值实现完全自定义的流动性模型（如Uniswap v2的恒定乘积模型）。
- **低成本实验**：开发者无需重新实现AMM协议即可测试新策略，复用V4的安全性和基础设施。

---

## 1.2 代码架构

### 关键合约交互流程

`PoolManager.sol`管理所有的pool state,并允许以下操作：

1. **锁定与回调**：通过 call`PoolManager.unlock()` 来进行pool actions，具体而言，integrator call `PoolManager.unlock()`， `PoolMananger` call back `unlockCallback`，执行以下其中被允许的pool actions: 

   - swap
   - modifyLiquidity
   - donate
   - take
   - settle
   - mint
   - burn

   最后`unlock()`中的`NonzeroDeltaCount.read()`通过瞬态存储跟踪未结算的Delta数量，确保在`unlock()`结束时所有余额变动已通过`settle()`处理归0，防止资金不一致。

   

2. **钩子集成**：池初始化时可以绑定钩子合约，但是在初始化后钩子的的callback逻辑就不能变了。hook可以实现预定义的回调函数，包括：

   - {before,after}Initialize
   - {before,after}AddLiquidity
   - {before,after}RemoveLiquidity
   - {before,after}Swap
   - {before,after}Donate
```solidity
// 示例：集成钩子的合约
contract MyHook {
    function beforeSwap(address sender, bytes calldata data) external {
        // 自定义逻辑（如动态费用调整）
    }
}
```



# TOB 团队报告总结

[stream record](https://www.youtube.com/watch?v=CwvD8dmTsRc&t=4317s)

[Testing Code](https://github.com/trailofbits/v4-core/tree/add-stateful-properties/test/trailofbits)

[Reports](https://github.com/trailofbits/publications/blob/master/reviews/2024-07-uniswap-v4-core-securityreview.pdf)

## 1. 初始测试

他们团队首先是对E2E简单的测试(`test/trailofbits/end2end`)，基本上复用了uniswap团队的一些测试代码来熟悉框架。

非常有意思的是，他们将Actor动作结合到一个合约中去













- 模型-End2End -> Asset=>Liabilities

**Asset**: Tokens in the contract+ Flashloan Receivables

**Liabilities**: Liquidity owed to LPs+Fees Owed to LPers+Tokens owed to users



foundry.toml中via-ir选项关闭，能跳过中间编译选项，加快编译速度

甚至加入了测试ActionHarness的测试（A Test to test your test harness)- **how?**



`utils/Deployer.sol`:负责搭建完整的测试环境，涵盖合约部署、代币管理、池初始化和测试操作封装。

`V4StateMachine.sol`:继承`Deployer`，主要解决部分不可通过任何getter function获得的数据，比如LP获得的fees。其方法是通过重写uniswap v4中的池中交易，来获取数据流水。

`ShadowAccounting.sol`：继承`Deployer`，作为一个账本记录系统中的balances, deltas, and remittances(汇款)；合约由多个mapping和更新数据的helper function组成。

`ActionFuzzBase.sol`：继承`V4StateMachine`与`ShadowAccounting`





```markdown
# V4StateMachine 合约在测试框架中的核心作用解析

## 概述
`V4StateMachine` 是 Uniswap v4 测试框架中的**状态机模拟器**，通过重建协议内部状态并模拟核心操作（如流动性调整、交换计算），实现对复杂场景的精确验证。其核心目标是**绕过协议 Gas 优化导致的不可观测性**，直接验证链下不可见的核心逻辑（如费用累计、流动性变化）。具体功能如下：

---

## 核心功能与设计原理

### 1. **完整状态重建**
- **数据结构**：
  - **Tick 位图与信息**：通过 `TickBitmapProxy` 和 `TickInfos` 映射重建池的 tick 初始化状态（如流动性净值和总流动性）。
  - **流动性头寸跟踪**：`PoolPositions` 记录每个池的所有流动性头寸（包括已清空的头寸），用于还原真实流动性分布。
- **状态同步**：每次模拟前调用 `_updateTickBitmapProxy`，基于当前流动性头寸重新生成 tick 位图和 tick 信息，确保模拟环境与链上状态一致。

### 2. **费用计算模拟**
- **精确费用预测**：`_calculateExpectedLPAndProtocolFees` 函数模拟完整的交换流程，逐步计算：
  - **LP 费用**：基于流动性分布和费率动态计算。
  - **协议费用**：根据协议费率分拆部分费用。
- **溢出处理**：`_calculateExpectedFeeGrowth` 考虑费用累计溢出的极端场景，确保计算逻辑与协议实现完全一致。

### 3. **交换流程验证**
- **分步模拟**：`_calculateSteps` 函数复现 `Pool.swap()` 的核心逻辑：
  1. **tick 边界探测**：通过 `nextInitializedTickWithinOneWord` 查找下一个有效 tick。
  2. **价格步进计算**：使用 `SwapMath.computeSwapStep` 计算单步交换的输入/输出量和费用。
  3. **流动性调整**：根据跨越 tick 的流动性净值更新全局流动性。
- **状态迭代**：循环处理每个价格区间，直到达到指定价格限制或完成全部交换量。

### 4. **复杂场景覆盖**
- **边界条件**：验证价格限制（如 `MIN_SQRT_PRICE` 和 `MAX_SQRT_PRICE`）下的交换终止逻辑。
- **协议费动态分拆**：处理协议费与 LP 费的叠加计算，确保费用分配正确性。
- **流动性头寸影响**：通过重建的 tick 信息验证流动性增减对价格滑点的影响。

---

## 关键测试场景

### 场景 1：多 tick 区间交换验证
- **步骤**：
  1. 添加多个不同 tick 范围的流动性头寸。
  2. 执行大额交换，触发多个 tick 跨越。
  3. 对比模拟结果与链上实际费用和流动性变化。
- **检测点**：模拟计算的费用与协议实际收取费用一致。

### 场景 2：费用溢出测试
- **步骤**：
  1. 长期运行高频率交换，使费用累计接近 `uint256` 上限。
  2. 触发费用溢出，验证 `_calculateExpectedFeeGrowth` 的溢出处理逻辑。
- **检测点**：费用累计正确回绕，不影响后续计算。

### 场景 3：极端价格限制操作
- **步骤**：
  1. 设置交换价格限制为 `MIN_SQRT_PRICE` 或 `MAX_SQRT_PRICE`。
  2. 执行交换，验证模拟器是否提前终止并返回正确结果。
- **检测点**：交换在价格限制处终止，未发生越界操作。

---

## 与测试框架的协同流程

1. **环境准备**：测试用例通过 `Deployers` 部署池并添加流动性。
2. **状态快照**：`V4StateMachine` 重建当前池的 tick 和流动性状态。
3. **操作模拟**：调用 `_calculateExpectedLPAndProtocolFees` 预测费用结果。
4. **实际执行**：在链上执行相同交换操作。
5. **结果比对**：断言模拟结果与链上实际数据一致。

---

## 设计意义
- **透明化黑盒逻辑**：通过重建内部状态，使 Gas 优化导致的不可观测逻辑（如 tick 位图、费用累计）变得可验证。
- **精准回归测试**：捕捉协议升级或参数调整对核心逻辑的细微影响。
- **复杂策略验证**：支持流动性提供策略、高频交易算法等复杂场景的端到端测试。

`V4StateMachine` 是 Uniswap v4 测试框架的**核心验证引擎**，通过精确的状态模拟和费用计算，确保协议在极端条件和复杂操作下的行为符合预期，为协议安全性和可靠性提供深度保障。🔍⚙️
```

