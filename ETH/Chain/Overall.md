[TOC]



# EVM

- **定义：**Ethereum Virtual Machine (EVM)是一个去中心化的虚拟环境（RunTime Environment），每个节点在EVM环境中运行合约；除了简单的账户之间转账之外，所有的计算任务都会EVM中进行。而gas则是其每次运行时的算力指标，以避免机器会在一个简单的While true中永远运行下去。

- **对比：**EVM和其他如.net或者JVM一样，都是为了满足跨平台需求的虚拟的状态机；Solidity和Vyper(python)等编程语言将代码编译特定的opcode，之后EVM会执行这些计算任务从而改变链上的状态。

- **单线程：**EVM不具备调度能力，这意味着它无法自主决定智能合约代码的执行顺序和时机，因为其执行顺序完全由外部的以太坊客户端来安排：客户端会按照已验证的区块交易顺序，确定哪些智能合约需要执行以及执行的先后次序。从这一角度来看，以太坊类似于 JavaScript，是单线程运行的，所有计算任务的调度逻辑都在 EVM 之外处理；这种架构保证了状态机和网络的稳定和一致性，但是又反过来限制了链上TPS的上限。



# Client

**以太坊的客户端: **An <u>implementation</u> of Ethereum that **verifies** **data** against the **protocol rules** and keeps the network secure

**客户端类别：**自从PoS后，一个节点需要运行两个客户端

- Execution Clien
- Consensus Client 

