---
type: concept
topic: Python+AI
aliases: [Options Pricing, Greeks Calculation, 期权定价]
status: done
created: "2026-06-28"
updated: "2026-06-28"
tags: [期权, Python, 定价]
related: "[[Python期权库入门]], [[Black-Scholes]]"
---

# 期权定价模型与 Greeks 计算

## 引言

期权定价是期权交易的核心。无论你是构建策略、管理风险还是寻找套利机会，都需要准确的定价模型和 Greeks 计算。本章将用 Python 从零实现主流的期权定价模型，包括 Black-Scholes 模型、二叉树模型（Binomial Tree）和蒙特卡洛模拟（Monte Carlo Simulation），并深入讲解 Greeks 的数值计算方法和隐含波动率的求解算法。

---

## 一、Black-Scholes 模型的 Python 实现

### 1.1 理论回顾

Black-Scholes 模型是期权定价的基石。对于不支付股息的欧式期权，看涨期权的定价公式为：

```
C = S * N(d1) - K * e^(-rT) * N(d2)
P = K * e^(-rT) * N(-d2) - S * N(-d1)

其中：
d1 = [ln(S/K) + (r + 0.5 * sigma^2) * T] / (sigma * sqrt(T))
d2 = d1 - sigma * sqrt(T)
```

参数说明：
- `S`：标的价格（Spot Price）
- `K`：执行价格（Strike Price）
- `T`：到期时间（年）
- `r`：无风险利率（Risk-Free Rate）
- `sigma`：波动率（Volatility）
- `N(x)`：标准正态分布的累积分布函数（CDF）

### 1.2 完整实现

```python
import numpy as np
from scipy.stats import norm
from dataclasses import dataclass
from typing import Optional

@dataclass
class OptionResult:
    """期权定价结果"""
    price: float
    delta: float
    gamma: float
    theta: float
    vega: float
    rho: float

class BlackScholes:
    """
    Black-Scholes 期权定价模型

    支持欧式看涨/看跌期权的定价和 Greeks 计算
    """

    def __init__(self, S: float, K: float, T: float, r: float, sigma: float):
        """
        初始化参数

        Args:
            S: 标的价格
            K: 执行价格
            T: 到期时间（年）
            r: 无风险利率（年化，如0.05表示5%）
            sigma: 波动率（年化，如0.20表示20%）
        """
        self.S = S
        self.K = K
        self.T = T
        self.r = r
        self.sigma = sigma

        # 预计算 d1 和 d2
        self._d1 = self._calc_d1()
        self._d2 = self._calc_d2()

    def _calc_d1(self) -> float:
        return (np.log(self.S / self.K) +
                (self.r + 0.5 * self.sigma**2) * self.T) / \
               (self.sigma * np.sqrt(self.T))

    def _calc_d2(self) -> float:
        return self._d1 - self.sigma * np.sqrt(self.T)

    def call_price(self) -> float:
        """计算看涨期权价格"""
        return (self.S * norm.cdf(self._d1) -
                self.K * np.exp(-self.r * self.T) * norm.cdf(self._d2))

    def put_price(self) -> float:
        """计算看跌期权价格"""
        return (self.K * np.exp(-self.r * self.T) * norm.cdf(-self._d2) -
                self.S * norm.cdf(-self._d1))

    def call_greeks(self) -> OptionResult:
        """计算看涨期权的所有 Greeks"""
        S, K, T, r, sigma = self.S, self.K, self.T, self.r, self.sigma
        d1, d2 = self._d1, self._d2
        sqrt_T = np.sqrt(T)
        exp_rT = np.exp(-r * T)
        nd1 = norm.pdf(d1)  # 标准正态概率密度

        delta = norm.cdf(d1)
        gamma = nd1 / (S * sigma * sqrt_T)
        theta = (-(S * nd1 * sigma) / (2 * sqrt_T) -
                 r * K * exp_rT * norm.cdf(d2)) / 365
        vega = S * nd1 * sqrt_T / 100
        rho = K * T * exp_rT * norm.cdf(d2) / 100

        return OptionResult(
            price=self.call_price(),
            delta=delta, gamma=gamma,
            theta=theta, vega=vega, rho=rho
        )

    def put_greeks(self) -> OptionResult:
        """计算看跌期权的所有 Greeks"""
        S, K, T, r, sigma = self.S, self.K, self.T, self.r, self.sigma
        d1, d2 = self._d1, self._d2
        sqrt_T = np.sqrt(T)
        exp_rT = np.exp(-r * T)
        nd1 = norm.pdf(d1)

        delta = norm.cdf(d1) - 1
        gamma = nd1 / (S * sigma * sqrt_T)
        theta = (-(S * nd1 * sigma) / (2 * sqrt_T) +
                 r * K * exp_rT * norm.cdf(-d2)) / 365
        vega = S * nd1 * sqrt_T / 100
        rho = -K * T * exp_rT * norm.cdf(-d2) / 100

        return OptionResult(
            price=self.put_price(),
            delta=delta, gamma=gamma,
            theta=theta, vega=vega, rho=rho
        )

    def put_call_parity_check(self) -> dict:
        """验证 Put-Call Parity: C - P = S - K * e^(-rT)"""
        c = self.call_price()
        p = self.put_price()
        lhs = c - p
        rhs = self.S - self.K * np.exp(-self.r * self.T)
        return {
            'call_price': c,
            'put_price': p,
            'C - P': lhs,
            'S - K*exp(-rT)': rhs,
            'difference': abs(lhs - rhs)
        }


# 使用示例
if __name__ == "__main__":
    # 参数设置
    S = 150.0      # 标的价格
    K = 155.0      # 执行价格
    T = 30 / 365   # 30天到期
    r = 0.05       # 5% 无风险利率
    sigma = 0.25   # 25% 波动率

    bs = BlackScholes(S, K, T, r, sigma)

    # 看涨期权结果
    call = bs.call_greeks()
    print(f"{'='*50}")
    print(f"Black-Scholes 期权定价")
    print(f"{'='*50}")
    print(f"标的价格: ${S}")
    print(f"执行价格: ${K}")
    print(f"到期天数: {int(T*365)}")
    print(f"波动率:   {sigma:.1%}")
    print(f"\n--- 看涨期权 ---")
    print(f"价格:  ${call.price:.2f}")
    print(f"Delta: {call.delta:.4f}")
    print(f"Gamma: {call.gamma:.6f}")
    print(f"Theta: {call.theta:.4f}")
    print(f"Vega:  {call.vega:.4f}")
    print(f"Rho:   {call.rho:.4f}")

    # Put-Call Parity 验证
    parity = bs.put_call_parity_check()
    print(f"\n--- Put-Call Parity 验证 ---")
    print(f"C - P = ${parity['C - P']:.4f}")
    print(f"S - Ke^(-rT) = ${parity['S - K*exp(-rT)']:.4f}")
    print(f"差异: ${parity['difference']:.10f}")
```

