

https://github.com/andreitoma8/learn-yul

[TOC]

# 基础

## 比特与字节

内存被分为一段段的"words",

每个''words'有32字节(bytes)，

32字节，1字节=8比特，即256bits，每个bit就是0或者1的二进制数。这也就是为什么会有`uint256`。

这32个字节，每个字节用**两个**16进制数（hex character）表示。也就是说内存中的数字字符一共其实有32x2，64个。

比如下面的：

`0x0000000000000000000000000000000000000000000000000000000000000028`

显示的是28，其十进制含义是2x16+8 =40。

在二进制中，28这个字节对应8比特应该是`0010_1000`。

> 怎么算的？
>
> 和十进制10^x一样，二进制就是几位当2^x，2^5是32，这里已经超过32，所以但没有超过64（2^6），还缺8，而第四位2（2^3）代表的就是8

<br/>

## 位运算 (Bitwise Operation)

**逻辑位运算：**

- 位与 (And)：`&`二进制中两个数对应位都为1时取1，否则则输出0。如`1010&1111 = 1010`
- 位或 (Or): `|`二进制中对应位的两个数有一个1则变1，若都为0则输出0。比如`1010|1111 = 1111`
- 异或(XOR): `^`，只有在两个数不同时采取1，否则为0。比如`1010^1111 =0101` （注意solidity中不能用^表示指数）
- 按位取反(Not): `~`，即1变0，0变1。比如`~1010 = 0101`

**位移运算：**

- 左移(<<): 将二进制数向左边移动，在yul中是`shl(n, x)`即将x左移n位。
  - 比如`32 << 1` 值为 64，就是将数据32（二进制中100000，十进制2^5，五个零）的位向左移动一个1位（这就变成了1000000，十进制2^6=64，6个零）
- 右移(>>): 将二进制数右移，yul中是`shr(n,x)`
  - 上述例子是`32>>1`值为16，即2^5变成2^4。



<br/>

## 二进制中的正负

计算机科学中，以一字节八位为例，有2^8这么多也就是换算十进制256个数可以储存。

如果数字是0-255，那么为区别正负，前一半算为正数，后一半算为负数。

此时0-255的bit中对应：

- 0-127是正数0-127，

- 128-255位则对应-128到-1。

这样在int8 `0000_0000`中，十进制2和正常的uint8一样，就是`0000_0010`

对于-2则需要让某个数加`0000_0010`为0即可。

我们可以得到-2其实就是`1111_1110`，正好和我们之前布置的-1为最高数`1111_1111`相符。

<br/>

如果我们要求一个负数的二进制数字，则求法为：

1. 找到对应正数的二进制数`0000_0010`（这叫原码）
2. 按位取反`1111_1101`（这叫反码）
3. 再加上1`1111_1110`（这叫补码）

或者之间最高位加上该负数的值转化为二进制

如：int8 =>256个数，-2就放在第256-2=254个数上，254转化二进制=>`1111_1110`





## 位运算应用

- **判断`int256`正负**

使用位与运算判断一个`int256`的正负：

```solidity
        uint256 constant TWO_COMPLEMENT_SIGN_MASK = 0x8000000000000000000000000000000000000000000000000000000000000000;
        
        assembly {
                //判断正负，mantissa是一个int256输入，
                //这个后面跟着的是掩码，16进制的800..(63个0)对应的是二进制1000...000(255个0，对应十进制2**255)
        		//如果mantissa的最大位为1，则为负，输出为1...0000
        		//在yul中，只要输出不是0000..000彻底的0，都会被认为是true。
                if and(mantissa, TWO_COMPLEMENT_SIGN_MASK) {
                	//负数情况
                    float := MANTISSA_SIGN_MASK
                    mantissa := sub(0, mantissa)
                }
            }
```

<br/>

- **提取部分信息内容**

Float128中bitmap如下：
| Bit range | Reserved for    |
| --------- | --------------- |
| 255 - 242 | EXPONENT        |
| 241       | L_MANTISSA_FLAG |
| 240       | MANTISSA_SIGN   |
| 239 - 0   | L_MANTISSA      |
| 128 - 0   | M_MANTISSA      |


