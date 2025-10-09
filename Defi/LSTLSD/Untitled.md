# Intro 

**LST（Liquid Staking Token）**：把你“委托/质押”的原生代币变成可流通、可用的代币化凭证（receipt token），同时原始资产继续在验证者上产生质押收益。典型例子：Lido 的 **stETH**、Rocket Pool 的 **rETH**、Ankr 的 **ankrETH/c** 等。

**LSD / LSDfi（Liquid Staking Derivative / LSD finance）**：通常指围绕这些衍生品构建的生态（包括借贷、做市、杠杆、收益聚合等），或者更广义地把 LST 称为 LSD（术语使用上常有交叉）。当中还衍生出“**liquid restaking / LRT**”等新类别（例如 EigenLayer 提供的 restaking 服务），把已质押或 LST 再次用于为其他协议担保以赚取额外收益

**两种常见会计/收益体现方式**：

1. **Rebasing（余额自动增加）**：持有地址里的 LST 数量会周期性增加来体现收益（Lido 的 stETH 就是采用 rebasing 的一种实现，Lido 会定期对持仓做 rebase）。stETH和ETH会尽量保持1：1，这样余额的增加就是奖励。但是rebase token引起许多安全隐患，wstETH被更多的采用

   [Lido Finance](https://blog.lido.fi/steth-the-mechanics-of-steth/)

2. **非 rebasing（固定余额，汇率上涨）**：钱包里 LST 数量不变，但每个 LST 能兑换回的底层资产份额随时间增长（Rocket Pool 的 rETH、Ankr 的 ankrETH/ aETHc 等多数用这种方式）。但是一般这种增长需要oracle支持（因为你不知道自己在beacon chain上赚了多少钱）以rETH为例，其`rETH`的价格由oracle反映ETH/rETH。

**头部项目方：**

1. **Lido (stETH / wstETH)** — 去中心化 liquid staking，stETH（rebasing）/ wstETH（wrapped non-rebase 版本）是其代表。广泛整合在 DeFi。[Lido Finance+1](https://blog.lido.fi/steth-the-mechanics-of-steth/?utm_source=chatgpt.com)

2. **Rocket Pool (rETH)** — 去中心化、低门槛（0.01 ETH）质押，rETH 是非-rebasing、其兑换率随时间增长。[docs.rocketpool.net](https://docs.rocketpool.net/guides/staking/overview.html?utm_source=chatgpt.com)

3. **Ankr (ankrETH / aETHc / aETH variants)** — CeFi/infra 风格的 liquid staking，发行 reward-bearing 代币（汇率上涨型）。[Ankr+1](https://www.ankr.com/docs/liquid-staking/eth/overview/?utm_source=chatgpt.com)

4. **Coinbase (cbETH)** — 交易所包装的 staked-ETH（中心化提供者发行的 LST）。[Coinbase](https://www.coinbase.com/cbeth?utm_source=chatgpt.com)

5. **StakeWise、Kiln、Frax、etc.** — 各有不同的设计（两代币模型、保险/补偿机制、集成 restaking 等）



项目特定风险：

- **Oracle Riks:** 由于通过oracle来报告资产底层ETH数量而产生的风险

- **LST De-Peg Risk:** ETH/LST 低于1的情况导致脱锚，其可能是由于

  - Low liquidity on DEXs.

  - Market panic or negative news.

  - Problems with the LST protocol or underlying staking risks（slashing）.


If an LST de-pegs significantly while you're using it as collateral, you could get liquidated!

- **Slashing Risk:** Pos中validator被罚没的情况，虽然多数协议由抵押物机制（node operator bonds, insurance funds like Lido's, or Rocket Pool's RPL collateral）。但不排除由于抵押物被消耗完导致的slash risk从而使得LST脱锚。



