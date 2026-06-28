---
type: concept
topic: Python+AI
aliases: [Signal Generation, Strategy Development, 信号生成]
status: done
created: "2026-06-28"
updated: "2026-06-28"
tags: [期权, Python, 策略]
related: "[[回测框架搭建]], [[IBKR API自动交易]]"
---

# 信号生成与策略开发

## 引言

在上一章中，我们构建了回测框架。本章将深入探讨如何生成有效的交易信号——这是连接市场分析和交易执行的桥梁。对于期权交易而言，信号不仅包括方向判断，更重要的是波动率判断、时间选择和合约选择。我们将覆盖技术指标信号、波动率信号、统计信号等多种方法，并介绍完整的策略开发流程。

---

## 一、信号生成概述

### 1.1 从市场数据到交易信号

交易信号的生成是一个从原始数据到决策的转化过程：

```
市场数据 → 指标计算 → 信号判断 → 合约选择 → 下单执行
  │           │           │           │
  │       技术指标       入场条件     Delta选择
  │       波动率指标     出场条件     到期日选择
  │       统计指标       仓位大小     行权价选择
  └───────────────────────────────────────────┘
```

### 1.2 期权信号的特殊性

与股票交易信号不同，期权信号需要考虑更多维度：

| 维度 | 股票信号 | 期权信号 |
|------|---------|---------|
| 方向 | 做多/做空 | 4种基本方向 + 组合 |
| 波动率 | 通常忽略 | 核心决策因素 |
| 时间 | 不敏感 | Theta 衰减是关键 |
| 合约 | 单一标的 | 多执行价、多到期日 |
| 杠杆 | 固定 | Delta 动态变化 |

### 1.3 信号质量评估

一个好的交易信号应该具备以下特征：

1. **统计显著性**：信号的预测能力应该在统计上显著
2. **稳定性**：信号在不同时间段应该保持一致的表现
3. **可执行性**：信号出现时应该有足够的流动性来执行
4. **低相关性**：不同信号之间应该尽量独立

---

## 二、技术指标信号

### 2.1 RSI（相对强弱指数）与期权

RSI 是最经典的动量指标之一。在期权交易中，RSI 可以帮助判断超买/超卖条件，从而选择卖出或买入期权的时机。