```solidity
//a是一个符合bitmap存储的float
//提取指数位的指数：
  uint constant EXPONENT_MASK = 0xfffc000000000000000000000000000000000000000000000000000000000000;
let aExp := and(a, EXPONENT_MASK)
```

已知指数存在255-242位，一共14位，那么使用掩码：

`1111_1111_1111_11000000....0000`

和`and`即可保留这段信息。

这里我们可以看到掩码和仅与逻辑即可暴露特点位置信息同时删除其他的无关信息。



<br/>

- **Float128**

Float128大量地使用了yul来进行算数，是一个快速学习yul的好教程

首先函数`toPackedFloat()`将位数和次幂转化为一个packFloat(内在uint256类型)

```solidity
function toPackedFloat(int mantissa, int exponent) internal pure returns (packedFloat float) {
        uint digitsMantissa;
        uint mantissaMultiplier;
        ///1. 判断mantissa的正负性
        if (mantissa != 0) {
            assembly {
                if and(mantissa, TWO_COMPLEMENT_SIGN_MASK) {//这段and(int,uint)来判断函数的正负
                    float := MANTISSA_SIGN_MASK
                    mantissa := sub(0, mantissa)
                }
            }
            // we normalize only if necessary
            if (
                !((mantissa <= int(MAX_M_DIGIT_NUMBER) && mantissa >= int(MIN_M_DIGIT_NUMBER)) ||
                    (mantissa <= int(MAX_L_DIGIT_NUMBER) && mantissa >= int(MIN_L_DIGIT_NUMBER)))
            ) {
                digitsMantissa = findNumberOfDigits(uint(mantissa));
                assembly {
                    mantissaMultiplier := sub(digitsMantissa, MAX_DIGITS_M)
                    let isResultL := slt(MAXIMUM_EXPONENT, add(exponent, mantissaMultiplier))
                    if isResultL {
                        mantissaMultiplier := sub(mantissaMultiplier, DIGIT_DIFF_L_M)
                        float := or(float, MANTISSA_L_FLAG_MASK)
                    }
                    exponent := add(exponent, mantissaMultiplier)
                    let negativeMultiplier := and(TWO_COMPLEMENT_SIGN_MASK, mantissaMultiplier)
                    if negativeMultiplier {
                        mantissa := mul(mantissa, exp(BASE, sub(0, mantissaMultiplier)))
                    }
                    if iszero(negativeMultiplier) {
                        mantissa := div(mantissa, exp(BASE, mantissaMultiplier))
                    }
                }
            } else if (
                (mantissa <= int(MAX_M_DIGIT_NUMBER) && mantissa >= int(MIN_M_DIGIT_NUMBER)) &&
                exponent > MAXIMUM_EXPONENT
            ) {
                assembly {
                    mantissa := mul(mantissa, BASE_TO_THE_DIGIT_DIFF)
                    exponent := sub(exponent, DIGIT_DIFF_L_M)
                    float := add(float, MANTISSA_L_FLAG_MASK)
                }
            } else if ((mantissa <= int(MAX_L_DIGIT_NUMBER) && mantissa >= int(MIN_L_DIGIT_NUMBER))) {
                assembly {
                    float := add(float, MANTISSA_L_FLAG_MASK)
                }
            }
            // final encoding
            assembly {
                float := or(float, or(mantissa, shl(EXPONENT_BIT, add(exponent, ZERO_OFFSET))))
            }
        }
    }
```



- Aegis

```solidity

  
  function verifyNonce(address sender, uint256 nonce) public view returns (uint256, uint256, uint256) {
    if (nonce == 0) revert InvalidNonce();
    //转为uint64，即64位的二进制数，之后向右移8位（减少8位，相当于除以2^8=256）
    //nonce0-255的slot为0，256-511是1，以此类推
    uint256 invalidatorSlot = uint64(nonce) >> 8;
    
    //uint8(nonce)相当于除余nonce % 256，因为>=256的数值都会被舍弃，此时二进制后面的数不断增加，作为循环
    //数据nonce = 1，则Bit = 1<<1 = 2 (0000...10)
    //	 nonce = 256,则除余为0，bit = 1 (0000...1)
    //这些操作相当于生成掩码
    uint256 invalidatorBit = 1 << uint8(nonce);
    
    
    uint256 invalidator = _orderBitmaps[sender][invalidatorSlot];
    ///如果之前nonce没在这里更新过，invalidator应该是0
    ///&与掩码数值也是0
    if (invalidator & invalidatorBit != 0) revert InvalidNonce();

    return (invalidatorSlot, invalidator, invalidatorBit);
  }
  
  
    function _deduplicateOrder(address sender, uint256 nonce) private {
    (uint256 invalidatorSlot, uint256 invalidator, uint256 invalidatorBit) = verifyNonce(sender, nonce);
    _orderBitmaps[sender][invalidatorSlot] = invalidator | invalidatorBit;
    //mapping通过or操作来更新掩码对应位置的数值
    //即sender的nonce0-255(sender->1)的mapping值为先前的mapping，在对应掩码位置上把0改1
  }
```