**客户端实现**：以太坊的客户端有多种语言多个团队所创建的多种实现([see clients list](https://ethereum.org/en/developers/docs/nodes-and-clients/#execution-clients))，但他们都遵守相同的规范

- [execution-specs](https://github.com/ethereum/execution-specs/)
- [consensus-specs](https://github.com/ethereum/consensus-specs)
- [EIP](https://eips.ethereum.org/) implemented in various [network upgrades](https://ethereum.org/en/history/)



**节点模型**：

以太坊的节点构成以太坊的网络，但是节点也有不同的类型；它们通过使用不同类型的客户端而采用以下三种保存和验证区块的策略：

- **Full Node:** 下载并验证完整的交易区块和状态信息，这些下载的区块会被定期修剪（pruned）比如只保留最近的128个区块。
- **Archive Node:** 下载并验证所有区块和历史状态，不会被修剪，通常是区块链浏览器或者钱包提供商运营这类节点客户端。
- **Light Node:** 只下载区块头，即每个区块的哈希值，来验证 hash(s1 +head)与下一个区块是否匹配；其他信息都从full node拿

运营上述的节点至少需要运营一个Execution Client？

**同步模型**：

为了验证当前的区块，客户端必须得同步最新的网络数据；同步的过程包括

1. 从peers中下载数据
2. 验证peers下载的数据
3. 构建本地区块链数据库





当用户发起一笔交易后，一笔交易通过rpc发送到execution client，此client将会先验证如签名是否正确/账户gas费是否足够，如果验证成功并在EVM中完成状态数的更新将会被gossip network发送给其他execution client的peers，此时此交易等待入块并会被放在memepool中，execution client也会收到其他节点的交易并同步到memepool等待出块，这些等待出块的交易通过engine api传输consensus client，之后consensus client中会根据算法选出validator，validator将memepool中交易打包出块，被打包出块的区块将在consensus client中gossip到其他的节点。当每一轮epoch结束后，其他的验证者将会进行验证并投票出正确的链，此链将会被finalize并成为以太坊上的一段数据。





# Validation

这里会以一个质押了ETH的人的视角来讲述staking的所有会涉及的流程以及专业名词

整个流程分为

1. Staking：如何将ETH存入
2. Earn and Exit：如何获得收益和质押退出机制。
3. Propose and Validation: 具体的validator工作流程



## Staking

### Validator

人类Operator在使用两个客户端的同时在consensus客户端加入validator client，一个validator client可以通过一个助记词生成多个key pairs同时，来控制多个validators。



而这个控制validator的key pair并不与EOA账户一致生成于`secp256k1`曲线；

而是每一个validator都有一个BLS密钥对，其公钥48字节私钥32字节，是信标链上的验证者主体的地址，由BLS12‑381曲线生成。

BLS12‑381生成的密钥有一个特点就是便于聚合，它可以将多个签名聚合成一个96 字节的聚合签名，验证时仍只需一次配对运算，这样就便于了共识中propse和attestations的操作。（[关于更多BLS的内容](https://eth2book.info/capella/part2/building_blocks/bls12-381/#curve-bls12-381)）



当你每次创建一个validator时都需要给执行链上`Deposit Contract`提交ETH，提交的同时需要明确`withdrawal_credentials`，这是一个32字节的输入，是每一个validator取款属性，用于告知beacon chain其取款类型信息。

<br/>

### Deposit Contract

`0x00000000219ab540356cBB839Cbe05303d7705Fa`是链上的stake接收地址(deposit Account)，staker通过将32个ETH转入来获得validator资格，

```solidity
   /// @notice Submit a Phase 0 DepositData object.
    /// @param pubkey A BLS12-381 public key.
    /// @param withdrawal_credentials Commitment to a public key for withdrawals.
    /// @param signature A BLS12-381 signature.
    /// @param deposit_data_root The SHA-256 hash of the SSZ-encoded DepositData object.
    /// Used as a protection against malformed input.
    function deposit(
        bytes calldata pubkey,
        bytes calldata withdrawal_credentials,
        bytes calldata signature,
        bytes32 deposit_data_root
    ) external payable;
```

当使用如`https://launchpad.ethereum.org/`进行最后的链上交易时，会调用这个合约的`deposit`存入32ETH，其中要求输入的参数为：

1. **pubkey：**当前validator用于后续签名处理出块事务时所用的BLS公钥
2. **withdrawal_credentials：**一个validator 账户类型的prefix（00/01/02）+（00填充）+（取款accounts公钥）的32字节拼接，对于00类BLS账户，会取00+hash(bls_withdrawal_pubkey)[1:]来填充（[see detail](https://ethereum.github.io/consensus-specs/specs/phase0/validator/#withdrawal-credentials)）。
3. **signature**：BLS12-381的签名
4. **deposit_data_root**：DepositData（`pubkey` + `withdrawal_credentials` + `depositAmount(gwei)` + `signature`）通过SSZ序列化后的SHA-256哈希值，**SSZ（Simple Serialize）** 是信标链为共识数据结构引入的一种序列化格式。



值得注意的是，如果使用的是00类的BLS账户作为`withdrawal_credentials`，对应输入的`pubkey`和此`withdrawal_credentials`**可以是不同的BLS公钥**，这是因为`pubkey`作为validator的信标链上对应的地址，会需要频繁给验证和投票等出块事务自动签名，一直都是保持`hot`。

而`withdrawal_credentials`对应的公钥只需要执行更改`withdraw_credentials`所对应EOA（CA）账户的行为的签名，这样就降低了密钥泄露的安全风险。

下面我们会讲这个修改`withdraw_credentials`的原因--validator类型的内容。

<br/>

### validator类型

*Pectra* 更新后有了三种validator类型，分冠以00/01/02的prefix来识别；validator account类型是由你在和deposit contract中输入的32字节`withdrawal_credentials`时所决定的。



总的来说`withdrawal_credentials`的格式只有两种

- **以00为prefix+BLS公钥的组合：**

​	A BLS withdrawal credential is the 32-byte hash of a 48-byte BLS public key, with the first byte replaced by `0x00`.

```bash
0x00_f50428677c60f997aadeab24aabf7fceaef491c96a52b463ae91f95611cf71
```

- **以01/02为prefix的+EOA/CA账户的组合**

  An Eth1 (execution) withdrawal credential has the [prefix](https://eth2book.info/capella/part3/config/constants/#withdrawal-prefixes) `0x01/02`, followed by 11 zero bytes, followed by the 20 bytes of a normal Ethereum address. That address is where all Ether from withdrawals will be sent.

```bash
0x01_00000000000000000000000_d369bb49efa5100fd3b86a9f828c55da04d2d50
```



而具体而言，不同的`withdrawal_credentials`产生三种类型的账户，对应不同的取款能力：

- **Type0**：

  又叫BLS账户/locked账户，当你输入此类账户，质押所产生的钱将会被锁定在Beacon chain上值到提供一个以太坊账户（`execution withdrawal address`）来转为下面两种validator类型（一旦提供则不可更改链上取款地址）。此时validator账户是由BLS key-pair所控制的。

- **Type1 (regular withdraw)**：

  正常的32ETH质押账户，当你的质押余额超过32ETH后，便会定期地将多出的余额发送给你的链上收款账户；此时你的validator账户是由此收款地址所控制。

- **Type2 (compounding)**：

  不同于前二者，此账户可以保留余额来复利，任意32-2048的ETH都会被接留在beacon chain上，但是用户也可以要求取出32余额之上的ETH到链上账户，此账户也被收款账户所控制。



## Earn and Exit

上述内容我们说过，对于01/02账户，超过了最大额度就会自动withdraw到指定的EOA/CA账户上，

每个区块最多有16个validator进行withdraw，从index为0的validator开始轮询，满16个为止，之后下一个区块按上一个区块的停止点继续，以此往复。

https://eth2book.info/capella/part2/deposits-withdrawals/withdrawal-processing/#withdrawal-processing



## Propose and Validate





## Reference

https://eth2book.info/capella/part2/deposits-withdrawals/staking/

https://ethereum.github.io/consensus-specs/specs/phase0/validator/


https://launchpad.ethereum.org/en/faq









# Beacon chain



https://beaconscan.com/

https://beaconcha.in/











## Reference

1. [ethereumbook](https://github.com/ethereumbook/ethereumbook/blob/develop/13evm.asciidoc)
2. [关于 EVM](https://www.evm.codes/about)
3. [The EVM From Scratch Book](https://www.evm-from-scratch.app/content/01_intro) ：使用python来写出一个简单的EVM，来实现各种opcode的执行
4. [Noxx EVM 深入研究](https://noxx.substack.com/p/evm-deep-dives-the-path-to-shadowy)
5. [Solidity 插槽数据解析](https://ethdebug.github.io/solidity-data-representation/)
6. [以太坊黄皮书](https://ethereum.github.io/yellowpaper/paper.pdf)
7. [beigepaper](https://github.com/chronaeon/beigepaper)
8. [Quillhash EVM Mastery](https://github.com/Quillhash/EVM-Mastery)
9. [EVM 实现示例](https://github.com/noxx3xxon/evm-by-example)
10. [PBS(Proposer-Builder Separation架构)](https://ethereum.org/en/roadmap/pbs/):
11. [Danksharding](https://www.ethereum.cn/ETh2/epochs-slots-and-beacon-blocks)









# DA

**Data availability(DA)**: DA的概念源于
