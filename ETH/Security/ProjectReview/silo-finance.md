# Silo









Avae，compound由于将所有的token都放在一个pool里面，所以所能借贷的和抵押的token其实会被严格险入而防止资产进入。而这种监管导致部分token被排除在外，没有借贷市场。

为了解决这个问题，Silo提出了 permissionless, isolated lending markets

核心机制：

- **风险隔离设计**：
  Silo 将每个借贷池（称为“筒仓”）独立化，每个筒仓仅包含两种资产：**一种目标代币**（如 USDC、wBTC）和 **桥接资产**（如 ETH 或 SiloDollar）。例如，USDC 借贷池仅允许用户存入 USDC 并借出 ETH，反之亦然。这种设计将单一资产的风险限制在对应筒仓内，避免系统性崩盘45。

- **桥接资产的角色**：
  所有跨资产借贷需通过桥接资产完成。例如，用户想存入代币 A 借取代币 B，需分两步操作：

  1. 存入代币 A → 借出桥接资产（如 ETH）；

  2. 存入 ETH → 借取代币 B。
     这种机制虽降低了资金利用率（最高约 25%），但增强了风险控制。

     

Silo Finance is a non-custodial money protocol that implements permissionless, isolated lending markets, known as silos. Silo Vault serves as the managed liquidity layer, channeling liquidity into Silo’s lending markets (silos) or any 4626 lending vaults.

- **Documentation**:
  - https://docs.morpho.org/morpho-vaults/overview
  - https://silo-finance.gitbook.io/private-space/silo-vaults

- **Where to Foucs**
  - Forked code: SiloVault is a fork of <u>MorphoVault</u>. Original code only supported deploying capital to Morpho protocol while SiloVault was changed to support any ERC4626 vault and highly configurable rewards. Admin role stayed unchanged. New admin function were added to manage rewards setup. It is a good start to make sure that code integrity is kept and no bugs were introduced during forking.
  - Rewards: SiloVault supports custom rewards distribution. It is important that rewards do not compromise the SiloVault logic in any way and are distributed correctly.



代码和狗屎一样



| File                                                         | nSLOC    |
| ------------------------------------------------------------ | -------- |
| /silo-vaults/contracts/IdleVault.sol                         | 33       |
| /silo-vaults/contracts/IdleVaultsFactory.sol                 | 22       |
| /silo-vaults/contracts/PublicAllocator.sol                   | 90       |
| /silo-vaults/contracts/SiloVault.sol                         | 554      |
| /silo-vaults/contracts/SiloVaultsFactory.sol                 | 37       |
| /silo-vaults/contracts/incentives/VaultIncentivesModule.sol  | 106      |
| /silo-vaults/contracts/incentives/claiming-logics/SiloIncentivesControllerCL.sol | 25       |
| /silo-vaults/contracts/incentives/claiming-logics/SiloIncentivesControllerCLFactory.sol | 11       |
| /silo-vaults/contracts/interfaces/IIncentivesClaimingLogic.sol | 5        |
| /silo-vaults/contracts/interfaces/INotificationReceiver.sol  | 3        |
| /silo-vaults/contracts/interfaces/IPublicAllocator.sol       | 20       |
| /silo-vaults/contracts/interfaces/ISiloIncentivesControllerCLFactory.sol | 5        |
| /silo-vaults/contracts/interfaces/ISiloVault.sol             | 15       |
| /silo-vaults/contracts/interfaces/ISiloVaultsFactory.sol     | 4        |
| /silo-vaults/contracts/interfaces/IVaultIncentivesModule.sol | 23       |
| /silo-vaults/contracts/libraries/ConstantsLib.sol            | 7        |
| /silo-vaults/contracts/libraries/ErrorsLib.sol               | 52       |
| /silo-vaults/contracts/libraries/EventsLib.sol               | 64       |
| /silo-vaults/contracts/libraries/PendingLib.sol              | 24       |
| /silo-vaults/contracts/libraries/SiloVaultActionsLib.sol     | 43       |
| /silo-core/contracts/incentives/SiloIncentivesController.sol | 78       |
| /silo-core/contracts/incentives/SiloIncentivesControllerFactory.sol | 11       |
| /silo-core/contracts/incentives/SiloIncentivesControllerGaugeLike.sol | 44       |
| /silo-core/contracts/incentives/SiloIncentivesControllerGaugeLikeFactory.sol | 11       |
| /silo-core/contracts/incentives/base/BaseIncentivesController.sol | 159      |
| /silo-core/contracts/incentives/base/DistributionManager.sol | 162      |
| /silo-core/contracts/incentives/interfaces/IDistributionManager.sol | 34       |
| /silo-core/contracts/incentives/interfaces/ISiloIncentivesController.sol | 28       |
| /silo-core/contracts/incentives/interfaces/ISiloIncentivesControllerFactory.sol | 4        |
| /silo-core/contracts/incentives/interfaces/ISiloIncentivesControllerGaugeLikeFactory.sol | 4        |
| /silo-core/contracts/incentives/lib/DistributionTypes.sol    | 19       |
| **Totals**                                                   | **1697** |







