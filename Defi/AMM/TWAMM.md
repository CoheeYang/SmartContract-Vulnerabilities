[TOC]



# 1. 算法推导

TWAMM作为大额交易的算法，模仿传统金融中的拆单模式，将订单金额以区块为单位，分别执行。

传统金融中是拆单每小时进行多少笔订单，而TWAMM则是在其内生AMM中拆成每区块多少金额资产买入/卖出

比如卖出2000USDC，拆成100个区块，从而每个区块AMM中收到20USDC。

拆成区块的好处还在每个区块发生交易导致price Impact时，这种资产价格波动会被套利者抹平，从而在下一个区块中以更优价格进行。



对于任意的卖出$X$ token的订单，其virtual order会保持如下的恒等式：
$$
(x + dX_{in}),(y - dY_{out}) = x y  \tag{1}
$$
展开得到:
$$
xy + x(-dY_{ out}) + ydX_{ in} - dX_{ in}dY_{ out} = xy
$$
化简可知：

$$
dY_{out}=\frac{y}{x}dX_{in} - \frac{dY_{out}dX_{in}}{x}
$$

二阶无穷小忽略时，我们知道此时此订单应该得到的$dY_{out}$ 是
$$
dY_{out}=\frac{y}{x}dX_{in} \tag{2}
$$
同理，对于任意卖出$Y$ token的订单，我们会有：
$$
dX_{out}=\frac{x}{y}dY_{in} \tag{3}
$$
而现在我们已知用户会在池子中下卖出代币的订单，假设在$t$区块内，池子中有卖出$X$ 代币的卖出速率为$r_x$，卖出$Y$的速率为$r_y$

>  就像之前例子中卖出2000USDC，拆成100个区块，从而每个区块AMM中收到20USDC，这20 USDC/Block 就是这里速率

那么我们很自然地就知道之前（2），（3）式的$dX_{in},dY_{in}$ 的表达式：
$$
\begin{cases}
dX_{in} = r_xdt 
\\
dY_{in} = r_ydt
\end{cases}
\tag{4}
$$
那么对于AMM整体来说，极限情况下的代币总**净收**为$dx， dy$ ，而这二者的表达式为：
$$
\begin{cases}
dx = dX_{in} -dX_{out}
\\
dy = dY_{in} -dY_{out}
\end{cases}
\tag{5}
$$
联立我们之前的（4），（2）和（3）这三个公式可以得到（以$dx$ 为例）：
$$
dx = r_xdt-\frac{x}{y}r_y d_t
$$
由于AMM是一个恒积式AMM，所以有$xy=k$，代入上式可得：
$$
dx= r_xdt-\frac{x^2}{k}r_y d_t \tag{6}
$$
**公式（6）是一个常微分方程，我们通过分离变量积分后可得**：
$$
\int \frac{dx}{r_x - \frac{r_y}{k}x^2} = \int dt \tag{7}
$$
等式的右边很简单，就是$t+Constant$

等式的左边则是一个反双曲正切函数（arctanh）的导数，即$\int \frac{du}{1-u^2}=\operatorname{arctanh} u + Constant$

设$A=r_x,B=\frac{r_y}{k}$，并令$x=u\sqrt{\frac{A}{B}}$ ，带入（7）左式中可得：
$$
\int \frac{dx}{A - Bx^2} 
== \frac{\sqrt{\frac{A}{B}} du}{A - B\Big(u^2\frac{A}{B}\Big)}
= \frac{\sqrt{\frac{A}{B}} du}{A - A u^2}
= \sqrt{\frac{A}{B}}\cdot\frac{du}{A(1-u^2)} \\
=\frac{1}{\sqrt{AB}}\cdot\frac{du}{1-u^2}.
$$
这样，我们通过（7）将左侧按$u$积分后带回$x$得到：
$$
\frac{1}{\sqrt{AB}}\operatorname{arctanh}\Big(x\sqrt{\tfrac{B}{A}}\Big)  = t+Constant \tag{8}
$$
值得注意的是`arctanh`的定义域是在（-1,1），所以我们得让$|x|\sqrt{\tfrac{B}{A}}<1$

（8）式是个不定积分的解，没什么实际意义，因为我们要求的是[0,t]区块时间段内，所需要的收到的代币数量

所以我们对（7）做定积分运算可得：
$$
\int_{x_0}^{x(t)}\frac{dx}{A-Bx^2}=\int_{0}^{t}dt 
$$

左侧按之前的求值可知：
$$
\frac{1}{\sqrt{AB}}\Big[
\operatorname{arctanh}\Big(x(t)\sqrt{\tfrac{B}{A}}\Big)- \operatorname{arctanh}\Big(x_0\sqrt{\tfrac{B}{A}}\Big)
  \Big]=t
$$
把常数移到右边，整理得
$$
\operatorname{arctanh}\Big(x(t)\sqrt{\tfrac{B}{A}}\Big)
  = t\sqrt{AB} + \operatorname{arctanh}\Big(x_0\sqrt{\tfrac{B}{A}}\Big) \tag{9}
$$

