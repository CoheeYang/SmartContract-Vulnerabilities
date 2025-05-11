

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