```python
import pandas as pd
import numpy as np
from typing import Tuple, Optional

class TechnicalSignals:
    """技术指标信号生成器"""

    @staticmethod
    def rsi(prices: pd.Series, period: int = 14) -> pd.Series:
        """
        计算 RSI（相对强弱指数）

        RSI = 100 - 100 / (1 + RS)
        RS = 平均上涨幅度 / 平均下跌幅度

        Args:
            prices: 收盘价序列
            period: 计算周期（默认14）

        Returns:
            RSI 序列（0-100）
        """
        delta = prices.diff()
        gain = delta.where(delta > 0, 0)
        loss = (-delta).where(delta < 0, 0)

        # 使用指数移动平均
        avg_gain = gain.ewm(alpha=1/period, min_periods=period).mean()
        avg_loss = loss.ewm(alpha=1/period, min_periods=period).mean()

        rs = avg_gain / avg_loss
        rsi = 100 - (100 / (1 + rs))

        return rsi

    @staticmethod
    def macd(prices: pd.Series, fast: int = 12,
             slow: int = 26, signal: int = 9) -> pd.DataFrame:
        """
        计算 MACD（移动平均收敛发散指标）

        MACD线 = EMA(12) - EMA(26)
        信号线 = EMA(MACD线, 9)
        柱状图 = MACD线 - 信号线
        """
        ema_fast = prices.ewm(span=fast, adjust=False).mean()
        ema_slow = prices.ewm(span=slow, adjust=False).mean()
        macd_line = ema_fast - ema_slow
        signal_line = macd_line.ewm(span=signal, adjust=False).mean()
        histogram = macd_line - signal_line

        return pd.DataFrame({
            'macd': macd_line,
            'signal': signal_line,
            'histogram': histogram
        })

    @staticmethod
    def bollinger_bands(prices: pd.Series, period: int = 20,
                         std_dev: float = 2.0) -> pd.DataFrame:
        """
        计算布林带（Bollinger Bands）

        中轨 = SMA(20)
        上轨 = 中轨 + 2 * 标准差
        下轨 = 中轨 - 2 * 标准差

        布林带宽度可以作为波动率的代理指标
        """
        middle = prices.rolling(window=period).mean()
        std = prices.rolling(window=period).std()
        upper = middle + std_dev * std
        lower = middle - std_dev * std

        # 布林带宽度（标准化）
        bandwidth = (upper - lower) / middle

        # 百分比带宽（%B）：价格在布林带中的位置
        pct_b = (prices - lower) / (upper - lower)

        return pd.DataFrame({
            'upper': upper,
            'middle': middle,
            'lower': lower,
            'bandwidth': bandwidth,
            'pct_b': pct_b
        })

    @staticmethod
    def atr(high: pd.Series, low: pd.Series,
            close: pd.Series, period: int = 14) -> pd.Series:
        """
        计算 ATR（平均真实波幅）

        TR = max(H-L, |H-Cp|, |L-Cp|)
        ATR = SMA(TR, period)

        ATR 可以用于动态调整期权策略的执行价格
        """
        prev_close = close.shift(1)
        tr1 = high - low
        tr2 = abs(high - prev_close)
        tr3 = abs(low - prev_close)
        true_range = pd.concat([tr1, tr2, tr3], axis=1).max(axis=1)
        atr = true_range.rolling(window=period).mean()
        return atr


class OptionSignalGenerator:
    """
    基于技术指标的期权信号生成器

    将股票的技术指标转化为期权交易信号
    """

    def __init__(self):
        self.tech = TechnicalSignals()

    def rsi_contrarian_signal(self, prices: pd.Series,
                               rsi_period: int = 14,
                               oversold: float = 30,
                               overbought: float = 70) -> pd.DataFrame:
        """
        RSI 逆势策略信号

        逻辑：
        - RSI < 30：超卖，看涨信号 → 卖出看跌期权 / 买入看涨期权
        - RSI > 70：超买，看跌信号 → 卖出看涨期权 / 买入看跌期权

        期权应用：
        - 超卖时卖出虚值看跌（Put），收取高波动率带来的高权利金
        - 超买时卖出虚值看涨（Call），利用均值回归特性
        """
        rsi = self.tech.rsi(prices, rsi_period)

        signals = pd.DataFrame(index=prices.index)
        signals['rsi'] = rsi
        signals['signal'] = 0

        # 超卖信号：RSI 从下方向上穿越 oversold 阈值
        signals.loc[
            (rsi < oversold) & (rsi.shift(1) >= oversold),
            'signal'
        ] = 1  # 看涨信号

        # 超买信号：RSI 从上方向下穿越 overbought 阈值
        signals.loc[
            (rsi > overbought) & (rsi.shift(1) <= overbought),
            'signal'
        ] = -1  # 看跌信号

        # 信号强度：RSI 偏离50的程度
        signals['strength'] = abs(rsi - 50) / 50

        # 建议的策略
        signals['strategy'] = 'none'
        signals.loc[signals['signal'] == 1, 'strategy'] = 'sell_put'
        signals.loc[signals['signal'] == -1, 'strategy'] = 'sell_call'

        return signals

    def macd_crossover_signal(self, prices: pd.Series) -> pd.DataFrame:
        """
        MACD 交叉策略信号

        逻辑：
        - MACD 向上穿越信号线：看涨信号
        - MACD 向下穿越信号线：看跌信号

        期权应用：
        - 看涨交叉：买入看涨价差（Bull Call Spread）
        - 看跌交叉：买入看跌价差（Bear Put Spread）
        """
        macd_data = self.tech.macd(prices)

        signals = pd.DataFrame(index=prices.index)
        signals['macd'] = macd_data['macd']
        signals['signal_line'] = macd_data['signal']
        signals['histogram'] = macd_data['histogram']

        # 交叉信号
        signals['signal'] = 0
        # 金叉：MACD 从下方穿越信号线
        signals.loc[
            (signals['macd'] > signals['signal_line']) &
            (signals['macd'].shift(1) <= signals['signal_line'].shift(1)),
            'signal'
        ] = 1
        # 死叉：MACD 从上方穿越信号线
        signals.loc[
            (signals['macd'] < signals['signal_line']) &
            (signals['macd'].shift(1) >= signals['signal_line'].shift(1)),
            'signal'
        ] = -1

        # 信号强度：柱状图的大小
        signals['strength'] = abs(signals['histogram']) / prices

        signals['strategy'] = 'none'
        signals.loc[signals['signal'] == 1, 'strategy'] = 'bull_call_spread'
        signals.loc[signals['signal'] == -1, 'strategy'] = 'bear_put_spread'

        return signals

    def bollinger_squeeze_signal(self, prices: pd.Series,
                                  squeeze_percentile: float = 20,
                                  period: int = 20) -> pd.DataFrame:
        """
        布林带挤压策略信号（Bollinger Squeeze）

        逻辑：
        - 布林带收窄到极低水平（低波动率），预示即将出现大幅波动
        - 挤压后突破上轨：看涨信号
        - 挤压后突破下轨：看跌信号

        期权应用：
        - 挤压阶段：卖出 Iron Condor（低波动率收取权利金）
        - 突破方向确认后：平仓 Iron Condor，改为方向性策略
        """
        bb = self.tech.bollinger_bands(prices, period)

        signals = pd.DataFrame(index=prices.index)
        signals['bandwidth'] = bb['bandwidth']
        signals['pct_b'] = bb['pct_b']
        signals['close'] = prices
        signals['upper'] = bb['upper']
        signals['lower'] = bb['lower']

        # 判断是否处于挤压状态（带宽低于历史分位数）
        bw_percentile = bb['bandwidth'].rolling(window=252).rank(pct=True) * 100
        signals['is_squeeze'] = bw_percentile < squeeze_percentile

        # 信号生成
        signals['signal'] = 0

        # 挤压状态下的突破信号
        squeeze_breakout_up = (
            signals['is_squeeze'].shift(1) &
            (prices > bb['upper'])
        )
        squeeze_breakout_down = (
            signals['is_squeeze'].shift(1) &
            (prices < bb['lower'])
        )

        signals.loc[squeeze_breakout_up, 'signal'] = 1
        signals.loc[squeeze_breakout_down, 'signal'] = -1

        signals['strategy'] = 'none'
        signals.loc[signals['is_squeeze'], 'strategy'] = 'iron_condor'
        signals.loc[squeeze_breakout_up, 'strategy'] = 'bull_call_spread'
        signals.loc[squeeze_breakout_down, 'strategy'] = 'bear_put_spread'

        return signals
```

### 2.2 多指标综合信号

单一指标往往存在较多噪音，将多个指标组合使用可以提高信号质量。

