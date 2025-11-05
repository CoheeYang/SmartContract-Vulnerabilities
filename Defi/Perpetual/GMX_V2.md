

[TOC]



## 1. 项目架构

![gmx_v2_contracts.png](https://github.com/Cyfrin/defi-gmx-v2/blob/main/notes/gmx_v2_contracts.png?raw=true)

[GMX_V2](https://github.com/gmx-io/gmx-synthetics/tree/main)中和uniswap用于使用Router作为用户层面的抽象，用来聚合一系列命令。

之后用handler来处理一个个板块的命令，是典型的Handler-Service-Storage模式。



```solidity
    function createDeposit(IDepositUtils.CreateDepositParams calldata params) external payable returns (bytes32);
    function cancelDeposit(bytes32 key) external payable;
    function createWithdrawal(IWithdrawalUtils.CreateWithdrawalParams calldata params) external payable returns (bytes32);
    function cancelWithdrawal(bytes32 key) external payable;
    function executeAtomicWithdrawal(
        IWithdrawalUtils.CreateWithdrawalParams calldata params,
        OracleUtils.SetPricesParams calldata oracleParams
    ) external payable;
    function createShift(IShiftUtils.CreateShiftParams calldata params) external payable returns (bytes32);
    function cancelShift(bytes32 key) external payable;
    function createOrder(IBaseOrderUtils.CreateOrderParams calldata params) external payable returns (bytes32);
    function updateOrder(
        bytes32 key,
        uint256 sizeDeltaUsd,
        uint256 acceptablePrice,
        uint256 triggerPrice,
        uint256 minOutputAmount,
        uint256 validFromTime,
        bool autoCancel
    ) external payable;
    function cancelOrder(bytes32 key) external payable;
}
```



## 2.角色设计与流程

在GMX的角色和动作

- Trader：create orders to
  - Swap
  - Long
  - Short
- LP
  - Deposit 
  - Withdraw 
- Keeper
  - Execute LP and traders orders
  - Set Oracle Prices

![gmx_v2.png](https://github.com/Cyfrin/defi-gmx-v2/blob/main/notes/gmx_v2.png?raw=true)

不同的角色对应了不同的流程和不同的机制，所以以下会按照角色顺序的逻辑来归类



### LP流程

LP会面临两种可提供流动性的池子：

1. **GMX Pool**：

   类似uniswap V2/V3 pool，但是可以提供<u>单边或者双边</u>任意流动性给指定的流动性池（如ETH/USD market），成功后获得对应的GM token。这种行为被称为`buy`

   当提供流动性时会被收取三种费用

   - **Price impact fee**：由于价格波动/单边流动性提供者不均匀，导致pair中的两类token的美元价值不均衡（比如现在100美元价值ETH和150美元价值USDC，缺ETH需要补ETH），如果LP继续恶化这种情况会被收取此费用，反之试图让二者平衡50：50则会受到奖励。
   - **Buy fee:** 买入时的手续费（此费用和PriceImpactFee都是按buy Amount的百分比计算）
   - **Network fee**：gas费用和keeper执行任务的gas费用，多的gas会退还

   当用户将GM token `sell`回池子时，会按原池子的现在代币对的比例收到代币对（不会收到单一的token）

   取出流动性时**只会被收取Sell fee 和 Network fee**，这是由于按比例返还代币对的算法会保证前后token的价值比例不变从而不收取Price Imapct Fee。

   

   除了`buy` & `sell`，**GMX还提供了`shift`，**即将一种GM token转为另一个池子的GM token，这种转移会被收取**Price impact fee**&**Network fee**



2. **GLV Vault**：

   这个vault是一个资管合约，本质上是在管理不同GM token以获得最大收益率，用户可以deposit GM token或者是backened Token Pair（如ETH/USDC），收到对应的shares，vault会对GM token进行资管。

   当用户burn shares时，会收到此GLV Vault的backened Token Pair，而非GM token。

   TODO具体怎么做得看代码。





### Trader流程



#### 1. Swap

**Swap类型：**

- Swap Market: Swap tokens at the current market price.
- Swap Limit: Swap tokens when the trigger price is reached.
- Swap TWAP: Swap tokens in evenly distributed parts over a specified time（拆单平分交易）

**费用：**PriceImpact + SwapFee +Network fee



#### 2. Short 

- Short Market: Increase a short position at the current price.
- Short Limit: Increase a short position when the price is above the trigger price.
- Short Stop Market: Increase a short position when the price is below the trigger price. 
- Short TP/SL: Decrease a short position when the trigger price is reached. **(Stop Loss，止损)**
- Short TWAP: Increase a short position in evenly distributed parts over a specified time.



### Keeper流程

1. **执行orders任务：**之所以需要keeper来执行任务而不是直接让用户执行任务是为了实现limit order，拆单分批执行等事件驱动型的特殊指令
2. **每次执行时标的价值的设定：**这个价格是爬的cex或者其他的外部较为新的价格，防止oracle产生stale price进而导致







## 3. 关键费率算法

费率有两种费率模型，

一种是one-time fee，一次性收取的无需多次计算。

一种是time-dependent fee，会随时间变化的费率

### 1. One-time fee

####  Network Fee&Protocal fee

这个没什么说的，在一个参数合约

中硬编码纯手算网络最拥堵的情况下每个调度任务需要执行的gas

再乘以一个1.2倍的系数。Admin可以设置这个参数，方便后续调整。



#### Price Impact Fee

此费用在swap和perpetual都有，其关键参数为Imbalance，此参数就是描述池子偏移了多少，作用是保持

Swap中：

- Imbalance = Long token(USD) -Short Token(USD)

Perpetual中

- Imbalance= OpenInterestLong -OpenInterestShort

但是一次提供流动性可能出现两种情况

1. SameSide：提供流动性后还是缺那个Token（补充的不够多）
2. CrossOver：提供的过多了，导致另外一个反而缺了



简要来说这个费用是通过两次Imbalance再乘以一个影响系数

- **始失衡值**：`d₀ = |初始长侧价值 - 初始短侧价值|`
- **最终失衡值**：`d₁ = |最终长侧价值 - 最终短侧价值|`
- **指数因子**：`e`（对失衡值进行指数调整）

最简化：(d0-d1)e 加上一堆判断条件来算。

### 2. Time-dependent fee

#### Funding fee

这个是只针对Trader的，为了保持多空平衡，且按时间计算。

其公式为：

FundingFeePerUSDSize = f*dt * largerSide(USD)/SmallerSide/Divisor

- `f`: The base funding fee factor per second (derived from the relevant utility function).
- `dt`: The time duration in seconds since the last funding fee calculation or payment.
- `divisor`: A value adjusted based on the relative open interest:
  - If Long OI > Short OI, `divisor = 2`.
  - If Long OI is not equal to Short OI (implying Short OI > Long OI or potentially if they are equal, though the exact logic can vary by protocol), `divisor = 1`.
- `size of larger side`: The total value (e.g., in USD) of open interest on the side (longs or shorts) with the greater amount.
- `size of smaller side`: The total value of open interest on the side with the lesser amount.



#### Borrowing fee

由于多空借走了LP的token一段时间而

类似Aave v3中的浮动利率计息方式，每秒都可能随市场变化，且有kink（拐点）。

Borrowing fee和funding fee都是每秒计息的模型。

每次市场有交易变化（有用户调用我们的函数）都会调用一个updateXXXstate的函数，计算一个accumlativeValue，然后是一个updateUserXXXstate的函数，二者一除就知道用户累计的费率。