vault incentives 

### 激励分配类型

Silo vault支持两种激励分配类型：即时分配（收到的激励可立即提取）和其他类型（如基于时间的激励控制器或计量器gauge，或两者兼具 ）。

其中vault incentives module是解决激励分配的关键

- **Vault incentives module**:由list of incentives claiming and distribution *<u>logics&solutions</u>*组成
  - logics: 每个vault market可拥有0，1或者更多个相关逻辑
  - solutions:Solution 由Incentives controller和Gauge组成，controller会处理不同的激励计划。Vault incentives module运行多种solutions，同时silo-vault会通知share token balance change？

<br/>

每个在silo-vault中的有效市场都可以有不同的激励计划，这种不同不光是说采用的solution的模块的不同，也包括了激励的起始和终止时间。为了满足这种需求，每个market都有可以自主选择的claming logcs(在一个独立的合约中实现)。

Vault通过Delegate call 



每个可用的市场可能有不同的激励计划。这些计划不仅在使用的解决方案上有差异，在起始和结束日期等方面也不同。这就要求能从新来源申领激励，或停止从已结束的激励计划申领。每个市场都有可配置的奖励申领逻辑，通过委托调用在单独的智能合约中执行。文档以 Market A、Market B 等为例，展示了不同市场对应的激励申领逻辑。

### 激励申领和分配逻辑执行（图 3）

- Silo 智能合约包含 Vault 和 SiloIncentivesController 。Vault 通过委托调用（delegateCall）到 incentives module 执行申领逻辑，incentives module 包含各个市场的激励申领逻辑。
- 第三方智能合约包括 Gauge、Merkl 和 Aave incentives controller 等 。激励申领逻辑通过特定功能从 SiloIncentivesController 进行即时激励分配。









为什么有些echidna的fuzzing要拿receivable?





奖励机制

用户应该是给了钱，但是给哪个合约？

`Silo.sol`继承`ShareCollateralToken.sol`(也继承`shareToken.sol`)，它们有`_afterTokenTransfer(address _sender, address _recipient, uint256 _amount)`函数被使用，这个函数会传给`GaugeHookReciever.sol::afterAction()`，之后这个函数会调起`Gauage`的afterAction。





Invariant：

对于incentivePlan来说，由于没有增长的地方，同时也没有手续费

我为自己deposit了多少，得了多少share，最后通过相同的amount赎回多少









## Report

缺乏share处理的逻辑

在用户对Silo进行deposit后，用户的assetToken会被转到Silo合约中，同时也会被Mint获得相关的shareToken。动作完成后调用`GaugeHookReceiver::afterAction()`来调用incentive中的GaugeLike的`afterTokenTransfer()`。