```python
class CompositeSignal:
    """
    综合信号生成器

    将多个技术指标的信号进行加权组合，
    生成更稳健的综合交易信号
    """

    def __init__(self):
        self.tech = TechnicalSignals()

    def generate(self, prices: pd.Series,
                 high: pd.Series = None,
                 low: pd.Series = None,
                 weights: dict = None) -> pd.DataFrame:
        """
        生成综合信号

        Args:
            prices: 收盘价
            high: 最高价（用于ATR）
            low: 最低价（用于ATR）
            weights: 各指标权重
        """
        if weights is None:
            weights = {
                'rsi': 0.30,
                'macd': 0.30,
                'bollinger': 0.20,
                'momentum': 0.20
            }

        signals = pd.DataFrame(index=prices.index)
        composite = pd.Series(0.0, index=prices.index)

        # RSI 信号
        rsi = self.tech.rsi(prices)
        rsi_signal = pd.Series(0.0, index=prices.index)
        rsi_signal[rsi < 30] = 1.0
        rsi_signal[rsi > 70] = -1.0
        rsi_signal[(rsi >= 30) & (rsi <= 70)] = (50 - rsi[(rsi >= 30) & (rsi <= 70)]) / 50
        signals['rsi'] = rsi_signal
        composite += rsi_signal * weights['rsi']

        # MACD 信号
        macd_data = self.tech.macd(prices)
        macd_norm = macd_data['histogram'] / prices.rolling(20).std()
        macd_signal = np.clip(macd_norm, -1, 1)
        signals['macd'] = macd_signal
        composite += macd_signal * weights['macd']

        # 布林带信号
        bb = self.tech.bollinger_bands(prices)
        bb_signal = pd.Series(0.0, index=prices.index)
        bb_signal[bb['pct_b'] > 1] = -1.0
        bb_signal[bb['pct_b'] < 0] = 1.0
        bb_signal[(bb['pct_b'] >= 0) & (bb['pct_b'] <= 1)] = \
            0.5 - bb['pct_b'][(bb['pct_b'] >= 0) & (bb['pct_b'] <= 1)]
        signals['bollinger'] = bb_signal
        composite += bb_signal * weights['bollinger']

        # 动量信号（20日收益率的Z-Score）
        returns = prices.pct_change()
        momentum = returns.rolling(20).mean() / returns.rolling(20).std()
        mom_signal = np.clip(momentum, -1, 1)
        signals['momentum'] = mom_signal
        composite += mom_signal * weights['momentum']

        # 综合信号
        signals['composite'] = composite
        signals['signal'] = 0
        signals.loc[signals['composite'] > 0.3, 'signal'] = 1
        signals.loc[signals['composite'] < -0.3, 'signal'] = -1

        # 信号强度
        signals['strength'] = abs(signals['composite'])

        return signals
```

---

## 三、波动率信号

波动率是期权交易的核心。与方向性信号相比，波动率信号对期权策略的影响更为直接和重要。

### 3.1 IV Rank 与 IV Percentile