# 抽象语法树(Abstract syntax tree)

















## StorageLayout in Solidity

Mapping存储方法：当mapping在





# 密码学

## SECP256k1

- 素数（prime number）：是除了1和本身外不能其他数整除的数，如1，3，5，7

- 素数域（Prime Field）：

  即一个素数取余运算中的所有结果的集合，比如5的素数域是{0,1,2,3,4}，因为任意整数X, `X mod 5`的结果只可能有这些。

$$
F_p= \{\, 0,1,2,…,p−1 \}
$$

在这个集合里的加法、减法、乘法，再对p取余，其结果总能返回到[0,p-1]的范围之中（此被称为素数域的**封闭性**）。

但是，这个离散集合没有除法怎么办？

这就有集合里每个非零元素`a` 都有一个唯一的**逆元(inverse)** `a^−1`满足
$$
a \times a^{-1} \mod p = 1
$$
比如根据定义F7中，5的逆元是3，即`5*3 mod 7 =1`这是唯一确定的，从而构造离散集合中的除法。这也就意味着比如素数域中的运算`2/3 (mod 7)`等同于2乘以3的逆元5，最终等于10。

求逆元可以通过`扩展欧几里得算法`或`费马小定理`来解。

- 扩展欧几里得算法如下

$$
ax+py=gcd(a,p)=1
\\
其中gcd是 greatest\ common\ divisor（最大公约数），
\\比如：gcd(14,21)=7
$$

比如求p=13时，5的逆元即 `5x + 13y = 1`，因为最大公约数为1，其中的x和y有一个数肯定不在域内，此时另外一个就是我们要的逆元。（这个在图像中更好展示，逆元必须得在0-12中，我们可以画个图来框住x,y来找到符合条件的数）

此时其中的一个结果就是x=8,y=-3

这个结果的8在域内，我们就知道8是5的inverse。

------

Secp256k1 就是一个定义在素数域上的椭圆曲线，具体而言应该是说曲线上的所有点的集合。

这个曲线的方程为：
$$
y² = x³ + 7 \ (mod\ \ p) \\
其中\ p= 2^{256} -2^{32} -977
$$


素数域上的点都是离散的，我们就会考虑曲线上的点(x,y)到底有多少个，而这个关于曲线上有多少个点我们称为阶，标记为*n*.

16进制下Scep256k1的*n* = `0xFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFEBAAEDCE6AF48A03BBFD25E8CD0364141`

**私钥和公钥的生成由以下步骤构成：**

1. 选取私钥 d，满足d，1  ≤  d  <  n

2. 基点 G：G=(Gx,Gy) ，它是曲线上公开的固定点（Generate point）。这个点的特性是对它做“重复加法”会生成曲线上的所有（或几乎所有）点。

3. 公钥就是Q  =  d×G, 表示椭圆曲线上的标量乘法运算d和G，即把 G重复加d遍，这样我们就能能映射所有的点

4. 得到新的坐标 (x_Q,y_Q)，这对数值即是公钥的“坐标表示”。

- 在网络协议中，公钥通常序列化为：
  - **未压缩格式**（65 字节）：`0x04 || x_Q(32B) || y_Q(32B)`（即04+x+y坐标，不是位运算而是直接连着拼接）
  - **压缩格式**（33 字节）：`0x02/03 || x_Q(32B)`，由于已知曲线方程，所以拿x就知道了y的坐标，而又因为椭圆曲线是关于x坐标对称的，同一个x可能存在两个不同的y，而为什么是一奇一偶；假设y是方程的一个解，则-y也是方程的一个解，在模运算规则下，-y ≡ p - y (mod p) ,所以p-y 是方程的解。两个解的和是y + (p - y) = p，p是素数，自然也是奇数，所以两个解肯定是一奇数，一偶数。



