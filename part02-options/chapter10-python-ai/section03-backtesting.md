---
type: concept
topic: Python+AI
aliases: [Backtesting, 回测框架]
status: done
created: "2026-06-28"
updated: "2026-06-28"
tags: [期权, Python, 回测]
related: "[[Python期权库入门]], [[信号生成与策略开发]]"
---

# 期权策略回测框架

## 引言

回测（Backtesting）是量化交易策略开发的核心环节。它通过在历史数据上模拟策略的执行过程，评估策略的盈利能力、风险特征和稳健性。对于期权策略而言，回测比股票策略更加复杂——因为期权有到期日、执行价格、Greeks 等多维度参数，且流动性、行权方式等因素都会影响实际结果。本章将从零构建一个专业的期权策略回测框架。

---

## 一、回测的重要性

### 1.1 为什么需要回测

在投入真金白银之前，回测可以回答以下关键问题：

1. **策略是否有正期望值？** 历史表现是否支持策略的盈利逻辑？
2. **风险特征如何？** 最大回撤（Maximum Drawdown）、波动率、尾部风险（Tail Risk）是多少？
3. **参数敏感性如何？** 策略是否过度依赖特定参数（过拟合）？
4. **不同市场环境下的表现？** 牛市、熊市、震荡市分别表现如何？

### 1.2 期权回测的特殊挑战

与股票回测相比，期权回测面临额外的复杂性：

- **合约选择**：需要定义明确的规则来选择执行价格和到期日
- **展期逻辑**：到期前需要决定是否展期到下一个月
- **流动性建模**：买卖价差（Bid-Ask Spread）和滑点（Slippage）对卖方策略影响巨大
- **保证金计算**：不同策略的保证金要求不同，影响资金使用效率
- **提前行权**：美式期权可能被提前行权
- **数据质量**：历史期权数据可能存在幸存偏差（Survivorship Bias）

---

## 二、回测框架设计

### 2.1 事件驱动 vs 向量化

回测框架主要有两种架构：

**向量化回测（Vectorized Backtesting）**：
- 使用 Pandas/NumPy 的向量化操作一次性计算所有时间点的结果
- 优点：代码简洁、运行速度快
- 缺点：难以模拟复杂的交易逻辑、不适合实时决策
- 适合：简单策略的快速验证

**事件驱动回测（Event-Driven Backtesting）**：
- 模拟真实交易流程，逐个时间点处理事件
- 事件类型：市场数据更新、信号生成、订单提交、成交确认
- 优点：更贴近实际交易、逻辑灵活
- 缺点：代码复杂、运行速度较慢
- 适合：复杂策略的完整模拟

本章将实现一个兼顾两者优势的混合框架。

### 2.2 框架架构

```
┌─────────────────────────────────────────┐
│              回测引擎 (Engine)            │
├─────────────┬─────────────┬─────────────┤
│  数据管理器  │  策略引擎   │  执行引擎   │
│ DataManager │ StrategyEngine│ ExecutionEngine│
├─────────────┼─────────────┼─────────────┤
│ 历史数据加载 │ 信号生成     │ 订单管理    │
│ 期权链管理   │ 合约选择     │ 成交模拟    │
│ 数据对齐     │ 仓位管理     │ 保证金计算  │
├─────────────┴─────────────┴─────────────┤
│           绩效分析器 (Analyzer)           │
├─────────────────────────────────────────┤
│ 收益率 │ 夏普比率 │ 最大回撤 │ 胜率 │ Greeks│
└─────────────────────────────────────────┘
```

---

## 三、数据准备

### 3.1 历史期权数据获取与清洗

高质量的历史数据是回测的基础。期权数据需要包含以下字段：

- 日期（Date）
- 标的价格（Underlying Price）
- 执行价格（Strike）
- 到期日（Expiration）
- 类型（Call/Put）
- 开盘价/最高价/最低价/收盘价（OHLC）
- 成交量（Volume）
- 持仓量（Open Interest）
- 隐含波动率（Implied Volatility）
- Greeks（可选，可自行计算）