### 1.3 股息调整

对于支付股息的期权，需要对 Black-Scholes 模型进行调整。最常用的方法是将标的价格替换为去除股息现值后的价格：

```python
class BlackScholesDividend(BlackScholes):
    """支持股息的 Black-Scholes 模型"""

    def __init__(self, S, K, T, r, sigma, q=0.0):
        """
        Args:
            q: 连续股息率（年化，如0.02表示2%）
        """
        super().__init__(S, K, T, r, sigma)
        self.q = q
        # 重新计算 d1, d2
        self._d1 = (np.log(S / K) + (r - q + 0.5 * sigma**2) * T) / \
                   (sigma * np.sqrt(T))
        self._d2 = self._d1 - sigma * np.sqrt(T)

    def call_price(self):
        return (self.S * np.exp(-self.q * self.T) * norm.cdf(self._d1) -
                self.K * np.exp(-self.r * self.T) * norm.cdf(self._d2))

    def put_price(self):
        return (self.K * np.exp(-self.r * self.T) * norm.cdf(-self._d2) -
                self.S * np.exp(-self.q * self.T) * norm.cdf(-self._d1))

    def call_greeks(self):
        S, K, T, r, sigma, q = self.S, self.K, self.T, self.r, self.sigma, self.q
        d1, d2 = self._d1, self._d2
        sqrt_T = np.sqrt(T)
        nd1 = norm.pdf(d1)
        eqT = np.exp(-q * T)

        delta = eqT * norm.cdf(d1)
        gamma = eqT * nd1 / (S * sigma * sqrt_T)
        theta = (-(S * eqT * nd1 * sigma) / (2 * sqrt_T) +
                 q * S * eqT * norm.cdf(d1) -
                 r * K * np.exp(-r * T) * norm.cdf(d2)) / 365
        vega = S * eqT * nd1 * sqrt_T / 100
        rho = K * T * np.exp(-r * T) * norm.cdf(d2) / 100

        return OptionResult(
            price=self.call_price(),
            delta=delta, gamma=gamma,
            theta=theta, vega=vega, rho=rho
        )

# 示例：AAPL 期权（考虑股息）
bs_div = BlackScholesDividend(S=150, K=155, T=30/365, r=0.05, sigma=0.25, q=0.006)
call_div = bs_div.call_greeks()
print(f"含股息调整的看涨期权价格: ${call_div.price:.2f}")
print(f"Delta: {call_div.delta:.4f}")
```

---

## 二、二叉树模型（Binomial Tree）实现

### 2.1 模型原理

二叉树模型是一种离散化的期权定价方法，由 Cox、Ross 和 Rubinstein 于 1979 年提出。其核心思想是将期权的到期时间分割为多个时间步，每一步标的价格只能向上或向下变动。通过从到期日向前递推，可以计算出当前的期权价格。

二叉树模型的优势在于：
- 可以为美式期权定价（提前行权）
- 直观易理解
- 可以处理复杂的期权类型（如百慕大期权）

### 2.2 完整实现

```python
import numpy as np
from typing import Literal

class BinomialTree:
    """
    CRR 二叉树期权定价模型

    支持欧式和美式期权，看涨和看跌
    """

    def __init__(self, S: float, K: float, T: float,
                 r: float, sigma: float, N: int):
        """
        Args:
            S: 标的价格
            K: 执行价格
            T: 到期时间（年）
            r: 无风险利率
            sigma: 波动率
            N: 时间步数
        """
        self.S = S
        self.K = K
        self.T = T
        self.r = r
        self.sigma = sigma
        self.N = N

        # CRR 参数
        self.dt = T / N
        self.u = np.exp(sigma * np.sqrt(self.dt))   # 上涨因子
        self.d = np.exp(-sigma * np.sqrt(self.dt))  # 下跌因子
        self.p = (np.exp(r * self.dt) - d) / (u - d)  # 风险中性概率
        self.discount = np.exp(-r * self.dt)

    def price(self, option_type: Literal['call', 'put'] = 'call',
              style: Literal['european', 'american'] = 'european') -> float:
        """
        计算期权价格

        Args:
            option_type: 'call' 或 'put'
            style: 'european' 或 'american'
        """
        N = self.N
        u, d, p = self.u, self.d, self.p

        # 构建到期日的标的价格
        # 使用向量化计算提高效率
        stock_prices = self.S * u ** np.arange(N, -1, -1) * d ** np.arange(0, N+1)

        # 计算到期日期权内在价值
        if option_type == 'call':
            option_values = np.maximum(stock_prices - self.K, 0)
        else:
            option_values = np.maximum(self.K - stock_prices, 0)

        # 从到期日向前递推
        for i in range(N - 1, -1, -1):
            # 当前节点的标的价格
            stock_prices = self.S * u ** np.arange(i, -1, -1) * d ** np.arange(0, i+1)

            # 风险中性期望值
            option_values = self.discount * (
                p * option_values[:-1] + (1 - p) * option_values[1:]
            )

            # 美式期权：检查提前行权
            if style == 'american':
                if option_type == 'call':
                    intrinsic = np.maximum(stock_prices - self.K, 0)
                else:
                    intrinsic = np.maximum(self.K - stock_prices, 0)
                option_values = np.maximum(option_values, intrinsic)

        return option_values[0]

    def greeks(self, option_type='call', style='european',
               dS=0.01, dt_step=1):
        """
        使用有限差分法计算 Greeks

        Args:
            dS: 标的价格变动量（用于 Delta, Gamma）
            dt_step: 时间步长（天，用于 Theta）
        """
        S, K, T, r, sigma, N = self.S, self.K, self.T, self.r, self.sigma, self.N

        # 当前期权价格
        V = self.price(option_type, style)

        # Delta = dV/dS
        tree_up = BinomialTree(S + dS, K, T, r, sigma, N)
        tree_down = BinomialTree(S - dS, K, T, r, sigma, N)
        V_up = tree_up.price(option_type, style)
        V_down = tree_down.price(option_type, style)
        delta = (V_up - V_down) / (2 * dS)

        # Gamma = d^2V/dS^2
        gamma = (V_up - 2 * V + V_down) / (dS ** 2)

        # Theta = dV/dt
        T_short = max(T - dt_step / 365, 1/365)
        tree_short = BinomialTree(S, K, T_short, r, sigma, N)
        V_short = tree_short.price(option_type, style)
        theta = (V_short - V) / dt_step  # 每天的Theta

        # Vega = dV/dsigma
        tree_vol_up = BinomialTree(S, K, T, r, sigma + 0.01, N)
        tree_vol_down = BinomialTree(S, K, T, r, sigma - 0.01, N)
        vega = (tree_vol_up.price(option_type, style) -
                tree_vol_down.price(option_type, style)) / 2

        # Rho = dV/dr
        tree_r_up = BinomialTree(S, K, T, r + 0.01, sigma, N)
        tree_r_down = BinomialTree(S, K, T, r - 0.01, sigma, N)
        rho = (tree_r_up.price(option_type, style) -
               tree_r_down.price(option_type, style)) / 2

        return {
            'price': V,
            'delta': delta,
            'gamma': gamma,
            'theta': theta,
            'vega': vega / 100,   # 标准化为1%变动
            'rho': rho / 100      # 标准化为1%变动
        }

    def price_tree(self, option_type='call', style='european'):
        """返回完整的二叉树（用于可视化和调试）"""
        N = self.N
        u, d, p = self.u, self.d, self.p

        # 构建价格树
        stock_tree = np.zeros((N + 1, N + 1))
        for i in range(N + 1):
            for j in range(i + 1):
                stock_tree[j, i] = self.S * u**(i-j) * d**j

        # 构建期权价值树
        option_tree = np.zeros((N + 1, N + 1))

        # 到期日
        for j in range(N + 1):
            if option_type == 'call':
                option_tree[j, N] = max(stock_tree[j, N] - self.K, 0)
            else:
                option_tree[j, N] = max(self.K - stock_tree[j, N], 0)

        # 倒推
        for i in range(N - 1, -1, -1):
            for j in range(i + 1):
                hold = self.discount * (p * option_tree[j, i+1] +
                                        (1-p) * option_tree[j+1, i+1])
                if style == 'american':
                    if option_type == 'call':
                        exercise = max(stock_tree[j, i] - self.K, 0)
                    else:
                        exercise = max(self.K - stock_tree[j, i], 0)
                    option_tree[j, i] = max(hold, exercise)
                else:
                    option_tree[j, i] = hold

        return stock_tree, option_tree


# 使用示例
if __name__ == "__main__":
    # 参数
    S, K, T, r, sigma = 100, 100, 1.0, 0.05, 0.20

    # 比较不同步数的收敛性
    print("=== 二叉树模型收敛性分析 ===")
    print(f"{'步数':>6} {'欧式看涨':>10} {'美式看涨':>10} {'美式看跌':>10}")
    print("-" * 45)

    for N in [10, 50, 100, 200, 500]:
        bt = BinomialTree(S, K, T, r, sigma, N)
        eu_call = bt.price('call', 'european')
        am_call = bt.price('call', 'american')
        am_put = bt.price('put', 'american')
        print(f"{N:>6} {eu_call:>10.4f} {am_call:>10.4f} {am_put:>10.4f}")

    # 计算 Greeks
    bt = BinomialTree(S, K, T, r, sigma, N=200)
    greeks = bt.greeks('call', 'european')
    print(f"\n=== Greeks (欧式看涨, N=200) ===")
    for key, value in greeks.items():
        print(f"  {key}: {value:.6f}")

    # 美式看跌期权的 Greeks
    greeks_am = bt.greeks('put', 'american')
    print(f"\n=== Greeks (美式看跌, N=200) ===")
    for key, value in greeks_am.items():
        print(f"  {key}: {value:.6f}")
```