```python
class VolatilitySignals:
    """
    波动率信号生成器

    波动率信号是期权策略的核心，
    因为期权价格直接由波动率决定
    """

    @staticmethod
    def iv_rank(current_iv: pd.Series, lookback: int = 252) -> pd.Series:
        """
        IV Rank（隐含波动率排名）

        IV Rank = (当前IV - 最低IV) / (最高IV - 最低IV)

        范围：0-100
        - IV Rank > 80：高波动率环境，适合卖方策略
        - IV Rank < 20：低波动率环境，适合买方策略

        Args:
            current_iv: 当前隐含波动率序列
            lookback: 回看周期（默认252个交易日，即一年）

        Returns:
            IV Rank 序列（0-100）
        """
        rolling_min = current_iv.rolling(window=lookback, min_periods=30).min()
        rolling_max = current_iv.rolling(window=lookback, min_periods=30).max()

        iv_range = rolling_max - rolling_min
        iv_rank = (current_iv - rolling_min) / iv_range.replace(0, np.nan) * 100

        return iv_rank.fillna(50)

    @staticmethod
    def iv_percentile(current_iv: pd.Series, lookback: int = 252) -> pd.Series:
        """
        IV Percentile（隐含波动率百分位）

        IV Percentile = 当前IV低于历史数据的百分比

        与 IV Rank 的区别：
        - IV Rank 关注的是极值范围内的位置
        - IV Percentile 关注的是分布中的位置
        - IV Percentile 对异常值更稳健
        """
        def percentile_rank(series):
            if len(series) < 30:
                return np.nan
            current = series.iloc[-1]
            return (series < current).sum() / len(series) * 100

        return current_iv.rolling(window=lookback, min_periods=30).apply(
            percentile_rank, raw=False
        )

    @staticmethod
    def iv_hv_spread(iv: pd.Series, hv: pd.Series,
                      period: int = 20) -> pd.Series:
        """
        IV-HV 价差（隐含波动率 - 历史波动率）

        正价差（IV > HV）：期权相对昂贵，适合卖出
        负价差（IV < HV）：期权相对便宜，适合买入

        这是波动率均值回归策略的核心指标
        """
        return iv - hv

    @staticmethod
    def historical_volatility(prices: pd.Series,
                               period: int = 20) -> pd.Series:
        """
        计算历史波动率（已实现波动率）

        使用对数收益率的标准差年化

        HV = std(ln(P_t / P_{t-1})) * sqrt(252)
        """
        log_returns = np.log(prices / prices.shift(1))
        hv = log_returns.rolling(window=period).std() * np.sqrt(252)
        return hv

    @staticmethod
    def volatility_regime(iv: pd.Series,
                           low_threshold: float = 20,
                           high_threshold: float = 80) -> pd.Series:
        """
        波动率状态识别

        将市场分为三种波动率状态：
        - 'low'：低波动率（IV Rank < 20）
        - 'medium'：中等波动率
        - 'high'：高波动率（IV Rank > 80）

        不同状态适合不同的策略
        """
        iv_rank = VolatilitySignals.iv_rank(iv)

        regime = pd.Series('medium', index=iv.index)
        regime[iv_rank < low_threshold] = 'low'
        regime[iv_rank > high_threshold] = 'high'

        return regime

    @staticmethod
    def vix_term_structure(vix_near: pd.Series,
                            vix_far: pd.Series) -> pd.Series:
        """
        VIX 期限结构

        正常状态（Contango）：远月VIX > 近月VIX
        恐慌状态（Backwardation）：近月VIX > 远月VIX

        Contango → 市场平静，适合卖方策略
        Backwardation → 市场恐慌，波动率可能继续上升
        """
        spread = vix_near - vix_far
        ratio = vix_near / vix_far

        regime = pd.Series('contango', index=vix_near.index)
        regime[spread > 0] = 'backwardation'

        return pd.DataFrame({
            'spread': spread,
            'ratio': ratio,
            'regime': regime
        })


class VolatilityStrategy:
    """
    基于波动率的期权策略

    核心逻辑：波动率具有均值回归特性，
    当波动率偏离均值过远时进行反向交易
    """

    def __init__(self, iv_high_threshold: float = 80,
                 iv_low_threshold: float = 20,
                 mean_reversion_target: float = 50):
        self.iv_high = iv_high_threshold
        self.iv_low = iv_low_threshold
        self.target = mean_reversion_target

    def generate_signals(self, iv: pd.Series,
                          prices: pd.Series) -> pd.DataFrame:
        """
        生成基于波动率的交易信号

        策略逻辑：
        1. 计算 IV Rank
        2. IV Rank > 80：卖出策略（Iron Condor / Short Strangle）
        3. IV Rank < 20：买入策略（Long Straddle / Long Strangle）
        4. IV Rank 20-80：观望或中性策略
        """
        vs = VolatilitySignals()

        signals = pd.DataFrame(index=iv.index)
        signals['iv'] = iv
        signals['iv_rank'] = vs.iv_rank(iv)
        signals['iv_percentile'] = vs.iv_percentile(iv)
        signals['hv'] = vs.historical_volatility(prices)
        signals['iv_hv_spread'] = vs.iv_hv_spread(iv, signals['hv'])
        signals['regime'] = vs.volatility_regime(iv, self.iv_low, self.iv_high)

        # 生成交易信号
        signals['signal'] = 0
        signals['strategy'] = 'none'

        # 高波动率：卖出策略
        high_vol = signals['iv_rank'] > self.iv_high
        signals.loc[high_vol, 'signal'] = -1  # 卖出波动率
        signals.loc[high_vol, 'strategy'] = 'short_volatility'

        # 低波动率：买入策略
        low_vol = signals['iv_rank'] < self.iv_low
        signals.loc[low_vol, 'signal'] = 1  # 买入波动率
        signals.loc[low_vol, 'strategy'] = 'long_volatility'

        # IV-HV 价差信号
        iv_premium = signals['iv_hv_spread'] > 0.05  # IV 显著高于 HV
        iv_discount = signals['iv_hv_spread'] < -0.05  # IV 显著低于 HV

        signals.loc[iv_premium & (signals['signal'] == 0), 'signal'] = -0.5
        signals.loc[iv_premium & (signals['strategy'] == 'none'), 'strategy'] = 'sell_premium'
        signals.loc[iv_discount & (signals['signal'] == 0), 'signal'] = 0.5
        signals.loc[iv_discount & (signals['strategy'] == 'none'), 'strategy'] = 'buy_premium'

        # 信号强度
        signals['strength'] = abs(signals['iv_rank'] - 50) / 50

        return signals
```

### 3.2 波动率均值回归策略

波动率的均值回归特性是期权卖方策略的理论基础。

```python
class MeanReversionVolStrategy:
    """
    波动率均值回归策略

    核心假设：波动率围绕长期均值波动，
    当偏离过大时会回归均值。

    策略逻辑：
    1. 计算 IV Rank
    2. IV Rank > 阈值 → 卖出宽跨式（Short Strangle）
    3. IV Rank < 阈值 → 买入宽跨式（Long Straddle）
    4. IV Rank 回归中位数 → 平仓
    """

    def __init__(self, entry_high: float = 80,
                 entry_low: float = 20,
                 exit_level: float = 50,
                 hold_days: int = 30):
        self.entry_high = entry_high
        self.entry_low = entry_low
        self.exit_level = exit_level
        self.hold_days = hold_days

    def backtest(self, iv: pd.Series, prices: pd.Series,
                 option_data: pd.DataFrame = None) -> pd.DataFrame:
        """
        回测波动率均值回归策略

        简化版本：使用 IV Rank 作为信号，
        使用标的收益率乘以近似 Delta 来估算策略收益
        """
        vs = VolatilitySignals()
        iv_rank = vs.iv_rank(iv)

        results = pd.DataFrame(index=iv.index)
        results['iv'] = iv
        results['iv_rank'] = iv_rank
        results['price'] = prices
        results['position'] = 0  # 1=做多波动率, -1=做空波动率, 0=空仓
        results['pnl'] = 0.0

        position = 0
        entry_date = None
        entry_iv = 0

        for i in range(1, len(results)):
            date = results.index[i]
            current_rank = iv_rank.iloc[i]
            current_iv = iv.iloc[i]

            # 入场逻辑
            if position == 0:
                if current_rank > self.entry_high:
                    position = -1  # 卖出波动率
                    entry_date = date
                    entry_iv = current_iv
                elif current_rank < self.entry_low:
                    position = 1   # 买入波动率
                    entry_date = date
                    entry_iv = current_iv

            # 出场逻辑
            elif position != 0:
                hold = (date - entry_date).days if entry_date else 0

                # 条件1：IV Rank 回归到退出区间
                if position == -1 and current_rank < self.exit_level:
                    position = 0
                elif position == 1 and current_rank > self.exit_level:
                    position = 0

                # 条件2：持有时间超过上限
                elif hold >= self.hold_days:
                    position = 0

                # 条件3：止损（IV 继续扩大）
                elif position == -1 and current_rank > 95:
                    position = 0
                elif position == 1 and current_rank < 5:
                    position = 0

            results.iloc[i, results.columns.get_loc('position')] = position

            # 计算PnL（简化：使用IV变化作为收益代理）
            if position != 0:
                iv_change = current_iv - iv.iloc[i-1]
                # 做空波动率：IV下降赚钱
                # 做多波动率：IV上升赚钱
                daily_pnl = -position * iv_change * 100
                results.iloc[i, results.columns.get_loc('pnl')] = daily_pnl

        results['cumulative_pnl'] = results['pnl'].cumsum()
        results['equity'] = 100000 + results['cumulative_pnl']

        return results
```

