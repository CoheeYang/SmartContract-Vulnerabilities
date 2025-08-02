[TOC]

# 以太坊基本架构


自从The Merge之后，以太坊就将共识端（Consensus Layer,CL）和执行端（Execution Layer,EL）进行了分离，这种模块化的处理便于以太坊后续的升级与维护，而最直接的结果就是我们能看见Ethereum是由两个客户端组成：

- Consensus Client，也可以被称为beachon chain

- Execution Client

而两类客户端又由不同的组织通过不同的语言进行实现以保证Client Diversity，避免由于一个客户端的bug导致整个网络出现问题。而每个不同的Client均会根据其layer的类型遵循

- [execution-specs](https://github.com/ethereum/execution-specs/)：

  此repo是一个由Python所写的客户端，其src/ethereum目录下记录着各式版本的Execution Layer以供研究，

  同时提供了可读的HTML文档来了解其codebase和升级的diff ([rendered spec](https://ethereum.github.io/execution-specs/))。

- [consensus-specs](https://github.com/ethereum/consensus-specs)

  此repo也是由python所写，记录了从Beacon Chain从诞生以来的所有详细记录。








如果要从用户角度来看，一笔用户的交易会经历以下流程：

>当用户发起一笔交易后，一笔交易通过rpc发送到execution client，此client将会先验证如签名是否正确/账户gas费是否足够，如果验证成功并在EVM中完成状态数的更新将会被gossip network发送给其他execution client的peers，此时此交易等待入块并会被放在memepool中，execution client也会收到其他节点的交易并同步到memepool等待出块，这些等待出块的交易通过engine api传输consensus client，之后consensus client中会根据算法选出validator，validator将memepool中交易打包出块，被打包出块的区块将在consensus client中gossip到其他的节点。当每一轮epoch结束后，其他的验证者将会进行验证并投票出正确的链，此链在后面justified并在后续被finalize并成为以太坊上的一段数据。





# Nodes & Validator



## Reward

https://docs.lido.fi/#lido-on-ethereum-apr

https://ethereum.org/en/developers/docs/consensus-mechanisms/pos/rewards-and-penalties/






# Reference

以下是一些是学习以太坊整个架构的重要的参考资料：

- [EPF wiki](https://epf.wiki/#/eps/intro)

- [ETH2book](https://eth2book.info/latest/)

- [ethereum官方文档](https://ethereum.org/en/developers/docs/)

还有一些值得看的资料：

- [vitalik's note](https://vitalik.eth.limo/)