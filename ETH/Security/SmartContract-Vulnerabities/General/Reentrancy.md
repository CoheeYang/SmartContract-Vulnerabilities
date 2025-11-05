- ## 重入攻击

  重入攻击是指当合同中的漏洞可能允许恶意合同在原始函数执行期间意外地重新进入合同。这可以使得智能合同中耗尽资金。就智能合同黑客攻击造成的资金损失而言，重入攻击可能是**影响最大**的漏洞，应引起相应的重视。[重入攻击列表](<https://github.com/pcaversaccio/reentrancy-attacks>)

  

  ### 外部调用

  如果存在一个外部调用call，攻击者可以利用它来执行重入攻击。外部调用允许被调用方使用`fallback`执行任意代码。最经典的外部调用是合约中使用`call`转账，它将触发合同中实现的`receive`或`fallback`函数。攻击者可以在`fallback`方法中编写任何任意逻辑，使得每当合同收到转账时，该逻辑就会被执行。

  ```solidity
  // 送钱的写法
  function withdraw() external {
      uint256 amount = balances[msg.sender];
      (bool success,) = msg.sender.call{value: balances[msg.sender]}("");
      require(success);
      balances[msg.sender] = 0;
  }
  ```

  **然而,并不是所有转账都会导致重入**，`transfer`和`send`这种转账方式由于2300的gas限制，所以无法完成逻辑复杂的reentrancy.

  

  值得注意的是，**外部调用的存在并不总是显而易见的**，因此，重要的是要意识到智能合同中可能执行外部调用的任何方式。比如在ERC721的lib中，每次你需要将ERC721转到一个地址时，`onERC721Received`函数就会产生潜在的威胁。

  还有比如，OpenZeppelin的[`ERC721._safeMint`](<https://github.com/OpenZeppelin/openzeppelin-contracts/blob/3f610ebc25480bf6145e519c96e2f809996db8ed/contracts/token/ERC721/ERC721.sol#L244>) & [`ERC721._safeTransfer`](<https://github.com/OpenZeppelin/openzeppelin-contracts/blob/3f610ebc25480bf6145e519c96e2f809996db8ed/contracts/token/ERC721/ERC721.sol#L190>) 函数也是一个难以察觉的外部调用的例子。

  ```solidity
  /**
    * @dev Same as {xref-ERC721-_safeMint-address-uint256-}[`_safeMint`], with an additional `data` parameter which is
    * forwarded in {IERC721Receiver-onERC721Received} to contract recipients.
    */
  function _safeMint(
      address to,
      uint256 tokenId,
      bytes memory _data
  ) internal virtual {
      _mint(to, tokenId);
      require(
          _checkOnERC721Received(address(0), to, tokenId, _data),
          "ERC721: transfer to non ERC721Receiver implementer"
      );
  }
  ```

  该函数被称为`_safeMint`，因为它通过首先检查合同是否实现了ERC721Receiver来防止token被无意中铸造到合同中，即标记自己为愿意接收NFT的接收者。这看起来似乎没有问题，但`_checkOnERC721Received`是对外部接收合同的调用，允许任意执行。

  

  ### 跨函数重入

  跨函数重入(cross-function reentrancy)是同一过程的更复杂版本。当易受攻击的函数与攻击者可以利用的函数共享状态时，会发生跨函数重入。

  ```solidity
  // 送钱童子
  function transfer(address to, uint amount) external {
    if (balances[msg.sender] >= amount) {
      balances[to] += amount;
      balances[msg.sender] -= amount;
    }
  }
  
  function withdraw() external {
    uint256 amount = balances[msg.sender];
    (bool success,) = msg.sender.call{value: balances[msg.sender]}("");
    require(success);
    balances[msg.sender] = 0;
  }
  ```

  在这个例子中，黑客可以通过让`fallbakc`在`withdraw()`函数中调用`transfer()`来利用这个合同，在余额设置为0之前转移已花费的资金。  

  当然这只是个简单的例子，实际上的例子可能更加复杂，比如下面代码:

  ```solidity
  // SPDX-License-Identifier: GPL-3.0
  pragma solidity 0.8.17;
  
  contract TwoStepSwap {
      struct Swap {
          address user;
          uint256 amount;
          address[] swapPath;
          bool unwrapNativeToken;
      }
  
      uint256 swapNonce;
      mapping(uint256 => Swap) pendingSwaps;
  
      function createSwap(uint256 _amount, address[] calldata _swapPath, bool _unwrapNativeToken) external {
          IERC20(_swapPath[0]).safeTransferFrom(msg.sender, address(this), _amount);
          pendingSwaps[++swapNonce] = Swap({
              user: msg.sender,
              amount: _amount,
              swapPath: _swapPath,
              unwrapNativeToken: _unwrapNativeToken
          });
      }
  
      function cancelSwap(uint256 _id) external {
          Swap memory swap = pendingSwaps[_id];
          require(swap.user == msg.sender);
          delete pendingSwaps[_id];
          IERC20(swap.swapPath[0]).safeTransfer(swap.user, swap.amount);
      }
  
      function executeSwap(uint256 _id) external onlySwapExecutor nonReentrant {
          Swap memory swap = pendingSwaps[_id];
          // If swapPath ends in WETH and unwrapNativeToken is true, send ether to the user
          ISwapper(swap.swapPath[swap.swapPath.length - 1]).swap(swap.user, swap.amount, swap.swapPath, swap.unwrapNativeToken);
          delete pendingSwaps[_id];
      }
  }
  ```

  这个代码设计之初是让用户执行一个两步走的swap(这种两步走的swap一般是为了避免front-running或者是为了现价单等情况而设计)；具体而言，用户通过`createSwap`创建swap，而具体的执行机器人将调用`executeSwap`执行swap。

  在这个代码中，尽管`executeSwap`被各种限制，但是其中调用的`swap`函数包括了一个这样的逻辑：当swap最后的resulting token为WETH时，用户可以将最后收到的WETH指定为ETH，从而实现一笔外部调用转账。

  这个给了攻击者可乘之机，因为攻击者可以在其合约的`fallback`函数中再次调用 `cancelSwap`，使得在swap完成前取消订单，并转回所有的资金。

  而这就导致用户即退了钱，也通过swap换到了ETH。

  类似的重入攻击还有很多，但究其原因都是在不同的函数/合约之中使用了或者潜在使用了外部调用。

  所以检查所有external/public函数并且评估每一个external call的风险是非常有必要的。

  

  ### 只读重入

  只读重入(Read-only Reentrancy)是一种只针对view函数的重入攻击，view函数一般由于不修改以太坊上的状态而被人经常忽视，但是如果合约中有非常依赖Oracle/PriceFeed的返回值，且存在外部调用的函数，则这个漏洞将非常明显。

  ```solidity
  // 有漏洞的合约
  contract VulnerableBank {
      mapping(address => uint) public balances;
      bool private locked;
      
      // 存款函数
      function deposit() external payable {
          balances[msg.sender] += msg.value;
      }
      
      // 取款函数 - 有重入漏洞
      function withdraw(uint amount) external {
          require(balances[msg.sender] >= amount, "Insufficient balance");
          
          // 先更新余额
          balances[msg.sender] -= amount;
          
          // 然后转账 - 这里可能触发重入
          (bool success, ) = msg.sender.call{value: amount}("");
          require(success, "Transfer failed");
      }
      
      // 只读函数 - 计算总余额
      function getTotalBalance() external view returns (uint) {
          return address(this).balance;
      }
      
      // 获取用户信息和总余额
      function getUserInfo(address user) external view returns (uint userBalance, uint totalBalance) {
          userBalance = balances[user];
          totalBalance = this.getTotalBalance(); // 这里可能被重入攻击！
      }
  }
  
  // 攻击者合约
  contract Attacker {
      VulnerableBank public bank;
      bool private isAttacking;
      
      constructor(address _bank) {
          bank = VulnerableBank(_bank);
      }
      
      // 开始攻击
      function startAttack() external payable {
          require(msg.value > 0, "Need ETH to attack");
          bank.deposit{value: msg.value}();
          bank.withdraw(msg.value);
      }
      
      // 回退函数 - 重入入口
      receive() external payable {
          if (!isAttacking) {
              isAttacking = true;
              
              // 在银行合约处于不一致状态时调用只读函数
              (uint myBalance, uint totalBalance) = bank.getUserInfo(address(this));
              
              // 此时银行的状态是不一致的：
              // - 我们的余额已经减少了（在withdraw中）
              // - 但ETH还没有转出（正在执行转账）
              // 所以getTotalBalance()会返回错误的值
              
              console.log("During reentrancy - My balance:", myBalance);
              console.log("During reentrancy - Total balance:", totalBalance);
              
              // 可以基于这些错误信息进行其他攻击...
              // 比如调用依赖view函数的合约
          }
      }
      
      // 检查攻击后的状态
      function checkState() external view {
          (uint myBalance, uint totalBalance) = bank.getUserInfo(address(this));
          console.log("After attack - My balance:", myBalance);
          console.log("After attack - Total balance:", totalBalance);
      }
  }
  ```
  
  正如我们从上面的例子中看到的，尽管两个函数都有`nonReentrant`修饰符，但攻击者仍然可以在`A.withdraw`的回调中调用`B.claim`，由于攻击者的余额尚未更新，执行成功。
  
  
  
  ### 重入预防
  
  最简单的重入预防机制是使用[`ReentrancyGuard`](<https://github.com/OpenZeppelin/openzeppelin-contracts/blob/master/contracts/utils/ReentrancyGuard.sol>)，它允许您向可能易受攻击的函数添加修饰符，例如`nonReentrant`。尽管它对大多数形式的重入都有效，但是只读重入可能可以绕过这一点，
  
  为了获得最佳安全性，请使用**检查-效果-交互模式**（CEI），这是一种简单的智能合同函数排序规则。
  
  这种结构对重入攻击有效，因为当攻击者重新进入函数时，状态更改已经完成。例如：
  
  ```solidity
  function withdraw() external {
    //check
    uint256 amount = balances[msg.sender];
    //effect
    balances[msg.sender] = 0;
    //interact
    (bool success,) = msg.sender.call{value: balances[msg.sender]}("");
    require(success);
  }
  ```
  
  由于在进行任何交互之前将余额设置为0，如果合同被递归调用，在第一次转账后将没有任何东西可以发送。



### Further Reading

  - [智能合同重入攻击：渗透测试人员的最佳实践](<https://consensys.github.io/smart-contract-best-practices/attacks/reentrancy/> )
  - [智能合同重入攻击：如何识别可利用的漏洞以及一个攻击示例](<https://medium.com/@gus_tavo_guim/reentrancy-attack-on-smart-contracts-how-to-identify-the-exploitable-and-an-example-of-an-attack-4470a2d8dfe4> )
  - <https://medium.com/coinmonks/protect-your-solidity-smart-contracts-from-reentrancy-attacks-9972c3af7c21>
  - <https://medium.com/coinmonks/protect-your-solidity-smart-contracts-from-reentrancy-attacks-9972c3af7c21>   
  - https://lab.guardianaudits.com/encyclopedia-of-solidity-attack-vectors/reentrancy
  - https://github.com/coinspect/learn-evm-attacks/tree/master/test/Reentrancy/ReadOnlyReentrancy