---

## 四、统计信号

### 4.1 Z-Score 与均值回归

```python
class StatisticalSignals:
    """统计信号生成器"""

    @staticmethod
    def z_score(series: pd.Series, lookback: int = 20) -> pd.Series:
        """
        Z-Score 标准化

        Z = (X - mean) / std

        在期权交易中，Z-Score 可以用于：
        - 判断价格偏离程度
        - 识别波动率异常
        - 评估价差的相对价值
        """
        mean = series.rolling(window=lookback).mean()
        std = series.rolling(window=lookback).std()
        return (series - mean) / std

    @staticmethod
    def rolling_correlation(series1: pd.Series, series2: pd.Series,
                             lookback: int = 60) -> pd.Series:
        """
        滚动相关系数

        可用于监控：
        - 标的与VIX的负相关性
        - 不同期权策略之间的相关性
        - 配对交易机会
        """
        return series1.rolling(window=lookback).corr(series2)

    @staticmethod
    def hurst_exponent(series: pd.Series, max_lag: int = 20) -> float:
        """
        Hurst 指数

        H > 0.5：趋势性序列（动量策略有效）
        H = 0.5：随机游走
        H < 0.5：均值回归序列（均值回归策略有效）

        这对于选择期权策略类型非常重要
        """
        lags = range(2, max_lag + 1)
        tau = []

        for lag in lags:
            # 计算不同时间间隔的方差
            diffs = series.diff(lag).dropna()
            tau.append(np.std(diffs))

        # 对数回归
        log_lags = np.log(list(lags))
        log_tau = np.log(tau)

        # OLS 回归
        coeffs = np.polyfit(log_lags, log_tau, 1)
        hurst = coeffs[0]

        return hurst

    @staticmethod
    def half_life(series: pd.Series) -> float:
        """
        均值回归半衰期

        使用 Ornstein-Uhlenbeck 过程估计：
        dX = theta * (mu - X) * dt + sigma * dW

        半衰期 = ln(2) / theta

        这个值告诉我们在均值回归策略中预期的持仓时间
        """
        y = series.diff().dropna()
        x = series.shift(1).dropna()

        # 对齐索引
        common_idx = y.index.intersection(x.index)
        y = y.loc[common_idx]
        x = x.loc[common_idx]

        # OLS 回归：dy = theta * (mu - y_prev) + epsilon
        # 即 dy = a + b * y_prev
        if len(x) < 10:
            return np.nan

        x_with_const = np.column_stack([np.ones(len(x)), x.values])
        coeffs = np.linalg.lstsq(x_with_const, y.values, rcond=None)[0]
        b = coeffs[1]

        if b >= 0:
            return np.nan  # 不是均值回归过程

        half_life = -np.log(2) / b
        return half_life

    @staticmethod
    def spread_z_score(series1: pd.Series, series2: pd.Series,
                        lookback: int = 60, hedge_ratio: float = None) -> dict:
        """
        价差 Z-Score（配对交易信号）

        适用于：
        - 相关标的之间的期权价差
        - 不同到期日的波动率价差
        - ETF 与其成分之间的套利
        """
        if hedge_ratio is None:
            # 使用滚动OLS估计对冲比率
            hedge_ratio = (series1.rolling(lookback).cov(series2) /
                          series2.rolling(lookback).var())

        spread = series1 - hedge_ratio * series2
        z = StatisticalSignals.z_score(spread, lookback)

        return {
            'spread': spread,
            'z_score': z,
            'hedge_ratio': hedge_ratio,
            'half_life': StatisticalSignals.half_life(spread)
        }
```

### 4.2 动量与反转信号

