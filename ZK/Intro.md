[TOC]



# Group Theory

## 1. Definition

**Group**就是一个集合(如`{1,2,3}`)，加上一种“运算”（一般只有加/乘法，运算一般抽象为`⋅`来表示），满足四个条件（就像一个小型的代数世界）：

1. **封闭性（Closure）**: 集合里的任意两个元素相加/相乘（取决于运算），结果还在这个集合里。
    👉比如整数集合 **Z**对加法是封闭的：任意两个整数相加，还是整数。
2. **结合律（Associativity）**:
    运算的顺序不影响结果：(a⋅b)⋅c=a⋅(b⋅c)。
3. **单位元（Identity element）**:
    有一个特殊的元素 ***I***，和任何元素运算后结果还是那个元素,即`a·I=a` for any a belongs to the group。
   👉 在整数加法群里，这个单位元就是 0。
4. **逆元（Inverse element）**: 群中的每个元素a都有一个逆元 b，使得`a·b = I`。👉在整数加法群里，5的逆元就是-5 

> 这样一个集合 + 运算就叫做 **群**。
>  在密码学里常用的是 **有限群**，特别是模 *p* 的加法群 `(Zp,+)`或者椭圆曲线群。
>
> 📌 你可以先把群想象成一个“对称性集合”，或者一个“合法的运算规则空间”。

我们刚才说了群的四个基本条件，现在在群的基础上，如果再加上一个条件，就得到了 **交换群（commutative group）**，也叫 **阿贝尔群（Abelian group）**。

这个额外条件就是交换律（Commutativity）： `a·b = b·a` holds for any a,b in the group

- 整数加法群和模素数加法群 `(Zp, +)`就符合这个条件，比如`(2 + 7) mod p = (7 + 2) mod p`
- 可逆矩阵的乘法群就不符合此条件：即`AB ≠ BA`存在



## 2. Modular Arithmetic

- **定义**：
   当两个整数 `a` 和 `b` 除以 `n` 的余数相同时，我们说
   `a ≡ b (mod n)`
- **例子**：
  - `17 ≡ 5 (mod 12)`，因为 17 和 5 除以 12 余数都是 5
  - `-7 ≡ 3 (mod 10)`，因为 -7 和 3 除以 10 余数都是 3
- **规则**

1. **加法规则**
    `(a + b) mod n = ((a mod n) + (b mod n)) mod n`
2. **乘法规则**
    `(a × b) mod n = ((a mod n) × (b mod n)) mod n`

模运算就是「在一个有限的数轴上绕圈子」。
 比如在 `mod 5` 下，所有数都会被折叠到 `{0,1,2,3,4}` 这 5 个余数上。

------

现在给你一个练习：
 在 `mod 7` 下，`4 + 5` 等于多少？







# References

[(37) Cryptography 101 for Blockchain Developers Part 1/3: Group Theory - YouTube](https://www.youtube.com/watch?v=jnhjM_2hDJE&t=199s)

