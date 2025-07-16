注意tx.origin的使用，别用中间变量来判断，比如用tx.origin == owner来判断owner，因为如果owner被钓鱼诱导调用了黑客的合约导致tx.origin==owner成立，会导致资金损失.

Tx.origin基本很少用，只有在区分CA和EOA账户时才会使用这个

```solidity
  function isContract(address account) public view returns (bool) {
        uint256 size;
        assembly {
            ///判断地址上的代码长度，长度为0则为EOA，反之CA
            size := extcodesize(account)
        }
        return size > 0;
    }

    bool public pwned = false;

    // 确保仅有 EOA 能调⽤
    function protected() external {
        require(!isContract(msg.sender), "no contract allowed");
        pwned = true;
    }


///但是如果是constructor中的代码先调用这个函数，就会绕过这个检查
////最佳判断EOA/CA的方法是
require(tx.origin == msg.sender,"no contract allowed")

```

