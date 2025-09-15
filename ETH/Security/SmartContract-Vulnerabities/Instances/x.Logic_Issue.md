# 1. 允许用户输入别人的账号导致的问题
# 2. typo off
1. eary return
[morpheus](https://code4rena.com/reports/2025-08-morpheus#m-04-yield-withdrawal-blocked-by-zero-reward-early-return)
函数的return写早了，导致后续的状态变量不会更新的情况。

其起因是`distributeRewards() public`函数会在最后更新`lastUnderlyingBalance`等状态变量，但是只有在reward != 0时才能更新。问题是`getPeriodRewards`只在特定时间发送reward，其余时间都是0，导致状态变量的不更新。

```solidity
uint256 rewards_ = IRewardPool(rewardPool).getPeriodRewards(
    rewardPoolIndex_,
    lastCalculatedTimestamp_,
    uint128(block.timestamp)
);
if (rewards_ == 0) return;

....更新逻辑
```