```python
import pandas as pd
import numpy as np
from datetime import datetime, timedelta
from dataclasses import dataclass, field
from typing import List, Optional, Dict, Tuple
from enum import Enum

class OptionType(Enum):
    CALL = 'call'
    PUT = 'put'

@dataclass
class OptionContract:
    """期权合约数据"""
    date: datetime
    expiry: datetime
    strike: float
    option_type: OptionType
    bid: float
    ask: float
    last: float
    volume: int
    open_interest: int
    implied_vol: float
    underlying_price: float

    @property
    def mid(self) -> float:
        return (self.bid + self.ask) / 2

    @property
    def dte(self) -> int:
        """到期天数（Days to Expiration）"""
        return (self.expiry - self.date).days

    @property
    def spread(self) -> float:
        return self.ask - self.bid

    @property
    def spread_pct(self) -> float:
        if self.mid > 0:
            return self.spread / self.mid * 100
        return float('inf')


class DataManager:
    """
    数据管理器

    负责加载、清洗和管理期权历史数据
    """

    def __init__(self):
        self.option_data: Dict[str, pd.DataFrame] = {}
        self.underlying_data: Dict[str, pd.Series] = {}

    def load_option_data(self, filepath: str, ticker: str):
        """
        加载期权历史数据

        支持 CSV 格式，预期列名：
        date, expiry, strike, type, bid, ask, last,
        volume, open_interest, implied_vol, underlying_price
        """
        df = pd.read_csv(filepath, parse_dates=['date', 'expiry'])

        # 数据清洗
        df = self._clean_data(df)
        self.option_data[ticker] = df

        print(f"加载 {ticker} 期权数据: {len(df)} 条记录")
        print(f"  日期范围: {df['date'].min()} ~ {df['date'].max()}")
        print(f"  到期日数量: {df['expiry'].nunique()}")
        print(f"  执行价格数量: {df['strike'].nunique()}")

    def load_underlying_data(self, filepath: str, ticker: str):
        """加载标的历史价格数据"""
        df = pd.read_csv(filepath, parse_dates=['date'])
        df.set_index('date', inplace=True)
        self.underlying_data[ticker] = df['close']

    def generate_synthetic_data(self, ticker: str, start_date: str,
                                 end_date: str, S0: float = 100,
                                 sigma: float = 0.20):
        """
        生成合成期权数据用于测试

        注意：合成数据仅用于框架开发和调试，
        实际策略评估应使用真实市场数据
        """
        from scipy.stats import norm

        dates = pd.bdate_range(start_date, end_date)
        expiries = self._generate_monthly_expiries(dates)

        records = []
        S = S0

        for date in dates:
            # 模拟标的价格变动
            S = S * np.exp(np.random.normal(0, sigma/np.sqrt(252)))

            for expiry in expiries:
                if expiry <= date:
                    continue

                dte = (expiry - date).days
                T = dte / 365

                # 选择合理的执行价格范围
                atm_strike = round(S / 5) * 5
                strikes = np.arange(atm_strike - 20, atm_strike + 25, 5)

                for strike in strikes:
                    for opt_type in ['call', 'put']:
                        # 简化的 IV 模型（微笑 + 期限结构）
                        moneyness = np.log(S / strike)
                        iv = sigma * (1 + 0.5 * moneyness**2 +
                                      0.02 * np.sqrt(T))

                        # Black-Scholes 价格
                        d1 = (np.log(S/strike) + (0.05 + 0.5*iv**2)*T) / \
                             (iv*np.sqrt(T))
                        d2 = d1 - iv*np.sqrt(T)

                        if opt_type == 'call':
                            price = S*norm.cdf(d1) - strike*np.exp(-0.05*T)*norm.cdf(d2)
                        else:
                            price = strike*np.exp(-0.05*T)*norm.cdf(-d2) - S*norm.cdf(-d1)

                        # 添加买卖价差
                        spread = max(0.05, price * 0.02)
                        bid = max(0.01, price - spread/2)
                        ask = price + spread/2

                        records.append({
                            'date': date,
                            'expiry': expiry,
                            'strike': strike,
                            'type': opt_type,
                            'bid': round(bid, 2),
                            'ask': round(ask, 2),
                            'last': round(price, 2),
                            'volume': int(np.random.exponential(500)),
                            'open_interest': int(np.random.exponential(2000)),
                            'implied_vol': round(iv, 4),
                            'underlying_price': round(S, 2)
                        })

        df = pd.DataFrame(records)
        self.option_data[ticker] = df

        # 标的价格
        underlying = df.groupby('date')['underlying_price'].first()
        self.underlying_data[ticker] = underlying

        print(f"生成合成数据: {len(df)} 条期权记录, {len(dates)} 个交易日")

    def _clean_data(self, df: pd.DataFrame) -> pd.DataFrame:
        """数据清洗"""
        initial_len = len(df)

        # 删除无效数据
        df = df.dropna(subset=['bid', 'ask', 'strike', 'expiry'])

        # 删除 bid <= 0 的记录
        df = df[df['bid'] > 0]

        # 删除 bid > ask 的异常数据
        df = df[df['bid'] <= df['ask']]

        # 删除深度虚值且无成交的期权
        df = df[~((df['volume'] == 0) & (df['open_interest'] == 0))]

        # 按日期和执行价格排序
        df = df.sort_values(['date', 'expiry', 'strike', 'type'])

        cleaned = initial_len - len(df)
        if cleaned > 0:
            print(f"  数据清洗: 删除 {cleaned} 条无效记录")

        return df.reset_index(drop=True)

    def _generate_monthly_expiries(self, dates):
        """生成每月第三个周五作为到期日"""
        expiries = []
        seen_months = set()
        for date in dates:
            month_key = (date.year, date.month)
            if month_key not in seen_months:
                # 找到当月第三个周五
                first_day = date.replace(day=1)
                friday_count = 0
                for day in range(1, 32):
                    try:
                        d = first_day.replace(day=day)
                        if d.weekday() == 4:  # Friday
                            friday_count += 1
                            if friday_count == 3:
                                expiries.append(d)
                                seen_months.add(month_key)
                                break
                    except ValueError:
                        break
        return sorted(expiries)

    def get_option_chain(self, ticker: str, date: datetime,
                         expiry: datetime) -> pd.DataFrame:
        """获取指定日期和到期日的期权链"""
        df = self.option_data[ticker]
        mask = (df['date'] == date) & (df['expiry'] == expiry)
        return df[mask].copy()

    def get_nearest_expiry(self, ticker: str, date: datetime,
                           min_dte: int = 30) -> Optional[datetime]:
        """获取最近的合适到期日"""
        df = self.option_data[ticker]
        future_expiries = df[
            (df['date'] == date) & ((df['expiry'] - date).dt.days >= min_dte)
        ]['expiry'].unique()

        if len(future_expiries) == 0:
            return None
        return min(future_expiries)
```

---

## 四、策略信号定义

### 4.1 入场与出场条件

策略信号是回测的核心。对于期权策略，信号不仅包括方向判断，还包括合约选择规则。

