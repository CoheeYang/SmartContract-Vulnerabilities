# Context

智能合约中的51% attack一般出现在存在vote的DAO相关协议中。这种攻击的共同特点是当假设有人持有51%的voting power后，它们提出一个对他们有利的proposal，再通过投票决议通过，使得它们利用这些proposal进行一些对它们有利的call。



| No. &Severity | Description                                                  | Link                                                         | Platform  |
| ------------- | ------------------------------------------------------------ | ------------------------------------------------------------ | --------- |
| 1-H           | 持有51%投票权后通过再铸造超级投票权，以通过那些需要几乎100%投票权才能完成的proposal | [Party-Protocal](https://solodit.cyfrin.io/issues/h-01-the-51-majority-can-hijack-the-partys-precious-tokens-through-an-arbitrary-call-proposal-if-the-addpartycardsauthority-contract-is-added-as-an-authority-in-the-party-code4rena-party-protocol-party-protocol-git) | Code4Rena |
| 2-H           | ERC20作为NFT持有股权，通过建立任意proposal将其所有的ERC20转给attacker，之后attacker拿这些ERC20换整个NFT | [PartyDAO](https://solodit.cyfrin.io/issues/h-06-a-majority-attack-can-steal-precious-nft-from-the-party-by-crafting-and-chaining-two-proposals-code4rena-partydao-partydao-contest-git) | Code4Rena |
| 3-M           | lack zero address check 导致潜在的51% attack                 | [Veto]( https://solodit.cyfrin.io/issues/m-11-loss-of-veto-power-can-lead-to-51-attack-code4rena-nouns-builder-nouns-builder-contest-git) | Code4Rena |













