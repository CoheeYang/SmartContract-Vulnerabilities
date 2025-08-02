[TOC]



# EVM虚拟机介绍

在虚拟机中，主要有三种虚拟机架构：

- Register Based Virtual Machines（寄存器）

- Stack Based Virtual Machines（堆栈）
- 混合架构（如**WebAssembly**）

而EVM是Stack-Based虚拟机，Solana则是基于寄存器的rBPF虚拟机运行的。**栈式虚拟机**胜在**简单性**和**可移植性**，适合快速实现和内存敏感场景。**寄存器型虚拟机**赢在**性能**和**优化潜力**，适合高性能计算和底层系统。





# Transient Storage

cancun的更新使得EIP-1153得到落地，此提议为EVM增加了瞬态存储的存储地址，比较三种存储位置：

1. **Memory**：临时只在一次调用（call frame）中存在；
2. **Storage**：永久跨交易存在；
    EIP‑1153 增加了第三种：
3. **Transient Storage**：在一次**交易**（transaction）中跨所有调用都可访问，到交易结束即清零。 [docs.chainstack.com](https://docs.chainstack.com/docs/ethereum-dencun-rundown-with-examples?utm_source=chatgpt.com)

**生命周期**

- **创建时机**：在 EVM 开始处理一次 state transition（交易执行）时初始化；
- **清理时机**：交易完全执行结束后，瞬态存储的所有内容都会被丢弃，不会写回链上。

**使用场景**

- 记录“跨内部函数调用的中间状态”或“防重入标志”之类的临时变量；
- 不必为一次性数据付出 `SSTORE` 的高 gas 开销，又比 `memory` 更长久（memory 只在单个函数里存活）。

比如新的OZ合约中的重入锁可以这样写：

```solidity
pragma solidity ^0.8.28;

contract TReentrant {
    mapping(address => bool) claimed;
    bool transient locked;//原来得在storage中存储，浪费空间和gas

    modifier nonReentrant {
        require(!locked, "Reentrancy attempt");

        locked = true;
        _;
        locked = false;
    }
    
     function claim() nonReentrant public {
        require(!claimed[msg.sender], "Already claimed");
        claimed[msg.sender] = true;
    }
    
}
```

对于assembly下面是一个Lido中人为创建一个hash map的使用案例：
```solidity
function create() internal returns (TransientUintUintMap self) {
    // 1. 预先计算好的“锚点”槽位（anchor）
    uint256 anchor = 0x6e38e7eaa4307e6ee6c66720337876ca65012869fbef035f57219354c1728400;

    assembly ("memory-safe") {
        // 2. 从瞬态存储中读取上一次创建的“地址”（prev）
        let prev := tload(anchor)

        // 3. 把 anchor 和 prev 拼到内存 [0x00 .. 0x40)
        mstore(0x00, anchor)  // 前 32 字节
        mstore(0x20, prev)    // 后 32 字节

        // 4. 对这 64 字节做 keccak256，得到新的“地址” self
        self := keccak256(0x00, 0x40)

        // 5. 在anchor的瞬时槽下存入这个self哈希
        tstore(anchor, self)
    }
}
```



# EOF

EOF是pectra升级中对EVM最重大的升级内容，包含了11个EIP落地：

- [EIP-3540](https://github.com/ethereum/EIPs/blob/master/EIPS/eip-3540.md): EOF — EVM Object Format v1
- [EIP-3670](https://github.com/ethereum/EIPs/blob/master/EIPS/eip-3670.md): EOF — Code Validation
- [EIP-4200](https://github.com/ethereum/EIPs/blob/master/EIPS/eip-4200.md): EOF — Static relative jumps
- [EIP-4750](https://github.com/ethereum/EIPs/blob/master/EIPS/eip-4750.md): EOF — Functions
- [EIP-5450](https://github.com/ethereum/EIPs/blob/master/EIPS/eip-5450.md): EOF — Stack Validation
- [EIP-663](https://github.com/ethereum/EIPs/blob/master/EIPS/eip-663.md): SWAPN, DUPN and EXCHANGE instructions
- [EIP-6206](https://github.com/ethereum/EIPs/blob/master/EIPS/eip-6206.md): EOF — JUMPF and non-returning functions
- [EIP-7069](https://github.com/ethereum/EIPs/blob/master/EIPS/eip-7069.md): Revamped CALL instructions
- [EIP-7480](https://github.com/ethereum/EIPs/blob/master/EIPS/eip-7480.md): EOF — Data section access instructions
- [EIP-7620](https://github.com/ethereum/EIPs/blob/master/EIPS/eip-7620.md): EOF Contract Creation
- [EIP-7698](https://github.com/ethereum/EIPs/blob/master/EIPS/eip-7698.md): EOF — Creation transaction
- 



# Reference

https://www.evm.codes/about

https://blog.blockmagnates.com/ethereum-object-format-what-s-new-a933c9d20afd