```python
from abc import ABC, abstractmethod

@dataclass
class TradeSignal:
    """交易信号"""
    date: datetime
    ticker: str
    action: str  # 'open' / 'close' / 'roll'
    strategy_name: str
    legs: List[dict]  # 每条腿的信息
    reason: str
    metadata: dict = field(default_factory=dict)

@dataclass
class Leg:
    """期权策略的一条腿"""
    option_type: OptionType
    strike: float
    expiry: datetime
    quantity: int       # 正数=买入，负数=卖出
    entry_price: float
    current_price: float = 0.0

    @property
    def is_long(self) -> bool:
        return self.quantity > 0

    @property
    def pnl(self) -> float:
        return (self.current_price - self.entry_price) * self.quantity * 100


class BaseStrategy(ABC):
    """策略基类"""

    def __init__(self, name: str):
        self.name = name
        self.positions: List[dict] = []

    @abstractmethod
    def check_entry(self, date, data_manager, ticker) -> Optional[TradeSignal]:
        """检查入场条件"""
        pass

    @abstractmethod
    def check_exit(self, date, position, data_manager, ticker) -> Optional[TradeSignal]:
        """检查出场条件"""
        pass

    @abstractmethod
    def select_contracts(self, date, data_manager, ticker, expiry) -> List[dict]:
        """选择具体的期权合约"""
        pass
```

### 4.2 Iron Condor 策略实现

Iron Condor 是最受欢迎的中性期权策略之一，由同时卖出一个看涨信用价差（Bear Call Spread）和一个看跌信用价差（Bull Put Spread）组成。

```python
class IronCondorStrategy(BaseStrategy):
    """
    Iron Condor 策略

    适用于低波动率、横盘震荡的市场环境。
    策略逻辑：
    1. 选择合适的到期日（30-45 DTE）
    2. 卖出虚值看涨和看跌期权（Delta ≈ -0.16 / +0.16）
    3. 买入更远虚值的期权作为保护
    4. 设定止损和止盈规则
    """

    def __init__(self, short_delta: float = 0.16,
                 wing_width: float = 5.0,
                 min_dte: int = 30,
                 max_dte: int = 45,
                 profit_target: float = 0.50,
                 stop_loss: float = 2.0,
                 min_credit: float = 0.30,
                 max_spread_pct: float = 5.0):
        super().__init__("IronCondor")
        self.short_delta = short_delta
        self.wing_width = wing_width
        self.min_dte = min_dte
        self.max_dte = max_dte
        self.profit_target = profit_target  # 盈利目标（占最大利润的百分比）
        self.stop_loss = stop_loss          # 止损倍数（占初始信用的倍数）
        self.min_credit = min_credit        # 最小收取的信用
        self.max_spread_pct = max_spread_pct

    def check_entry(self, date, data_manager, ticker) -> Optional[TradeSignal]:
        """检查入场条件"""
        # 基本入场条件：可以是固定的日期规则或基于指标的信号
        # 这里使用简化逻辑：每月第一个交易日入场

        # 检查是否已有持仓
        if len(self.positions) > 0:
            return None

        # 找到合适的到期日
        expiry = data_manager.get_nearest_expiry(ticker, date, self.min_dte)
        if expiry is None:
            return None

        dte = (expiry - date).days
        if dte > self.max_dte:
            return None

        # 获取期权链
        chain = data_manager.get_option_chain(ticker, date, expiry)
        if len(chain) == 0:
            return None

        # 选择合约
        legs = self.select_contracts(date, data_manager, ticker, expiry)
        if legs is None or len(legs) != 4:
            return None

        # 计算总信用
        total_credit = sum(leg['credit'] for leg in legs if leg['quantity'] < 0)
        total_debit = sum(leg['debit'] for leg in legs if leg['quantity'] > 0)
        net_credit = total_credit - total_debit

        if net_credit < self.min_credit:
            return None

        return TradeSignal(
            date=date,
            ticker=ticker,
            action='open',
            strategy_name=self.name,
            legs=legs,
            reason=f"Iron Condor 入场, 净信用={net_credit:.2f}, DTE={dte}",
            metadata={'net_credit': net_credit, 'expiry': expiry, 'dte': dte}
        )

    def check_exit(self, date, position, data_manager, ticker) -> Optional[TradeSignal]:
        """检查出场条件"""
        expiry = position['expiry']
        entry_credit = position['net_credit']

        # 条件1：到期日平仓
        if date >= expiry:
            return TradeSignal(
                date=date, ticker=ticker, action='close',
                strategy_name=self.name,
                legs=position['legs'],
                reason="到期平仓"
            )

        # 条件2：盈利目标
        current_value = self._calculate_position_value(
            date, position, data_manager, ticker
        )
        if current_value is not None:
            profit = entry_credit - current_value
            if profit >= entry_credit * self.profit_target:
                return TradeSignal(
                    date=date, ticker=ticker, action='close',
                    strategy_name=self.name,
                    legs=position['legs'],
                    reason=f"达到盈利目标: {profit:.2f}"
                )

            # 条件3：止损
            loss = current_value - entry_credit
            if loss >= entry_credit * self.stop_loss:
                return TradeSignal(
                    date=date, ticker=ticker, action='close',
                    strategy_name=self.name,
                    legs=position['legs'],
                    reason=f"触发止损: 亏损 {loss:.2f}"
                )

        # 条件4：提前平仓（剩余 DTE 少于总 DTE 的 20%）
        remaining_dte = (expiry - date).days
        total_dte = (expiry - position['entry_date']).days
        if remaining_dte <= total_dte * 0.20:
            return TradeSignal(
                date=date, ticker=ticker, action='close',
                strategy_name=self.name,
                legs=position['legs'],
                reason=f"提前平仓: 剩余 DTE={remaining_dte}"
            )

        return None

    def select_contracts(self, date, data_manager, ticker, expiry):
        """选择 Iron Condor 的四条腿"""
        chain = data_manager.get_option_chain(ticker, date, expiry)
        if len(chain) == 0:
            return None

        spot = chain['underlying_price'].iloc[0]

        # 分离看涨和看跌
        calls = chain[chain['type'] == 'call'].copy()
        puts = chain[chain['type'] == 'put'].copy()

        if len(calls) == 0 or len(puts) == 0:
            return None

        # 计算近似 Delta
        calls['approx_delta'] = self._estimate_delta(
            spot, calls['strike'], calls['implied_vol'],
            (expiry - date).days / 365, 'call'
        )
        puts['approx_delta'] = self._estimate_delta(
            spot, puts['strike'], puts['implied_vol'],
            (expiry - date).days / 365, 'put'
        )

        # 选择卖出行权价：最接近目标 Delta 的合约
        # 看涨侧：卖出行权价 ≈ Delta 0.16 的 call
        short_call = calls.iloc[
            (calls['approx_delta'] - self.short_delta).abs().argsort().iloc[0]
        ]
        # 看跌侧：卖出行权价 ≈ Delta -0.16 的 put
        short_put = puts.iloc[
            (puts['approx_delta'] + self.short_delta).abs().argsort().iloc[0]
        ]

        # 选择买入行权价：wing_width 之外
        long_call_strike = short_call['strike'] + self.wing_width
        long_put_strike = short_put['strike'] - self.wing_width

        long_call = calls[calls['strike'] == long_call_strike]
        long_put = puts[puts['strike'] == long_put_strike]

        if long_call.empty or long_put.empty:
            return None

        long_call = long_call.iloc[0]
        long_put = long_put.iloc[0]

        # 构建四条腿
        legs = [
            {
                'option_type': 'put', 'strike': short_put['strike'],
                'expiry': expiry, 'quantity': 1,
                'credit': short_put['bid'], 'debit': 0,
                'label': 'Short Put'
            },
            {
                'option_type': 'put', 'strike': long_put['strike'],
                'expiry': expiry, 'quantity': -1,
                'credit': 0, 'debit': long_put['ask'],
                'label': 'Long Put'
            },
            {
                'option_type': 'call', 'strike': short_call['strike'],
                'expiry': expiry, 'quantity': 1,
                'credit': short_call['bid'], 'debit': 0,
                'label': 'Short Call'
            },
            {
                'option_type': 'call', 'strike': long_call['strike'],
                'expiry': expiry, 'quantity': -1,
                'credit': 0, 'debit': long_call['ask'],
                'label': 'Long Call'
            },
        ]

        return legs

    def _estimate_delta(self, S, strikes, ivs, T, opt_type):
        """快速估算 Delta"""
        from scipy.stats import norm
        r = 0.05
        d1 = (np.log(S / strikes) + (r + 0.5 * ivs**2) * T) / (ivs * np.sqrt(T))
        if opt_type == 'call':
            return norm.cdf(d1)
        else:
            return norm.cdf(d1) - 1

    def _calculate_position_value(self, date, position, data_manager, ticker):
        """计算当前持仓价值"""
        chain = data_manager.get_option_chain(
            ticker, date, position['expiry']
        )
        if len(chain) == 0:
            return None

        total_value = 0
        for leg in position['legs']:
            mask = (chain['strike'] == leg['strike']) & \
                   (chain['type'] == leg['option_type'])
            contract = chain[mask]
            if len(contract) == 0:
                return None

            mid = (contract.iloc[0]['bid'] + contract.iloc[0]['ask']) / 2
            if leg['quantity'] > 0:  # 卖出
                total_value += mid * abs(leg['quantity'])
            else:  # 买入
                total_value -= mid * abs(leg['quantity'])

        return total_value
```