---

## 三、Greeks 的数值计算方法

除了从定价公式解析推导 Greeks 外，数值方法（有限差分法）在实际应用中更为通用，尤其适用于复杂期权的定价模型。

### 3.1 有限差分法（Finite Difference Method）

有限差分法的核心思想是通过微小的参数变动来近似计算偏导数。这种方法适用于任何可以计算价格的定价模型。

```python
import numpy as np
from scipy.stats import norm

def bs_price(S, K, T, r, sigma, option_type='call'):
    """Black-Scholes 定价函数"""
    if T <= 0:
        if option_type == 'call':
            return max(S - K, 0)
        else:
            return max(K - S, 0)

    d1 = (np.log(S / K) + (r + 0.5 * sigma**2) * T) / (sigma * np.sqrt(T))
    d2 = d1 - sigma * np.sqrt(T)

    if option_type == 'call':
        return S * norm.cdf(d1) - K * np.exp(-r * T) * norm.cdf(d2)
    else:
        return K * np.exp(-r * T) * norm.cdf(-d2) - S * norm.cdf(-d1)


class NumericalGreeks:
    """
    使用有限差分法计算 Greeks

    支持前向差分、后向差分和中心差分三种方法
    """

    def __init__(self, pricing_func, S, K, T, r, sigma, option_type='call'):
        self.pricing_func = pricing_func
        self.S = S
        self.K = K
        self.T = T
        self.r = r
        self.sigma = sigma
        self.option_type = option_type

        # 基准价格
        self.V = pricing_func(S, K, T, r, sigma, option_type)

    def _price(self, S=None, K=None, T=None, r=None, sigma=None):
        """计算给定参数下的期权价格"""
        return self.pricing_func(
            S if S is not None else self.S,
            K if K is not None else self.K,
            T if T is not None else self.T,
            r if r is not None else self.r,
            sigma if sigma is not None else self.sigma,
            self.option_type
        )

    def delta(self, h=0.01, method='central'):
        """
        Delta = dV/dS

        衡量标的价格变动1单位时期权价格的变动量
        """
        if method == 'forward':
            return (self._price(S=self.S + h) - self.V) / h
        elif method == 'backward':
            return (self.V - self._price(S=self.S - h)) / h
        else:  # central
            return (self._price(S=self.S + h) - self._price(S=self.S - h)) / (2 * h)

    def gamma(self, h=0.01):
        """
        Gamma = d^2V/dS^2 = d(Delta)/dS

        衡量 Delta 的变化速度，是期权凸性的度量
        """
        V_up = self._price(S=self.S + h)
        V_down = self._price(S=self.S - h)
        return (V_up - 2 * self.V + V_down) / (h ** 2)

    def theta(self, dt=1/365):
        """
        Theta = dV/dt

        衡量时间流逝对期权价格的影响（通常是负值）
        注意：这里计算的是每天的 Theta
        """
        if self.T - dt <= 0:
            return -self.V  # 到期时价值归零
        return (self._price(T=self.T - dt) - self.V) / dt

    def vega(self, d_sigma=0.01):
        """
        Vega = dV/d(sigma)

        衡量波动率变动对期权价格的影响
        注意：返回的是波动率变动1%时的价格变动
        """
        return (self._price(sigma=self.sigma + d_sigma) -
                self._price(sigma=self.sigma - d_sigma)) / 2

    def rho(self, dr=0.01):
        """
        Rho = dV/dr

        衡量利率变动对期权价格的影响
        注意：返回的是利率变动1%时的价格变动
        """
        return (self._price(r=self.r + dr) -
                self._price(r=self.r - dr)) / 2

    def vanna(self, d_sigma=0.01, h=0.01):
        """
        Vanna = d^2V/dS/dsigma = d(Delta)/d(sigma)

        衡量标的价格变动对 Vega 的影响，也称为 DdeltaDvol
        """
        V_up_vol = self._price(S=self.S + h, sigma=self.sigma + d_sigma)
        V_down_vol = self._price(S=self.S - h, sigma=self.sigma + d_sigma)
        V_up_base = self._price(S=self.S + h, sigma=self.sigma - d_sigma)
        V_down_base = self._price(S=self.S - h, sigma=self.sigma - d_sigma)

        delta_up = (V_up_vol - V_up_base) / (2 * d_sigma)
        delta_down = (V_down_vol - V_down_base) / (2 * d_sigma)

        return (delta_up - delta_down) / (2 * h)

    def volga(self, d_sigma=0.01):
        """
        Volga = d^2V/d(sigma)^2 = d(Vega)/d(sigma)

        衡量 Vega 对波动率变动的敏感度
        """
        V_up = self._price(sigma=self.sigma + d_sigma)
        V_down = self._price(sigma=self.sigma - d_sigma)
        return (V_up - 2 * self.V + V_down) / (d_sigma ** 2)

    def charm(self, dt=1/365, h=0.01):
        """
        Charm = d(Delta)/dt = d^2V/dS/dt

        衡量 Delta 随时间的变化率
        """
        delta_now = self.delta(h)
        delta_future = NumericalGreeks(
            self.pricing_func, self.S, self.K,
            self.T - dt, self.r, self.sigma, self.option_type
        ).delta(h)
        return (delta_future - delta_now) / dt

    def all_greeks(self):
        """计算所有 Greeks"""
        return {
            'price': self.V,
            'delta': self.delta(),
            'gamma': self.gamma(),
            'theta': self.theta(),
            'vega': self.vega(),
            'rho': self.rho(),
            'vanna': self.vanna(),
            'volga': self.volga(),
            'charm': self.charm()
        }


# 使用示例
if __name__ == "__main__":
    S, K, T, r, sigma = 150, 155, 30/365, 0.05, 0.25

    print("=== 数值 Greeks 计算 ===\n")

    # 看涨期权
    ng_call = NumericalGreeks(bs_price, S, K, T, r, sigma, 'call')
    greeks_call = ng_call.all_greeks()

    print("看涨期权 Greeks:")
    for name, value in greeks_call.items():
        print(f"  {name:>8}: {value:>10.6f}")

    # 看跌期权
    ng_put = NumericalGreeks(bs_price, S, K, T, r, sigma, 'put')
    greeks_put = ng_put.all_greeks()

    print("\n看跌期权 Greeks:")
    for name, value in greeks_put.items():
        print(f"  {name:>8}: {value:>10.6f}")
```

