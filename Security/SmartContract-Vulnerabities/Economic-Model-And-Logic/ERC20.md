https://github.com/d-xo/weird-erc20?tab=readme-ov-file#weird-erc20-tokens

[TOC]

# WiredList

### 1. 重入调用（Reentrant Calls）

- 常见于ERC777，这个代币在transfer和transferFrom的方法中存在一个对外call的hook，导致潜在的重入攻击[ERC777](https://learnblockchain.cn/article/6087)
- **风险**：可在 Uniswap、借贷协议中重复抽取资金，如 imBTC Uniswap 池被清空、dForce 借贷池被掏空

> [!NOTE]
>
> 



### 2. Missing Return Values & Return false in failure

1. **Miss return value**

`USDT,BNG,OMG`这样的代币在transfer/transfeFrom不返回bool。BNB最特殊，它在`transfer()`不返回，但是`tranferFrom`返回。有一些sb的token，即使转账成功了也可能return false(如[Tether God](https://etherscan.io/address/0x4922a015c4407f87432b179bb209e125432e4a2a#code))



2. **Return false in failure**

正常OZ库的`transfer`会在余额不足时revert `ERC20InsufficientBalance`，`transferFrom()`额度不够时revert `ERC20InsufficientAllowance`。

但是一些token(e.g. [ZRX](https://etherscan.io/address/0xe41d2489571d322189246dafa5ebde1f4699f498#code), [EURS](https://etherscan.io/token/0xdb25f211ab05b1c97d595516f45794528a807ad8#code))会 return false，而不revert。

```solidity
   //return false when falure.
   function transfer(address dst, uint wad) external returns (bool) {
        return transferFrom(msg.sender, dst, wad);
    }
    function transferFrom(address src, address dst, uint wad) virtual public returns (bool) {
        if (balanceOf[src] < wad) return false;                        // insufficient src bal
        if (balanceOf[dst] >= (type(uint256).max - wad)) return false; // dst bal too high

        if (src != msg.sender && allowance[src][msg.sender] != type(uint).max) {
            if (allowance[src][msg.sender] < wad) return false;        // insufficient allowance
            allowance[src][msg.sender] = allowance[src][msg.sender] - wad;
        }

        balanceOf[src] = balanceOf[src] - wad;
        balanceOf[dst] = balanceOf[dst] + wad;

        emit Transfer(src, dst, wad);
        return true;
    }
```



### 3. 转账费（Fee on Transfer）

- **示例**：`TransferFee.sol` 对每次 `transfer` 扣除一定比例费用。
- **风险**：某些代币（如 SHIB、Safemoon 等）在 transfer 时会从 transferred amount 中抽取一部分到合约或某个地址。若合约内部只关注“transfer 事件发生且转移了 amount”而没有再查询 `balanceOf`，就可能错误认为“转账成功并且收到了 amount”，但实际上收到的只有 `amount – fee`。合约按指定 `amount` 更新内部余额，但实际到账少于期望，导致会计不平衡。历史上 STA 费用曾导致 Balancer 池损失 US$500k[GitHub](https://github.com/d-xo/weird-erc20?utm_source=chatgpt.com)[LinkedIn](https://www.linkedin.com/posts/dineshshetty1_erc20-smartcontract-tokendevelopment-activity-7041886918791323648-Np_g?utm_source=chatgpt.com)。



### 4. 非转账途径修改余额（Rebasing / Airdrops）

有一些代币会在链上自动或手动“改变某个地址的余额”而没有对应的 `transfer()` 调用，例如：

- **Rebasing（重基数）代币**：
   以 Ampleforth（AMPL）为代表，它会定期（比如每日）根据市场价格使所有持仓按一定比例“自动膨胀或收缩”。比如你原本有 10 AMPL，Rebase 之后变成 12 AMPL（或者 8 AMPL）。这并非发生在某一次 `transfer()`；而是合约在 `rebase()` 函数调用时遍历（或通过比例计算）直接批量修改每个持仓余额。
- **Governance Airdrop（治理代币空投）**：
   Compound 在每个区块/周期会把 COMP 代币“自动空投”到某些借贷用户钱包，这通常由合约自身在后台调用 `mint(用户, amt)`；并非用户主动发起 transfer。
- **可增发/可销毁代币**：
   有些项目合约随时可以对特定账户执行 `mint` 或 `burn`，改变持仓，而不会触发常规的 `transfer` 流程。

这些场景都属于“持仓在合约内部被修改，但并非通过常规 transfer”——也就是说，**余额隔壁“悄然”发生了变化**。




与此同时很多 DeFi 协议为了节约 Gas、提高效率，会在合约内部缓存某些地址或池子的代币余额。例如：

- **Uniswap V2/Liquidity Pool**
   每次有人在池子里做 swap、add/remove liquidity 时，合约会事先读取 tokenA.balanceOf(poolAddress)、tokenB.balanceOf(poolAddress)，并将这两个数值存到本地变量 `reserve0`、`reserve1`（池子内部状态）。后续的定价、滑点计算都基于这两个“缓存的”数值。如果你认为「除了 swap/add/remove 这几种操作，余额不会变化」，就根本不会在其它地方重新调用 `balanceOf()`。
- **Balancer 等多资产池**
   同理也往往会在每次交易或流动性调整时，把各资产的 `balanceOf(pool)` 读一次并写进合约状态，然后后续就用这个「上一次读取的值」做很多计算。
- **借贷协议中的抵押回报计算**
   有些协议会缓存用户抵押代币数量，或者借贷时某个存款代币在用户合约中的余额（代表它在合约里质押了多少），如果不重新对齐链上 `balanceOf()`，也会出错。

**风险：**

1. <u>定价计算、滑点预测完全失真</u>

以 重基数（rebasing）代币 为例：

1. 池子里最初有 1,000 个重基数代币（假设叫 R）。每个 R 价值 1 美元，所以池子估值 1,000 美元。
2. 某天触发 rebase，合约将所有 R 持有者的余额都按照 +10% 的比例膨胀，也就是说，池子里从 1,000 R 变成了 1,100 R。
3. 但是，Uniswap V2 池子内部的 `reserveR` 仍然记录的是“**1,000**”——它并不知道链上余额突然变成了 1,100。
4. 如果此时有人调用 `swapExactTokensForTokens(amountIn, ...)`，Uniswap 会基于“池子里还是 1,000 R”去计算价格，而实际上池子里有 1,100 R；买家极有可能按照错误的报价进行交易，造成套利者“把 1,100 R 捞走”或“出价过低买入池子里 R”，最终造成池子资金损失。

为了解决这个问题，Uniswap V2 团队特意为 Ampleforth (AMPL) 一类的重基数代币在其 Factory/Pair 里加入了“**Rebasing Listener**”机制：只要有 rebase 发生，就会在相同的事务里把 `reserveR` 更新为 `balanceOf(pool)`，保持“池内缓存”与“链上真实余额”一致，否则会出现上述定价失真。

2. <u>流动性池份额计算错乱</u>

在重基数代币加入池子时，LP 份额的计算常常需要依据「您投入了多少 token X、多少 token Y」来分配 LP Token。如果这个 token X 是重基数代币，且在您加入/退出期间发生了 rebase，而合约又没有及时刷新缓存值，那就会导致：

- 您实际投入时的“对等价值”被搞错，拿到的 LP 份额要么多拿，要么少拿；
- 退出时合约依旧基于错误的缓存（reserves）去熔断，可能拿回的资产与您预期大相径庭。

3. <u>合约内余额检查、贷款清算出错</u>

一些借贷协议会先检查 `balanceOf(user)` 来判断“用户是否真的有足够抵押”或“是否能借更多”。如果该 token 是可随时 mint/burn、或有空投逻辑的，那在合约做完抵押检查（缓存）之后，用户的余额可能瞬间被空投增加，再去转移就造成“多转”或“少转”问题；若空投期间余额减少，清算逻辑也可能不会及时察觉，从而引发潜在坏账。



### 5. 可升级代币（Upgradable Tokens）

- **示例**：`Upgradable.sol` 允许管理员随时替换逻辑。
- **风险**：代币语义可在任意时刻变更，可能破坏依赖其旧行为的合约。推荐检测并在升级时冻结交互[GitHub](https://github.com/d-xo/weird-erc20?utm_source=chatgpt.com)。

### 6. 闪电铸造（Flash Mintable）

- **示例**：未提供示例，但 DAI 闪电铸造模块可在单交易内 mint 巨量 DAI，须在交易末尾归还。
- **风险**：总供应可瞬时膨胀至 `2^256-1`，若合约忽视借还流程，可能发生控制权或余额操纵[arXiv](https://arxiv.org/abs/2309.04700?utm_source=chatgpt.com)。

### 7. 黑名单（Blocklists） & 可暂停（Pausable）

- **示例**：
  - `BlockList.sol`：基于 USDT/USDC 黑名单，不允许被列入黑名单的地址转账。
  - `Pausable.sol`：可由管理员暂停所有或部分操作。
- **风险**：恶意或受控管理员可冻结用户资产或锁死合约资金，引发服务中断或敲诈[GitHub](https://github.com/d-xo/weird-erc20?utm_source=chatgpt.com)[OpenZeppelin 博客](https://blog.openzeppelin.com/workshop-recap-secure-development-workshop-1?utm_source=chatgpt.com)。

### 8. Approval Race

Approval Race是部分如`USDT`, `KNC`不允许token在已有allowance `M > 0`的情况下进行二次approve `N>0`。代码如下

```solidity
// Copyright (C) 2020 d-xo
// SPDX-License-Identifier: AGPL-3.0-only

pragma solidity >=0.6.12;

import {ERC20} from "./ERC20.sol";

contract ApprovalRaceToken is ERC20 {
    // --- Init ---
    constructor(uint _totalSupply) ERC20(_totalSupply) public {}

    // --- Token ---
    function approve(address usr, uint wad) override public returns (bool) {
        require(allowance[msg.sender][usr] == 0, "unsafe-approve");
        return super.approve(usr, wad);
    }
}
```

这是源于一个[ERC20攻击场景](https://docs.google.com/document/d/1YLPtQxZu1UAvO9cZ1O2RPXBbT0mooh4DYKjA_jp-RLM/edit?tab=t.0)，即当A有了B的allowance `M`时，当B想将A的allowance 从`M`变成`N,N>0`时，A可以在其approve执行前，抢跑先用掉`M`，再等allowance为`N`时再用掉`N`，这样A就用掉了`M+N`的token额度。



### 9. 零地址或零值操作异常

1. 零地址转账/approve都会revert （OZ标准）
2. `BNB` approve()为0值时会revert。
3. `Lend` 转账为0值时会revert。





### 10. 多地址代理（Proxied Tokens）

- **示例**：`Proxied.sol` 假设单一地址，无法遮蔽同一逻辑代理的多个实现。
- **风险**：所有代理地址都需纳入白名单，否则 `rescueFunds` 可能被 owner 利用窃取资金[GitHub](https://github.com/d-xo/weird-erc20?utm_source=chatgpt.com)[bkrem.github.io](https://bkrem.github.io/awesome-solidity/?utm_source=chatgpt.com)。



### 11. 非字符串元数据（Bytes32 Metadata）

有些token（MKR）的name和symbol是byte32而非string。当你扒取这些信息时可能存在口径不对



### 12. 高/低小数位（Decimals）

- **示例**：`LowDecimals.sol` (`USDC` 6 位)、`HighDecimals.sol` (`YAM-V2` 24 位)。
- **风险**：计算精度或乘除溢出导致意外 revert 或金额误差[GitHub](https://github.com/d-xo/weird-erc20?utm_source=chatgpt.com)。

### 13. `transferFrom` 特殊语义

有一些token（DSToken）在使用`transferFrom`时规定 当`from == msg.sender`时不降低allowance，使得它和`transfer`一样。而OZ标准，Uniswap-v2是没有这个的。



### 14. Revert on Large Approvals & Transfers

Some tokens (e.g. `UNI`, `COMP`) revert if the value passed to `approve` or `transfer` is larger than `uint96`.

Both of the above tokens have special case logic in `approve` that sets `allowance` to `type(uint96).max` if the approval amount is `uint256(-1)`, which may cause issues with systems that expect the value passed to `approve` to be reflected in the `allowances` mapping.



### 15. Code injection

某些token，利用交易所前端需要读取链上token name的漏洞，往token名中加入js代码，浏览器如果“直接把合约里返回的名字渲染到页面上”，就可能遇到“跨站脚本（XSS）攻击”：如果某个合约的 `name()` 返回值里包含了恶意的 JavaScript 片段，那么前端就会把这段脚本插入页面，导致脚本在用户浏览器里执行、读取用户数据。

https://hackernoon.com/how-one-hacker-stole-thousands-of-dollars-worth-of-cryptocurrency-with-a-classic-code-injection-a3aba5d2bff0





### 16. 非标准 `permit`

([DAI, RAI, GLM, STAKE, CHAI, HAKKA, USDFL, HNY](https://github.com/yashnaman/tokensWithPermitFunctionList/blob/master/hasDAILikePermitFunctionTokenList.json))  等部分代币的 `permit()` 与 EIP‑2612 不兼容，甚至不 revert，可能令下游调用流程逻辑出错，

[Uniswap permit2](https://github.com/Uniswap/permit2)提供了一个兼容的wrapper。



### 17. Transfer `type(Uint256).max`

某些代币在 `transfer(address to, uint256 amount)` 中做了一个特殊判断：如果传入的 `amount` 等于 `type(uint256).max`（也就是最大无符号 256 位整数），就不按常规“扣除 `amount` 再转给 `to`”的逻辑去做，而是直接把持有者**所有**余额一次性转给接受方。这样一来，传 `amount = 2**256–1`（也就是 `uint256.max`）能够让持有者“把自己账户里目前实际拥有的全部代币”一次性转出，而不需要先查余额再传一个具体数字。

以 **cUSDCv3**（Compound v3 上的 USDC 借贷代币）为例，其 `transfer` 实现里就包含了这种“传 `uint256.max` → 实际只转出持有人当前余额”的特殊分支。伪代码大致像这样：

```solidity
function transfer(address recipient, uint256 amount) external returns (bool) {
    if (amount == type(uint256).max) {
        // “一键取出所有余额”的快捷分支
        uint256 balance = balances[msg.sender];
        balances[msg.sender] = 0;
        balances[recipient] += balance;
        return true;
    }
    // 否则按常规操作：检查余额是否 ≥ amount，再扣除 amount 转给 recipient
    require(balances[msg.sender] >= amount, "INSUFFICIENT_BALANCE");
    balances[msg.sender] -= amount;
    balances[recipient] += amount;
    return true;
}
```

这样看似只是“给用户一个方便的快速取光余额功能”，但如果调用方的逻辑直接信任传入的 `amount`，就会出现“**转入并没有真实达到调用者期望的数量**”的差额。具体风险场景常见于“Vault 型”或“Pool 型”合约，比如：

1. **用户先调用 `token.transfer(vaultAddress, amount)`**

   - 用户原本打算把自己账户里的 `amount=uint256.max`（陷阱值）转给 Vault，以便“赎回所有代币”或“退出仓位”。
   - Vault 合约看到 `transfer(...)` 返回成功，就认为“用户确实向我转来了 `amount` 数量的代币”。

2. **Vault 把“传入数量”直接记入自己的内部账本**

   - 典型 Vault 会在 `deposit()` 或 `redeem()` 里写出类似下面逻辑：

   ```
   solidity复制编辑function redeem(uint256 amount) external {
     // ① 用户调用 cUSDCv3.transfer(vault, amount)
     // ② Vault 认为到账量 = amount，于是内部记录 users[msg.sender].balance -= amount;
     // ③ Vault 根据 amount 给用户记账或给用户发某些“份额”或分配回报
   }
   ```

   但事实上，当 `amount == uint256.max` 时，cUSDCv3 并不会转出一个等于 `uint256.max` 的数值（它根本没那么多）；它只会把用户**目前真实拥有**的余额（假设是 1,000 cUSDCv3）转给 Vault。
    结果就是：

   - **Vault 以为它收到了 `amount`（也就是 2^256–1）单位代币，实际上只拿到 1,000。**
   - Vault 随后可能直接把 “2^256–1” 写进它自己的内部会计账本里（比如给用户贷记了这么多份额），导致 Vault 账户系统和链上实际余额巨幅不一致。

3. **最终出现的恶果**

   1. **Vault 账本里显示它收了天文数字的代币** （`uint256.max`），而链上只实际得到持有人真实余额（比如 1,000）。
   2. **Vault 随后可能给用户等值算出的份额或可提资产** 按照“用户已存入的数额”来计算。
   3. 当用户或其他人尝试提现时，Vault 发现自己链上根本没这么多代币可供支付，就会造成**卡死（out‐of‐funds）\**或\**不可预知的清算错误**。
   4. 进一步说，如果 Vault 管理的资产池还承担流动性对赌（如发行 LP 份额、配对做市、借贷凭证等），这“账面多入、实物少入”的漏洞甚至能让攻击者从中**骗走大量剩余代币**，或者让 Vault 彻底陷入资不抵债。





### 18. ERC-20 Representation of Native Currency

有一些链，它有一个原生的合约来记录原生代币的账户情况，当使用正常的原生代币转账时，对应的原生合约也会同步增减余额。

比如：

- Celo with [CELO](https://celoscan.io/address/0x471ece3750da237f93b8e339c536989b8978a438) (address `0x471EcE3750Da237f93B8E339c536989b8978a438`).
- Polygon with [POL](https://polygonscan.com/token/0x0000000000000000000000000000000000001010) (address `0x0000000000000000000000000000000000001010`).
- zkSync Era with [ETH](https://era.zksync.network/token/0x000000000000000000000000000000000000800a) (address `0x000000000000000000000000000000000000800A`).

这就可能造成double spending，比如下面的[uniswap v4](https://blog.openzeppelin.com/uniswap-v4-core-audit#erc-20-representation-of-native-currency-can-be-used-to-drain-native-currency-pools)的案例：



假定攻击者钱包里有 **1,000 CELO**，准备在一次交易里把它“算成”2,000 CELO。操作顺序如下（均在一个 transaction 里）：

1. **调用 `manager.sync(ERC20_CELO)`**

   ```solidity
   manager.sync(0x471EcE3750Da237f93B8E339c536989b8978a438);
   ```

   - 这一步把合约里目前“ERC-20_CELO” 余额读到暂存：假设先前合约里就有 0 CELO，所以暂存的 “lastRecordedBalance” 也就是 0。
   - `sync` 并不改变任何 delta，只是为下一步 `settle` 奠基，让合约记下“上次 ERC-20_CELO 余额 = 0”。

2. **调用 `manager.settle{value:1000}(NativeMarker)`**

   ```solidity
   manager.settle{value: 1000}(address(0x0));
   ```

   - 由于传入的是 native 标记（通常是 `address(0)` 或合约里定义的 NATIVE 常量），`settle` 函数就会把 `msg.value=1000` 直接累加到“Native delta”上。
   - 同时，这 1,000 CELO 完全通过普通的 `msg.value` 进账到 `PoolManager`，合约里真实持有的 native CELO 增加了 1,000。
   - 到目前为止，合约内部状态：
     - **Native delta** = +1000
     - **ERC-20_CELO delta** = +0（还没操作）
     - 合约余额 = 1000 CELO（存放在合约的原生 CELO 里）

3. **调用 `manager.settle(ERC20_CELO)`**

   ```
   manager.settle(0x471EcE3750Da237f93B8E339c536989b8978a438);
   ```

   - 这次 `settle` 检测到 “ERC-20_CELO” 不是 native，而是一个真正的 ERC-20 合约地址，所以它会按以下逻辑：
     1. 读取当前 `PoolManager` 对 `ERC-20_CELO` 的 `balanceOf`，发现刚才 native 入口 `msg.value=1000` 也同时增加了合约的 ERC-20_CELO （因为在 Celo 上，native CELO 会自动使 ERC-20_CELO 合约里调用 `receive()`）。换句话说，当你把原生 CELO 放进合约，合约的 ERC-20_CELO 余额也同步加上了这 1,000。
     2. “余额差值” = 当前 ERC-20_CELO 余额（1,000） −“上次 sync 时记录的余额”（0） = 1,000。
     3. 于是 `settle` 就把这 1,000 计入 “ERC-20_CELO delta”。
   - 到此为止，合约内部状态：
     - **Native delta** = +1000
     - **ERC-20_CELO delta** = +1000
     - 合约里真金白银（合约持有的 CELO，记在同一个地址上）总共是“1,000 CELO”，但账户 delta 总共可提是 2,000。



# Best Practice

**链上白名单（Allowlist）**
 仅信任已验证过的 ERC20 合约地址，拒绝所有其它代币操作请求，以避免未知行为的风险[GitHub](https://github.com/d-xo/weird-erc20?utm_source=chatgpt.com)。

**边界封装（Wrapper Contracts）**
 在系统边缘使用专门的封装层与外部代币交互，将各种非标准情况（重入、挂起、失败返回等）集中处理，如OZ的`safeERC20`确保核心逻辑中代币语义一致性[GitHub](https://github.com/d-xo/weird-erc20?utm_source=chatgpt.com)[bkrem.github.io](https://bkrem.github.io/awesome-solidity/?utm_source=chatgpt.com)。

**离线与前端提示**
 对于无法链上管理的场景（如完全去中心化的 AMM），可在官方 UI 中维护一份“离线白名单”，提示普通用户避免与风险代币交互[加密开发中心](https://cryptodevhub.io/weird-erc-20-tokens?utm_source=chatgpt.com)。