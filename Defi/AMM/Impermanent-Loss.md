

要理解 **impermanent loss（IL，非永久性损失）**，必须先弄清楚：作为 Liquidity Provider (LP)，你在 AMM 池里“赚的是什么”，以及“为什么会出现和单纯持有不同的结果”。

---

## 1. LP 的获利来源

当你往 AMM 池里存入流动性（比如 1 BTC + \$100,000 稳定币，假设 BTC 当时价格正好是 \$100,000），你会收到 LP Token。这个 LP Token 代表你在池子中的 **份额**。
你作为 LP 的收入主要来自：

1. **交易手续费**

   * 池子里有用户不断在 BTC ↔ 稳定币之间做交易。
   * 每一笔交易要付交易费（例如 0.04%）。
   * 这些费用按比例分给 LP，所以交易量越大、池子越深，收益越多。

2. **可能的激励 / 挖矿奖励**

   * 有的池子会额外发放奖励（例如 CRV、治理代币等），作为补贴。

所以：**LP 并不是通过资产价格上涨本身赚钱的，而是通过“别人交易付费”赚钱的。**

---

## 2. 为什么会出现 Impermanent Loss（IL）

Impermanent loss 是和 **“如果我只是单纯持有（HODL）”** 相比，LP 的资产价值少了多少。

根源在于 AMM 的自动再平衡机制：

* 池子使用公式（比如常见的 `x * y = k`）保持两边资产的比例。
* 当 BTC 价格上涨时，套利者会不断买走 BTC、放进稳定币，让池子价格和外部市场对齐。
* 结果：**池子里的 BTC 数量会越来越少，稳定币数量越来越多**。
* 当你取出流动性时，你得到的 **BTC 比单纯持有时更少**，**稳定币比单纯持有时更多**。

这就是 IL：你“错过了”如果单纯拿着 BTC，本可以获得更多涨幅的收益。

---

## 3. 举个例子

初始条件：

* 你存入 **1 BTC + \$100,000 USDC**，BTC 当时价格 = \$100,000。
* 池子一开始总量：10 BTC + \$1,000,000 USDC（假设）。

你占池子 **10% 份额**。

---

### 情况 A：BTC 价格不动

* 你一直拿着 LP Token，最后拿回相同数量的 BTC+USDC，加上交易费收益。
* 没有 IL，赚的就是手续费。

---

### 情况 B：BTC 翻倍到 \$200,000

* **单纯持有**：你的资产 = 1 BTC (\$200k) + \$100k = **\$300k**。
* **做 LP**：池子会不断被套利调整，最终比例会接近 50% BTC + 50% USDC。

  * 你取出时，大约得到 **0.707 BTC + \$141,000 USDC ≈ \$283,000**。
* 差额 ≈ **\$17k (5.7%)**，就是 IL。
* 不过在此期间，你还赚了交易费（比如几千美金）。如果手续费收入 > 17k，那么整体还是正收益。

---

### 情况 C：BTC 跌到 \$50,000

* **单纯持有**：1 BTC (\$50k) + \$100k = **\$150k**。
* **做 LP**：池子再平衡后，你会得到更多 BTC（大约 1.414 BTC + \$70,700 USDC ≈ \$141k）。
* 差额 ≈ -\$9k (≈ -6%)，依然是 IL。
* 但一样，可能手续费弥补。

---

## 4. 小结

* **为什么有 IL**：因为 AMM 池要保持两边资产的平衡，当价格变动时，你的资产配置会被动再平衡，导致相对“单纯持有”少赚（涨的时候少赚，跌的时候反而多拿 BTC，但价值还是低）。
* **LP 如何获利**：靠 **手续费 + 奖励** 来弥补 IL，如果交易足够频繁，手续费收入就能盖过 IL，从而获得净收益。

---

👉 所以当你提供流动性时，你本质上是在**拿资产价格风险 + 承担 IL**，换取**交易费收益**。

下面一段图片能更加直观地对比LP在将资产放入流动性池时和直接HOLD时其portfolio的价值对比。可以看到LP的头寸与P是呈现一个√p的关系



![alt text](../../resources/ImermanentLoss-Example.png)
```python
# plotting LP vs HODL value for a constant-product AMM (no fees) and presenting a small table
import numpy as np
import pandas as pd
import matplotlib.pyplot as plt
from math import sqrt
# helper from the jupyter tools (available in this environment)
try:
    from caas_jupyter_tools import display_dataframe_to_user
except Exception as e:
    display_dataframe_to_user = None

# Parameters (the example you gave)
btc0 = 1.0               # initial BTC provided by the LP
usd0 = 100000.0          # initial USD/stablecoin provided by the LP
p0 = usd0 / btc0         # initial price USD per BTC (100,000)
k = btc0 * usd0          # constant-product k

# price multipliers relative to p0 (we'll sweep from down 0.2x to up 2.0x)
multipliers = np.linspace(0.2, 2.0, 181)  # step 0.01 roughly
prices = p0 * multipliers

# compute new reserves for a constant-product AMM after price change
rb_new = np.sqrt(k / prices)      # BTC reserve after price change (if pool sole reserves = user's deposit)
ru_new = np.sqrt(k * prices)      # USD reserve after price change

# values
lp_values = rb_new * prices + ru_new           # value of LP holdings (BTC*price + USD)
hodl_values = btc0 * prices + usd0             # value if we just held BTC+USD

# impermanent loss (fractional)
il_fraction = (lp_values - hodl_values) / hodl_values

# package into dataframe for a few highlighted points
highlight_multipliers = [0.5, 0.70710678, 1.0, 1.41421356, 2.0]  # some canonical points including sqrt(2)
rows = []
for m in highlight_multipliers:
    p = p0 * m
    rb = sqrt(k / p)
    ru = sqrt(k * p)
    lp_val = rb * p + ru
    hodl_val = btc0 * p + usd0
    il_pct = (lp_val - hodl_val) / hodl_val * 100
    rows.append({
        "multiplier": m,
        "price": p,
        "btc_withdrawn": rb,
        "usd_withdrawn": ru,
        "LP_value": lp_val,
        "HODL_value": hodl_val,
        "IL_percent": il_pct
    })
df_highlight = pd.DataFrame(rows)

# full table for display (sampled every 10th point to keep size reasonable)
sample_df = pd.DataFrame({
    "multiplier": multipliers,
    "price": prices,
    "LP_value": lp_values,
    "HODL_value": hodl_values,
    "IL_percent": il_fraction * 100
})
sample_df_sampled = sample_df.iloc[::10].reset_index(drop=True)

# Plot (single figure) - do not set colors or styles explicitly
fig, ax = plt.subplots(figsize=(9,5))
ax.plot(multipliers, hodl_values, label="HODL value (1 BTC + $100k)")
ax.plot(multipliers, lp_values, label="LP value (constant-product, no fees)")
ax.set_xlabel("Price multiplier (relative to initial price)")
ax.set_ylabel("Portfolio value (USD)")
ax.set_title("HODL vs LP value for 1 BTC + $100k in a constant-product AMM (no fees)")
ax.legend()
ax.grid(True)

# show the figure (this cell's output will include the plot)
plt.tight_layout()
plt.show()

# display the highlighted table to the user via the interactive dataframe helper when available
if display_dataframe_to_user is not None:
    display_dataframe_to_user("Highlighted values for selected price multipliers", df_highlight)

# also return the sampled numeric table as output
sample_df_sampled.head(40)


```

