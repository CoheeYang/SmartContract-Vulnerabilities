# Yieldoor项目回顾

项目地址：https://github.com/CoheeYang/2025-2-yieldDoor?tab=readme-ov-file （PrivateForNow）

这是个Sherlock项目，记得复制并保存原仓库



项目特点：这是个关联uniswap v3的strategy协议，重度依赖外部dex的合约会面临以下问题；

- 选择fork还是本地mock整个dex？
  - 如果选择fork，使用echidna进行测试会非常不方便，因为echidna并不支持如`deal()`这样的cheatcode，无法给角色的token账户充值。echidna唯一的补救措施是在dex中通过WETH换钱,可能会破坏dex状态。
  - 如果选择mock，费时费力
  
- 编译问题
  - Uniswap v3与其依赖多数是compiler verison 低于0.8.0的，echidna并不能像foundry一样做到动态识别编译，你会需要将代码中的编译pragma修改至一个适合的版本（如>=0.7.6 <0.9.0），这种编译器的改动是否引入潜在问题尚不明确。
  - 选择foundry意味着失去corpus等功能
  
- 依赖问题

  - uniswap中会用oz的依赖，由于是hardhat项目，其依赖基本在node_module中，通过`yarn install`下载，而yield也会使用其在lib/. 使用oz。这会产生remapping冲突，foundry给出了[解决方法](https://book.getfoundry.sh/projects/dependencies#remapping-conflicts)

    > 具体而言添加remapping.txt文件，并输入以下内容：
    >
    > lib/v3-core/:@openzeppelin/=lib/v3-core//node_modules/@openzeppelin/
    >
    > lib/v3-periphery/:@openzeppelin/=lib/v3-periphery/node_modules/@openzeppelin/



## Contest Report

[Contest Report](https://audits.sherlock.xyz/contests/791/report)

Report总结：

| Index | RootCause                                                    | Classification                                   |
| ----- | ------------------------------------------------------------ | ------------------------------------------------ |
| H1    | Leverager.sol中的liquidation逻辑中<br/>`uint256 borrowedValue = owedAmount * bPrice / ERC20(up.denomination).decimals();`<br/>除法问题，它的被除数是decimals位数(8/18这样的)<br/>**正确的操作**是使用`10^decimals()` | Decimal issue                                    |
| H2    | `Strategy:checkPoolActivity()`中<br/>`uint256 index = (observationCardinality + currentIndex - i) % observationCardinality;`会出现Overflow<br/>这是因为`observationCardinality `和`currentIndex`来自uniswap v3 slot，而他们两个都是uint16的参数，相加会导致超出2^16。<br/>而`observationCardinality `可以通过`UniswapV3Pool#increaseObservationCardinalityNext`来手动增加Observation的容量，防止oracle manipulation。这样就导致两个Unit16数据相加会产生overflow<br/>**解决方法**是用`uint32(observationCardinality)`去和`currentIndex`相加。此时相加计算的ceiling是uin32，不会产生overflow | DOS (Overflow)                                   |
| H3    | 没看懂                                                       |                                                  |
| H4    | `Leverager:openLeveragedPosition()`中调用了Uniswap Router中的`exactOutput(ExactOutputParams *calldata* params)`，虽然对相关的输入参数`params`的`tokenIn`swap中存入的token进行了检查，**但是没有对出来的token进行检查**<br/>这意味着用户可以先转WETH，但是对params中写入WBTC，设置recipient为自己，来伪装一笔WETH的交易，但是却可以成功盗取WBTC。 | AccessControl (insufficient checks)              |
| H5    | `Leverager:openLeveragedPosition()`中无法实现2x杠杆，虽然系统期望可以实现。<br/>这因为任何两倍杠杆的头寸都符合其内部函数`isLiquidateable()`中`owedAmount > totalDenom`的check条件，并被视为可以清算的头寸<br/>**Tips:以后多看看那些允许两倍杠杆的协议会不会有相同的问题** | AccessControl (Contradicted Checks--overchecked) |
| H6    | `CollectFee()`中的逻辑写错<br/>if (ongoingVestingPosition) { collectPositionFees(vestPosition.tickLower, <u>**mainPosition**.tickUpper</u>);   } | LogicError（写错）                               |
| H7    | `Leverager:withdraw()`中欠款逻辑如下<br/>` uint256 totalOwedAmount = up.borrowedAmount * bIndex / up.borrowedIndex;`<br/>其内部借款额是按`bIndex`除以仓位记录的`borrowedIndex`来算的<br/>其中`bIndex`是其他的关于时间的逻辑一直增长的，而`borrowedIndex`是按借款时间来产生的。<br/>但是如果只取一部分钱会触发一个更新`borrowedIndex=bIndex`的逻辑，这意味着<br/>取1%的钱，这部分钱算利息。而之后取99%钱，这部分钱没有利息<br/>**解决方法：**去掉这段更新逻辑 | LogicError (ReduntantLogic)                      |
| H8    | Leverager中未设置FeeRecipient且没有设置的相关函数            | LogicError（写错）                               |
| M1    | `Vault::_calcDeposit()`中出现了**过多的连续乘法运算**；在计算shares时，它间接使用了`depositAmount * bal * totalSupply`，导致如果三个数的量级为1e18，就总共占用1e54个的量级，还剩下1e23。如果totalSupply和bal的量级分别为1e9，这就导致只剩下1e5的量级。<br/>也就是说大于10_000元的存款会失败。 | Overflow                                         |
| M2    | `strategy:_setMainTicks(int)`中，求余逻辑`int module= tick% tickSpacing`中如果tickSpacing为1，且tick为负数的情况，会导致`isLowersided=module<(tickSpacing/2)`在此情况下为false(0<0)，使得后续对tickBorder增加tickSpacing，导致与当前tick不对称的价格上下限。最大化手续费收入失败造成损失。（**Calculation Edge cases**） | LogicError（calculation）                        |
| M3    | ``Strategy::checkPoolActivity` `                             |                                                  |
| M4    | `_setSecondaryPositionsTicks`没有对module为负数时的情况进行判断和矫正，导致可能secondaryPosition在错误情况下开仓 | LogicError（calculation）                        |
| M5    | `isLiquidateable()`中对于清算的MaxLeverager**口径没对齐**，函数中有两个maxLeverage，一个是`vp.maxTimesLeverage`作为清算的基准，另外一个是来自lendingPool的`maxLevTimes`。而合理的杠杆率根据`_checkWithinlimits()`是二者中最小的那个，但是真实到了清算时它武断地使用了`vp.maxTimesLeverage`作为基准。<br/>也就是说如果实际上`maxLevTimes`更小，则会导致不符合`checkWithinlimits()`中杠杆率要求的头寸存在。 | LogicError(口径问题)                             |
| M6    | `Strategy::checkPoolActivity`                                |                                                  |
| M7    | copy-paste error，本应该用amountOut1的地方复制成了amountOut0 | LogicError（写错）                               |
| M8    | `openLeveragedPosition()`中调用` _getTokenIn(swapParams.path)`来检查传入数据中的tokenIn是否是denomination代币。<br/>但是`_getTokenIn`中<br/>`while (path.hasMultiplePools()) {path.skipToken();}`  这种调用方法用于无法更新path的值(合约中对bytes使用了path lib)，对于有multi-hop的swap将产生无限循环DoS<br/>**正确做法**是使用`{path = path.skipToken()}`更新path的值 | DoS (infinite-loop)                              |

H1是绝对能通过fuzzing来测到的，H2也可以通过fuzzing测到，但是需要Mock uniswap v3合约，并且加入uniswap v3的相关动作



我觉得还是得拿MD文档来记录，当recon对每个可测试函数列举后，我把它们都抄到这里，并把所有对于外部函数的思考写在这里






我遇到的第一个问题就是太多的可修改的参数情况了 

- 另外一个问题是我该把角色限制死吗？什么角色做什么事情，万一这个人刚deposit再open leverage会不会导致bug

- 我们能直接放个有struct函数的函数给echidna使用吗？答案是可以

- 我在这里遇见了钻石继承的问题，具体而言，所有的pro合约都继承了setfunction。然后这些合约将会被另外一个合约继承
- 我应该把require的数字限制都改成between吗？



我有一个系统性的invariant A=>L, in terms of token amount

在这个合约中，Asset有什么组成？

用户对`vault.deposit()`交互时，`strategy`增加assetToken，一部分作为`idleBalance`，另外一部分存到了`uniswap`作为流动性

LendingPool中也有一部分的来自Lender的资产

另外一部分在borrower借出的资产（借出的资产也被vault转到strategy了，不能算）

+其他的protocal盈利部分



负债是什么组成：

lender+depositors上述存的钱

还有欠他们的手续费



- 寻找Invariant
- 利用那个表试一下





迭代流程：

我先提出一个system-level的invariant (A>=L)，再通过把所有可能的动作写上，并在动作中加入一些简单的black-box test。

之后测完这波，再加入white-box tests，以及是否有可能有更多的system-level的invariant？





Recon脚手架不好用

before&after基本没啥屌用

property也是基本空着不太可能会用

IHEVM居然没有startPrank

为数不多有用的就是assert，ihevm的写法和兼容foundry的办法

 



编译成为一个非常傻逼的问题！