```python
class MomentumReversalSignals:
    """动量与反转信号"""

    @staticmethod
    def momentum_score(prices: pd.Series,
                        periods: list = None) -> pd.Series:
        """
        多周期动量评分

        综合多个时间周期的动量信号：
        - 短期动量（5天）
        - 中期动量（20天）
        - 长期动量（60天）

        评分范围：-1（强看跌）到 +1（强看涨）
        """
        if periods is None:
            periods = [5, 20, 60]

        scores = pd.DataFrame(index=prices.index)

        for period in periods:
            returns = prices.pct_change(period)
            # 标准化为 Z-Score
            z = (returns - returns.rolling(252).mean()) / returns.rolling(252).std()
            scores[f'mom_{period}'] = np.clip(z, -1, 1)

        # 加权平均
        weights = [0.4, 0.35, 0.25]  # 短期权重更高
        composite = sum(scores[f'mom_{p}'] * w
                        for p, w in zip(periods, weights))

        return composite

    @staticmethod
    def mean_reversion_signal(prices: pd.Series,
                                lookback: int = 20,
                                threshold: float = 2.0) -> pd.DataFrame:
        """
        均值回归信号

        当价格偏离移动均线超过阈值个标准差时，
        预期价格将回归均值。

        期权应用：
        - 价格大幅上涨后 → 卖出看涨期权
        - 价格大幅下跌后 → 卖出看跌期权
        """
        ma = prices.rolling(window=lookback).mean()
        std = prices.rolling(window=lookback).std()
        z = (prices - ma) / std

        signals = pd.DataFrame(index=prices.index)
        signals['price'] = prices
        signals['ma'] = ma
        signals['z_score'] = z

        signals['signal'] = 0
        signals.loc[z > threshold, 'signal'] = -1   # 价格过高，预期回落
        signals.loc[z < -threshold, 'signal'] = 1    # 价格过低，预期反弹

        # 信号强度与偏离程度成正比
        signals['strength'] = np.clip(abs(z) / threshold, 0, 1)

        # 建议的期权策略
        signals['strategy'] = 'none'
        signals.loc[signals['signal'] == 1, 'strategy'] = 'sell_put_or_buy_call'
        signals.loc[signals['signal'] == -1, 'strategy'] = 'sell_call_or_buy_put'

        return signals
```

---

## 五、策略开发流程

### 5.1 从假设到验证的完整流程

一个严谨的策略开发应该遵循以下流程：

```
1. 假设形成 (Hypothesis)
   └── 基于市场观察或理论推导

2. 数据收集 (Data Collection)
   └── 获取足够的历史数据

3. 信号定义 (Signal Definition)
   └── 将假设转化为可量化的规则

4. 样本内回测 (In-Sample Backtest)
   └── 在训练集上验证策略逻辑

5. 参数优化 (Parameter Optimization)
   └── 寻找最优参数组合（但要避免过拟合）

6. 样本外验证 (Out-of-Sample Validation)
   └── 在未见过的数据上验证

7. 纸上交易 (Paper Trading)
   └── 在模拟环境中实时验证

8. 小额实盘 (Small Position Live)
   └── 用小资金在真实市场中测试

9. 正式交易 (Full Deployment)
   └── 逐步增加仓位到目标规模
```

### 5.2 参数优化与防止过拟合

```python
class ParameterOptimizer:
    """
    参数优化器

    使用网格搜索和 Walk-Forward 分析来寻找稳健的参数组合
    """

    def __init__(self, strategy_func, metric: str = 'sharpe'):
        """
        Args:
            strategy_func: 策略函数，接受参数字典，返回回测结果
            metric: 优化目标指标
        """
        self.strategy_func = strategy_func
        self.metric = metric

    def grid_search(self, param_grid: dict, data: pd.DataFrame) -> pd.DataFrame:
        """
        网格搜索

        Args:
            param_grid: 参数网格，如 {'period': [10, 20, 30], 'threshold': [1.5, 2.0, 2.5]}

        Returns:
            参数组合及其对应的绩效指标
        """
        from itertools import product

        keys = list(param_grid.keys())
        values = list(param_grid.values())
        combinations = list(product(*values))

        results = []

        for combo in combinations:
            params = dict(zip(keys, combo))
            try:
                perf = self.strategy_func(params, data)
                perf['params'] = str(params)
                results.append(perf)
            except Exception as e:
                print(f"参数 {params} 出错: {e}")

        results_df = pd.DataFrame(results)
        results_df = results_df.sort_values(self.metric, ascending=False)

        return results_df

    def walk_forward_optimize(self, param_grid: dict,
                                data: pd.DataFrame,
                                n_splits: int = 5,
                                train_ratio: float = 0.7) -> dict:
        """
        Walk-Forward 优化

        将数据分为多个窗口，每个窗口内：
        1. 在训练集上用网格搜索找最优参数
        2. 在测试集上验证该参数的表现
        3. 记录样本外绩效

        这是防止过拟合的最有效方法之一
        """
        n = len(data)
        split_size = n // n_splits
        train_size = int(split_size * train_ratio)
        test_size = split_size - train_size

        oos_results = []  # 样本外结果

        for i in range(n_splits):
            start = i * split_size
            train_end = start + train_size
            test_end = min(train_end + test_size, n)

            train_data = data.iloc[start:train_end]
            test_data = data.iloc[train_end:test_end]

            # 在训练集上优化
            train_results = self.grid_search(param_grid, train_data)
            best_params = eval(train_results.iloc[0]['params'])

            # 在测试集上验证
            oos_perf = self.strategy_func(best_params, test_data)
            oos_perf['window'] = i + 1
            oos_perf['best_params'] = str(best_params)
            oos_results.append(oos_perf)

        oos_df = pd.DataFrame(oos_results)

        print("=== Walk-Forward 优化结果 ===")
        for _, row in oos_df.iterrows():
            print(f"窗口 {int(row['window'])}: "
                  f"Sharpe={row.get('sharpe', 'N/A'):.2f}, "
                  f"Return={row.get('total_return', 'N/A'):.2%}, "
                  f"参数={row['best_params']}")

        # 稳健性评估
        avg_sharpe = oos_df['sharpe'].mean()
        std_sharpe = oos_df['sharpe'].std()
        print(f"\n平均夏普比率: {avg_sharpe:.2f} (+/- {std_sharpe:.2f})")

        return {
            'oos_results': oos_df,
            'avg_sharpe': avg_sharpe,
            'std_sharpe': std_sharpe,
            'is_robust': std_sharpe < avg_sharpe * 0.5  # 标准差小于均值的一半
        }
```