---

## 五、回测执行引擎

### 5.1 事件驱动引擎

```python
from collections import defaultdict

@dataclass
class Position:
    """持仓记录"""
    id: str
    entry_date: datetime
    expiry: datetime
    strategy_name: str
    legs: List[dict]
    net_credit: float
    status: str  # 'open' / 'closed'
    exit_date: Optional[datetime] = None
    exit_price: Optional[float] = None
    pnl: float = 0.0

@dataclass
class TradeRecord:
    """成交记录"""
    date: datetime
    ticker: str
    strategy: str
    action: str
    strike: float
    option_type: str
    quantity: int
    price: float
    commission: float


class BacktestEngine:
    """
    回测执行引擎

    模拟真实的交易执行流程，包括：
    - 订单管理
    - 成交模拟（含滑点和手续费）
    - 保证金计算
    - 持仓跟踪
    """

    def __init__(self, initial_capital: float = 100000,
                 commission_per_contract: float = 0.65,
                 slippage_pct: float = 0.5,
                 max_positions: int = 5):
        """
        Args:
            initial_capital: 初始资金
            commission_per_contract: 每张合约手续费
            slippage_pct: 滑点百分比
            max_positions: 最大同时持仓数
        """
        self.initial_capital = initial_capital
        self.capital = initial_capital
        self.commission = commission_per_contract
        self.slippage_pct = slippage_pct / 100
        self.max_positions = max_positions

        self.positions: List[Position] = []
        self.closed_positions: List[Position] = []
        self.trade_history: List[TradeRecord] = []
        self.equity_curve: List[dict] = []
        self.position_counter = 0

    def execute_signal(self, signal: TradeSignal) -> bool:
        """执行交易信号"""
        if signal.action == 'open':
            return self._open_position(signal)
        elif signal.action == 'close':
            return self._close_position(signal)
        elif signal.action == 'roll':
            return self._roll_position(signal)
        return False

    def _open_position(self, signal: TradeSignal) -> bool:
        """开仓"""
        if len(self.positions) >= self.max_positions:
            return False

        # 计算净信用
        net_credit = signal.metadata.get('net_credit', 0)

        # 计算保证金要求（简化估算）
        max_risk = self._estimate_max_risk(signal.legs)
        if max_risk > self.capital * 0.5:  # 单笔不超过50%资金
            return False

        # 创建持仓
        self.position_counter += 1
        position = Position(
            id=f"POS-{self.position_counter:04d}",
            entry_date=signal.date,
            expiry=signal.metadata.get('expiry'),
            strategy_name=signal.strategy_name,
            legs=signal.legs,
            net_credit=net_credit,
            status='open'
        )

        self.positions.append(position)

        # 记录交易
        for leg in signal.legs:
            price = leg.get('credit', 0) or leg.get('debit', 0)
            adjusted_price = self._apply_slippage(price, leg['quantity'])
            self.trade_history.append(TradeRecord(
                date=signal.date,
                ticker=signal.ticker,
                strategy=signal.strategy_name,
                action='SELL' if leg['quantity'] > 0 else 'BUY',
                strike=leg['strike'],
                option_type=leg['option_type'],
                quantity=abs(leg['quantity']),
                price=adjusted_price,
                commission=self.commission
            ))

        return True

    def _close_position(self, signal: TradeSignal) -> bool:
        """平仓"""
        # 找到匹配的持仓
        position = None
        for pos in self.positions:
            if pos.status == 'open' and pos.strategy_name == signal.strategy_name:
                # 匹配到期日
                if any(leg.get('expiry') == pos.expiry for leg in signal.legs):
                    position = pos
                    break

        if position is None:
            return False

        # 计算平仓成本/收入
        close_cost = 0
        for leg in signal.legs:
            price = leg.get('debit', 0) or leg.get('credit', 0)
            adjusted_price = self._apply_slippage(price, -leg['quantity'])
            close_cost += adjusted_price * abs(leg['quantity']) * 100

            self.trade_history.append(TradeRecord(
                date=signal.date,
                ticker=signal.ticker,
                strategy=signal.strategy_name,
                action='BUY' if leg['quantity'] > 0 else 'SELL',
                strike=leg['strike'],
                option_type=leg['option_type'],
                quantity=abs(leg['quantity']),
                price=adjusted_price,
                commission=self.commission
            ))

        # 计算盈亏
        entry_value = position.net_credit * 100  # 每张合约100股
        total_commission = self.commission * len(signal.legs) * 2
        pnl = entry_value - close_cost - total_commission

        position.status = 'closed'
        position.exit_date = signal.date
        position.exit_price = close_cost
        position.pnl = pnl

        self.capital += pnl
        self.positions.remove(position)
        self.closed_positions.append(position)

        return True

    def _roll_position(self, signal: TradeSignal) -> bool:
        """展期操作：平旧仓 + 开新仓"""
        # 先平旧仓
        close_signal = TradeSignal(
            date=signal.date, ticker=signal.ticker,
            action='close', strategy_name=signal.strategy_name,
            legs=signal.legs, reason='展期平仓'
        )
        self._close_position(close_signal)

        # 再开新仓（使用新到期日的合约）
        open_signal = TradeSignal(
            date=signal.date, ticker=signal.ticker,
            action='open', strategy_name=signal.strategy_name,
            legs=signal.metadata.get('new_legs', []),
            reason='展期开仓', metadata=signal.metadata
        )
        return self._open_position(open_signal)

    def _apply_slippage(self, price: float, quantity: int) -> float:
        """应用滑点"""
        if quantity > 0:  # 买入，价格上升
            return price * (1 + self.slippage_pct)
        else:  # 卖出，价格下降
            return price * (1 - self.slippage_pct)

    def _estimate_max_risk(self, legs: List[dict]) -> float:
        """估算最大风险（用于保证金检查）"""
        # 对于 Iron Condor：最大风险 = wing_width - 净信用
        wing_width = 0
        net_credit = 0

        strikes = sorted([leg['strike'] for leg in legs])
        if len(strikes) >= 4:
            wing_width = (strikes[-1] - strikes[-2])  # call wing
            net_credit = sum(leg.get('credit', 0) for leg in legs) - \
                        sum(leg.get('debit', 0) for leg in legs)

        max_loss_per_contract = wing_width - net_credit
        return max_loss_per_contract * 100  # 每张合约100股

    def update_equity(self, date, data_manager, ticker):
        """更新每日权益曲线"""
        # 计算未实现盈亏
        unrealized_pnl = 0
        for position in self.positions:
            # 简化：使用 net_credit 估算
            pass  # 完整实现需要获取当前市场价格

        total_equity = self.capital + unrealized_pnl

        self.equity_curve.append({
            'date': date,
            'equity': total_equity,
            'cash': self.capital,
            'unrealized_pnl': unrealized_pnl,
            'n_positions': len(self.positions)
        })

    def run(self, strategy: BaseStrategy, data_manager: DataManager,
            ticker: str, start_date: datetime, end_date: datetime):
        """
        运行回测

        Args:
            strategy: 策略实例
            data_manager: 数据管理器
            ticker: 标的代码
            start_date: 回测开始日期
            end_date: 回测结束日期
        """
        print(f"\n{'='*60}")
        print(f"开始回测: {strategy.name}")
        print(f"期间: {start_date.date()} ~ {end_date.date()}")
        print(f"初始资金: ${self.initial_capital:,.2f}")
        print(f"{'='*60}")

        # 获取交易日列表
        df = data_manager.option_data[ticker]
        trading_days = sorted(df[
            (df['date'] >= start_date) & (df['date'] <= end_date)
        ]['date'].unique())

        for date in trading_days:
            # 1. 检查现有持仓是否需要平仓
            positions_to_close = []
            for position in self.positions:
                exit_signal = strategy.check_exit(
                    date, position.__dict__, data_manager, ticker
                )
                if exit_signal:
                    self.execute_signal(exit_signal)
                    positions_to_close.append(position)

            # 2. 检查是否需要开新仓位
            entry_signal = strategy.check_entry(date, data_manager, ticker)
            if entry_signal:
                self.execute_signal(entry_signal)

            # 3. 更新权益
            self.update_equity(date, data_manager, ticker)

        # 回测结束，强制平仓所有持仓
        for position in self.positions.copy():
            close_signal = TradeSignal(
                date=end_date, ticker=ticker,
                action='close', strategy_name=strategy.name,
                legs=position.legs, reason="回测结束强制平仓"
            )
            self.execute_signal(close_signal)

        print(f"\n回测完成: {len(self.closed_positions)} 笔交易")
        return self.generate_report()

    def generate_report(self) -> pd.DataFrame:
        """生成回测报告"""
        if len(self.equity_curve) == 0:
            return pd.DataFrame()

        equity_df = pd.DataFrame(self.equity_curve)
        equity_df.set_index('date', inplace=True)

        return equity_df
```