### 3.2 Greeks 的实际解读

理解每个 Greek 的实际含义对于期权交易至关重要：

| Greek | 符号 | 含义 | 典型应用 |
|-------|------|------|---------|
| Delta | Δ | 标的价格变动$1时期权价格变动量 | 对冲比率、方向性敞口 |
| Gamma | Γ | Delta 的变化率 | Gamma Scalping、凸性收益 |
| Theta | Θ | 每天的时间衰减 | 卖方策略的时间收益 |
| Vega | ν | 波动率变动1%时期权价格变动量 | 波动率交易的核心指标 |
| Rho | ρ | 利率变动1%时期权价格变动量 | 长期期权的利率风险 |
| Vanna | — | Delta 对波动率的敏感度 | 波动率偏斜交易 |
| Volga | — | Vega 对波动率的敏感度 | Vega 凸性 |
| Charm | — | Delta 对时间的敏感度 | 隔夜 Delta 对冲调整 |

---

## 四、隐含波动率的反推计算

### 4.1 Newton-Raphson 方法

隐含波动率（Implied Volatility, IV）是使期权模型价格等于市场价格的波动率参数。Newton-Raphson 方法利用 Vega 作为导数来快速收敛。

```python
import numpy as np
from scipy.stats import norm

def implied_vol_newton(market_price, S, K, T, r, option_type='call',
                       initial_guess=0.20, tolerance=1e-8, max_iterations=100):
    """
    使用 Newton-Raphson 方法计算隐含波动率

    收敛条件：|f(sigma)| < tolerance
    其中 f(sigma) = BS_price(sigma) - market_price

    Args:
        market_price: 市场期权价格
        S: 标的价格
        K: 执行价格
        T: 到期时间（年）
        r: 无风险利率
        option_type: 'call' 或 'put'
        initial_guess: 初始猜测值
        tolerance: 收敛容差
        max_iterations: 最大迭代次数

    Returns:
        (implied_vol, iterations, converged)
    """
    sigma = initial_guess

    for i in range(max_iterations):
        # 计算当前 sigma 下的价格和 Vega
        d1 = (np.log(S / K) + (r + 0.5 * sigma**2) * T) / (sigma * np.sqrt(T))
        d2 = d1 - sigma * np.sqrt(T)

        if option_type == 'call':
            price = S * norm.cdf(d1) - K * np.exp(-r * T) * norm.cdf(d2)
        else:
            price = K * np.exp(-r * T) * norm.cdf(-d2) - S * norm.cdf(-d1)

        # Vega (注意这里没有除以100)
        vega = S * norm.pdf(d1) * np.sqrt(T)

        # 避免除零
        if abs(vega) < 1e-15:
            return sigma, i, False

        # Newton-Raphson 更新
        diff = price - market_price
        sigma_new = sigma - diff / vega

        # 确保 sigma 为正
        sigma_new = max(sigma_new, 1e-6)

        # 检查收敛
        if abs(diff) < tolerance:
            return sigma_new, i + 1, True

        sigma = sigma_new

    return sigma, max_iterations, False


def implied_vol_brent(market_price, S, K, T, r, option_type='call'):
    """
    使用 Brent 方法计算隐含波动率（更稳健）

    Brent 方法结合了二分法、割线法和逆二次插值，
    虽然可能比 Newton-Raphson 慢，但更加稳健
    """
    from scipy.optimize import brentq

    def objective(sigma):
        d1 = (np.log(S / K) + (r + 0.5 * sigma**2) * T) / (sigma * np.sqrt(T))
        d2 = d1 - sigma * np.sqrt(T)

        if option_type == 'call':
            price = S * norm.cdf(d1) - K * np.exp(-r * T) * norm.cdf(d2)
        else:
            price = K * np.exp(-r * T) * norm.cdf(-d2) - S * norm.cdf(-d1)

        return price - market_price

    try:
        iv = brentq(objective, 0.001, 10.0, xtol=1e-10)
        return iv
    except ValueError:
        return np.nan


def batch_implied_vol(prices, strikes, S, T, r, option_type='call'):
    """
    批量计算隐含波动率

    Args:
        prices: 期权价格数组
        strikes: 执行价格数组
        S: 标的价格
        T: 到期时间
        r: 无风险利率
        option_type: 'call' 或 'put'

    Returns:
        隐含波动率数组
    """
    ivs = []
    for price, K in zip(prices, strikes):
        iv, iters, converged = implied_vol_newton(
            price, S, K, T, r, option_type
        )
        if not converged:
            # 回退到 Brent 方法
            iv = implied_vol_brent(price, S, K, T, r, option_type)
        ivs.append(iv)
    return np.array(ivs)


# 使用示例
if __name__ == "__main__":
    S = 150
    T = 30 / 365
    r = 0.05

    # 市场数据
    strikes = np.array([130, 135, 140, 145, 150, 155, 160, 165, 170])
    market_prices = np.array([21.50, 17.20, 13.10, 9.50, 6.50, 4.20, 2.50, 1.40, 0.70])

    print("=== 隐含波动率计算 ===")
    print(f"{'执行价格':>8} {'市场价格':>8} {'IV (Newton)':>12} {'迭代次数':>8}")
    print("-" * 45)

    ivs = []
    for K, price in zip(strikes, market_prices):
        iv, iters, converged = implied_vol_newton(price, S, K, T, r)
        ivs.append(iv)
        status = "OK" if converged else "FAIL"
        print(f"${K:>7} ${price:>7.2f} {iv:>11.2%} {iters:>7} {status}")

    print(f"\nIV 范围: {min(ivs):.2%} - {max(ivs):.2%}")
    print(f"IV 均值: {np.mean(ivs):.2%}")
```

