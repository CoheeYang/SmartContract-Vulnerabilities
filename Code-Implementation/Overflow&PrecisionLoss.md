### 溢出与精度损失 

注意除法就好了，solidity没有小数

```Solidity
// SPDX-License-Identifier: MIT
pragma solidity 0.8.20;



contract Overflow {
    uint8 public count;

    // uint8 has a max value of 255, so if we add 1 to 255, we get 0 if it's unchecked!
    // Versions prior to 0.8 of solidity also have this issue
    function increment(uint8 amount) public {
        unchecked {
            count = count + amount;
        }
    }
}



contract Underflow {
    uint8 public count;

    // uint8 has a min value of 0, but if we subtract 1 from 0, we get 255 if it's unchecked!
    // Versions prior to 0.8 of solidity also have this issue
    function decrement() public {
        unchecked {
            count--;
        }
    }
}



//除不尽，没有小数，会向下取整（Rounding-Down）
contract PrecisionLoss {
    uint256 public moneyToSplitUp = 225;
    uint256 public users = 4;

    // This function will return 56, but we want it to return 56.25
    function shareMoney() public view returns (uint256) {
        return moneyToSplitUp / users;
    }
}
```