---

## 六、绩效评估

### 6.1 核心绩效指标

```python
import numpy as np
import pandas as pd

class PerformanceAnalyzer:
    """
    绩效分析器

    计算回测的核心绩效指标
    """

    def __init__(self, equity_curve: pd.DataFrame,
                 risk_free_rate: float = 0.05):
        self.equity = equity_curve
        self.rf = risk_free_rate

        # 计算日收益率
        self.equity['daily_return'] = self.equity['equity'].pct_change()
        self.equity = self.equity.dropna()

    def total_return(self) -> float:
        """总收益率"""
        return (self.equity['equity'].iloc[-1] /
                self.equity['equity'].iloc[0]) - 1

    def annualized_return(self) -> float:
        """年化收益率"""
        n_days = len(self.equity)
        total = self.total_return()
        return (1 + total) ** (252 / n_days) - 1

    def annualized_volatility(self) -> float:
        """年化波动率"""
        return self.equity['daily_return'].std() * np.sqrt(252)

    def sharpe_ratio(self) -> float:
        """
        夏普比率（Sharpe Ratio）

        衡量每单位风险所获得的超额收益
        Sharpe = (R_p - R_f) / sigma_p
        """
        excess_return = self.annualized_return() - self.rf
        vol = self.annualized_volatility()
        if vol == 0:
            return 0
        return excess_return / vol

    def sortino_ratio(self) -> float:
        """
        索提诺比率（Sortino Ratio）

        只考虑下行风险，比夏普比率更适合非对称收益分布
        """
        excess_return = self.annualized_return() - self.rf
        downside_returns = self.equity['daily_return'][
            self.equity['daily_return'] < 0
        ]
        downside_vol = downside_returns.std() * np.sqrt(252)
        if downside_vol == 0:
            return 0
        return excess_return / downside_vol

    def max_drawdown(self) -> dict:
        """
        最大回撤（Maximum Drawdown）

        衡量从最高点到最低点的最大跌幅
        """
        equity = self.equity['equity']
        peak = equity.expanding().max()
        drawdown = (equity - peak) / peak

        max_dd = drawdown.min()
        max_dd_end = drawdown.idxmin()
        max_dd_start = equity[:max_dd_end].idxmax()

        # 回撤恢复时间
        recovery = equity[max_dd_end:]
        recovery_date = recovery[recovery >= equity[max_dd_start]].first_valid_index()

        return {
            'max_drawdown': max_dd,
            'start_date': max_dd_start,
            'end_date': max_dd_end,
            'recovery_date': recovery_date,
            'recovery_days': (recovery_date - max_dd_end).days if recovery_date else None
        }

    def calmar_ratio(self) -> float:
        """
        卡尔玛比率（Calmar Ratio）

        = 年化收益率 / 最大回撤绝对值
        """
        mdd = abs(self.max_drawdown()['max_drawdown'])
        if mdd == 0:
            return float('inf')
        return self.annualized_return() / mdd

    def win_rate(self, trades: List[Position]) -> float:
        """胜率"""
        if len(trades) == 0:
            return 0
        wins = sum(1 for t in trades if t.pnl > 0)
        return wins / len(trades)

    def profit_factor(self, trades: List[Position]) -> float:
        """
        盈亏比（Profit Factor）

        = 总盈利 / 总亏损
        """
        gross_profit = sum(t.pnl for t in trades if t.pnl > 0)
        gross_loss = abs(sum(t.pnl for t in trades if t.pnl < 0))
        if gross_loss == 0:
            return float('inf')
        return gross_profit / gross_loss

    def avg_trade_pnl(self, trades: List[Position]) -> dict:
        """平均交易盈亏统计"""
        pnls = [t.pnl for t in trades]
        wins = [p for p in pnls if p > 0]
        losses = [p for p in pnls if p < 0]

        return {
            'total_trades': len(trades),
            'avg_pnl': np.mean(pnls) if pnls else 0,
            'avg_win': np.mean(wins) if wins else 0,
            'avg_loss': np.mean(losses) if losses else 0,
            'largest_win': max(wins) if wins else 0,
            'largest_loss': min(losses) if losses else 0,
            'win_rate': len(wins) / len(pnls) if pnls else 0,
        }

    def monthly_returns(self) -> pd.DataFrame:
        """月度收益率"""
        monthly = self.equity['daily_return'].resample('M').apply(
            lambda x: (1 + x).prod() - 1
        )
        return monthly

    def generate_full_report(self, trades: List[Position] = None) -> dict:
        """生成完整的绩效报告"""
        mdd = self.max_drawdown()

        report = {
            '--- 收益指标 ---': '',
            '总收益率': f"{self.total_return():.2%}",
            '年化收益率': f"{self.annualized_return():.2%}",
            '年化波动率': f"{self.annualized_volatility():.2%}",

            '--- 风险指标 ---': '',
            '最大回撤': f"{mdd['max_drawdown']:.2%}",
            '回撤开始': str(mdd['start_date'].date()),
            '回撤结束': str(mdd['end_date'].date()),
            '回撤恢复': str(mdd['recovery_date'].date()) if mdd['recovery_date'] else 'N/A',

            '--- 风险调整收益 ---': '',
            '夏普比率': f"{self.sharpe_ratio():.2f}",
            '索提诺比率': f"{self.sortino_ratio():.2f}",
            '卡尔玛比率': f"{self.calmar_ratio():.2f}",
        }

        if trades:
            avg = self.avg_trade_pnl(trades)
            report.update({
                '--- 交易统计 ---': '',
                '总交易次数': avg['total_trades'],
                '胜率': f"{avg['win_rate']:.1%}",
                '平均盈亏': f"${avg['avg_pnl']:.2f}",
                '平均盈利': f"${avg['avg_win']:.2f}",
                '平均亏损': f"${avg['avg_loss']:.2f}",
                '最大单笔盈利': f"${avg['largest_win']:.2f}",
                '最大单笔亏损': f"${avg['largest_loss']:.2f}",
                '盈亏比': f"{self.profit_factor(trades):.2f}",
            })

        return report

    def print_report(self, trades: List[Position] = None):
        """打印绩效报告"""
        report = self.generate_full_report(trades)

        print(f"\n{'='*50}")
        print(f"{'回测绩效报告':^40}")
        print(f"{'='*50}")

        for key, value in report.items():
            if value == '':
                print(f"\n{key}")
            else:
                print(f"  {key:.<30} {value}")

        print(f"\n{'='*50}")

    def plot_equity_curve(self, benchmark: pd.Series = None):
        """绘制权益曲线"""
        import matplotlib.pyplot as plt

        fig, axes = plt.subplots(3, 1, figsize=(14, 10), height_ratios=[3, 1, 1])

        # 权益曲线
        ax1 = axes[0]
        ax1.plot(self.equity.index, self.equity['equity'],
                 'b-', linewidth=1.5, label='Strategy')
        if benchmark is not None:
            normalized = benchmark / benchmark.iloc[0] * self.equity['equity'].iloc[0]
            ax1.plot(normalized.index, normalized,
                     'gray', linewidth=1, alpha=0.7, label='Benchmark')
        ax1.set_title('权益曲线')
        ax1.set_ylabel('权益 ($)')
        ax1.legend()
        ax1.grid(True, alpha=0.3)

        # 回撤
        ax2 = axes[1]
        equity = self.equity['equity']
        peak = equity.expanding().max()
        drawdown = (equity - peak) / peak
        ax2.fill_between(drawdown.index, drawdown, 0,
                          color='red', alpha=0.3)
        ax2.set_ylabel('回撤 (%)')
        ax2.set_title('回撤')
        ax2.grid(True, alpha=0.3)

        # 月度收益
        ax3 = axes[2]
        monthly = self.monthly_returns()
        colors = ['green' if x >= 0 else 'red' for x in monthly]
        ax3.bar(monthly.index, monthly, width=20, color=colors, alpha=0.7)
        ax3.set_ylabel('月度收益 (%)')
        ax3.set_title('月度收益')
        ax3.grid(True, alpha=0.3)

        plt.tight_layout()
        plt.savefig('backtest_report.png', dpi=150, bbox_inches='tight')
        plt.show()
```

