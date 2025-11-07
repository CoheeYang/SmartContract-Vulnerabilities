[TOC]



# 1.Do you really know Uniswap V2

UniswapV2可以说是AMM中最基础的项目，但是你真的熟悉uniswapV2吗？

如果你真的觉得自己是Defi大师，请回答我以下的问题：

1. ETH/USDC池中，如果ETH价格上涨一倍，请问IL（无常损失）是多少？
2. 告诉我Uniswap中的最优价格路径算法是怎么样的
3. 解释 Pair 合约中的 `sync()` 与 `skim()` 的区别与使用场景
4. UniswapV2中如何分配项目方和众多LP的手续费？



# 2.UniswapV2详解

## 2.1架构分析

UniswapV2的架构相对简单直白：

- peripheral+core的模式分离核心逻辑与用户交互方面的优化逻辑

- core中factory+pair是经典的工厂模式以用一个factory合约控制多个pair合约。





## 2.2 代码解析

### 2.2.3 Core

#### 2.2.3.1 简单地完成一个pair

> 注意！下面的代码都经过一系列简化方便阅读，其安全性和gas优化都会打折扣，生产级别代码参考uniswap源码

假设你是uniswap的开发者，你要开发一个流动性池，你需要开发的功能有哪些？想必会有

1. **提供流动性和取出流动性的函数（Mint/Burn）**
2. **面对trader进行代币互换的函数（Swap）**

现在我们试着解决最简单的`Mint`函数。这个函数应该是LP们提供流动性后，会Mint一些ERC20 token给它作为凭证，以获得手续费。

那么首先我们得思考`Mint`函数该如何分配给LP多少 token，还有edge case的第一次Mint时，我们需要给第一个LP多少 token？

我们先从简单的开始，假设LP已经发钱来了，`Mint`首先得看看用户到底转了多少钱，这个好办，我们只需要算一下与上次记录时，`reserve`变量的差值。

```solidity
function Mint() external {
 (uint256 reserve0, uint256 reserve1) = getReserve();
 uint256 diff0 = balanceOf(token0) - reserve0;
 uint256 diff1 = balanceOf(token1) - reserve1;
 ...
}

```

之后我们得想想该mint给用户多少token？既然我们知道`xy=k`的`k`是两个代币数量的积，那不如用`sqrt(k)`作为这个LP token所mint的数量吧。自然地我们有

```solidity
function Mint(address to) external {
 (uint256 reserve0, uint256 reserve1) = getReserve();
 uint256 diff0 = token0.balanceOf(address(this)) - reserve0;
 uint256 diff1 = token1.balanceOf(address(this)) - reserve1;
 ---- new part----
 uint256 liquidity = sqrt(diff0*diff1);
 require(liquidity>0,"zero liquidity");
 _mint(to,liquidity);//继承ERC20
 
 _update(token0.balanceOf(address(this)),token1.balanceOf(address(this)));
}
function _update(uint256 amount0,uint256 amount1) internal {
	reserve0 = amount0; //reserve0,1 as state variable
    reserve1 = amount1;
}

```

那`burn`函数不是更好写？我们只需要和一般的ERC20一样burn掉它就可以了，之后按比例地返回用户所获得的token就好了：

```solidity
function burn(uint256 amount) external {
	uint256 Userbalance = balanceOf(msg.sender);
	require(Userbalance>=amount,"not enough LP token");
	//evenly give back the tokens
	uint256 shareOfUser = Userbalance*10**decimal / totalSupply();
	
	uint256 token0_balance = token0.balanceOf(address(this));
	uint256 token1_balance = token1.balanceOf(address(this));
	
	uint256 amount0 = token0_balance*shareOfUser/(10*token0.decimal()*10**decimal);
	uint256 amount1 = token1_balance*shareOfUser/(10*token1.decimal()*10**decimal);
	
	token0.safeTransfer(msg.sender,amount0);
	token1.safeTransfer(msg.sender,amount1);
	
}
```

但是你按上面写你会发现如下问题：

- 使用`sqrt(diff0*diff1);`作为mint的流动性代币会导致token decimal的不一致，比如USDC*WETH，最终decimal位数是（6+18）/2 = 12。这种不一致导致`burn`的时候，使用`10**decimal`（比如erc20合约设decimal为18）时多除了6位。

- `sqrt(diff0*diff1);`的记录LP的方式会让不同价值的资产组合获得相同数量的LP。假设某coin的decimal为18位，资产价格为0.1美元。

  1USDC+10coin 和 0.01USDC + 1000coin 两个资产组合获得相同的LP。

  但是前者资产价格为2美元，后者却价值100美元。同时用户还花了非常多的钱扭曲了池子中的流动性，即超额提供了coin。



对于第一个问题，只需要简单修改一下代码顺序即可：

