[TOC]

我认为过去的分类方法简直不科学，很多维度都非常混乱，比如DOS，griefing其实更多是结果维度的定义，而类似front run是attacker的攻击路径的维度。
但是我们现在需要找bug是在找攻击的根源与路径。
什么是漏洞的根源？
比如access control的不严格，比如off by one 的顺序错误

##

1. 存在验证的条件涉及state variable，但是此状态变量可以由用户随意更改

2. balanceOf引起的问题

3. external call引起的问题
4. for loop中的问题

5. 外部依赖的问题
wired erc20

6. unchecked input
7. 改变交易顺序导致的MEV
8. Signature
9. calculations
10. Hash Collisions(EncodePacked)







## 列表

1. 重入
2. front run
3. Wired ERC20
4. Flash-Staking
5. Oracle Manipulation
6. DoS
7. Miss Behavior
8. greifing-Attack
9. Signature Shit
10. Calculation Related Issues
12. access control
13. Upgradable
14. ERC4626
15. Low Level
16. Governance Attack
17. Logic Error(Protocal Specific)



### ReentrancyAttack



### FrontRun
front run经常出现在交易顺序related issues，如果改变交易的顺序是否会影响用户的利益
比如
AMM中的`swap()`，然后类似使用xy=k的买卖交易都可能面临这种问题。

Perpetual Dex中由于fundingFee的空多开仓顺序需要keeper来执行