---

## 七、常见陷阱

### 7.1 过拟合（Overfitting）

过拟合是回测中最常见也最危险的陷阱。当策略参数过度拟合历史数据时，它在样本内表现优异，但在样本外往往表现糟糕。

**识别过拟合的方法**：
1. **样本外测试**：将数据分为训练集和测试集
2. **参数稳定性**：检查参数微小变动是否导致结果剧烈变化
3. **交叉验证**：使用 Walk-Forward 分析
4. **逻辑合理性**：策略是否有合理的经济逻辑支撑

```python
def walk_forward_analysis(strategy_class, data_manager, ticker,
                          train_months=6, test_months=2, n_splits=5):
    """
    Walk-Forward 分析

    将历史数据分为多个滚动窗口，每个窗口内：
    - 前 train_months 用于参数优化
    - 后 test_months 用于样本外验证

    这比简单的 train/test split 更适合时间序列数据
    """
    df = data_manager.option_data[ticker]
    dates = sorted(df['date'].unique())
    total_days = len(dates)
    train_days = train_months * 21  # 约21个交易日/月
    test_days = test_months * 21
    window_days = train_days + test_days

    results = []

    for i in range(n_splits):
        start_idx = i * test_days
        end_idx = start_idx + window_days

        if end_idx > total_days:
            break

        train_start = dates[start_idx]
        train_end = dates[start_idx + train_days - 1]
        test_start = dates[start_idx + train_days]
        test_end = dates[min(end_idx - 1, total_days - 1)]

        print(f"\n窗口 {i+1}: 训练 {train_start.date()} ~ {train_end.date()}, "
              f"测试 {test_start.date()} ~ {test_end.date()}")

        # 在训练集上优化参数（这里简化为使用默认参数）
        strategy = strategy_class()

        # 在测试集上评估
        engine = BacktestEngine(initial_capital=100000)
        equity = engine.run(strategy, data_manager, ticker, test_start, test_end)

        if len(equity) > 0:
            analyzer = PerformanceAnalyzer(equity)
            results.append({
                'window': i + 1,
                'train_period': f"{train_start.date()} ~ {train_end.date()}",
                'test_period': f"{test_start.date()} ~ {test_end.date()}",
                'total_return': analyzer.total_return(),
                'sharpe': analyzer.sharpe_ratio(),
                'max_drawdown': analyzer.max_drawdown()['max_drawdown'],
                'win_rate': analyzer.win_rate(engine.closed_positions)
            })

    results_df = pd.DataFrame(results)
    print("\n=== Walk-Forward 分析结果 ===")
    print(results_df.to_string(index=False))

    return results_df
```

