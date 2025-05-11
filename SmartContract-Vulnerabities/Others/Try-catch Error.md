

## 情景介绍

Solidity中的try catch的捕捉是一个需要关注的问题，因为有些交易所中可能会存在try尝试交易，catch指定error失败时重新尝试交易的需求，比如下面的代码：


```solidity
function executeOrder(
    bytes32 key,
    OracleUtils.SetPricesParams memory oracleParams
) external onlyOrderKeeper {
    uint256 startingGas = gasleft();

    try this._executeOrder(
        key,
        oracleParams,
        msg.sender,
        startingGas
    ) {
    } catch Error(string memory reason) {//只捕捉revert,require导致的Error(string)
        bytes32 reasonKey = keccak256(abi.encodePacked(reason));
        // revert instead of cancel if the reason for failure is due to oracle params
        // or order requirements not being met
        if (
            reasonKey == Keys.ORACLE_ERROR_KEY ||
            reasonKey == Keys.EMPTY_POSITION_ERROR_KEY ||
            reasonKey == Keys.INSUFFICIENT_SWAP_OUTPUT_AMOUNT_ERROR_KEY ||
            reasonKey == Keys.UNACCEPTABLE_USD_ADJUSTMENT_ERROR_KEY
        ) {
            revert(reason);
        }

        OrderUtils.cancelOrder(
            dataStore,
            orderStore,
            key,
            msg.sender,
            startingGas
        );
    }
}
```

上述代码出现的问题是，当尝试`_executeOrder()`中，可能会出现非其他的error，比如由于assert 失败或算术异常导致的`panic(uint)`的错误，和定义的`custom error`.

`catch Error(string memory reason)`是精准捕捉`revert("…") 或 require(…, "…")` 抛出的错误。

这也就造成用户可以通过触发如`panic error`，导致解析成string的error内容与if条件中都不存匹配内容，而取消order重试。

**最佳的改进方式是直接使用**`catch (bytes memory data)`，之后解析内容进行匹配。




----事实上这不是个bug，最后到最后的代码你就会知道为什么

我觉得可能存在bug的原因是require(false)和直接revert()，而没有匹配的内容可能会出现问题

```solidity
pragma solidity ^0.8.0;

interface IExternal {
    function foo(uint x) external returns (uint);
}

contract Demo {
    function callFoo(address ext, uint x) external returns (string memory) {
        try IExternal(ext).foo(x) returns (uint result) {
            return string(abi.encodePacked("OK:", Strings.toString(result)));
        } 
        
    ////三种不同的catch方式： 
        
        catch Error(string memory reason) {
            // 精确捕获 revert("…") 或 require(…, "…") 抛出的错误
            return string(abi.encodePacked("Error:", reason));
        } catch Panic(uint code) {
            // 精确捕获 assert 失败或算术异常
            return string(abi.encodePacked("Panic code:", Strings.toString(code)));
        } catch (bytes memory data) {
            // 所有错误
            return string(abi.encodePacked("LowLevel:", data.length > 0 ? string(data) : "no data"));
        }
    }
}

```

## 底层原理

https://www.rareskills.io/post/try-catch-solidity

- 须知：Solidity的编译器会处理相关的try-catch逻辑，当有low-level call出现错误时，代码会告知错误。所以，这种处理错误的方式只限于solidity的complier，当换一种语言或者使用汇编时都其捕捉逻辑会存在不一致。

当Low-level call对外部进行call失败的时候会return一个bool `false`.

这有可能是因为：

- The called contract reverts
- The called contract does an illegal operation (like dividing by zero or accessing an out-of-bounds array index)
- The called contract uses up all the gas

接下来看看`call`在不同情况下失败后都会返回怎么样的message

1. Revert & Rquire
```solidity
// SPDX-License-Identifier: GPL-3.0
pragma solidity >=0.7.0 <0.9.0;

import "hardhat/console.sol";

contract ContractA {
    function mint() external pure {
        revert();
    }
}

contract ContractB {
    function call_failure(address contractAAddress) external {
        (, bytes memory err) = contractAAddress.call(
            abi.encodeWithSignature("mint()")
        );

        console.logBytes(err);
    }
}
//1.
//The revert() error will be triggered no data will be returned
//最后显示的结果是`0x`
//这种返回结果对于 require(false)，没有后续标注的string也是一样的

//2. 
//如果revert中有内容，如revert("Unauthorized");
//则最后返回的是 bytes:
0x08c379a0       <- function selector Error(string)
0000000000000000000000000000000000000000000000000000000000000020  <-offset
000000000000000000000000000000000000000000000000000000000000000c  <-length
556e617574686f72697a65640000000000000000000000000000000000000000  <-data (string Unauthorized)
//对于 require(false,"Unauthorized")，也是一样，返回Error(string) ... 的bytes内容


//3.
//如果是custom error，如revert Unauthorized(msg.sender);
//就会返回 bytes:
0x8e4a23d6  <- Unauthorized(address) selector
0000000000000000000000009c84abe0d64a1a27fc82821f88adae290eab5e07  <-address内容msg.seder
//这种和require(false, Unauthorized());返回的结果一样

```


