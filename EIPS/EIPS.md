

- **[ERC7201](https://learnblockchain.cn/article/10350)**

  （可升级合约的槽位管理）由于多项继承的imp合约中的state variable可能在升级的后产生槽位的冲突而做出的namespace方法

Solidity存储数据的Location`L`可以是：

1. 根节点本身（`L=root`）
2. L加上任意一一个数值( `L=L+n` 即正常累加排列)
3. 对`L`值进行 keccak256 哈希运算的结果(`keccak256(L)`-动态数组，位置在L，数据在`keccak256(L)`，`keccak256(L+1)`...)
4. 对哈希值（键 k）与根节点进行异或运算结果再做 keccak256 哈希运算的结果(`keccak256(h(k). L)`--Mapping)。

由于root基本上是所有L计算的根本，如果想要避免冲突，就只需要改变起始的root的值不再是0即可。

solidity默认会将root设为0，





- **EIP712**