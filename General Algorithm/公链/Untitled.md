公链架构：

1. **网络层（P2P）**

- 节点通过点对点协议相互发现与通信（gossip 协议常用）。
- 负责交易与区块的传播、节点发现、带宽控制与防攻击机制（如消息过滤、防DDoS）。

2. **数据层（区块与状态）**

- 区块：包含区块头（前区块哈希、Merkle root、时间戳、Nonce/高度等）与交易列表。
- 存储模型：
  - UTXO（比特币）：每笔输出可被完全花费，适合简单货币模型。
  - 账户模型（以太坊）：全局账户状态，方便合约与余额操作。
- 状态证明：Merkle / Patricia Merkle 树，用于高效的状态/交易证明。

3. **共识层**

- 解决在不可信网络中“谁来打包/确定区块”的问题。
- 两大类思路：
  - Nakamoto-style（PoW/某些PoS变体）：最终性概率性（多区块后认为确认）。
  - 经典BFT（Tendermint、HotStuff）：确定性最终性（一次达成多数同意即最终）。
- 常见算法：PoW、PoS（权益证明）、DPoS、BFT（PBFT、Tendermint）、以及 DAG 等。

4. **交易池（mempool）**

- 接受未上链交易，节点进行初步验证（签名、nonce、费用等），按策略传播与打包。

5. **执行/合约层**

- 执行机（虚拟机）负责对交易进行状态变更：例如 EVM、WASM、原生程序运行环境。
- 智能合约、交易脚本在此执行；要求**确定性**（相同输入不同节点应得同样输出）。

6. **对外接口与生态**

- RPC/API：节点提供 JSON-RPC、gRPC 等供钱包、链上服务、区块浏览器调用。
- 节点类型：全节点、归档节点（archive）、轻客户端（light）、验证节点（validator/miner）。
- 工具链：钱包、浏览器、索引器（The Graph 类）、或acles、跨链桥、Layer-2。



差异方向：

**数据模型**：Bitcoin（UTXO） vs Ethereum（Account）。

**执行环境**：EVM（以太） vs WASM（许多新链） vs 原生高性能运行时（Solana）。

**共识与最终性**：PoW（比特币，概率性） vs PoS/BFT（Cosmos/Tendermint，确定性）。

**扩容方案**：链内扩容（更大区块、并行执行） vs 链外（Layer-2：Rollups、State Channels）。