2. Assert & illegal operations

```solidity
import "hardhat/console.sol";

contract ContractB {
    function mint() external pure {
        assert(false); // we will test what this returns
    }

}

contract ContractA {
    function call_failure(address contractBAddress) external {
        (, bytes memory err) = contractBAddress.call(
            abi.encodeWithSignature("mint()")
        );

        console.logBytes(err);
    }
}

//call的return结果：
0x4e487b71 // <- the function selector `Panic(uint256)`
0000000000000000000000000000000000000000000000000000000000000001 // the error code，代表Assert导致的错误

//panic会出现在Assert失败和illegal operations出现时，1是assert对应的error code，
//其他的illegal operations such as division by zero, popping an empty array, or an array-out-of-bounds
//对应的error code为：
0x32 out-of-bounds array error.
0x12 division by zero
...
see more in `https://docs.soliditylang.org/en/latest/control-structures.html#panic-via-assert-and-error-via-require`


值得注意的是division by zero在assembly中不会revert，他会直接变成0，只有solidity层面出现这种错误才会报错

result := div(numerator, denominator)

```


3. out-of-gas

```solidity 
contract E {

    function outOfGas() external pure {
        while (true) {}
    }

}

contract C {

    function call_outOfGas(address e) external returns (bytes memory err) {
        (, err) = e.call{gas: 2300}(abi.encodeWithSignature("outOfGas()"));
    }

}

// The variable err will be empty. Due to the 63/64 rule for gas, contract C will still have 1/64th of the original gas left, so the transaction to the function call_outOfGas itself will not necessarily revert due to out-of-gas, even if you try to forward all available gas to contract E.
```




### Try catch的工作原理

下面的代码会展示，catch flow如何正确地catch各种不同的error。
当尝试删除一部分catch flow时，对于不匹配的error，catch block将不会产生反应


```solidity
// SPDX-License-Identifier: GPL-3.0

pragma solidity >=0.7.0 <0.9.0;

import "hardhat/console.sol";

contract ContractB {

    error CustomError(uint256 balance);

    uint256 public balance = 10;

    function decrementBalance() external {
        require(balance > 0, "Balance is already zero");
        balance -= 1;
    }

    function revertTest() external view {
        if (balance == 9) {
            // revert without a message
            revert();
        }

        if (balance == 8) {
            uint256 a = 1;
            uint256 b = 0;
            // This is an illegal operation and should cause a panic (Panic(uint256)) due to division by zero
            a / b;
        }

        if (balance == 7) {
            // revert with a message
            revert("not allowed");
        }

        if (balance == 6) {
            // revert with a message
            revert CustomError(100);
        }
    }
}


contract ContractA {

    event Errorhandled(uint256 balance);

    ContractB public contractB;

    constructor(address contractBAddress) {
        contractB = ContractB(contractBAddress);
    }

    function callContractB() external view {

        try contractB.revertTest() {
            // Handle the success case if needed
        } catch Panic(uint256 errorCode) {
            // handle illegal operation and `assert` errors
            console.log("error occurred with this error code: ", errorCode);
        } catch Error(string memory reason) {
            // handle revert with a reason
            console.log("error occured with this reason: ", reason);
        } catch (bytes memory lowLevelData) {
            // revert without a message
            if (lowLevelData.length == 0) {
                console.log("revert without a message occured");
            }

            // Decode the error data to check if it's the custom error
            if (                bytes4(abi.encodeWithSignature("CustomError(uint256)")) ==                bytes4(lowLevelData)
            ) {
                // handle custom error
                console.log("CustomError occured here");
            }
        }
    }
}

```

When try/catch fails? 

see [blog](https://www.rareskills.io/post/try-catch-solidity#What%20are%20the%20scenarios%20try/catch%20will%20fail%20to%20handle%20your%20errors?:~:text=What%20are%20the%20scenarios%20try/catch%20will%20fail%20to%20handle%20your%20errors%3F)