### 4.2 初始猜测值的优化

Newton-Raphson 方法的收敛速度高度依赖于初始猜测值。以下是几种常用的初始猜测值策略：

```python
def smart_initial_guess(market_price, S, K, T, r, option_type='call'):
    """
    智能初始猜测值

    使用 Brenner-Subrahmanyam (1988) 近似公式作为初始猜测
    对于平值期权（ATM），该近似非常准确
    """
    # Brenner-Subrahmanyam 近似
    if option_type == 'call':
        # ATM call 近似: sigma ≈ sqrt(2*pi/T) * C/S
        atm_approx = np.sqrt(2 * np.pi / T) * market_price / S
    else:
        # ATM put 近似
        atm_approx = np.sqrt(2 * np.pi / T) * market_price / S

    # 对于深度实值或虚值期权，需要调整
    moneyness = np.log(S / K)

    if abs(moneyness) < 0.1:  # 近似 ATM
        return max(atm_approx, 0.10)
    else:
        # 使用 Jäckel (2015) 的更精确近似
        # 这里简化处理，使用二分法快速获得一个合理估计
        from scipy.optimize import brentq

        def obj(sigma):
            d1 = (np.log(S / K) + (r + 0.5 * sigma**2) * T) / (sigma * np.sqrt(T))
            d2 = d1 - sigma * np.sqrt(T)
            if option_type == 'call':
                p = S * norm.cdf(d1) - K * np.exp(-r * T) * norm.cdf(d2)
            else:
                p = K * np.exp(-r * T) * norm.cdf(-d2) - S * norm.cdf(-d1)
            return p - market_price

        try:
            return brentq(obj, 0.01, 5.0, xtol=1e-4)  # 粗略求解
        except:
            return 0.30  # 默认值
```

---

## 五、波动率曲面拟合

### 5.1 波动率曲面的概念

波动率曲面（Volatility Surface）是隐含波动率关于执行价格和到期时间的三维曲面。由于 Black-Scholes 模型假设波动率为常数，但实际市场中不同执行价格和到期日的期权具有不同的隐含波动率，这就形成了波动率微笑（Volatility Smile）和波动率期限结构（Term Structure）。

### 5.2 三次样条插值

```python
import numpy as np
from scipy.interpolate import CubicSpline, RectBivariateSpline
import matplotlib.pyplot as plt

class VolatilitySurface:
    """
    波动率曲面拟合与插值

    使用三次样条（Cubic Spline）进行平滑插值
    """

    def __init__(self, strikes, expiries, iv_matrix):
        """
        Args:
            strikes: 执行价格数组 (M,)
            expiries: 到期时间数组（年） (N,)
            iv_matrix: 隐含波动率矩阵 (N, M)
        """
        self.strikes = np.array(strikes)
        self.expiries = np.array(expiries)
        self.iv_matrix = np.array(iv_matrix)

        # 创建插值器
        self.interpolator = RectBivariateSpline(
            self.expiries, self.strikes, self.iv_matrix,
            kx=min(3, len(self.expiries)-1),
            ky=min(3, len(self.strikes)-1)
        )

    def get_iv(self, strike, expiry):
        """获取指定执行价格和到期时间的隐含波动率"""
        return float(self.interpolator(expiry, strike))

    def get_smile(self, expiry, strike_range=None):
        """获取指定到期时间的波动率微笑"""
        if strike_range is None:
            strike_range = np.linspace(
                self.strikes.min(), self.strikes.max(), 100
            )
        ivs = self.interpolator(expiry, strike_range)[0]
        return strike_range, ivs

    def get_term_structure(self, strike, expiry_range=None):
        """获取指定执行价格的期限结构"""
        if expiry_range is None:
            expiry_range = np.linspace(
                self.expiries.min(), self.expiries.max(), 100
            )
        ivs = self.interpolator(expiry_range, strike)[:, 0]
        return expiry_range, ivs

    def plot_surface(self):
        """绘制波动率曲面"""
        strike_grid = np.linspace(self.strikes.min(), self.strikes.max(), 50)
        expiry_grid = np.linspace(self.expiries.min(), self.expiries.max(), 50)
        K_mesh, T_mesh = np.meshgrid(strike_grid, expiry_grid)
        IV_mesh = self.interpolator(expiry_grid, strike_grid)

        fig = plt.figure(figsize=(12, 8))
        ax = fig.add_subplot(111, projection='3d')
        surf = ax.plot_surface(K_mesh, T_mesh * 365, IV_mesh,
                               cmap='viridis', alpha=0.8)
        ax.set_xlabel('执行价格')
        ax.set_ylabel('到期天数')
        ax.set_zlabel('隐含波动率')
        ax.set_title('波动率曲面')
        fig.colorbar(surf, shrink=0.5, aspect=5)
        plt.tight_layout()
        return fig

    def extrapolate(self, strike, expiry, method='nearest'):
        """
        外溢处理：当请求超出数据范围时的处理策略

        method:
            'nearest' - 使用最近的数据点
            'flat' - 使用边界值（常数外溢）
            'linear' - 线性外溢
        """
        if method == 'nearest':
            # 找最近的已知数据点
            k_idx = np.argmin(np.abs(self.strikes - strike))
            t_idx = np.argmin(np.abs(self.expiries - expiry))
            return self.iv_matrix[t_idx, k_idx]
        elif method == 'flat':
            # 限制在数据范围内
            strike_clipped = np.clip(strike,
                                      self.strikes.min(),
                                      self.strikes.max())
            expiry_clipped = np.clip(expiry,
                                      self.expiries.min(),
                                      self.expiries.max())
            return float(self.interpolator(expiry_clipped, strike_clipped))
        else:
            return float(self.interpolator(expiry, strike))


# 使用示例
if __name__ == "__main__":
    # 模拟市场数据
    strikes = np.array([90, 95, 100, 105, 110])
    expiries = np.array([7, 30, 60, 90, 180]) / 365

    # 生成波动率微笑 + 期限结构
    iv_data = np.array([
        [0.28, 0.24, 0.20, 0.22, 0.26],  # 7天
        [0.26, 0.23, 0.19, 0.21, 0.25],  # 30天
        [0.25, 0.22, 0.19, 0.20, 0.24],  # 60天
        [0.24, 0.22, 0.18, 0.20, 0.23],  # 90天
        [0.23, 0.21, 0.18, 0.19, 0.22],  # 180天
    ])

    surface = VolatilitySurface(strikes, expiries, iv_data)

    # 查询特定点的 IV
    print(f"K=102, T=45天: IV = {surface.get_iv(102, 45/365):.2%}")

    # 获取波动率微笑
    k_range, smile = surface.get_smile(30/365)
    print(f"\n30天波动率微笑:")
    for k, iv in zip(k_range[::10], smile[::10]):
        print(f"  K={k:.0f}: IV={iv:.2%}")
```

