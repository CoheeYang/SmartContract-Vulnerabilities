

由于多段执行任务导致的bug



## 1. 允许用户输入别人的账号导致的问题

impersonate other people

[Etherspot-GasTankPaymasterModule](https://github.com/shieldify-security/audits-portfolio-md/blob/main/Etherspot-GasTankPaymasterModule-Extended-Security-Review.md#h-01-sessionkey-owner-can-impersonate-another-session-key-owner-for-the-same-smart-wallet)



## 2. typo off

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



gaspay是垫付的，而用户可以在执行结束后withdraw其资金

[Etherspot-GasTankPaymasterModule-H3](https://github.com/shieldify-security/audits-portfolio-md/blob/main/Etherspot-GasTankPaymasterModule-Extended-Security-Review.md#h-03-users-can-escape-paying-for-the-tx-gas)



## 3. Parallel Data

https://github.com/trailofbits/publications/blob/master/reviews/2025-10-radiustechnology-evmauth-securityreview.pdf

这个项目就是经典的多个平行数据



## 4. Missing checks

steal approval amount



missing duplicate token/user/address/ checks: [audits-portfolio-md/Etherspot-GasTankPaymasterModule-Extended-Security-Review.md at main · shieldify-security/audits-portfolio-md](https://github.com/shieldify-security/audits-portfolio-md/blob/main/Etherspot-GasTankPaymasterModule-Extended-Security-Review.md#m-01-session-with-duplicate-tokens-can-not-be-settled)

missing condition checks 导致读取旧数据

[audits-portfolio-md/Etherspot-GasTankPaymasterModule-Extended-Security-Review.md at main · shieldify-security/audits-portfolio-md](https://github.com/shieldify-security/audits-portfolio-md/blob/main/Etherspot-GasTankPaymasterModule-Extended-Security-Review.md#m-03-solver-offboarding-can-block-invoice-settlement)



Restriction Bypass类型：

Redeem cap bypassed using `swap()`，赎回的部分有cap检查，但是swap却忘了这个检查

[Covenant-security-review](https://github.com/pashov/audits/blob/master/team/md/Covenant-security-review_2025-08-18.md#m-04-redeem-cap-bypassed-using-swap)



## 5. Edge Case

极端的输入值

[audits-portfolio-md/Etherspot-GasTankPaymasterModule-Extended-Security-Review.md at main · shieldify-security/audits-portfolio-md](https://github.com/shieldify-security/audits-portfolio-md/blob/main/Etherspot-GasTankPaymasterModule-Extended-Security-Review.md#m-02-solver-can-receive-zero-settlement-when-fee-exceeds-token-amount)



极端的情况

比如solver下线了，但是这里的settlement却需要一个activeSolver。类似的可以是比如某一个vault被移除了，但是用户存入的钱时，取出的函数却需要active Vault类似，导致vault中的钱无法取出。无法成功退出

[audits-portfolio-md/Etherspot-GasTankPaymasterModule-Extended-Security-Review.md at main · shieldify-security/audits-portfolio-md](https://github.com/shieldify-security/audits-portfolio-md/blob/main/Etherspot-GasTankPaymasterModule-Extended-Security-Review.md#m-03-solver-offboarding-can-block-invoice-settlement)



