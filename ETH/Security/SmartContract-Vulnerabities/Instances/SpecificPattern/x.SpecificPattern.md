[TOC]



某些项目中使用了一些特定的系统模式，这些特定的模式常常会产生特定的bug。

比如多段式执行一个特定的任务；"用户A发出deposit/withdraw请求，但是这个请求会被keeper或者admin来执行，并非调用deposit/withdraw后直接执行." 这种模式就得考虑如果有cancel/expired request或者被特定的输入卡住request（比如输入一个USDC backlisted address作为接收者）会怎么样。

1. **Multi-Step**:通常这些用户请求的任务会在后续异步执行，而不是直接请求直接执行的任务

2. **State Machine**：涉及了全局/部分的状态，使用enum来管理这些状态的情况。

3. **DAO**

4. **Batching Process**:批量执行

5. **Cross-Chain**

7. **Liquidation：**杠杆/保证金模式下清算的漏洞（稳定币/借贷/杠杆理财项目中，关于资产不抵债情况的例子）







### Multi-Step

异步执行常常出现在以下的项目中：

- DEX
- 



1. 是否能大量地输入垃圾请求造成DoS？
2. 是否可以使得请求/执行前后资产价格不一致来套利？
3. 





### Batching Process


我们之前讨论的批量处理模式（Batch Processing）确实有一些特有的漏洞。以下是一些典型的漏洞类型：

1. **原子性破坏**：在批量操作中，如果其中一个操作失败，整个交易应该回滚。但有时设计不当，可能导致部分操作成功，部分失败，从而破坏原子性。
2. **排序依赖**：批量操作中的顺序可能影响最终状态。如果顺序可以被操纵，或者顺序依赖的状态在操作之间被改变，可能导致非预期结果。
3. **Gas限制与循环**：批量操作通常涉及循环，如果循环次数过多，可能达到Gas限制，导致整个交易失败。此外，如果循环中的每个操作消耗Gas不固定，可能难以预估Gas。
4. **前端运行**：由于批量操作可能包含多个步骤，攻击者可能在操作之间插入自己的交易，从而获利。
5. **重复处理**：如果没有适当的防止重复的机制，同一个操作可能被多次执行。
6. **权限扩散**：在批量操作中，可能一次操作涉及多个用户，需要确保每个子操作都通过了适当的权限检查。
7. **状态不一致**：批量操作可能读取同一状态多次，如果在操作过程中状态发生变化，可能导致不一致。
8. msg.value的漏洞





### Liquidation

mint中会检查`_checkUnderCollateralized`全局债务

```
攻击者在市场早期执行：
1. 作为首批流动性提供者，存入少量baseToken
2. 获得aToken(杠杆代币)和zToken(债务代币)
3. 将大部分aToken通过swap换成zToken
   → 结果：攻击者持有不成比例的大量zToken(债务)
   一旦市场下跌就会导致mint被`_checkUnderCollateralized`所卡住，从而使得其他人无法提供流动性
```

[Early market can be DoSed by over-minting debt](https://github.com/pashov/audits/blob/master/team/md/Covenant-security-review_2025-08-18.md#m-05-early-market-can-be-dosed-by-over-minting-debt)
