

[TOC]





# EIP 7002

**EIP‑7002** (“Execution layer triggerable withdrawals”) 的核心在于：

它为以太坊验证者增加了一条新通道，使得持有执行层（0x01）取款凭证的密钥（通常是冷钱包或 DAO 多签），可以 **直接发起退出（exit）和部分提现（partial withdrawal）** 操作，而无需依赖“活跃签名密钥”（active BLS key）本身去提交退出请求[Ethereum Improvement Proposals](https://eips.ethereum.org/EIPS/eip-7002?utm_source=chatgpt.com)。

- 之前的设计中，只有验证者的BLS密钥可以发起退出，造成在多方托管、密钥分离等场景下，取款凭证持有者无法独立控制流动性。

- 此提议在Pectra升级中落地

  





# **[ERC7201](https://learnblockchain.cn/article/10350)**

（可升级合约的槽位管理）由于多项继承的imp合约中的state variable可能在升级的后产生槽位的冲突而做出的namespace方法

Solidity存储数据的Location`L`可以是：

1. 根节点本身（`L=root`）
2. L加上任意一一个数值( `L=L+n` 即正常累加排列)
3. 对`L`值进行 keccak256 哈希运算的结果(`keccak256(L)`-动态数组，位置在L，数据在`keccak256(L)`，`keccak256(L+1)`...)
4. 对哈希值（键 k）与根节点进行异或运算结果再做 keccak256 哈希运算的结果(`keccak256(h(k). L)`--Mapping)。

由于root基本上是所有L计算的根本，如果想要避免冲突，就只需要改变起始的root的值不再是0即可。

solidity默认会将root设为0，但是我们可以使用struct来打包变量，并且只需要改变struct的存储位置，那么其余的变量位置也会一同改变。

这是由于在 Solidity 里，**struct 本身并不会自动给你分配一个固定的 slot**，它只是把一组成员按顺序、按打包规则，映射到「它所处的那个变量的起始 slot」往后的若干个 slot 上。只有当你**把一个 struct 当作合约的 state 变量**来声明时，编译器才会帮你把它分配到 slot 0、slot 1…这样的连续存储单元里

```solidity
contract A {
    struct S { uint x; uint y; }//声明struct类型并不会占用slot
    
    //只有声明一个struct变量才会占用slot
    S public data;   // data.x 在 slot 0，data.y 在 slot 1
    uint public z;   // z 在 slot 2
    …
}
```

这样，我们就能结合利用assembly的slot，来改变声明的struct的变量的slot，以改变起始的root，从而避免storage collision。

```solidity
   Struct StrategyData {....}
   
   function logicFunction () external {
        // Cache storage pointer.
        StrategyData storage S = _strategyStorage();//接收stora变量，并进行后续处理
        ....
        }
        
   function _strategyStorage() internal pure returns (StrategyData storage S) {
   //return中声明storage变量，同时assembly改变slot位置，返回该变量
        bytes32 slot = BASE_STRATEGY_STORAGE;
        assembly {
            S.slot := slot
        }
    }
    
    bytes32 internal constant BASE_STRATEGY_STORAGE =
        bytes32(uint256(keccak256("yearn.base.strategy.storage")) - 1);

```







# [ERC 7512](https://eips.ethereum.org/EIPS/eip-7512)

链上审计标准合约，披露合约审计的信息



# **EIP712**