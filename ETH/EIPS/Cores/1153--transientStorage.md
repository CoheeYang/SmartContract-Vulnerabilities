[EIP-1153: Transient storage opcodes](https://eips.ethereum.org/EIPS/eip-1153)

[Moody Salem - Applications of Transient Storage (EIP-1153)](https://www.youtube.com/watch?v=xFp8RlRq0qU)

[base class for the "till" pattern · Issue #4361 · OpenZeppelin/openzeppelin-contracts](https://github.com/OpenZeppelin/openzeppelin-contracts/issues/4361#issuecomment-1595095135)

# Till pattern

如果用户需要在你的协议中进行资产买卖：

1. Protocal 1.0时期

```bash
customer wants to buy 2 items and sell 1 item
 call #buy(item NO.1)
 call #buy(item NO.2)
 call #sell(itme No.3)
```

缺点：多笔call，导致多笔基础费用浪费gas和时间

2. Protocal 2.0时期

```bash
call #buy([item NO.1, item NO.2])
call #sell([item NO.3])
```

解决了多次call的毛病，将交易允许用array打包

缺点：如果用户想卖掉item3的钱来买item 1&2怎么办，同时还是有多笔sell和buy的交易分离导致gas问题



3. protocal 3.0

```bash
call #executeOrder([buy1,buy2,sell3])
-先transfer item3拿到payement
-之后直接算净额结算资金，以避免多笔交易
```

缺点：如果用户需要条件订单怎么办？比如某一个价格我执行buy/sell，某一个价格我只进行sell这样的。此类的协议就无法满足用户的定制化需求



4. protocal 4.0 --- till pattern

```bash
使用till pattern，即直到用户所有的callback结束后才进行所有的检查
 Customer calls Market#lock
 Market calls Customer#lockCallBack
 - customer calls Market#buy(1) if condition is met
 - customer calls Market#buy(2) if condition is met
 - customer calls Market#sell(3) if condition is met
 - 用户可以在自定义条件时call任何函数
 期间用transient记录一个需要支付的差额和资产
 Customer calls Market #pay{value:difference}()
 Customer calls #transfer(item3)
 
 最后合约只需要检查invarient是否被满足即可
 Market checks the balance is settled
 
```