```solidity
function burn(uint256 amount) external {
	uint256 Userbalance = balanceOf(msg.sender);
	require(Userbalance>=amount,"not enough LP token");
	//evenly give back the tokens
	
	uint256 token0_balance = token0.balanceOf(address(this));
	uint256 token1_balance = token1.balanceOf(address(this));
	
	uint256 amount0 = token0_balance*amount/totalSupply;
	uint256 amount1 = token1_balance*amount/totalSupply;
	require(amount0 > 0 && amount1 > 0, 'UniswapV2: INSUFFICIENT_LIQUIDITY_BURNED');
	
	token0.safeTransfer(msg.sender,amount0);
	token1.safeTransfer(msg.sender,amount1);
	
}
```

而对于第二个问题，我们需要在第一次`mint`后取最小值

```solidity
function Mint(address to) external {
 (uint256 reserve0, uint256 reserve1) = getReserve();
 uint256 diff0 = balanceOf(token0) - reserve0;
 uint256 diff1 = balanceOf(token1) - reserve1;
 if (totalSupply == 0) {
   //第一次mint，L=sqrt(xy) - MINIMUM_LIQUIDITY
   liquidity = Math.sqrt(diff0*diff1) - MINIMUM_LIQUIDITY;
   _mint(address(0), MINIMUM_LIQUIDITY); // permanently lock the first MINIMUM_LIQUIDITY tokens
   } else {
   //后续的mint会按最小比例来计算
   //L = min(amount0*totalSupply/x, amount1*totalSupply/y)
   liquidity = Math.min(diff0*totalSupply/reserve0, diff1*totalSupply/reserve1);
    }
    
 require(liquidity>0,"zero liquidity");
 _mint(to,liquidity);//继承ERC20
 
 _update(diff0,diff1);
}
```

其中使用$ΔS =S⋅min(Δx/x, Δy/y)$ 来计算新增流动性token。

虽然这个方法保留了让不同价值的组合可能获得相同流动性的问题。但是却让LP们自己寻找一个平衡位置来提供流动性；**即最优的token提供方案应该是：$Δx/x = Δy/y$ 的情况，否则多余的任何x/y 代币都会是无效的。**



现在做完了`burn/mint`函数，接下来就是面对LP们的swap方法了。swap方法首先得明确我们是swap是输入outAmount还是inAmount，在uniswapV2中，采用的是outAmount

```solidity
function swap(uint256 amount0Out,uint256 amount1Out,address to) external {
        require(amount0Out > 0 || amount1Out > 0, 'UniswapV2: INSUFFICIENT_OUTPUT_AMOUNT');
        (uint256 _reserve0, uint256 _reserve1,) = getReserves(); // gas savings
        require(amount0Out < _reserve0 && amount1Out < _reserve1, 'UniswapV2: INSUFFICIENT_LIQUIDITY');
        //转账
        token0.safeTrasnfer(to,amount0Out);
        token1.safeTransfer(to,amount1Out);
       	//对比前后转账的变化
       	uint256 balance0 = IERC20(_token0).balanceOf(address(this));
        uint256 balance1 = IERC20(_token1).balanceOf(address(this));
        
        uint amount0In = balance0 > _reserve0 - amount0Out ? balance0 - (_reserve0 - amount0Out) : 0;
        uint amount1In = balance1 > _reserve1 - amount1Out ? balance1 - (_reserve1 - amount1Out) : 0;
        require(amount0In > 0 || amount1In > 0, 'UniswapV2: INSUFFICIENT_INPUT_AMOUNT');
        
        //保持xy=k前后成立
        require(balance0*balance1 == _reserve0*_reserve1,"invariant does not hold" );

		_update(balance0,balance1);//更新reserve
}
```







#### 2.2.3.2 完成pair中的其他需求

上一节中我们似乎完成了一个简单的swap，它有面向LP的函数和swap函数，但是关键的费率还是缺少，毕竟大家都要赚钱的。

于是现在我们有以下的需求：

- LP和协议都从swap中获得交易者的手续费
- LP分5/6，协议分1/6

**思考一下如何去分这个钱？**

首先一个简单暴力的分钱方式是每次swap的时候，我们抽取一部分的钱放在一个指定的treasury合约中，LP们凭着自己持有的代币进行均分，每次分钱时LP们主动call函数`collect()`，`collect()`总是要分1/6的LP获得的金额给协议从而使得LP总拿5/6，而协议会拿到1/6。

但是这种设计方案会导致以下问题：

- **时间相关的问题：**用户可能在$t_1$时调用了`swap`，此时的手续费应该是归给$t_1$时刻前提供流动性的用户。于是我们必然需要记录这笔费用发生的时刻，但是手续费用又是如此的微不足道，以至于每次记录费用的gas都可能比手续费贵。而上万笔交易有上万个交易时间戳，每次我们都要算这笔交易的手续费应该归属哪些LP们。这个相关的问题可能比较头疼。
- **Gas问题：**上面也说了，多出的transfer+记录所用的gas费用可能会增加trader的损耗