对于（9）式我们可以用$ \tanh(\operatorname{arctanh}(z))=z$ 来简化，在此之前我们先简化符号，定义$\beta 和 S $为： 
$$
\beta:=\sqrt{AB}=\sqrt{\frac{r_xr_y}{k}},\qquad
S:=\sqrt{\frac{A}{B}}=\sqrt{\frac{r_x k}{r_y}}.
$$
（9）式带入新的符号为：
$$
\operatorname{arctanh}\Big(\frac{x(t)}{S}\Big)
= \beta t + \operatorname{arctanh}\Big(\frac{x_0}{S}\Big).
$$
对两边做 tanh（双曲正切）的逆运算，得
$$
\frac{x(t)}{S}=\tanh\Big(\beta t + \operatorname{arctanh}\big(\tfrac{x_0}{S}\big)\Big)  \tag{10}
$$

此时我们用tanh的加法公式：$\tanh(\alpha+\gamma)=\frac{\tanh\alpha+\tanh\gamma}{1+\tanh\alpha\tanh\gamma}$再将上面式子简化
$$
\boxed{\frac{x(t)}{S}=\frac{tanh(\beta t) +\frac{x_0}{S}}{1+tanh(\beta t)\frac{x_0}{S}}}  \tag{11}
$$

在tanh的乘法公式中表达：$\tanh(\beta t)=\frac{e^{2\beta t}-1}{e^{2\beta t}+1}$  

**下面我们对（11）进行化简：**


记 $u_0 :=\dfrac{x_0}{S}, E:=e^{2\beta t}$ 

把 $\tanh(\beta t)$ 代入分式（11）右边的分子和分母：

**分子：**
$$
\frac{E-1}{E+1}+u_0=\frac{E-1 + u_0(E+1)}{E+1}
=\frac{(1+u_0)E + (u_0-1)}{E+1}.
$$

**分母：**
$$
1+u_0\frac{E-1}{E+1}=\frac{E+1 + u_0(E-1)}{E+1}
=\frac{(1+u_0)E + (1-u_0)}{E+1}.
$$

因此整个分式化为
$$
\frac{(1+u_0)E + (u_0-1)}{(1+u_0)E + (1-u_0)}.
$$

把分子分母同时除以 $1+u_0$，并记$c:=\frac{1-u_0}{1+u_0}$
$$
\frac{E + \dfrac{u_0-1}{1+u_0}}{E + \dfrac{1-u_0}{1+u_0}}
= \frac{E - c}{E + c},
$$
（11）式整理可得（12）式
$$
x(t)=S\cdot\frac{E - c}{E + c} \tag{12}
$$


这正是你图片中采用的排列（仅是 (c) 定义的符号不同）。现在把 (u_0) 展开回初始量并把 (E) 写回原来的常量：

记回原始量：
$$
S:=\sqrt{\frac{r_x k}{r_y}}\\
\beta t:=\sqrt{\frac{r_xr_y}{k}}t\\
E=e^{2\beta t}= \exp\Big(2\sqrt{\frac{r_xr_y}{k}}t)\\
u_0 :=\dfrac{x_0}{\sqrt{\frac{r_x k}{r_y}}}\\
c=(1-\dfrac{x_0}{\sqrt{\frac{r_x k}{r_y}}})/(1+\dfrac{x_0}{\sqrt{\frac{r_x k}{r_y}}})
$$
其中$\beta t:=\sqrt{\frac{r_xr_y}{k}}t = \sqrt{\frac{(x_{in}/t)(y_{in}/t)}{k}t^2}$ 可以将时间变量$t$ 约掉，（$x_{in},y_{in}$是这段时间的代币总流入）

类似地，对于$c=(1-\dfrac{x_0}{\sqrt{\frac{r_x k}{r_y}}})/(1+\dfrac{x_0}{\sqrt{\frac{r_x k}{r_y}}})$ 也可以将$r_x,r_y$ 替换成$x_{in},y_{in}$ ，从而避免时间单位的计算，而只计算总量即可，以便于lazy evaluation。

最终我们化简可得到（13）式，即在任意连续$t$ 时块的区块的$X$ token reserve应该是
$$
\boxed{x_{t}
= \sqrt{\frac{kx_{\rm in}}{y_{\rm in}}}
\frac{e^{2\sqrt{\dfrac{x_{\rm in}y_{\rm in}}{k}}}+c}
{e^{2\sqrt{\dfrac{x_{\rm in}y_{\rm in}}{k}}}-c}}\tag{13}
\qquad\\ 
\\
where  \qquad
c=\frac{\sqrt{x_0y_{\rm in}}-\sqrt{y_0x_{\rm in}}}
{\sqrt{x_0y_{\rm in}}+\sqrt{y_0x_{\rm in}}}.
$$

通过（13）式，我们可以联立恒积式不变量，得到此时对应的$Y$ token reserve。并将$t$时刻的reserve和之前时刻$t'$ 来相减，从而获得这段时间需要输出多少token.



当我们知道了总量，只需要按类似流动性奖励的算法来算出分配给每个买卖者所得到和花费的token即可。

# 代码





# ref

[TWAMM - Paradigm](https://www.paradigm.xyz/2021/07/twamm)