看上去非常公钥得出来方法非常简单，似乎是一个简单的`加法`就能完成，那为什么公钥倒推私钥如此困难呢？

原因在于几个点:

- 加法的定义并非1+1=2，而是如下面P+P =2P，即相同的两个数相加就是其点在椭圆曲线上的切线与曲线的另外一个交点的关于X轴对称的那个点（切线倍点规则）

![img](https://user-images.githubusercontent.com/35583758/229375008-254586ef-177c-4111-9aa6-60d42ff6a251.png)

- 椭圆曲线的阶数足够大，接近2^256的计算，如果暴力穷举，按这个加法的算法基本上是天文时间。 



在比特币和以太坊等使用 secp256k1 椭圆曲线的区块链系统中，`v`, `r`, `s` 是构成 **ECDSA签名** 的三个核心组成部分。它们共同证明了签名者拥有与特定公钥（或地址）对应的私钥，且同意交易（或消息）的内容。

以下是 `v`, `r`, `s` 的详细解释：

1.  **`r` 和 `s`：签名的核心数值**，二者都是32字节：它们是由 ECDSA 签名算法根据 **私钥 (`d`)**, 被签名的 **交易/消息的哈希值 (`z`)**, 以及一个临时的、随机生成的 **随机数 (`k`)** 计算得出的。
    *   **如何生成 (简化流程)：**
        1.  选择一个密码学安全的随机数 `k` (在 [1, n-1] 范围内，`n` 是 secp256k1 曲线的阶)。
        2.  计算临时点 `R = k * G`（X_r, Y_r） 
        3.  取 `R` 的 x 坐标作为 `r` (`r = R.x mod n`)。如果 `r = 0`，需重新选 `k`。
        4.  计算消息哈希 `z` 。(`Keccak256(message)` )
        5.  用私钥`d`计算 `s = k⁻¹ * (z + r * d) mod n`。如果 `s = 0` 或 `s` 过小（这也是为什么会出现签名可塑性，在oz的签名库中 `s` 必须大于 `n/2` 以防范此攻击，因为你可以用另外一个合法的`s`和`v`来配对同一个签名而成功计算出公钥地址），需重新选 `k`。
    *   **作用：**
        *   `r` 和 `s` 是数学证明的核心。它们以一种只有私钥持有者才能产生，但任何人都能用公钥验证的方式，将签名与特定消息和特定私钥绑定在一起。
        *   **验证者** 使用签名者的 **公钥 (`Q`)**, 消息哈希 `z`, 以及 `r` 和 `s`，通过一系列椭圆曲线运算，可以验证签名是否有效，而无需知道私钥 `d`。
2.  **`v` (恢复标识符 / 恢复ID)：**
    *   `v` 是一个很小的值（通常是 0, 1, 27, 28 或类似的 1 字节值）。
    *   它的核心作用是 **帮助确定签名验证时需要的正确公钥**。在 ECDSA 验证公式中，需要用到计算 `r` 时产生的临时点 `R` 的 **完整坐标 (x, y)**。然而，签名本身只包含了 `R` 的 x 坐标（即 `r`）。一个 x 坐标在椭圆曲线上通常对应 **两个** 可能的 y 坐标（一奇一偶）。为了在验证时准确地重建出临时点 `R`，需要知道 `R` 的 y 坐标是 **偶数还是奇数**（或者更一般地说，是压缩格式中的某个特定值）。

 

`v`, `r`, `s`如何恢复公钥：

由`s = k⁻¹ * (z + r * d) mod n`变形可得:

`k = s⁻¹ * (z + r * d) mod n`

`r`是点`R`的x坐标，而`d`又是私钥，我们知道私钥`d*G=Q` 而`r`是由 `R = k * G`得来的，

我们就能得到:

`R=s⁻¹⋅(z⋅G+r⋅Q)`

整理可得关于公钥`Q`的公式：

`Q=(s⋅R−z⋅G)⋅r⁻¹ (mod n)`

这样我们就能根据输入的`r`,`s`解得正确的公钥地址。