```solidity
    /**
     * @dev Called by the corresponding asset on any update that affects the rewards distribution
     * @param _incentivesProgramId The id of the incentives program being updated
     * @param _user The address of the user
     * @param _totalSupply The total supply of the asset in the lending pool
     * @param _userBalance The balance of the user of the asset in the lending pool
     */
    function _handleAction(
        bytes32 _incentivesProgramId,
        address _user,
        uint256 _totalSupply,
        uint256 _userBalance
    ) internal virtual {
        uint256 accruedRewards = _updateUserAssetInternal(_incentivesProgramId, _user, _userBalance, _totalSupply);

        if (accruedRewards != 0) {
            uint256 newUnclaimedRewards = _usersUnclaimedRewards[_user][_incentivesProgramId] + accruedRewards;
            _usersUnclaimedRewards[_user][_incentivesProgramId] = newUnclaimedRewards;

            emit RewardsAccrued(
                _user,
                incentivesPrograms[_incentivesProgramId].rewardToken,
                getProgramName(_incentivesProgramId),
                newUnclaimedRewards
            );
        }
    }
    
    
        /**
     * @dev Updates the state of an user in a distribution
     * @param incentivesProgramId The id of the incentives program being updated
     * @param user The user's address
     * @param stakedByUser Amount of tokens staked by the user in the distribution at the moment
     * @param totalStaked Total tokens staked in the distribution
     * @return The accrued rewards for the user until the moment
     */
    function _updateUserAssetInternal(
        bytes32 incentivesProgramId,
        address user,
        uint256 stakedByUser,
        uint256 totalStaked
    ) internal virtual returns (uint256) {
        uint256 userIndex = incentivesPrograms[incentivesProgramId].users[user];
        uint256 accruedRewards = 0;

        uint256 newIndex = _updateAssetStateInternal(incentivesProgramId, totalStaked);

        if (userIndex != newIndex) {
            if (stakedByUser != 0) {
                accruedRewards = _getRewards(stakedByUser, newIndex, userIndex);
            }

            incentivesPrograms[incentivesProgramId].users[user] = newIndex;

            emit UserIndexUpdated(user, getProgramName(incentivesProgramId), newIndex);
        }

        return accruedRewards;
    }
    
    
    
    
       /**
     * @dev Updates the state of one distribution, mainly rewards index and timestamp
     * @param incentivesProgramId The id of the incentives program being updated
     * @param totalStaked Current total of staked assets for this distribution
     * @return The new distribution index
     */
    function _updateAssetStateInternal(
        bytes32 incentivesProgramId,
        uint256 totalStaked
    ) internal virtual returns (uint256) {
        uint256 oldIndex = incentivesPrograms[incentivesProgramId].index;
        uint256 emissionPerSecond = incentivesPrograms[incentivesProgramId].emissionPerSecond;
        uint256 lastUpdateTimestamp = incentivesPrograms[incentivesProgramId].lastUpdateTimestamp;
        uint256 distributionEnd = incentivesPrograms[incentivesProgramId].distributionEnd;

        if (block.timestamp == lastUpdateTimestamp) {
            return oldIndex;
        }

        uint256 newIndex = _getIncentivesProgramIndex(
            oldIndex,
            emissionPerSecond,
            lastUpdateTimestamp,
            distributionEnd,
            totalStaked
        );

        if (newIndex != oldIndex) {
            incentivesPrograms[incentivesProgramId].index = newIndex;
            incentivesPrograms[incentivesProgramId].lastUpdateTimestamp = uint40(block.timestamp);

            emit IncentivesProgramIndexUpdated(getProgramName(incentivesProgramId), newIndex);
        } else {
            incentivesPrograms[incentivesProgramId].lastUpdateTimestamp = uint40(block.timestamp);
        }

        return newIndex;
    }
    
    
    
    
    
    

```

我一开始之模仿一部分交互的方法实在有点问题，但是到底是哪里出了错误，让我在使用这种方法后浪费了这么多时间







Notifer是一个接受deposit的合约，当他接受deposit，会根据call afterTokenTransfer，填入recipient的信息

如果有人从这个vault中withdraw任何流动性，也会call afterTokenTransfer，填入







当有时间条件的时候就变得非常麻烦



amountToClaim一直是0，

如何增加amountToClaim?

来源有两个

1. _usersUnclaimedRewards 的mapping被增加

2. _accrueRewardsForPrograms(msg.sender, programIds)得到的accrudedReward不为0



第一个`_usersUnclaimedRewards`是在`_handleAction`中`_updateUserAssetInternal`输出的`accruedInterest`不为0时增加

`_updateUserAssetInternal`中又套用另外一个`_updateAssetStateInternal(incentivesProgramId, totalStaked)`输出newIndex

只有newIndex不为原来的Index，函数才会对第一个输出一个`accruedInterest`

最后发现关键是改

`lastUpdateTimestamp == block.timestamp ||lastUpdateTimestamp >= distributionEnd`不成立才可以





Attack

When reward token is transfered to the Controller

Attack would can do following path to front run

1. deposit A big amount of token first to initialize the balance 
2. deposit 0 amount after that



currentTotalStake is 

100*10^18
