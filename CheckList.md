## 列表

1. 重入
2. front run
3. Wired ERC20
4. Flash-Staking
5. Oracle Manipulation
6. DoS
7. 不小心写错的傻逼
8. greifing-Attack
9. Signature Shit
10. Precision Loss and panic error in Divion
11. Overflow/Underflow error in calculation(比如连乘过大的数导致overflow，减法没有检查数据大小直接相减)
12. access control
13. Upgradable
14. ERC4626
15. Low Level
16. Logic Error(Protocal Specific)











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
see: https://solodit.cyfrin.io/issues/h-05-the-implementation-of-pulltokenswithpermit-poses-a-risk-allowing-malicious-actors-to-steal-tokens-code4rena-bakerfi-bakerfi-git
```





## Reference

https://scsfg.io/hackers/