---

## 六、蒙特卡洛模拟（Monte Carlo）定价

### 6.1 基本原理

蒙特卡洛方法通过大量随机模拟标的价格路径来估计期权价格。它特别适用于路径依赖型期权（如亚式期权、障碍期权）和多标的期权。

```python
import numpy as np
from scipy.stats import norm

class MonteCarloPricer:
    """
    蒙特卡洛期权定价引擎

    支持欧式期权、亚式期权和障碍期权
    """

    def __init__(self, S, K, T, r, sigma, n_paths=100000, n_steps=252,
                 seed=42):
        self.S = S
        self.K = K
        self.T = T
        self.r = r
        self.sigma = sigma
        self.n_paths = n_paths
        self.n_steps = n_steps
        self.seed = seed

    def _simulate_paths(self, antithetic=True):
        """
        模拟标的价格路径

        使用几何布朗运动（GBM）：
        dS = mu*S*dt + sigma*S*dW

        Args:
            antithetic: 是否使用对偶变量法减少方差
        """
        np.random.seed(self.seed)
        dt = self.T / self.n_steps

        if antithetic:
            # 对偶变量法：生成一半路径，另一半为镜像
            half_paths = self.n_paths // 2
            Z = np.random.standard_normal((self.n_steps, half_paths))
            Z = np.concatenate([Z, -Z], axis=1)
        else:
            Z = np.random.standard_normal((self.n_steps, self.n_paths))

        # 几何布朗运动
        drift = (self.r - 0.5 * self.sigma**2) * dt
        diffusion = self.sigma * np.sqrt(dt) * Z
        log_returns = drift + diffusion

        # 价格路径矩阵 (n_steps+1, n_paths)
        log_prices = np.zeros((self.n_steps + 1, self.n_paths))
        log_prices[0] = np.log(self.S)
        log_prices[1:] = np.log(self.S) + np.cumsum(log_returns, axis=0)

        return np.exp(log_prices)

    def european_option(self, option_type='call'):
        """
        欧式期权定价

        Args:
            option_type: 'call' 或 'put'

        Returns:
            (price, standard_error)
        """
        paths = self._simulate_paths()
        S_T = paths[-1]  # 到期价格

        if option_type == 'call':
            payoffs = np.maximum(S_T - self.K, 0)
        else:
            payoffs = np.maximum(self.K - S_T, 0)

        discount = np.exp(-self.r * self.T)
        discounted_payoffs = discount * payoffs

        price = np.mean(discounted_payoffs)
        std_error = np.std(discounted_payoffs) / np.sqrt(self.n_paths)

        return price, std_error

    def asian_option(self, option_type='call', average_type='arithmetic'):
        """
        亚式期权定价（Asian Option）

        payoff = max(AVG(S) - K, 0) for call
        payoff = max(K - AVG(S), 0) for put

        Args:
            average_type: 'arithmetic' 或 'geometric'
        """
        paths = self._simulate_paths()

        # 计算路径平均值（不包括初始价格）
        if average_type == 'arithmetic':
            avg_prices = np.mean(paths[1:], axis=0)
        else:  # geometric
            avg_prices = np.exp(np.mean(np.log(paths[1:]), axis=0))

        if option_type == 'call':
            payoffs = np.maximum(avg_prices - self.K, 0)
        else:
            payoffs = np.maximum(self.K - avg_prices, 0)

        discount = np.exp(-self.r * self.T)
        discounted_payoffs = discount * payoffs

        price = np.mean(discounted_payoffs)
        std_error = np.std(discounted_payoffs) / np.sqrt(self.n_paths)

        return price, std_error

    def barrier_option(self, barrier, barrier_type='down-and-out',
                       option_type='call'):
        """
        障碍期权定价（Barrier Option）

        barrier_type:
            'down-and-out': 价格触及下障碍则作废
            'down-and-in': 价格触及下障碍才生效
            'up-and-out': 价格触及上障碍则作废
            'up-and-in': 价格触及上障碍才生效
        """
        paths = self._simulate_paths()
        S_T = paths[-1]

        # 检查是否触及障碍
        if 'down' in barrier_type:
            touched = np.any(paths <= barrier, axis=0)
        else:  # up
            touched = np.any(paths >= barrier, axis=0)

        # 基础期权 payoff
        if option_type == 'call':
            payoffs = np.maximum(S_T - self.K, 0)
        else:
            payoffs = np.maximum(self.K - S_T, 0)

        # 应用障碍条件
        if 'out' in barrier_type:
            payoffs = np.where(touched, 0, payoffs)
        else:  # in
            payoffs = np.where(touched, payoffs, 0)

        discount = np.exp(-self.r * self.T)
        discounted_payoffs = discount * payoffs

        price = np.mean(discounted_payoffs)
        std_error = np.std(discounted_payoffs) / np.sqrt(self.n_paths)

        return price, std_error

    def price_with_confidence(self, option_type='call', confidence=0.95):
        """带置信区间的定价"""
        price, std_error = self.european_option(option_type)
        z = norm.ppf(1 - (1 - confidence) / 2)
        ci_lower = price - z * std_error
        ci_upper = price + z * std_error

        return {
            'price': price,
            'std_error': std_error,
            'ci_lower': ci_lower,
            'ci_upper': ci_upper,
            'confidence': confidence
        }


# 使用示例
if __name__ == "__main__":
    S, K, T, r, sigma = 100, 100, 1.0, 0.05, 0.20

    mc = MonteCarloPricer(S, K, T, r, sigma, n_paths=500000, n_steps=252)

    # 欧式看涨
    result = mc.price_with_confidence('call')
    print("=== 蒙特卡洛定价结果 ===")
    print(f"欧式看涨: ${result['price']:.4f}")
    print(f"标准误差: ${result['std_error']:.4f}")
    print(f"95%置信区间: [${result['ci_lower']:.4f}, ${result['ci_upper']:.4f}]")

    # 亚式看涨
    asian_price, asian_se = mc.asian_option('call', 'arithmetic')
    print(f"\n亚式看涨(算术平均): ${asian_price:.4f} (SE: ${asian_se:.4f})")

    # 障碍期权
    barrier_price, barrier_se = mc.barrier_option(
        barrier=80, barrier_type='down-and-out', option_type='call'
    )
    print(f"下跌敲出看涨 (B=80): ${barrier_price:.4f} (SE: ${barrier_se:.4f})")

    # 与 Black-Scholes 解析解比较
    d1 = (np.log(S/K) + (r + 0.5*sigma**2)*T) / (sigma*np.sqrt(T))
    d2 = d1 - sigma*np.sqrt(T)
    bs_price = S*norm.cdf(d1) - K*np.exp(-r*T)*norm.cdf(d2)
    print(f"\nBlack-Scholes 解析解: ${bs_price:.4f}")
    print(f"蒙特卡洛偏差: ${result['price'] - bs_price:.4f}")
```