---

## 六、策略组合

### 6.1 多策略整合

单一策略往往存在周期性失效的风险。将多个低相关性的策略组合在一起，可以提高整体的风险调整收益。

```python
class MultiStrategyPortfolio:
    """
    多策略组合管理器

    将多个独立的期权策略组合在一起，
    通过分散化降低整体风险
    """

    def __init__(self, strategies: dict, weights: dict = None):
        """
        Args:
            strategies: 策略名称到策略实例的映射
            weights: 各策略的权重（默认等权）
        """
        self.strategies = strategies
        self.names = list(strategies.keys())

        if weights is None:
            n = len(self.names)
            self.weights = {name: 1/n for name in self.names}
        else:
            self.weights = weights

    def combine_signals(self, individual_signals: dict) -> pd.DataFrame:
        """
        组合各策略的信号

        使用加权平均方法：
        composite_signal = sum(weight_i * signal_i)
        """
        combined = pd.DataFrame()

        for name in self.names:
            if name in individual_signals:
                signal = individual_signals[name]
                combined[f'{name}_signal'] = signal * self.weights[name]

        combined['composite'] = combined.sum(axis=1)

        # 综合信号判断
        combined['action'] = 'hold'
        combined.loc[combined['composite'] > 0.3, 'action'] = 'buy'
        combined.loc[combined['composite'] < -0.3, 'action'] = 'sell'

        return combined

    def analyze_correlation(self, returns_dict: dict) -> pd.DataFrame:
        """
        分析策略之间的相关性

        低相关性的策略组合效果更好
        """
        returns_df = pd.DataFrame(returns_dict)
        corr_matrix = returns_df.corr()

        print("=== 策略收益相关性矩阵 ===")
        print(corr_matrix.round(3))

        # 计算平均相关系数（排除对角线）
        n = len(corr_matrix)
        avg_corr = (corr_matrix.values.sum() - n) / (n * (n - 1))
        print(f"\n平均相关系数: {avg_corr:.3f}")

        if avg_corr < 0.3:
            print("相关性较低，组合效果预期较好")
        elif avg_corr < 0.6:
            print("相关性中等，建议进一步分散")
        else:
            print("相关性较高，组合效果可能有限")

        return corr_matrix

    def optimize_weights(self, returns_dict: dict,
                          risk_free_rate: float = 0.05) -> dict:
        """
        优化策略权重

        使用均值-方差优化（Markowitz），
        目标：最大化夏普比率
        """
        from scipy.optimize import minimize

        returns_df = pd.DataFrame(returns_dict)
        n = len(self.names)

        mean_returns = returns_df.mean() * 252
        cov_matrix = returns_df.cov() * 252

        def neg_sharpe(weights):
            port_return = np.dot(weights, mean_returns)
            port_vol = np.sqrt(np.dot(weights.T, np.dot(cov_matrix, weights)))
            return -(port_return - risk_free_rate) / port_vol

        constraints = {'type': 'eq', 'fun': lambda w: np.sum(w) - 1}
        bounds = tuple((0, 0.5) for _ in range(n))  # 单策略最多50%
        init = np.array([1/n] * n)

        result = minimize(neg_sharpe, init, method='SLSQP',
                         bounds=bounds, constraints=constraints)

        optimal_weights = dict(zip(self.names, result.x))
        optimal_sharpe = -result.fun

        print("\n=== 最优策略权重 ===")
        for name, weight in optimal_weights.items():
            print(f"  {name}: {weight:.1%}")
        print(f"最优夏普比率: {optimal_sharpe:.2f}")

        return optimal_weights
```

---

## 七、代码示例：基于 IV Rank 的卖方策略

