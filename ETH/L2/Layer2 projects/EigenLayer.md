# Intro

## EigenLayer

EigenLayer offers`Security as a Service`。

EigenLayer的主要面对客户有三类：

1. 资金散户
2. Operator（算力设备供应方）
3. AVS（Autonomous Verifiable Service），即应用开发者

它主要解决的痛点是：

- 散户质押后的staking token手中持有而无收益的问题
- Operator多余的设备没地方用
- AVS们寄希望创建off-chain service但是却需要面对宣发吸引散户和operator执行节点等高额成本

**实现手段：**

EigenLayer主要是通过了`restaking`的方法，吸引散户将ETH和如stETH的LST（Liquid Staking Token）再质押，这些质押的token会delegate一个用户指定的operator，operator节点会运行AVS的应用，同时AVS通过线上其自己部署的验证合约来验证operator们没有进行恶意行为。如果出现了作恶，operator的staked token会被slash.



**AVS类型**：

The scope of AVS design is broad. It includes **any off-chain service** that can be verified on-chain. 

 The only requirement is that some evidence for the off-chain service’s execution is posted on-chain to enable verification of the service.

Examples of these services include rollup services, co-processors, cryptography services, zk Proof services, and [more](https://docs.eigencloud.xyz/products/eigenlayer/developers/concepts/avs-developer-guide#what-can-you-build-as-an-avs).



## EigenLayer Middleware