### 6.2 方差缩减技术

蒙特卡洛方法的一个主要问题是收敛速度慢（标准误差与 1/sqrt(N) 成正比）。方差缩减技术可以在不增加模拟次数的情况下提高估计精度。

```python
class VarianceReduction:
    """方差缩减技术集合"""

    @staticmethod
    def control_variate(S, K, T, r, sigma, n_paths=100000):
        """
        控制变量法（Control Variate）

        使用标的价格 S_T 作为控制变量，
        因为 E[S_T] = S * exp(rT) 已知
        """
        np.random.seed(42)
        dt = T / 252
        Z = np.random.standard_normal((252, n_paths))

        log_returns = (r - 0.5*sigma**2)*dt + sigma*np.sqrt(dt)*Z
        S_T = S * np.exp(np.sum(log_returns, axis=0))

        # 期权 payoff
        payoffs = np.maximum(S_T - K, 0)

        # 控制变量：调整后的 payoff
        # E[S_T] = S * exp(rT)
        S_T_expected = S * np.exp(r * T)
        beta = np.cov(payoffs, S_T)[0, 1] / np.var(S_T)
        adjusted_payoffs = payoffs - beta * (S_T - S_T_expected)

        price = np.exp(-r*T) * np.mean(adjusted_payoffs)
        return price

    @staticmethod
    def importance_sampling(S, K, T, r, sigma, n_paths=100000):
        """
        重要性采样（Importance Sampling）

        通过改变采样分布，使更多样本落在期权实值区域
        """
        np.random.seed(42)

        # 计算最优漂移量
        mu_star = np.log(K / S) / T - (r - 0.5*sigma**2)

        dt = T / 252
        Z = np.random.standard_normal((252, n_paths))

        # 使用偏移后的分布
        log_returns = (r - 0.5*sigma**2)*dt + sigma*np.sqrt(dt)*Z
        log_returns_shifted = log_returns + mu_star * dt

        S_T = S * np.exp(np.sum(log_returns_shifted, axis=0))
        payoffs = np.maximum(S_T - K, 0)

        # 计算似然比（Likelihood Ratio）
        Z_original = (np.sum(log_returns_shifted, axis=0) -
                      (r - 0.5*sigma**2)*T) / (sigma*np.sqrt(T))
        Z_shifted = Z_original - mu_star*np.sqrt(T)/sigma

        likelihood_ratio = np.exp(
            -0.5 * (Z_shifted**2 - Z_original**2)
        )

        weighted_payoffs = payoffs * likelihood_ratio
        price = np.exp(-r*T) * np.mean(weighted_payoffs)
        return price


# 比较不同方法的效率
print("=== 方差缩减技术比较 ===")
S, K, T, r, sigma = 100, 100, 1.0, 0.05, 0.20

mc_basic = MonteCarloPricer(S, K, T, r, sigma, n_paths=100000)
basic_price, basic_se = mc_basic.european_option('call')

cv_price = VarianceReduction.control_variate(S, K, T, r, sigma)
is_price = VarianceReduction.importance_sampling(S, K, T, r, sigma)

# 解析解
d1 = (np.log(S/K) + (r + 0.5*sigma**2)*T) / (sigma*np.sqrt(T))
d2 = d1 - sigma*np.sqrt(T)
bs_price = S*norm.cdf(d1) - K*np.exp(-r*T)*norm.cdf(d2)

print(f"Black-Scholes:   ${bs_price:.4f}")
print(f"基础蒙特卡洛:    ${basic_price:.4f} (偏差: ${basic_price-bs_price:.4f})")
print(f"控制变量法:      ${cv_price:.4f} (偏差: ${cv_price-bs_price:.4f})")
print(f"重要性采样:      ${is_price:.4f} (偏差: ${is_price-bs_price:.4f})")
```

---

## 七、完整的期权定价计算器

将以上所有内容整合为一个完整的、可直接使用的期权定价计算器：