Lending的`Liquidations`动作也会有抢跑的情况
1. liquidator之间的抢跑：一个要被清算的交易，谁先执行谁拿钱则会导致天然的MEV问题，如果后执行的人被revert了还好没什么，就怕后执行的人会因为被frontRun而亏钱
2. liquidator和liquidatee之间的抢跑:如liquidatee通过调用函数更新liquidator奖励的方式来进行(see example)[https://solodit.cyfrin.io/issues/trst-m-1-markliquidationstatus-may-cause-the-liquidator-to-lose-premium-trust-security-none-stella-markdown_]



Liquidity Pool中创建pool的抢跑[example](https://solodit.cyfrin.io/issues/h-12-attacker-can-get-extremely-cheap-synth-by-front-running-create-pool-code4rena-vader-protocol-vader-protocol-contest-git)
通过抢先创建pool，并定义xy=k的值，使得价格很低。
之前的报告中也有类似的抢跑createPool从而创建新的手续费很低（或者为0）的池子，损害协议的利益

Just In time:
Flash Staking in staking actions
[Morpho V1](https://solodit.cyfrin.io/issues/frontrunners-can-exploit-system-by-not-allowing-head-of-dll-to-match-in-p2p-spearbit-morpho-pdf)中按余额排名进行借贷导致大户能just in time，在借贷需求被上传到交易池时抢跑，把自己的钱全刷排名插队，然后退出剩余余额。



### Wired ERC20



### Flash-Staking



### Oracle Manipulation









### DoS



### Misbehavior

这多数是由于developer的不小心引起的错误，问题非常多而杂，无法集成为某一种类

[OZ-across](https://blog.openzeppelin.com/across-audit#spokepools-fill-function-performs-malformed-call)这个bug就是call的时候参数少写了一位parameter。

[OZ-across](https://blog.openzeppelin.com/across-audit#forwarder-and-withdrawal-helper-contracts-do-not-handle-eth-transfers-correctly)则少写了receive，导致该收钱的合约没法收钱。







折叠的mapping在删除的时候会出现问题

减法导致的revert

downcasting

No withdraw in an token/ether receive contract

parallel data structure

typos

yul中未更新的指针

decimal handling

msg.value in the loop

chainlink(freshness of data, heartbeat, sequencer check for L2)

off-by-one error(错误的执行顺序)

block -re-org

### Griefing-Attack

------

Griefing Attack指的是利用合约的漏洞，做出往往损害protocal或者协议的行为，比如常见的问题如在有deposit/withdraw限制的合约中，攻击者重复使用或者抢跑利用这些漏洞导致特定user无法存入或者取出。案例如下

```solidity
pragma solidity ^0.8.17;

contract DelayedWithdrawal {
    address beneficiary;
    uint256 delay;
    uint256 lastDeposit;

    constructor(uint256 _delay) {
        beneficiary = msg.sender;
        lastDeposit = block.timestamp;
        delay = _delay;
    }

    modifier checkDelay() {
        require(block.timestamp >= lastDeposit + delay, "Keep waiting");
        _;
    }

    function deposit() public payable {
        require(msg.value != 0);
        lastDeposit = block.timestamp;
    }

    function withdraw() public checkDelay {
        (bool success, ) = beneficiary.call{value: address(this).balance}("");
        require(success, "Transfer failed");
    }
}
```

真实发送在协议中的有

1. 稳定币项目：[Aegis](https://audits.sherlock.xyz/contests/799/report)通过大量的mint/redeem透支某段时间内的限额，导致其他用户无法换取稳定币。
2. Strategy项目：[Stability-contracts](https://cantina.xyz/code/e1c0be8d-0c3d-485a-a446-a582beb120b1/audits/initial-audit-stability-platform-v24.01.1-alpha.md)合约中withdraw和deposit都有个block.number锁，会更新每个用户的上次的存取时间，攻击者可以做到和上述案例类似的攻击。
3. Strategy项目：[这里](https://solodit.cyfrin.io/issues/h-02-griefing-attack-via-continuous-deposit-and-withdrawal-pashov-audit-group-none-radiant-june-markdown)的案例也说明通过频繁deposit/withdraw也会导致griefing，但是不是通过delay的锁，而是频繁的deposit限制了liquidity pool的正常运行
4. Lending项目中：[ZeroLend](https://solodit.cyfrin.io/issues/griefing-attack-to-cause-the-rewards-of-a-user-to-be-locked-and-when-users-claim-the-reward-after-maturity-date-user-will-suffer-the-penalty-immunefi-zerolend-git)的项目中出现了` getReward(address account IERC20 token)`的方法，这个方法会claim staker的reward，但是如果提前领取会记录penalty，这就给了attack提前替其他人claim reward，让别人能受到penalty的影响。





#### Gas-Griefing

Gas-Griefing则是其中一种特殊的griefing-attack，它常常通过操作gas相关的损害系统或者其他用户的利益。比如利用`63/64 rule`使得external-call 花费大量63/64的gas而失败，而剩下的`1/64`的gas被保留完成执行原函数的任务，使得合约错误地记录状态。

```solidity
pragma solidity ^0.8.17;

contract Relayer {
    mapping (bytes => bool) executed;
    address targetSwap;
	
	//这个函数用来执行一个Swap的行为，如果给出的_data命令target执行一个非常长的swap路径，
	//从而导致超过gasLimit而失败，那么这个人的地址就再无可能在被使用了
    function forward(bytes memory _data,address _onBehave) public {
    
        require(!executed[_onBehave], "Replay protection");   
        executed[_onBehave] = true;
        targetSwap.call(abi.encodeWithSignature("execute(bytes)", _data));
        //没有检查call成功与否，失败了反而记录了地址的信息，这样就导致了attack可以攻击grief其他用户
    }
}
```

1. CrossChain项目：[Accross](https://blog.openzeppelin.com/across-audit#griefing-attacks-are-possible-in-zk-adapters)中，用户通过可以设置特别高的 `maxFeePerGas` 与 `maxPriorityFeePerGas`，使得`tx.gasprice`巨大，从而搬空那个将垫付交易gas的合约`HubPool`

2. Dex 项目中：永续合约交易所[GMX V2](【6个关键漏洞 | 建立你的工具箱】 【精准空降到 30:23】 https://www.bilibili.com/video/BV1AjN9e8EaV/?share_source=copy_web&vd_source=03848fedb0d9e50472e7d3176f314e01&t=1823)使用没有检查swapPath的长度，同时GMX为了避免funding fee导致的frontRun，从而使用keeper来执行交易，这样用户可以先提交交易，此时记录过早的price价格，而keeper执行时通过callback中卡住gasLimit，使得keeper无法执行，而在用户发现价格有利时又改变callback合约中的函数，使得交易能够成功通过，此时交易价格为很早的价格，用户可以直接清空头寸，从而直接获利。

   

   

### Signature Shit

------

#### 1. Missing Validation

这个问题会出现在使用原生的`ecrecovery`中，即比如下面的`recover`得到的地址`signer`并没有进行后续的比如0地址的检查。

```solidity
`function recover(uint8 v, bytes32 r, bytes32 s, bytes32 hash) external {
    address signer = ecrecover(hash, v, r, s);
    //Do more stuff with the hash
}
A crucial check for address(0) is absent in this instance. This omission allows an attacker to submit invalid signatures with arbitrary payloads yet pass as valid. A simple yet effective solution to this issue would be to include a check like the following:

require(signer != address(0), "invalid signature");
Even better, OpenZeppelin's ECDSA library should be used because it automatically reverts when invalid signatures are encountered.
```



#### 2. Signature Malleability

也是使用了`ecrecovery`的方法进行解密，导致出现的`s`的奇偶解都能解出同一个`msg.sender`.

```solidity
pragma solidity 0.8.17;

contract signatureMealleable is Ownable {
    address token;
    mapping(bytes32 => bool) executed;

    function signedTransfer(
        address to,
        uint256 amount,
        uint8 v,
        bytes32 r,
        bytes32 s
    ) external {
        bytes32 msgHash = keccak256(abi.encode(msg.sender, to, amount));
@>      address signer = ecrecover(msgHash, v, r, s);
        require(signer == owner);

        bytes32 sigHash = keccak256(abi.encode(msgHash, v, r, s));
        require(!executed[sigHash]);
        executed[sigHash] = true;

        IERC20(token).safeTransfer(to, amount);
    }
}

```

最佳实践是使用OZ的ECDSA库，其规定了输入的s必须是大于n/2的数（椭圆曲线的上半部分的s）

```solidity
if (uint256(s) > 0x7FFFFFFFFFFFFFFFFFFFFFFFFFFFFFFF5D576E7357A4501DDFE92F46681B20A0) {
    return (address(0), RecoverError.InvalidSignatureS);
}
```

OZ库中 = 4.1.0 < 4.7.3版本的`ECDSA.recover(bytes32 hash, bytes memory signature)`和` tryRecover(bytes32 hash, bytes memory signature)`会受到一个签名可塑性攻击，此攻击是基于EIP-2098 定义的 **64 字节** “compact” `(r, vs)` 格式

具体而言：

- 传统的 **65 字节** `(r, s, v)` 签名格式
- EIP-2098 定义的 **64 字节** “compact” `(r, vs)` 格式

因为这两种格式在逻辑上是等价的——只不过把 `v`（恢复 ID）跟 `s` 的最高位压缩到同一个字节里——攻击者就可以拿到一次合法提交过的 65 字节签名，把它转换成 64 字节的 compact 形式，再次通过`recover`或者`tryRecover`来recover。

这是因为这些函数内部有一个if判别导致65/64字节的signature能两次输出同一个结果

```solidity
   function tryRecover(bytes32 hash, bytes memory signature) internal pure returns (address, RecoverError) {
        // Check the signature length
        // - case 65: r,s,v signature (standard)
        // - case 64: r,vs signature (cf https://eips.ethereum.org/EIPS/eip-2098) _Available since v4.1._
        if (signature.length == 65) {
            bytes32 r;
            bytes32 s;
            uint8 v;
            // ecrecover takes the signature parameters, and the only way to get them
            // currently is to use assembly.
            assembly {
                r := mload(add(signature, 0x20))
                s := mload(add(signature, 0x40))
                v := byte(0, mload(add(signature, 0x60)))
            }
            return tryRecover(hash, v, r, s);
        } else if (signature.length == 64) {
            bytes32 r;
            bytes32 vs;
            // ecrecover takes the signature parameters, and the only way to get them
            // currently is to use assembly.
            assembly {
                r := mload(add(signature, 0x20))
                vs := mload(add(signature, 0x40))
            }
            return tryRecover(hash, r, vs);
        } else {
            return (address(0), RecoverError.InvalidSignatureLength);
        }
    }
```

而在修复后，OZ的此类recovery只接受65字节的signature

```solidity
    function tryRecover(bytes32 hash, bytes memory signature) internal pure returns (address, RecoverError) {
        if (signature.length == 65) {
            bytes32 r;
            bytes32 s;
            uint8 v;
            // ecrecover takes the signature parameters, and the only way to get them
            // currently is to use assembly.
            /// @solidity memory-safe-assembly
            assembly {
                r := mload(add(signature, 0x20))
                s := mload(add(signature, 0x40))
                v := byte(0, mload(add(signature, 0x60)))
            }
            return tryRecover(hash, v, r, s);
        } else {
            return (address(0), RecoverError.InvalidSignatureLength);
        }
    }
```





这个bug只对4.7.3以下版本产生影响，且以`v`,`r`,`s`或者,`r`,`vs`分开输入的函数不会受此影响



#### 3. Replay Attacks

经典的攻击方式，只验证了的signer，而没有在后续对签名进行作废

```solidity
    function action(uint256 _param1, bytes32 _param2, bytes memory _sig) external {
        bytes32 hash = keccak256(abi.encodePacked(_param1, _param2));
        bytes32 signedHash = hash.toEthSignedMessageHash();
        address signer = signedHash.recover(_sig);

        require(signer == owner, "Invalid signature");
	
        // use `param1` and `param2` to perform authorized action
    }
```

正确修复的方法如下，加入了mapping禁止了重复的replay，但是同时又加入了`nonce`作为签名hash，从而保证user可以多次进行同一个合法动作：

```solidity
    function action(uint256 _param1, bytes32 _param2, uint256 _nonce, bytes memory _sig) external {
        bytes32 hash = keccak256(abi.encodePacked(_param1, _param2, _nonce));//加入nonce
+       require(!seenSignatures[hash], "Signature has been used");

        bytes32 signedHash = hash.toEthSignedMessageHash();
        address signer = signedHash.recover(_sig);
        require(signer == owner, "Invalid signature");

+       seenSignatures[hash] = true;

        // use `param1` and `param2` to perform authorized action
    }
```

但是，对于cross-chain的合约，这些都不够用，因为可能出现cross-chain-replay，比如nonce值为10的交易，在另一个链的合约上又重复使用同意的param进行了，这是因为首先这个`sig`可以成功验证，同时另外一个链上的合约mapping不可能会记载另外一个链上的交易。

所以一般得加如chainId作为signature的一部分来进行recover

```solidity
    function action(uint256 _param1, bytes32 _param2, uint256 _nonce, uint256 _chainId, bytes memory _sig) external {
        require(_chainId == block.chainid, "Invalid chain ID");//验证chainId

@>      bytes32 hash = keccak256(abi.encodePacked(_param1, _param2, _nonce, _chainId));
        require(!seenSignatures[hash], "Signature has been used");

        bytes32 signedHash = hash.toEthSignedMessageHash();
        address signer = signedHash.recover(_sig);
        require(signer == owner, "Invalid signature");

        seenSignatures[hash] = true;

        // use `param1` and `param2` to perform authorized action
    }
```



#### 4. Frontrunning

即使上述的问题都被cover到了，signature依然有frontRun的问题，比如上述的`action()`函数不需要使用`param1`作为message的一部分时，有人就能抢跑，并输入一个对自己有利的`param1`

```solidity
bytes32 hash = keccak256(abi.encodePacked(_param2, _nonce, _chainId));
```

还有比如BakerFi的案例，攻击者可以抢先用户执行该`pullTokensWithPermit`让用户转账给合约的`router`，然后又立马调用清理`router` token的函数`sweepTokens `来进行窃取

```solidity
   function pullTokensWithPermit(
        IERC20Permit token,
        uint256 amount,
        address owner,
        uint256 deadline,
        uint8 v,
        bytes32 r,
        bytes32 s
    ) internal virtual {
        // Permit the VaultRouter to spend tokens on behalf of the owner
        IERC20Permit(token).permit(owner, address(this), amount, deadline, v, r, s);

        // Transfer the tokens from the owner to this contract
        IERC20(address(token)).safeTransferFrom(owner, address(this), amount);
    }
//see: https://solodit.cyfrin.io/issues/h-05-the-implementation-of-pulltokenswithpermit-poses-a-risk-allowing-malicious-actors-to-steal-tokens-code4rena-bakerfi-bakerfi-git
```

### Calculaiton Related
加减乘除幂

加法：一般不会出现什么问题
乘：乘法可能因为连续的乘法中导致的overflow
除法：precision loss/panic division loss
减法：小减大导致的可能问题
幂：使用^而非**，忘记



### ERC4626
Inflation Attack

非标准ERC4626的[inflation attack](https://solodit.cyfrin.io/issues/trst-h-4-first-depositor-can-steal-asset-tokens-of-others-trust-security-none-stella-markdown_)



### Low Level

------

#### ABI Hash Collision
Import Information
checks the 

**ABI 编码（abi.encodePacked）**
 Solidity 提供两种常用的序列化函数：

1. `abi.encode(...)` ——会在每个动态类型前插入长度信息，保证不同参数间编码不混淆，但结果更大。
2. `abi.encodePacked(...)` ——将所有参数紧凑拼接，省去长度标记，节省 gas，却在多动态参数时容易“黏”在一起，导致拼接后的字节流有歧义。

```solidity
uint x = 10;
address addr = 0x7A58c0Be72BE218B41C608b7Fe7C5bB630736C71;
string name = "0xAA";
uint[2] array = [5, 6];

function encode() public view returns(bytes memory result) {
    result = abi.encode(x, addr, name, array);
}

0x
000000000000000000000000000000000000000000000000000000000000000a    // x
0000000000000000000000007a58c0be72be218b41c608b7fe7c5bb630736c71    // addr
00000000000000000000000000000000000000000000000000000000000000a0    // name 参数的偏移量
0000000000000000000000000000000000000000000000000000000000000005    // array[0]
0000000000000000000000000000000000000000000000000000000000000006    // array[1]
0000000000000000000000000000000000000000000000000000000000000004    // name 参数的长度为4字节
3078414100000000000000000000000000000000000000000000000000000000    // name

//encodePakced结果
0x
000000000000000000000000000000000000000000000000000000000000000a
7a58c0be72be218b41c608b7fe7c5bb630736c71
30784141
0000000000000000000000000000000000000000000000000000000000000005
0000000000000000000000000000000000000000000000000000000000000006

```

但是keccak256(abi.encodePacked())常被作为payload,mapping-key存在，而如果使用`abi.encodePacked()`在encode动态变量的时候，没有长度分割容易导致碰撞。
比如下面的案例：
```solidity
    function claimRewards(address[] calldata privileged, address[] calldata regular) external {
        bytes32 payoutKey = keccak256(abi.encodePacked(privileged, regular));
        require(allowedPayouts[payoutKey], "Unauthorized claim");
        allowedPayouts[payoutKey] = false;
        _payout(privileged, premiumPayout);
        _payout(regular, regularPayout);
    }

//当输入hash1和hash2，都能通过检验，但是具体输入payout的地址却完全不同了
hash1 = keccak256(abi.encodePacked([addr1], [addr2, addr3]));
hash2 = keccak256(abi.encodePacked([addr1, addr2], [addr3]));
require(hash1 == hash2);
```

除了mapping中会直接出现如此问题，签名中也可能出现同样的问题，比如下面的：

```solidity
    modifier signedOnly(bytes memory message, bytes calldata signature) {
        // Gets the address that signed the message with signature
        address messageSigner = ECDSA.recover(
            ECDSA.toEthSignedMessageHash(message),
            signature
        );

        require(hasRole(SIGNER_ROLE, messageSigner), "Signer not recognized");

        _;
    }


    function call(
        address instance,
        bytes calldata data,
        bytes calldata signature
    )
        external
        payable
        operatorOnly(instance)
        signedOnly(abi.encodePacked(msg.sender, instance, data), signature)
    {
        _call(instance, data, msg.value);
    }


```
除了动态，其他的类型都可能出现碰撞
```solidity
    //比如下面两个都是，但是拼装的变量顺序要调转，难度比较大，出现概率小
        uint32  a1 = 0x12345678;
        uint256 b1 = 0x99999999999999999999999999999999999999999999999999999999FFFFFFFF;
       
        // Pack 2
        uint256 a2 = 0x1234567899999999999999999999999999999999999999999999999999999999;
        uint32  b2 = 0xFFFFFFFF;
```

其他ABI相关问题：
插值 
```solidity
_id是个uint256类型
return string(abi.encodePacked(baseURI, _id, ".json"))
//插值插入链接，uint256是32字节的，插入会有一大串0，而非一个1或者2，3，4
//see https://solodit.cyfrin.io/issues/m-04-cidnft-broken-tokenuri-function-code4rena-canto-identity-protocol-canto-identity-protocol-contest-git
```

## Reference

https://scsfg.io/hackers/