### 7.2 生存偏差（Survivorship Bias）

生存偏差是指只使用当前仍然存在的股票/期权数据进行回测，忽略了已经退市的标的。这会导致回测结果偏高。

**解决方案**：
- 使用包含已退市标的的完整历史数据库
- 在合成数据中包含失败案例
- 使用 Survivorship-Free 数据源

### 7.3 前瞻偏差（Look-Ahead Bias）

前瞻偏差是指在回测中使用了在实际交易时尚不可获得的信息。

**常见场景**：
- 使用当日收盘价决定当日开盘时的交易
- 使用未来修订后的财务数据
- 使用当日的隐含波动率来选择当日到期的合约

**解决方案**：
- 严格使用 `shift(1)` 确保信号只使用历史数据
- 在数据对齐时使用 `asof` 而非 `merge`
- 建立严格的时间序列检查机制

### 7.4 交易成本忽视

对于期权卖方策略，交易成本（手续费 + 滑点 + 买卖价差）可能占到利润的很大比例。

```python
def sensitivity_to_costs(strategy, data_manager, ticker,
                          start_date, end_date):
    """分析交易成本对策略收益的影响"""
    cost_scenarios = [
        {'commission': 0.0, 'slippage': 0.0, 'label': '零成本'},
        {'commission': 0.65, 'slippage': 0.0, 'label': '仅手续费'},
        {'commission': 0.65, 'slippage': 0.3, 'label': '低滑点'},
        {'commission': 0.65, 'slippage': 0.5, 'label': '中等滑点'},
        {'commission': 0.65, 'slippage': 1.0, 'label': '高滑点'},
    ]

    print("=== 交易成本敏感性分析 ===")
    for scenario in cost_scenarios:
        engine = BacktestEngine(
            initial_capital=100000,
            commission_per_contract=scenario['commission'],
            slippage_pct=scenario['slippage']
        )
        equity = engine.run(strategy, data_manager, ticker, start_date, end_date)
        if len(equity) > 0:
            analyzer = PerformanceAnalyzer(equity)
            print(f"\n{scenario['label']}:")
            print(f"  总收益: {analyzer.total_return():.2%}")
            print(f"  夏普比率: {analyzer.sharpe_ratio():.2f}")
```

