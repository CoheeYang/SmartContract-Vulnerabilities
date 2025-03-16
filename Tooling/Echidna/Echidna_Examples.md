# Examples

## 1. Uniswap V2

本章将讲解如何使用echidna的external testing方法测试uniswap v2合约。

本章内容主要参考echidna官方团队的直播文档 [echidna-streaming-series/part3&part4](https://github.com/crytic/echidna-streaming-series/tree/main)

uniswap v2本身项目有两个repo，[v2-core](https://github.com/Uniswap/v2-core/tree/master/contracts)和[v2-peripheral](https://github.com/Uniswap/v2-periphery/tree/master/contracts)

v2-core中包含了三个重要的组成部分：ERC20，Factory，Pair

v2-peripheral则包含了其余的swap中路由所需的合约: Migrator，Router1，Router2

我们将在下面的内容中测试Core repo中的一部分重要的invariant



## 2. V2-core

我们将会通过下面的案例介绍如何使用External Testing的方法来测试uniswap v2中的一个重要的invariant：xy=k，且当xy增加，k也会对应增加。

接下来，将v2-core的仓库克隆，之后我们在目录中创建两个测试文件：`Setup.sol`和`EchidnaTest.sol`

`Setup.sol`:

```solidity
pragma solidity ^0.6.0;

import "../uni-v2/UniswapV2ERC20.sol";
import "../uni-v2/UniswapV2Pair.sol";
import "../uni-v2/UniswapV2Factory.sol";
import "../libraries/UniswapV2Library.sol";

///我们创建一个User合约，这个实例可以模拟用户对uniswap的交互过程，创建多个实例就能类似地创建多个用户。
contract Users {
    function proxy(address target, bytes memory data) public returns (bool success, bytes memory retData) {
        return target.call(data);
    }
}
///接下来进行基本的配置
contract Setup {
	
    UniswapV2Factory factory;
    UniswapV2Pair pair;
    UniswapV2ERC20 testToken1;
    UniswapV2ERC20 testToken2;
    Users user;
    bool completed;///标记我们函数是否完成precondition
    
    constructor() public {
    ///创建实例
        testToken1 = new UniswapV2ERC20();
        testToken2 = new UniswapV2ERC20();
        factory = new UniswapV2Factory(address(this));
        pair = UniswapV2Pair(factory.createPair(address(testToken1), address(testToken2)));
        // Sort the test tokens we just created, for clarity when writing invariant tests later
        (address testTokenA, address testTokenB) = UniswapV2Library.sortTokens(address(testToken1), address(testToken2));
        testToken1 = UniswapV2ERC20(testTokenA);
        testToken2 = UniswapV2ERC20(testTokenB);
        user = new Users();
        user.proxy(address(testToken1),abi.encodeWithSelector(testToken1.approve.selector, address(pair),uint(-1)));
        user.proxy(address(testToken2), abi.encodeWithSelector(testToken2.approve.selector,address(pair),uint(-1)));
    }



///以下均为helper function，帮助我们做precondition
    function _init(uint amount1, uint amount2) internal {
        testToken1.mint(address(user), amount1);
        testToken2.mint(address(user), amount2);
        completed = true;
    }


    function _between(uint val, uint low, uint high) internal pure returns(uint) {
        return low + (val % (high-low +1)); //将输入的val固定在[low,high]之间，类似于foundry中的bound()
    }

}
```

`Echidna.sol`:

```solidity
pragma solidity ^0.6.0;

import "./Setup.sol";
import "../libraries/UniswapV2Library.sol";

contract EchidnaTest is Setup {///继承setup合约，之后EchidnaTest合约就是我们echidna的目标测试合约
    event AmountsIn(uint amount0, uint amount1);
    event AmountsOut(uint amount0, uint amount1);
    event BalancesBefore(uint balance0, uint balance1);
    event BalancesAfter(uint balance0, uint balance1);
    event ReservesBefore(uint reserve0, uint reserve1);
    event ReservesAfter(uint reserve0, uint reserve1);


	//@notice: 检查user作为LP给uniswap提供流动性后，K值的变化
    function testProvideLiquidity(uint amount0, uint amount1) public {
        //1. Preconditions:
        amount0 = _between(amount0, 1000, uint(-1));//[1000,2^256 -1]
        amount1 = _between(amount1, 1000, uint(-1));

        if (!completed) {
            _init(amount0, amount1);
        }
        	//// State before
        uint lpTokenBalanceBefore = pair.balanceOf(address(user));//查看用户LP token的余额
        (uint reserve0Before, uint reserve1Before,) = pair.getReserves();//查看paire中token0与token1的余额
        uint kBefore = reserve0Before * reserve1Before;//得到K


        //2. Action:
        	//// Transfer tokens to UniswapV2Pair contract
        (bool success1,) = user.proxy(address(testToken1), abi.encodeWithSelector(testToken1.transfer.selector, address(pair), amount0));
        (bool success2,) = user.proxy(address(testToken2), abi.encodeWithSelector(testToken2.transfer.selector, address(pair), amount1));
        require(success1 && success2);
        	//// Mint LP token
        (bool success3,) = user.proxy(address(pair), abi.encodeWithSelector(bytes4(keccak256("mint(address)")), address(user)));

        //3. Postconditions:
        if (success3) {
            uint lpTokenBalanceAfter = pair.balanceOf(address(user));
            (uint reserve0After, uint reserve1After,) = pair.getReserves();
            uint kAfter = reserve0After * reserve1After;
            assert(lpTokenBalanceBefore < lpTokenBalanceAfter);
            assert(kBefore < kAfter);
        }
        else {
        	assert(false)
        }
        
        
    }


    function testBadSwap(uint amount0, uint amount1) public {///测试如果我用0个token进去，应该也是0个token出来

        if (!completed) {
            _init(amount0, amount1);
        }

        // Preconditions:
        pair.sync(); //matched the balances with reserves，如果echidna不小心触及了外露的transferToken的函数进行转账了，sync可以更新reserve，保证就和没转帐一样
        require(pair.balanceOf(address(user)) > 0); //there is liquidity for the swap

        // Action:
        (bool success,) = user.proxy(address(pair), abi.encodeWithSelector(pair.swap.selector, amount0, amount1, address(user), ""));

        // Post-condition:
        assert(!success); //从来没转过钱，这个call怎么样都得fail
    }
    
    
    function testSwap(uint amount0, uint amount1) public {
        // Preconditions:
        if (!completed) {
            _init(amount0, amount1);
        }
        require(pair.balanceOf(address(user)) > 0);//User有LPtoken
        require(amount0 > 0 && amount1 > 0);//amount0/amount1大于0

        uint balance0Before = testToken1.balanceOf(address(user));
        uint balance1Before = testToken2.balanceOf(address(user));

        (uint reserve0Before, uint reserve1Before,) = pair.getReserves();
        uint kBefore = reserve0Before * reserve1Before;
        emit ReservesBefore(reserve0Before, reserve1Before);
        
        uint amount0In = _between(amount0, 1, reserve0Before - 1);
        uint amount1In = _between(amount1, 1, reserve1Before - 1);
        // emit AmountsIn(amount0In, amount1In);
        if (amount0In > balance0Before) {
            testToken1.mint(address(user), amount0In - balance0Before);
            balance0Before = testToken1.balanceOf(address(user));
        }
        if (amount1In > balance1Before) {
            testToken2.mint(address(user), amount1In - balance1Before);
            balance1Before = testToken2.balanceOf(address(user));
        }
        require(amount0In <= balance0Before || amount1In <= balance1Before);
        emit BalancesBefore(balance0Before, balance1Before);
        
        uint amount0Out;
        uint amount1Out;

        /**
         * Precondition of UniswapV2Pair.swap is that we transfer the token we are swapping in first.
         * So, we pick the larger of the two input amounts to transfer, and also use
         * the Uniswap library to determine how much of the other we will receive in return.
         */
        if (amount0In > balance0Before) {
            amount0In = 0;
        } else if (amount1In > balance1Before) {
            amount1In = 0;
        }
        if (amount0In > amount1In) {
            require(amount0In <= balance0Before);
            amount1In = 0;
            amount0Out = 0;
            amount1Out = UniswapV2Library.getAmountOut(amount0In, reserve0Before, reserve1Before);
            require(amount1Out > 0);
            emit AmountsIn(amount0In, amount1In);
            emit AmountsOut(amount0Out, amount1Out);
            (bool success1,) = user.proxy(address(testToken1), abi.encodeWithSelector(testToken1.transfer.selector, address(pair), amount0In));
            require(success1);
        } else {
            require(amount1In <= balance1Before);
            amount0In = 0;
            amount1Out = 0;
            amount0Out = UniswapV2Library.getAmountOut(amount1In, reserve1Before, reserve0Before);
            require(amount0Out > 0);
            emit AmountsIn(amount0In, amount1In);
            emit AmountsOut(amount0Out, amount1Out);
            (bool success1,) = user.proxy(address(testToken2), abi.encodeWithSelector(testToken2.transfer.selector, address(pair), amount1In));
            require(success1);
        }

        // Action:
        (bool success2,) = user.proxy(address(pair), abi.encodeWithSelector(pair.swap.selector, amount0Out, amount1Out, address(user), ""));

        // Post-condition:
        /* 1. Swap should be successful */
        assert(success2);
        /* 2. Reserves may change, but k should be (relatively) constant */
        (uint reserve0After, uint reserve1After,) = pair.getReserves();
        uint kAfter = reserve0After * reserve1After;
        emit ReservesAfter(reserve0After, reserve1After);
        // assert(kBefore == kAfter);
        assert(kBefore <= kAfter);
        /* 3. The change in the user's token balances should match our expectations */
        uint balance0After = testToken1.balanceOf(address(user));
        uint balance1After = testToken2.balanceOf(address(user));
        emit BalancesAfter(balance0After, balance1After);
        if (amount0In > amount1In) {
            assert(balance0After == balance0Before - amount0In);
            assert(balance1After == balance1Before + amount1Out);
        } else {
            assert(balance1After == balance1Before - amount1In);
            assert(balance0After == balance0Before + amount0Out);
        }
    }

}
```