```python
"""
完整期权定价计算器
支持多种定价模型和 Greeks 计算
"""
import numpy as np
from scipy.stats import norm
from scipy.optimize import brentq
from dataclasses import dataclass
from typing import Optional, Literal

@dataclass
class FullOptionResult:
    """完整的期权定价结果"""
    # 输入参数
    S: float
    K: float
    T: float
    r: float
    sigma: float
    option_type: str
    style: str

    # 定价结果
    price: float
    intrinsic_value: float
    time_value: float

    # Greeks
    delta: float
    gamma: float
    theta: float
    vega: float
    rho: float

    # 高阶 Greeks
    vanna: Optional[float] = None
    volga: Optional[float] = None
    charm: Optional[float] = None


class OptionCalculator:
    """统一的期权计算器"""

    def __init__(self, S, K, T, r, sigma):
        self.S = S
        self.K = K
        self.T = T
        self.r = r
        self.sigma = sigma

    def calculate(self, option_type='call', style='european',
                  method='black_scholes', **kwargs) -> FullOptionResult:
        """
        统一计算接口

        Args:
            option_type: 'call' 或 'put'
            style: 'european' 或 'american'
            method: 'black_scholes', 'binomial', 或 'monte_carlo'
        """
        if method == 'black_scholes' and style == 'european':
            return self._bs_calculate(option_type)
        elif method == 'binomial':
            N = kwargs.get('n_steps', 200)
            return self._binomial_calculate(option_type, style, N)
        elif method == 'monte_carlo':
            n_paths = kwargs.get('n_paths', 100000)
            return self._mc_calculate(option_type, n_paths)
        else:
            raise ValueError(f"不支持的组合: {method}/{style}")

    def _bs_calculate(self, option_type) -> FullOptionResult:
        S, K, T, r, sigma = self.S, self.K, self.T, self.r, self.sigma
        sqrt_T = np.sqrt(T)
        d1 = (np.log(S/K) + (r + 0.5*sigma**2)*T) / (sigma*sqrt_T)
        d2 = d1 - sigma*sqrt_T
        nd1 = norm.pdf(d1)
        exp_rT = np.exp(-r*T)

        if option_type == 'call':
            price = S*norm.cdf(d1) - K*exp_rT*norm.cdf(d2)
            intrinsic = max(S - K, 0)
            delta = norm.cdf(d1)
            theta = (-(S*nd1*sigma)/(2*sqrt_T) - r*K*exp_rT*norm.cdf(d2))/365
            rho = K*T*exp_rT*norm.cdf(d2)/100
        else:
            price = K*exp_rT*norm.cdf(-d2) - S*norm.cdf(-d1)
            intrinsic = max(K - S, 0)
            delta = norm.cdf(d1) - 1
            theta = (-(S*nd1*sigma)/(2*sqrt_T) + r*K*exp_rT*norm.cdf(-d2))/365
            rho = -K*T*exp_rT*norm.cdf(-d2)/100

        gamma = nd1 / (S*sigma*sqrt_T)
        vega = S*nd1*sqrt_T / 100

        # 高阶 Greeks
        vanna = -nd1 * d2 / sigma / 100
        volga = vega * d1 * d2 / sigma
        charm_val = -(nd1 * (2*r*T - d2*sigma*sqrt_T)) / (2*T*sigma*sqrt_T)

        return FullOptionResult(
            S=S, K=K, T=T, r=r, sigma=sigma,
            option_type=option_type, style='european',
            price=price, intrinsic_value=intrinsic,
            time_value=price - intrinsic,
            delta=delta, gamma=gamma, theta=theta,
            vega=vega, rho=rho,
            vanna=vanna, volga=volga, charm=charm_val
        )

    def implied_vol(self, market_price, option_type='call'):
        """反推隐含波动率"""
        def obj(sigma):
            self.sigma = sigma
            result = self._bs_calculate(option_type)
            return result.price - market_price

        try:
            iv = brentq(obj, 0.001, 10.0, xtol=1e-10)
            self.sigma = iv  # 恢复
            return iv
        except:
            return np.nan

    @staticmethod
    def display(result: FullOptionResult):
        """格式化输出结果"""
        print(f"\n{'='*55}")
        print(f"  期权定价报告")
        print(f"{'='*55}")
        print(f"  标的价格:  ${result.S:.2f}")
        print(f"  执行价格:  ${result.K:.2f}")
        print(f"  到期天数:  {int(result.T*365)}")
        print(f"  波动率:    {result.sigma:.2%}")
        print(f"  无风险利率: {result.r:.2%}")
        print(f"  类型:      {result.option_type.upper()} ({result.style})")
        print(f"{'='*55}")
        print(f"  价格:      ${result.price:.4f}")
        print(f"  内在价值:  ${result.intrinsic_value:.4f}")
        print(f"  时间价值:  ${result.time_value:.4f}")
        print(f"{'='*55}")
        print(f"  Delta:     {result.delta:>10.6f}")
        print(f"  Gamma:     {result.gamma:>10.6f}")
        print(f"  Theta:     {result.theta:>10.6f} /天")
        print(f"  Vega:      {result.vega:>10.6f} /1%")
        print(f"  Rho:       {result.rho:>10.6f} /1%")
        if result.vanna is not None:
            print(f"  Vanna:     {result.vanna:>10.6f}")
        if result.volga is not None:
            print(f"  Volga:     {result.volga:>10.6f}")
        if result.charm is not None:
            print(f"  Charm:     {result.charm:>10.6f} /天")
        print(f"{'='*55}")


# 主程序
if __name__ == "__main__":
    calc = OptionCalculator(S=150, K=155, T=30/365, r=0.05, sigma=0.25)

    # 看涨期权
    call_result = calc.calculate('call', 'european', 'black_scholes')
    OptionCalculator.display(call_result)

    # 看跌期权
    put_result = calc.calculate('put', 'european', 'black_scholes')
    OptionCalculator.display(put_result)

    # 隐含波动率反推
    market_price = 3.50
    iv = calc.implied_vol(market_price, 'call')
    print(f"\n市场价格 ${market_price} -> 隐含波动率: {iv:.2%}")
```

---

## 小结

本章详细讲解了期权定价的核心内容：

- **Black-Scholes 模型**：解析解实现，包括股息调整版本
- **二叉树模型**：支持美式期权定价，理解离散化定价方法
- **Greeks 计算**：从解析推导到数值方法，涵盖一阶和高阶 Greeks
- **隐含波动率**：Newton-Raphson 和 Brent 方法，以及智能初始猜测值
- **波动率曲面**：三次样条插值与外溢处理
- **蒙特卡洛模拟**：欧式/亚式/障碍期权定价，方差缩减技术

掌握这些定价工具后，你就拥有了分析和交易期权的核心能力。

### 思考题

1. 修改二叉树模型，使其能够处理离散股息（Discrete Dividend），并在 AAPL 期权上验证结果。
2. 实现一个波动率曲面拟合器，使用 SVI（Stochastic Volatility Inspired）参数化方法代替纯插值。
3. 比较蒙特卡洛模拟中不同路径数（10K/50K/100K/500K）对定价精度的影响，并绘制收敛图。

---

**下一章：** [03-backtesting.md —— 期权策略回测框架](section03-backtesting.md)




---
**相关笔记：** [[Python期权库入门]], [[Black-Scholes]]
**返回：** [[chapter00-python-ai]] | [[chapter01-主页]]