---

## 八、完整的 Iron Condor 回测示例

```python
def run_full_backtest():
    """完整的 Iron Condor 回测流程"""

    # 1. 初始化数据管理器
    dm = DataManager()
    dm.generate_synthetic_data(
        ticker='SPY',
        start_date='2020-01-01',
        end_date='2024-12-31',
        S0=320,
        sigma=0.18
    )

    # 2. 初始化策略
    strategy = IronCondorStrategy(
        short_delta=0.16,
        wing_width=5.0,
        min_dte=30,
        max_dte=45,
        profit_target=0.50,
        stop_loss=2.0,
        min_credit=0.30
    )

    # 3. 初始化回测引擎
    engine = BacktestEngine(
        initial_capital=100000,
        commission_per_contract=0.65,
        slippage_pct=0.5,
        max_positions=3
    )

    # 4. 运行回测
    equity = engine.run(
        strategy=strategy,
        data_manager=dm,
        ticker='SPY',
        start_date=pd.Timestamp('2021-01-01'),
        end_date=pd.Timestamp('2024-12-31')
    )

    # 5. 绩效分析
    if len(equity) > 0:
        analyzer = PerformanceAnalyzer(equity)
        analyzer.print_report(engine.closed_positions)

        # 6. 绘制权益曲线
        analyzer.plot_equity_curve()

    return engine, analyzer


# 运行回测
if __name__ == "__main__":
    engine, analyzer = run_full_backtest()
```

---

## 小结

本章构建了一个完整的期权策略回测框架，涵盖了：

- **数据管理**：历史数据加载、清洗、合成数据生成
- **策略框架**：抽象基类和 Iron Condor 具体实现
- **执行引擎**：模拟下单、滑点、手续费
- **绩效评估**：收益率、夏普比率、最大回撤、胜率等核心指标
- **常见陷阱**：过拟合、生存偏差、前瞻偏差的识别与防范

回测不是目的，而是手段。好的回测应该帮助你理解策略的本质特征，而不是单纯追求漂亮的历史数字。

### 思考题

1. 实现 Covered Call 策略的回测，并与 Iron Condor 的结果进行对比分析。
2. 在 Walk-Forward 分析中，尝试不同的训练/测试窗口比例，观察参数稳定性如何变化。
3. 添加一个基于 VIX 指数的过滤器，当 VIX 高于某阈值时暂停 Iron Condor 开仓，比较过滤前后的绩效差异。

---

**下一章：** [04-signal-strategy.md —— 信号生成与策略开发](04-signal-strategy.md)




---
**相关笔记：** [[Python期权库入门]], [[信号生成与策略开发]]
**返回：** [[chapter00-python-ai]] | [[chapter01-主页]]