```python
def iv_rank_selling_strategy(iv_series: pd.Series,
                              prices: pd.Series,
                              entry_threshold: float = 70,
                              exit_threshold: float = 40,
                              max_hold_days: int = 45,
                              delta_target: float = 0.16) -> pd.DataFrame:
    """
    基于 IV Rank 的期权卖方策略

    完整实现包括：
    1. 波动率信号生成
    2. 合约选择逻辑
    3. 入场/出场规则
    4. 仓位管理
    5. 绩效跟踪

    Args:
        iv_series: 隐含波动率时间序列
        prices: 标的价格时间序列
        entry_threshold: IV Rank 入场阈值
        exit_threshold: IV Rank 出场阈值
        max_hold_days: 最大持有天数
        delta_target: 目标 Delta

    Returns:
        包含交易记录和绩效数据的 DataFrame
    """
    from scipy.stats import norm

    # 计算 IV Rank
    iv_rank = VolatilitySignals.iv_rank(iv_series)

    # 初始化结果 DataFrame
    results = pd.DataFrame(index=iv_series.index)
    results['price'] = prices
    results['iv'] = iv_series
    results['iv_rank'] = iv_rank
    results['position'] = 0       # 0=空仓, 1=持有空头波动率
    results['entry_price'] = 0.0
    results['pnl'] = 0.0
    results['cumulative_pnl'] = 0.0

    # 交易记录
    trades = []
    position = 0
    entry_date = None
    entry_price = 0
    entry_iv = 0

    for i in range(1, len(results)):
        date = results.index[i]
        current_price = prices.iloc[i]
        current_iv = iv_series.iloc[i]
        current_rank = iv_rank.iloc[i]

        daily_pnl = 0

        # 出场检查
        if position != 0:
            hold_days = (date - entry_date).days if entry_date else 0

            # 出场条件
            should_exit = False
            exit_reason = ''

            if current_rank < exit_threshold:
                should_exit = True
                exit_reason = 'IV Rank 回归到退出区间'
            elif hold_days >= max_hold_days:
                should_exit = True
                exit_reason = '达到最大持有天数'
            elif current_rank > 95:
                should_exit = True
                exit_reason = 'IV Rank 极端高位（止损）'

            if should_exit:
                # 计算盈亏
                iv_change = current_iv - entry_iv
                # 做空波动率的收益 ≈ -iv_change * vega
                trade_pnl = -iv_change * 1000  # 简化估算
                daily_pnl = trade_pnl

                trades.append({
                    'entry_date': entry_date,
                    'exit_date': date,
                    'hold_days': hold_days,
                    'entry_iv': entry_iv,
                    'exit_iv': current_iv,
                    'pnl': trade_pnl,
                    'exit_reason': exit_reason
                })

                position = 0
                entry_date = None

        # 入场检查
        if position == 0 and current_rank > entry_threshold:
            position = -1  # 做空波动率
            entry_date = date
            entry_price = current_price
            entry_iv = current_iv

        results.iloc[i, results.columns.get_loc('position')] = position
        results.iloc[i, results.columns.get_loc('pnl')] = daily_pnl

    results['cumulative_pnl'] = results['pnl'].cumsum()
    results['equity'] = 100000 + results['cumulative_pnl']

    # 输出统计
    trades_df = pd.DataFrame(trades) if trades else pd.DataFrame()

    print(f"\n{'='*55}")
    print(f"  IV Rank 卖方策略回测报告")
    print(f"{'='*55}")
    print(f"  入场阈值: IV Rank > {entry_threshold}")
    print(f"  出场阈值: IV Rank < {exit_threshold}")
    print(f"  最大持有天数: {max_hold_days}")

    if len(trades_df) > 0:
        print(f"\n  总交易次数: {len(trades_df)}")
        print(f"  胜率: {(trades_df['pnl'] > 0).mean():.1%}")
        print(f"  平均盈亏: ${trades_df['pnl'].mean():.2f}")
        print(f"  总盈亏: ${trades_df['pnl'].sum():.2f}")
        print(f"  平均持有天数: {trades_df['hold_days'].mean():.1f}")

        win_trades = trades_df[trades_df['pnl'] > 0]
        lose_trades = trades_df[trades_df['pnl'] < 0]

        if len(win_trades) > 0:
            print(f"  平均盈利: ${win_trades['pnl'].mean():.2f}")
        if len(lose_trades) > 0:
            print(f"  平均亏损: ${lose_trades['pnl'].mean():.2f}")

    print(f"{'='*55}")

    return results


# 运行示例
if __name__ == "__main__":
    # 生成模拟数据
    np.random.seed(42)
    dates = pd.bdate_range('2020-01-01', '2024-12-31', freq='B')
    n = len(dates)

    # 模拟标的价格（几何布朗运动）
    log_returns = np.random.normal(0.0003, 0.015, n)
    prices = pd.Series(300 * np.exp(np.cumsum(log_returns)), index=dates)

    # 模拟隐含波动率（均值回归过程）
    iv = pd.Series(0.0, index=dates)
    iv.iloc[0] = 0.20
    for i in range(1, n):
        iv.iloc[i] = iv.iloc[i-1] + 0.1 * (0.20 - iv.iloc[i-1]) + \
                     0.02 * np.random.randn()
        iv.iloc[i] = max(0.08, min(0.60, iv.iloc[i]))

    # 运行策略
    results = iv_rank_selling_strategy(
        iv, prices,
        entry_threshold=70,
        exit_threshold=40,
        max_hold_days=45
    )
```

---

## 小结

本章系统介绍了期权交易信号生成的方法：

- **技术指标信号**：RSI、MACD、布林带等经典指标在期权交易中的应用
- **波动率信号**：IV Rank、IV Percentile、IV-HV 价差等波动率特有指标
- **统计信号**：Z-Score、Hurst 指数、半衰期等统计工具
- **策略开发流程**：从假设形成到正式交易的完整路径
- **参数优化**：Walk-Forward 分析防止过拟合
- **多策略组合**：通过分散化提高整体表现

信号质量决定了策略的上限。好的信号应该有清晰的逻辑支撑、稳健的统计验证和可执行的交易规则。

### 思考题

1. 实现一个基于 Hurst 指数的策略选择器：当 Hurst > 0.5 时使用动量策略，当 Hurst < 0.5 时使用均值回归策略。
2. 设计一个"波动率套利"策略：当 IV 显著高于 HV 时卖出期权，当 IV 显著低于 HV 时买入期权。
3. 将本章的 IV Rank 策略与上一章的 Iron Condor 策略结合，实现一个"仅在高 IV 环境下开仓"的条件策略。

---

**下一章：** [05-ibkr-api.md —— IBKR API 与自动交易](section05-ibkr-api.md)




---
**相关笔记：** [[回测框架搭建]], [[IBKR API自动交易]]
**返回：** [[chapter00-python-ai]] | [[chapter01-主页]]
