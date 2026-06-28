---
type: concept
topic: Python+AI
aliases: [IBKR API, 自动交易]
status: done
created: "2026-06-28"
updated: "2026-06-28"
tags: [期权, Python, IBKR]
related: "[[API与自动化基础]], [[信号生成与策略开发]]"
---

# IBKR API 与自动交易

## 引言

前面几章我们学习了期权定价、策略回测和信号生成。本章将进入最关键的环节——实盘自动交易。Interactive Brokers（IBKR）是全球最大的在线券商之一，其 TWS API 为专业交易者提供了强大的程序化交易接口。我们将详细介绍如何使用 Python 的 `ib_insync` 库连接 IBKR API，获取实时数据、管理期权链、自动下单以及构建一个完整的交易机器人。

---

## 一、IBKR API 架构概述

### 1.1 TWS API vs Client Portal API

Interactive Brokers 提供两种主要的 API 接口：

**TWS API（Trader Workstation API）**：
- 基于 TCP Socket 的传统 API
- 需要运行 TWS 桌面软件或 IB Gateway
- 功能最完整，支持所有订单类型
- 延迟低，适合高频和算法交易
- 本章重点介绍此接口

**Client Portal Web API**：
- 基于 REST/JSON 的 Web API
- 不需要运行 TWS，但需要 Client Portal 软件
- 功能相对有限
- 更适合账户管理和查询
- 使用 HTTPS 协议，更易部署

### 1.2 API 通信模型

TWS API 使用异步通信模型：

```
┌──────────────┐     TCP Socket      ┌──────────────┐
│  Python 程序  │ <──────────────────> │  TWS/Gateway  │
│  (ib_insync) │                      │  (端口7497)   │
└──────────────┘                      └──────────────┘
       │                                      │
       │  1. 连接请求 (clientId)               │
       │ ──────────────────────────────────>   │
       │                                      │
       │  2. 订阅数据 (reqMktData)             │
       │ ──────────────────────────────────>   │
       │                                      │
       │  3. 实时报价回调                       │
       │ <──────────────────────────────────   │
       │                                      │
       │  4. 提交订单 (placeOrder)              │
       │ ──────────────────────────────────>   │
       │                                      │
       │  5. 订单状态更新                       │
       │ <──────────────────────────────────   │
```

### 1.3 端口与 clientId

- **端口号**：默认 7497（TWS 实盘）/ 7496（TWS 模拟）/ 4001（Gateway 实盘）/ 4002（Gateway 模拟）
- **clientId**：每个 API 连接需要一个唯一的客户端 ID（0-32）
  - 同一 clientId 不能被多个程序同时使用
  - 建议为不同功能使用不同的 clientId

---

## 二、ib_insync 库详解

### 2.1 为什么选择 ib_insync

`ib_insync` 是 IBKR TWS API 的 Python 封装库，由 Ewald de Wit 开发。相比官方的 `ibapi`，`ib_insync` 有以下优势：

1. **异步/同步统一**：使用 `asyncio` 事件循环，但提供了同步接口
2. **简洁的 API**：将复杂的回调模式转换为直观的函数调用
3. **自动重连**：网络断开后自动重新连接
4. **类型安全**：使用 Python dataclass 表示所有数据对象
5. **活跃的社区**：文档完善，问题解答及时

```python
# 安装 ib_insync
# pip install ib_insync

# 对比 ibapi 和 ib_insync
# ibapi（官方库，复杂）：
# class MyWrapper(EWrapper):
#     def tickPrice(self, reqId, tickType, price, attrib):
#         print(f"Price: {price}")
#
# ib_insync（简化版）：
# ticker = ib.reqMktData(contract)
# print(ticker.last)  # 直接获取价格
```

### 2.2 核心数据对象

`ib_insync` 将 IBKR API 的数据封装为 Python 对象：

```python
from ib_insync import *

# Contract（合约）对象
stock = Stock('AAPL', 'SMART', 'USD')        # 股票
option = Option('AAPL', '20240621', 150, 'C', 'SMART')  # 期权
future = Future('ES', '202406', 'CME')       # 期货
forex = Forex('EURUSD')                       # 外汇

# Order（订单）对象
market_order = MarketOrder('BUY', 100)
limit_order = LimitOrder('BUY', 100, 150.00)

# Trade（交易）对象 - 提交订单后返回
# trade = ib.placeOrder(stock, market_order)

# Ticker（行情）对象 - 订阅数据后更新
# ticker = ib.reqMktData(stock)

# Position（持仓）对象
# positions = ib.positions()

# AccountValue（账户价值）对象
# account = ib.accountValues()
```

---

## 三、连接与认证

### 3.1 建立连接

```python
from ib_insync import *
import asyncio
import logging

class IBKRConnection:
    """
    IBKR 连接管理器

    封装连接建立、断开重连和错误处理逻辑
    """

    def __init__(self, host: str = '127.0.0.1',
                 port: int = 7497,
                 client_id: int = 1,
                 account: str = ''):
        """
        Args:
            host: TWS 运行的主机地址
            port: API 端口号
            client_id: 客户端标识
            account: 账户号（留空则使用默认账户）
        """
        self.host = host
        self.port = port
        self.client_id = client_id
        self.account = account
        self.ib = IB()

        # 配置日志
        self.logger = logging.getLogger('IBKRConnection')
        self.logger.setLevel(logging.INFO)

    def connect(self, timeout: int = 20) -> bool:
        """
        建立与 TWS 的连接

        Args:
            timeout: 连接超时时间（秒）

        Returns:
            连接是否成功
        """
        try:
            self.ib.connect(
                host=self.host,
                port=self.port,
                clientId=self.client_id,
                timeout=timeout,
                account=self.account
            )

            # 注册事件处理器
            self.ib.errorEvent += self._on_error
            self.ib.disconnectedEvent += self._on_disconnected

            self.logger.info(
                f"成功连接到 TWS - "
                f"Server Version: {self.ib.serverVersion()}, "
                f"Accounts: {self.ib.managedAccounts()}"
            )

            return True

        except Exception as e:
            self.logger.error(f"连接失败: {e}")
            return False

    def disconnect(self):
        """断开连接"""
        if self.ib.isConnected():
            self.ib.disconnect()
            self.logger.info("已断开 TWS 连接")

    def is_connected(self) -> bool:
        """检查连接状态"""
        return self.ib.isConnected()

    def _on_error(self, reqId, errorCode, errorString, contract):
        """错误回调处理"""
        # 忽略信息性消息
        informational_codes = {2104, 2106, 2158}  # 数据连接状态消息
        if errorCode in informational_codes:
            self.logger.info(f"[信息] Code={errorCode}: {errorString}")
        else:
            self.logger.error(
                f"[错误] reqId={reqId}, Code={errorCode}: {errorString}"
            )

    def _on_disconnected(self):
        """断开连接回调"""
        self.logger.warning("TWS 连接已断开，尝试重连...")
        # 自动重连逻辑
        try:
            self.connect()
        except:
            self.logger.error("重连失败")

    def __enter__(self):
        self.connect()
        return self

    def __exit__(self, *args):
        self.disconnect()


# 使用示例
def test_connection():
    """测试连接"""
    conn = IBKRConnection(port=7497, client_id=1)

    if conn.connect():
        ib = conn.ib

        # 获取账户信息
        accounts = ib.managedAccounts()
        print(f"可用账户: {accounts}")

        # 获取账户价值
        account_values = ib.accountValues()
        for av in account_values:
            if av.tag in ['TotalCashValue', 'NetLiquidation',
                          'UnrealizedPPL', 'AvailableFunds']:
                print(f"  {av.tag}: {av.value} {av.currency}")

        conn.disconnect()
    else:
        print("连接失败，请确保 TWS 或 IB Gateway 正在运行")


# 运行测试
# test_connection()
```

### 3.2 连接配置最佳实践

```python
# config.py - IBKR 连接配置

IBKR_CONFIG = {
    # 实盘配置
    'live': {
        'host': '127.0.0.1',
        'port': 7497,          # TWS 实盘端口
        'client_id': 1,
        'account': '',         # 留空使用默认账户
    },
    # 模拟盘配置
    'paper': {
        'host': '127.0.0.1',
        'port': 7496,          # TWS 模拟端口
        'client_id': 1,
        'account': '',
    },
    # Gateway 配置
    'gateway': {
        'host': '127.0.0.1',
        'port': 4001,          # Gateway 实盘端口
        'client_id': 1,
        'account': '',
    }
}

# TWS API 设置建议：
# 1. File -> Global Configuration -> API -> Settings
# 2. 启用 "Enable ActiveX and Socket Clients"
# 3. 设置 "Socket port" 为 7497（实盘）或 7496（模拟）
# 4. 在 "Trusted IP Addresses" 中添加 127.0.0.1
# 5. 取消勾选 "Read-Only API"（如果需要下单）
```

---

## 四、获取实时报价

### 4.1 股票实时报价

```python
from ib_insync import *
import pandas as pd

class MarketDataProvider:
    """市场数据提供器"""

    def __init__(self, ib: IB):
        self.ib = ib
        self.subscriptions = {}

    def get_stock_quote(self, symbol: str,
                         exchange: str = 'SMART',
                         currency: str = 'USD') -> dict:
        """
        获取股票实时报价

        Args:
            symbol: 股票代码
            exchange: 交易所
            currency: 货币

        Returns:
            包含实时报价的字典
        """
        contract = Stock(symbol, exchange, currency)

        # 合约详情验证
        self.ib.qualifyContracts(contract)

        # 请求实时行情
        ticker = self.ib.reqMktData(contract, '', False, False)
        self.ib.sleep(2)  # 等待数据到达

        quote = {
            'symbol': symbol,
            'last': ticker.last,
            'bid': ticker.bid,
            'ask': ticker.ask,
            'bid_size': ticker.bidSize,
            'ask_size': ticker.askSize,
            'volume': ticker.volume,
            'high': ticker.high,
            'low': ticker.low,
            'open': ticker.open,
            'close': ticker.close,
            'vwap': ticker.vwap,
            'timestamp': ticker.time
        }

        # 取消订阅以释放资源
        self.ib.cancelMktData(contract)

        return quote

    def get_historical_data(self, symbol: str,
                            duration: str = '1 D',
                            bar_size: str = '1 min',
                            what_to_show: str = 'TRADES',
                            exchange: str = 'SMART') -> pd.DataFrame:
        """
        获取历史数据

        Args:
            symbol: 股票代码
            duration: 数据时长（如 '1 D', '1 W', '1 M', '1 Y'）
            bar_size: K线周期（如 '1 min', '5 mins', '1 hour', '1 day'）
            what_to_show: 数据类型（'TRADES', 'MIDPOINT', 'BID', 'ASK'）

        Returns:
            DataFrame 格式的历史数据
        """
        contract = Stock(symbol, exchange, 'USD')
        self.ib.qualifyContracts(contract)

        bars = self.ib.reqHistoricalData(
            contract,
            endDateTime='',
            durationStr=duration,
            barSizeSetting=bar_size,
            whatToShow=what_to_show,
            useRTH=True,    # 仅使用常规交易时段
            formatDate=1
        )

        df = util.df(bars)
        return df

    def get_market_depth(self, symbol: str,
                          exchange: str = 'SMART',
                          num_rows: int = 5) -> dict:
        """
        获取市场深度（Level 2 数据）

        Args:
            symbol: 股票代码
            num_rows: 显示的深度行数

        Returns:
            买卖盘深度数据
        """
        contract = Stock(symbol, exchange, 'USD')
        self.ib.qualifyContracts(contract)

        # 请求市场深度
        self.ib.reqMktDepth(contract, numRows=num_rows)
        self.ib.sleep(2)

        # 读取深度数据
        ticker = self.ib.ticker(contract)

        depth = {
            'bids': [(entry.price, entry.size) for entry in ticker.domBids],
            'asks': [(entry.price, entry.size) for entry in ticker.domAsks]
        }

        self.ib.cancelMktDepth(contract)
        return depth


# 使用示例
def demo_market_data():
    with IBKRConnection(port=7496, client_id=10) as conn:
        ib = conn.ib
        mdp = MarketDataProvider(ib)

        # 获取 AAPL 实时报价
        quote = mdp.get_stock_quote('AAPL')
        print(f"AAPL 实时报价:")
        print(f"  最新价: ${quote['last']}")
        print(f"  买一: ${quote['bid']} x {quote['bid_size']}")
        print(f"  卖一: ${quote['ask']} x {quote['ask_size']}")
        print(f"  成交量: {quote['volume']}")

        # 获取历史数据
        hist = mdp.get_historical_data('AAPL', duration='5 D', bar_size='1 hour')
        print(f"\n历史数据（最近5天，1小时K线）:")
        print(hist.tail())
```

### 4.2 期权实时报价

```python
class OptionDataProvider:
    """期权数据提供器"""

    def __init__(self, ib: IB):
        self.ib = ib

    def get_option_quote(self, symbol: str, expiry: str,
                          strike: float, right: str,
                          exchange: str = 'SMART') -> dict:
        """
        获取单个期权合约的实时报价

        Args:
            symbol: 标的代码
            expiry: 到期日（格式：'20240621'）
            strike: 执行价格
            right: 'C'（看涨）或 'P'（看跌）
            exchange: 交易所
        """
        contract = Option(symbol, expiry, strike, right, exchange)
        self.ib.qualifyContracts(contract)

        ticker = self.ib.reqMktData(contract, '', False, False)
        self.ib.sleep(2)

        # 计算隐含波动率和 Greeks（如果 API 未提供）
        from scipy.stats import norm
        import numpy as np

        underlying_price = ticker.underlying.last if ticker.underlying else None

        quote = {
            'symbol': symbol,
            'expiry': expiry,
            'strike': strike,
            'right': right,
            'bid': ticker.bid,
            'ask': ticker.ask,
            'last': ticker.last,
            'volume': ticker.volume,
            'open_interest': ticker.openInterest,
            'implied_vol': ticker.impliedVol if hasattr(ticker, 'impliedVol') else None,
            'delta': ticker.delta if hasattr(ticker, 'delta') else None,
            'gamma': ticker.gamma if hasattr(ticker, 'gamma') else None,
            'theta': ticker.theta if hasattr(ticker, 'theta') else None,
            'vega': ticker.vega if hasattr(ticker, 'vega') else None,
            'underlying_price': underlying_price
        }

        # 如果 API 没有提供 Greeks，自行计算
        if quote['delta'] is None and underlying_price:
            quote.update(self._calculate_greeks(
                underlying_price, strike, expiry, right,
                (quote['bid'] + quote['ask']) / 2 if quote['bid'] and quote['ask'] else quote['last']
            ))

        self.ib.cancelMktData(contract)
        return quote

    def _calculate_greeks(self, S, K, expiry_str, right, option_price):
        """计算 Greeks"""
        from scipy.stats import norm
        from datetime import datetime
        import numpy as np

        # 计算到期天数
        exp_date = datetime.strptime(expiry_str, '%Y%m%d')
        T = max((exp_date - datetime.now()).days / 365, 1/365)
        r = 0.05  # 无风险利率

        # 使用市场价反推隐含波动率
        from scipy.optimize import brentq

        def bs_diff(sigma):
            d1 = (np.log(S/K) + (r + 0.5*sigma**2)*T) / (sigma*np.sqrt(T))
            d2 = d1 - sigma*np.sqrt(T)
            if right == 'C':
                price = S*norm.cdf(d1) - K*np.exp(-r*T)*norm.cdf(d2)
            else:
                price = K*np.exp(-r*T)*norm.cdf(-d2) - S*norm.cdf(-d1)
            return price - option_price

        try:
            iv = brentq(bs_diff, 0.01, 5.0, xtol=1e-8)
        except:
            return {'implied_vol': None, 'delta': None,
                    'gamma': None, 'theta': None, 'vega': None}

        d1 = (np.log(S/K) + (r + 0.5*iv**2)*T) / (iv*np.sqrt(T))
        d2 = d1 - iv*np.sqrt(T)
        nd1 = norm.pdf(d1)

        if right == 'C':
            delta = norm.cdf(d1)
            theta = (-(S*nd1*iv)/(2*np.sqrt(T)) - r*K*np.exp(-r*T)*norm.cdf(d2))/365
        else:
            delta = norm.cdf(d1) - 1
            theta = (-(S*nd1*iv)/(2*np.sqrt(T)) + r*K*np.exp(-r*T)*norm.cdf(-d2))/365

        gamma = nd1 / (S*iv*np.sqrt(T))
        vega = S*nd1*np.sqrt(T) / 100

        return {
            'implied_vol': iv,
            'delta': delta,
            'gamma': gamma,
            'theta': theta,
            'vega': vega
        }
```

---

## 五、获取期权链

### 5.1 完整的期权链数据获取

```python
class OptionChainManager:
    """
    期权链管理器

    获取完整的期权链数据，包括所有到期日和执行价格
    """

    def __init__(self, ib: IB):
        self.ib = ib
        self.chain_cache = {}

    def get_expirations(self, symbol: str,
                         exchange: str = 'SMART') -> list:
        """
        获取所有可用的到期日

        Args:
            symbol: 标的代码

        Returns:
            到期日列表（按日期排序）
        """
        contract = Stock(symbol, exchange, 'USD')
        self.ib.qualifyContracts(contract)

        chains = self.ib.reqSecDefOptParams(
            underlyingSymbol=symbol,
            futFopExchange='',
            underlyingSecType='STK',
            underlyingConId=contract.conId
        )

        expirations = set()
        for chain in chains:
            expirations.update(chain.expirations)

        return sorted(list(expirations))

    def get_strikes(self, symbol: str, expiry: str,
                     exchange: str = 'SMART') -> list:
        """获取指定到期日的所有执行价格"""
        contract = Stock(symbol, exchange, 'USD')
        self.ib.qualifyContracts(contract)

        chains = self.ib.reqSecDefOptParams(
            underlyingSymbol=symbol,
            futFopExchange='',
            underlyingSecType='STK',
            underlyingConId=contract.conId
        )

        strikes = set()
        for chain in chains:
            if expiry in chain.expirations:
                strikes.update(chain.strikes)

        return sorted(list(strikes))

    def get_full_chain(self, symbol: str, expiry: str,
                        exchange: str = 'SMART',
                        include_greeks: bool = True) -> pd.DataFrame:
        """
        获取完整的期权链数据

        Args:
            symbol: 标的代码
            expiry: 到期日
            exchange: 交易所
            include_greeks: 是否计算 Greeks

        Returns:
            包含所有期权合约信息的 DataFrame
        """
        strikes = self.get_strikes(symbol, expiry, exchange)

        if not strikes:
            return pd.DataFrame()

        # 获取标的价格
        stock = Stock(symbol, exchange, 'USD')
        self.ib.qualifyContracts(stock)
        stock_ticker = self.ib.reqMktData(stock, '', False, False)
        self.ib.sleep(2)
        spot_price = stock_ticker.last
        self.ib.cancelMktData(stock)

        # 创建期权合约列表
        contracts = []
        for strike in strikes:
            call = Option(symbol, expiry, strike, 'C', exchange)
            put = Option(symbol, expiry, strike, 'P', exchange)
            contracts.extend([call, put])

        # 批量验证合约
        qualified = self.ib.qualifyContracts(*contracts)

        # 批量请求行情
        tickers = []
        for contract in qualified:
            ticker = self.ib.reqMktData(contract, '', False, False)
            tickers.append((contract, ticker))

        # 等待数据到达
        self.ib.sleep(3)

        # 构建 DataFrame
        records = []
        for contract, ticker in tickers:
            mid = (ticker.bid + ticker.ask) / 2 if ticker.bid and ticker.ask else None

            record = {
                'symbol': symbol,
                'expiry': expiry,
                'strike': contract.strike,
                'right': contract.right,
                'bid': ticker.bid,
                'ask': ticker.ask,
                'mid': mid,
                'last': ticker.last,
                'volume': ticker.volume,
                'open_interest': ticker.openInterest,
                'underlying': spot_price,
                'itm': (spot_price > contract.strike) if contract.right == 'C'
                       else (spot_price < contract.strike),
                'moneyness': contract.strike / spot_price
            }

            records.append(record)

        # 取消所有订阅
        for contract, _ in tickers:
            self.ib.cancelMktData(contract)

        df = pd.DataFrame(records)
        df = df.sort_values(['right', 'strike']).reset_index(drop=True)

        # 计算 Greeks
        if include_greeks and len(df) > 0:
            df = self._add_greeks(df, spot_price, expiry)

        return df

    def _add_greeks(self, df: pd.DataFrame, spot: float,
                     expiry: str) -> pd.DataFrame:
        """为期权链添加 Greeks"""
        from scipy.stats import norm
        from datetime import datetime
        import numpy as np

        exp_date = datetime.strptime(expiry, '%Y%m%d')
        T = max((exp_date - datetime.now()).days / 365, 1/365)
        r = 0.05

        deltas, gammas, thetas, vegas, ivs = [], [], [], [], []

        for _, row in df.iterrows():
            mid = row['mid']
            K = row['strike']
            right = row['right']

            if mid and mid > 0:
                # 反推隐含波动率
                from scipy.optimize import brentq

                def obj(sigma):
                    d1 = (np.log(spot/K) + (r + 0.5*sigma**2)*T) / (sigma*np.sqrt(T))
                    d2 = d1 - sigma*np.sqrt(T)
                    if right == 'C':
                        p = spot*norm.cdf(d1) - K*np.exp(-r*T)*norm.cdf(d2)
                    else:
                        p = K*np.exp(-r*T)*norm.cdf(-d2) - spot*norm.cdf(-d1)
                    return p - mid

                try:
                    iv = brentq(obj, 0.01, 5.0, xtol=1e-8)
                except:
                    iv = np.nan

                if not np.isnan(iv):
                    d1 = (np.log(spot/K) + (r + 0.5*iv**2)*T) / (iv*np.sqrt(T))
                    d2 = d1 - iv*np.sqrt(T)
                    nd1 = norm.pdf(d1)

                    if right == 'C':
                        delta = norm.cdf(d1)
                        theta = (-(spot*nd1*iv)/(2*np.sqrt(T)) -
                                 r*K*np.exp(-r*T)*norm.cdf(d2)) / 365
                    else:
                        delta = norm.cdf(d1) - 1
                        theta = (-(spot*nd1*iv)/(2*np.sqrt(T)) +
                                 r*K*np.exp(-r*T)*norm.cdf(-d2)) / 365

                    gamma = nd1 / (spot*iv*np.sqrt(T))
                    vega = spot*nd1*np.sqrt(T) / 100
                else:
                    delta = gamma = theta = vega = np.nan
            else:
                iv = delta = gamma = theta = vega = np.nan

            ivs.append(iv)
            deltas.append(delta)
            gammas.append(gamma)
            thetas.append(theta)
            vegas.append(vega)

        df['implied_vol'] = ivs
        df['delta'] = deltas
        df['gamma'] = gammas
        df['theta'] = thetas
        df['vega'] = vegas

        return df


# 使用示例
def demo_option_chain():
    with IBKRConnection(port=7496, client_id=11) as conn:
        ib = conn.ib
        ocm = OptionChainManager(ib)

        # 获取到期日
        expirations = ocm.get_expirations('AAPL')
        print(f"可用到期日: {expirations[:5]}...")

        # 获取最近到期日的完整期权链
        if expirations:
            expiry = expirations[2]  # 选择第3个到期日
            chain = ocm.get_full_chain('AAPL', expiry, include_greeks=True)

            print(f"\n{expiry} 期权链:")
            print(f"总计 {len(chain)} 个合约")

            # 显示看涨期权
            calls = chain[chain['right'] == 'C']
            print(f"\n看涨期权:")
            print(calls[['strike', 'bid', 'ask', 'volume',
                         'implied_vol', 'delta']].to_string(index=False))
```

---

## 六、自动下单

### 6.1 订单类型

```python
class OrderManager:
    """
    订单管理器

    支持各种订单类型的创建和提交
    """

    def __init__(self, ib: IB):
        self.ib = ib
        self.active_orders = {}

    def place_market_order(self, contract: Contract,
                            action: str, quantity: int,
                            order_type: str = 'MKT') -> Trade:
        """
        市价单（Market Order）

        优点：确保成交
        缺点：价格不确定，滑点风险大

        Args:
            contract: 合约对象
            action: 'BUY' 或 'SELL'
            quantity: 数量（期权合约数）
        """
        order = MarketOrder(action, quantity)
        order.tif = 'DAY'  # 当日有效

        trade = self.ib.placeOrder(contract, order)
        self.active_orders[trade.order.orderId] = trade

        print(f"市价单已提交: {action} {quantity} {contract.symbol} "
              f"{contract.strike}{contract.right}")
        return trade

    def place_limit_order(self, contract: Contract,
                           action: str, quantity: int,
                           limit_price: float) -> Trade:
        """
        限价单（Limit Order）

        优点：价格确定
        缺点：可能无法成交

        Args:
            limit_price: 限价价格
        """
        order = LimitOrder(action, quantity, limit_price)
        order.tif = 'DAY'

        trade = self.ib.placeOrder(contract, order)
        self.active_orders[trade.order.orderId] = trade

        print(f"限价单已提交: {action} {quantity} {contract.symbol} "
              f"{contract.strike}{contract.right} @ ${limit_price}")
        return trade

    def place_stop_order(self, contract: Contract,
                          action: str, quantity: int,
                          stop_price: float) -> Trade:
        """
        止损单（Stop Order）

        当价格触及止损价时，以市价成交
        """
        order = Order()
        order.action = action
        order.totalQuantity = quantity
        order.orderType = 'STP'
        order.auxPrice = stop_price
        order.tif = 'DAY'

        trade = self.ib.placeOrder(contract, order)
        self.active_orders[trade.order.orderId] = trade
        return trade

    def place_combo_order(self, contracts: list,
                           actions: list, quantities: list,
                           limit_price: float = None,
                           order_type: str = 'LMT') -> Trade:
        """
        组合订单（Combo/Spread Order）

        将多个期权合约作为一个订单提交，
        确保所有腿同时成交。

        Args:
            contracts: 合约列表（每条腿一个合约）
            actions: 每条腿的买卖方向
            quantities: 每条腿的数量
            limit_price: 组合限价（净信用/净支出）
            order_type: 'LMT'（限价）或 'MKT'（市价）
        """
        # 创建组合合约
        combo = ComboOrder()
        combo.action = actions[0]  # 主方向
        combo.totalQuantity = quantities[0]
        combo.orderType = order_type

        if limit_price is not None:
            combo.lmtPrice = limit_price

        combo.tif = 'DAY'

        # 构建组合合约
        bag = Bag(
            symbol=contracts[0].symbol,
            exchange=contracts[0].exchange,
            currency='USD'
        )

        # 组合的组成部分
        combo_parts = []
        for contract, action, qty in zip(contracts, actions, quantities):
            part = ComboLeg(
                conId=contract.conId,
                ratio=qty,
                action=action,
                exchange=contract.exchange
            )
            combo_parts.append(part)

        bag.comboLegs = combo_parts

        trade = self.ib.placeOrder(bag, combo)
        self.active_orders[trade.order.orderId] = trade

        print(f"组合订单已提交: {len(contracts)} 条腿, "
              f"限价=${limit_price}")
        return trade

    def place_iron_condor(self, symbol: str, expiry: str,
                           short_put_strike: float,
                           long_put_strike: float,
                           short_call_strike: float,
                           long_call_strike: float,
                           quantity: int,
                           net_credit: float,
                           exchange: str = 'SMART') -> Trade:
        """
        Iron Condor 组合订单

        将 Iron Condor 的四条腿作为一个订单提交
        """
        # 创建四条腿的合约
        short_put = Option(symbol, expiry, short_put_strike, 'P', exchange)
        long_put = Option(symbol, expiry, long_put_strike, 'P', exchange)
        short_call = Option(symbol, expiry, short_call_strike, 'C', exchange)
        long_call = Option(symbol, expiry, long_call_strike, 'C', exchange)

        # 验证合约
        contracts = [short_put, long_put, short_call, long_call]
        qualified = self.ib.qualifyContracts(*contracts)

        # 检查所有合约是否验证成功
        if len(qualified) != 4:
            raise ValueError("部分合约验证失败")

        # Iron Condor 的买卖方向
        # 卖出 put + 买入 put + 卖出 call + 买入 call
        actions = ['SELL', 'BUY', 'SELL', 'BUY']
        quantities = [quantity] * 4

        return self.place_combo_order(
            qualified, actions, quantities,
            limit_price=net_credit,
            order_type='LMT'
        )

    def modify_order(self, trade: Trade, new_price: float = None,
                      new_quantity: int = None):
        """修改订单"""
        order = trade.order

        if new_price is not None:
            if hasattr(order, 'lmtPrice'):
                order.lmtPrice = new_price
            elif hasattr(order, 'auxPrice'):
                order.auxPrice = new_price

        if new_quantity is not None:
            order.totalQuantity = new_quantity

        self.ib.placeOrder(trade.contract, order)
        print(f"订单已修改: orderId={order.orderId}")

    def cancel_order(self, trade: Trade):
        """取消订单"""
        self.ib.cancelOrder(trade.order)
        if trade.order.orderId in self.active_orders:
            del self.active_orders[trade.order.orderId]
        print(f"订单已取消: orderId={trade.order.orderId}")

    def cancel_all_orders(self):
        """取消所有活跃订单"""
        self.ib.reqGlobalCancel()
        self.active_orders.clear()
        print("所有订单已取消")

    def get_order_status(self, trade: Trade) -> dict:
        """获取订单状态"""
        return {
            'order_id': trade.order.orderId,
            'status': trade.orderStatus.status,
            'filled': trade.orderStatus.filled,
            'remaining': trade.orderStatus.remaining,
            'avg_fill_price': trade.orderStatus.avgFillPrice,
            'last_fill_price': trade.orderStatus.lastFillPrice,
            'perm_id': trade.order.permId
        }
```

### 6.2 高级订单功能

```python
class AdvancedOrderManager(OrderManager):
    """高级订单管理器"""

    def place_bracket_order(self, contract: Contract,
                             action: str, quantity: int,
                             entry_price: float,
                             take_profit_price: float,
                             stop_loss_price: float) -> list:
        """
        括号订单（Bracket Order）

        一个入场单 + 一个止盈单 + 一个止损单
        当入场单成交后，止盈和止损单自动激活。
        其中一个触发后，另一个自动取消。

        Args:
            entry_price: 入场限价
            take_profit_price: 止盈限价
            stop_loss_price: 止损触发价
        """
        # 入场订单
        entry = LimitOrder(action, quantity, entry_price)
        entry.orderId = self.ib.client.getReqId()
        entry.transmit = False  # 不立即发送

        # 止盈订单
        tp_action = 'SELL' if action == 'BUY' else 'BUY'
        take_profit = LimitOrder(tp_action, quantity, take_profit_price)
        take_profit.orderId = self.ib.client.getReqId()
        take_profit.parentId = entry.orderId
        take_profit.transmit = False

        # 止损订单
        stop_loss = Order()
        stop_loss.action = tp_action
        stop_loss.totalQuantity = quantity
        stop_loss.orderType = 'STP'
        stop_loss.auxPrice = stop_loss_price
        stop_loss.orderId = self.ib.client.getReqId()
        stop_loss.parentId = entry.orderId
        stop_loss.transmit = True  # 最后一个订单触发发送

        # 提交所有订单
        trades = []
        for order in [entry, take_profit, stop_loss]:
            trade = self.ib.placeOrder(contract, order)
            trades.append(trade)

        print(f"括号订单已提交:")
        print(f"  入场: ${entry_price}")
        print(f"  止盈: ${take_profit_price}")
        print(f"  止损: ${stop_loss_price}")

        return trades

    def place_trailing_stop(self, contract: Contract,
                             action: str, quantity: int,
                             trail_stop_price: float,
                             trailing_amount: float) -> Trade:
        """
        移动止损单（Trailing Stop Order）

        止损价随价格上涨而上移，但不会随价格下跌而下移
        """
        order = Order()
        order.action = action
        order.totalQuantity = quantity
        order.orderType = 'TRAIL'
        order.trailStopPrice = trail_stop_price
        order.auxPrice = trailing_amount
        order.tif = 'DAY'

        trade = self.ib.placeOrder(contract, order)
        return trade

    def place_oca_group(self, orders_data: list,
                         oca_group_name: str) -> list:
        """
        OCA（One Cancels All）订单组

        当其中一个订单成交后，自动取消组中其他所有订单

        Args:
            orders_data: [{'contract': ..., 'action': ..., 'quantity': ..., 'price': ...}, ...]
            oca_group_name: OCA 组名称
        """
        trades = []

        for data in orders_data:
            order = LimitOrder(
                data['action'],
                data['quantity'],
                data['price']
            )
            order.ocaGroup = oca_group_name
            order.ocaType = 1  # 1 = 取消剩余订单

            trade = self.ib.placeOrder(data['contract'], order)
            trades.append(trade)

        print(f"OCA 订单组 '{oca_group_name}' 已提交, "
              f"包含 {len(trades)} 个订单")
        return trades
```

---

## 七、持仓与账户管理

### 7.1 实时监控

```python
class AccountManager:
    """账户管理器"""

    def __init__(self, ib: IB):
        self.ib = ib

    def get_account_summary(self) -> dict:
        """获取账户摘要"""
        account = self.ib.managedAccounts()[0]
        summary = self.ib.accountSummary(account)

        result = {}
        for item in summary:
            if item.tag in [
                'TotalCashValue', 'NetLiquidation', 'UnrealizedPPL',
                'RealizedPPL', 'AvailableFunds', 'MaintMarginReq',
                'InitMarginReq', 'GrossPositionValue'
            ]:
                result[item.tag] = float(item.value)

        return result

    def get_positions(self) -> pd.DataFrame:
        """获取所有持仓"""
        positions = self.ib.positions()

        records = []
        for pos in positions:
            records.append({
                'account': pos.account,
                'symbol': pos.contract.symbol,
                'sec_type': pos.contract.secType,
                'strike': getattr(pos.contract, 'strike', None),
                'right': getattr(pos.contract, 'right', None),
                'expiry': getattr(pos.contract, 'lastTradeDateOrContractMonth', None),
                'position': pos.position,
                'avg_cost': pos.avgCost,
                'market_value': pos.position * pos.avgCost
            })

        return pd.DataFrame(records)

    def get_open_orders(self) -> list:
        """获取所有未成交订单"""
        trades = self.ib.openOrders()

        orders = []
        for trade in trades:
            orders.append({
                'order_id': trade.order.orderId,
                'symbol': trade.contract.symbol,
                'action': trade.order.action,
                'quantity': trade.order.totalQuantity,
                'order_type': trade.order.orderType,
                'limit_price': getattr(trade.order, 'lmtPrice', None),
                'status': trade.orderStatus.status,
                'filled': trade.orderStatus.filled
            })

        return orders

    def get_portfolio(self) -> pd.DataFrame:
        """获取投资组合详情"""
        account = self.ib.managedAccounts()[0]
        portfolio = self.ib.portfolio(account)

        records = []
        for item in portfolio:
            records.append({
                'symbol': item.contract.symbol,
                'sec_type': item.contract.secType,
                'strike': getattr(item.contract, 'strike', None),
                'right': getattr(item.contract, 'right', None),
                'position': item.position,
                'market_price': item.marketPrice,
                'market_value': item.marketValue,
                'avg_cost': item.averageCost,
                'unrealized_pnl': item.unrealizedPNL,
                'realized_pnl': item.realizedPNL
            })

        return pd.DataFrame(records)

    def get_pnl(self) -> dict:
        """获取盈亏数据"""
        account = self.ib.managedAccounts()[0]

        # 订阅盈亏数据
        pnl = self.ib.reqPnL(account)
        self.ib.sleep(1)

        result = {
            'daily_pnl': pnl.dailyPnL,
            'unrealized_pnl': pnl.unrealizedPnL,
            'realized_pnl': pnl.realizedPnL
        }

        self.ib.cancelPnL(pnl.reqId)
        return result

    def monitor_account(self, interval: int = 60):
        """
        持续监控账户状态

        Args:
            interval: 更新间隔（秒）
        """
        import time

        print("开始账户监控...")
        print(f"{'时间':>10} {'净值':>12} {'日盈亏':>10} {'未实现':>10} {'持仓数':>6}")
        print("-" * 55)

        while True:
            try:
                summary = self.get_account_summary()
                positions = self.get_positions()
                pnl = self.get_pnl()

                timestamp = pd.Timestamp.now().strftime('%H:%M:%S')
                print(f"{timestamp:>10} "
                      f"${summary.get('NetLiquidation', 0):>11,.2f} "
                      f"${pnl['daily_pnl']:>9,.2f} "
                      f"${pnl['unrealized_pnl']:>9,.2f} "
                      f"{len(positions):>6}")

                time.sleep(interval)

            except KeyboardInterrupt:
                print("\n监控已停止")
                break
            except Exception as e:
                print(f"监控错误: {e}")
                time.sleep(5)
```

---

## 八、错误处理与日志

### 8.1 稳健的错误处理

```python
import logging
from datetime import datetime
from functools import wraps
import time

def setup_logging(log_file: str = 'ibkr_trading.log'):
    """配置交易日志"""
    logging.basicConfig(
        level=logging.INFO,
        format='%(asctime)s [%(levelname)s] %(name)s: %(message)s',
        handlers=[
            logging.FileHandler(log_file, encoding='utf-8'),
            logging.StreamHandler()
        ]
    )
    return logging.getLogger('TradingBot')

def retry_on_error(max_retries: int = 3, delay: float = 5.0):
    """
    重试装饰器

    在网络错误或其他临时错误时自动重试
    """
    def decorator(func):
        @wraps(func)
        def wrapper(*args, **kwargs):
            for attempt in range(max_retries):
                try:
                    return func(*args, **kwargs)
                except ConnectionError as e:
                    logging.warning(f"连接错误 (尝试 {attempt+1}/{max_retries}): {e}")
                    if attempt < max_retries - 1:
                        time.sleep(delay)
                except Exception as e:
                    logging.error(f"执行错误: {e}")
                    raise
            raise ConnectionError(f"重试 {max_retries} 次后仍然失败")
        return wrapper
    return decorator


class TradingErrorHandler:
    """交易错误处理器"""

    # IBKR 错误代码分类
    NETWORK_ERRORS = {502, 503, 504}  # 连接相关错误
    DATA_ERRORS = {162, 200, 354}      # 数据相关错误
    ORDER_ERRORS = {103, 104, 105, 106, 107, 109, 110, 114, 115, 116, 117, 118, 119}
    INFORMATIONAL = {2100, 2103, 2104, 2105, 2106, 2107, 2108, 2158}

    @staticmethod
    def handle_error(error_code: int, error_msg: str) -> dict:
        """处理 IBKR API 错误"""
        result = {
            'code': error_code,
            'message': error_msg,
            'severity': 'unknown',
            'action': 'none'
        }

        if error_code in TradingErrorHandler.INFORMATIONAL:
            result['severity'] = 'info'
            result['action'] = 'log_only'

        elif error_code in TradingErrorHandler.NETWORK_ERRORS:
            result['severity'] = 'critical'
            result['action'] = 'reconnect'

        elif error_code in TradingErrorHandler.ORDER_ERRORS:
            result['severity'] = 'error'
            result['action'] = 'cancel_and_retry'

        elif error_code in TradingErrorHandler.DATA_ERRORS:
            result['severity'] = 'warning'
            result['action'] = 'retry_data_request'

        else:
            result['severity'] = 'warning'
            result['action'] = 'investigate'

        return result
```

---

## 九、完整的 Iron Condor 自动交易机器人

```python
class IronCondorBot:
    """
    Iron Condor 自动交易机器人

    功能：
    1. 监控波动率环境
    2. 自动选择合约
    3. 提交 Iron Condor 订单
    4. 监控持仓并自动平仓
    5. 风险管理和止损
    """

    def __init__(self, config: dict):
        self.config = config
        self.ib = None
        self.logger = setup_logging()
        self.is_running = False

        # 策略参数
        self.symbol = config.get('symbol', 'SPY')
        self.iv_rank_entry = config.get('iv_rank_entry', 70)
        self.iv_rank_exit = config.get('iv_rank_exit', 40)
        self.delta_target = config.get('delta_target', 0.16)
        self.wing_width = config.get('wing_width', 5)
        self.profit_target = config.get('profit_target', 0.50)
        self.stop_loss = config.get('stop_loss', 2.0)
        self.min_dte = config.get('min_dte', 30)
        self.max_dte = config.get('max_dte', 45)
        self.max_positions = config.get('max_positions', 2)

        # 状态
        self.positions = []
        self.iv_history = []

    def start(self):
        """启动机器人"""
        self.logger.info("=" * 60)
        self.logger.info("Iron Condor 交易机器人启动")
        self.logger.info(f"标的: {self.symbol}")
        self.logger.info(f"IV Rank 入场: {self.iv_rank_entry}")
        self.logger.info(f"IV Rank 出场: {self.iv_rank_exit}")
        self.logger.info("=" * 60)

        # 连接 IBKR
        conn = IBKRConnection(
            port=self.config.get('port', 7496),
            client_id=self.config.get('client_id', 5)
        )

        if not conn.connect():
            self.logger.error("无法连接到 TWS，机器人退出")
            return

        self.ib = conn.ib
        self.is_running = True

        try:
            self._main_loop()
        except KeyboardInterrupt:
            self.logger.info("收到中断信号，正在停止...")
        except Exception as e:
            self.logger.error(f"机器人异常: {e}", exc_info=True)
        finally:
            self.stop()

    def stop(self):
        """停止机器人"""
        self.is_running = False
        if self.ib and self.ib.isConnected():
            self.ib.disconnect()
        self.logger.info("机器人已停止")

    def _main_loop(self):
        """主循环"""
        check_interval = 300  # 每5分钟检查一次

        while self.is_running:
            try:
                # 1. 更新市场数据
                current_iv = self._get_current_iv()
                iv_rank = self._calculate_iv_rank(current_iv)
                self.iv_history.append({
                    'time': pd.Timestamp.now(),
                    'iv': current_iv,
                    'iv_rank': iv_rank
                })

                # 2. 检查现有持仓
                self._check_positions()

                # 3. 检查入场信号
                if len(self.positions) < self.max_positions:
                    if iv_rank > self.iv_rank_entry:
                        self._open_iron_condor(current_iv, iv_rank)

                # 4. 状态日志
                self.logger.info(
                    f"IV={current_iv:.2%}, IV Rank={iv_rank:.0f}, "
                    f"持仓={len(self.positions)}"
                )

                # 等待下一次检查
                self.ib.sleep(check_interval)

            except Exception as e:
                self.logger.error(f"主循环错误: {e}", exc_info=True)
                self.ib.sleep(30)

    def _get_current_iv(self) -> float:
        """获取当前隐含波动率（使用 VIX 或近月 ATM IV）"""
        # 使用 VIX 作为 IV 的代理
        vix = Index('VIX', 'CBOE')
        try:
            self.ib.qualifyContracts(vix)
            ticker = self.ib.reqMktData(vix, '', False, False)
            self.ib.sleep(2)
            iv = ticker.last / 100 if ticker.last else 0.20
            self.ib.cancelMktData(vix)
            return iv
        except:
            return 0.20  # 默认值

    def _calculate_iv_rank(self, current_iv: float) -> float:
        """计算 IV Rank"""
        if len(self.iv_history) < 30:
            return 50  # 数据不足时返回中位

        ivs = [h['iv'] for h in self.iv_history[-252:]]
        min_iv = min(ivs)
        max_iv = max(ivs)

        if max_iv == min_iv:
            return 50

        return (current_iv - min_iv) / (max_iv - min_iv) * 100

    def _open_iron_condor(self, current_iv: float, iv_rank: float):
        """开仓 Iron Condor"""
        self.logger.info(f"尝试开仓 Iron Condor, IV Rank={iv_rank:.0f}")

        try:
            ocm = OptionChainManager(self.ib)

            # 获取合适的到期日
            expirations = ocm.get_expirations(self.symbol)
            target_expiry = None

            for exp in expirations:
                exp_date = datetime.strptime(exp, '%Y%m%d')
                dte = (exp_date - datetime.now()).days
                if self.min_dte <= dte <= self.max_dte:
                    target_expiry = exp
                    break

            if not target_expiry:
                self.logger.warning("未找到合适的到期日")
                return

            # 获取期权链
            chain = ocm.get_full_chain(self.symbol, target_expiry)
            if len(chain) == 0:
                return

            # 选择合约
            calls = chain[(chain['right'] == 'C') & (chain['mid'].notna())]
            puts = chain[(chain['right'] == 'P') & (chain['mid'].notna())]

            if len(calls) == 0 or len(puts) == 0:
                return

            # 选择 Delta ≈ 0.16 的卖出腿
            calls['delta_diff'] = abs(calls['delta'].abs() - self.delta_target)
            puts['delta_diff'] = abs(puts['delta'].abs() - self.delta_target)

            short_call = calls.loc[calls['delta_diff'].idxmin()]
            short_put = puts.loc[puts['delta_diff'].idxmin()]

            # 选择 wing 保护腿
            long_call_strike = short_call['strike'] + self.wing_width
            long_put_strike = short_put['strike'] - self.wing_width

            long_call = calls[calls['strike'] == long_call_strike]
            long_put = puts[puts['strike'] == long_put_strike]

            if long_call.empty or long_put.empty:
                self.logger.warning("未找到合适的保护腿合约")
                return

            # 计算净信用
            net_credit = (short_call['bid'] + short_put['bid'] -
                         long_call.iloc[0]['ask'] - long_put.iloc[0]['ask'])

            if net_credit < 0.30:
                self.logger.warning(f"净信用过低: ${net_credit:.2f}")
                return

            # 提交订单
            om = OrderManager(self.ib)
            trade = om.place_iron_condor(
                symbol=self.symbol,
                expiry=target_expiry,
                short_put_strike=short_put['strike'],
                long_put_strike=long_put_strike,
                short_call_strike=short_call['strike'],
                long_call_strike=long_call_strike,
                quantity=1,
                net_credit=net_credit
            )

            # 记录持仓
            position = {
                'entry_time': datetime.now(),
                'expiry': target_expiry,
                'short_put': short_put['strike'],
                'long_put': long_put_strike,
                'short_call': short_call['strike'],
                'long_call': long_call_strike,
                'net_credit': net_credit,
                'trade': trade,
                'iv_rank_entry': iv_rank
            }
            self.positions.append(position)

            self.logger.info(
                f"Iron Condor 已开仓: "
                f"SP={short_put['strike']}/{long_put_strike}P "
                f"{short_call['strike']}/{long_call_strike}C, "
                f"净信用=${net_credit:.2f}"
            )

        except Exception as e:
            self.logger.error(f"开仓失败: {e}", exc_info=True)

    def _check_positions(self):
        """检查持仓状态，执行平仓逻辑"""
        positions_to_close = []

        for pos in self.positions:
            expiry = datetime.strptime(pos['expiry'], '%Y%m%d')
            days_to_exp = (expiry - datetime.now()).days

            # 检查出场条件
            should_close = False
            reason = ''

            # 条件1：到期前5天
            if days_to_exp <= 5:
                should_close = True
                reason = f"临近到期 ({days_to_exp} DTE)"

            # 条件2：盈利目标（使用IV Rank作为代理）
            current_iv_rank = self._calculate_iv_rank(self._get_current_iv())
            if current_iv_rank < self.iv_rank_exit:
                should_close = True
                reason = f"IV Rank 回归 ({current_iv_rank:.0f})"

            if should_close:
                self.logger.info(f"平仓信号: {reason}")
                positions_to_close.append(pos)

        # 执行平仓
        for pos in positions_to_close:
            self._close_position(pos)

    def _close_position(self, position: dict):
        """平仓 Iron Condor"""
        try:
            # 提交平仓订单（简化版）
            self.logger.info(f"平仓 Iron Condor: {position['expiry']}")
            self.positions.remove(position)
        except Exception as e:
            self.logger.error(f"平仓失败: {e}", exc_info=True)


# 配置和启动示例
def main():
    config = {
        'symbol': 'SPY',
        'port': 7496,           # 模拟盘端口
        'client_id': 5,
        'iv_rank_entry': 70,
        'iv_rank_exit': 40,
        'delta_target': 0.16,
        'wing_width': 5,
        'profit_target': 0.50,
        'stop_loss': 2.0,
        'min_dte': 30,
        'max_dte': 45,
        'max_positions': 2
    }

    bot = IronCondorBot(config)
    bot.start()


if __name__ == "__main__":
    main()
```

---

## 小结

本章详细介绍了 IBKR API 自动交易的完整流程：

- **连接管理**：TWS API 的架构、端口配置、clientId 管理
- **ib_insync 库**：简化的 API 调用方式，核心数据对象
- **数据获取**：实时报价、历史数据、期权链
- **订单管理**：市价单、限价单、组合订单、括号订单
- **账户监控**：持仓管理、盈亏跟踪
- **错误处理**：日志记录、重试机制、错误分类
- **完整示例**：Iron Condor 自动交易机器人

自动交易将策略从理论变为实践，但也带来了新的挑战：网络稳定性、订单执行速度、风险控制等。建议在模拟盘上充分测试后再投入实盘。

### 思考题

1. 修改 Iron Condor 机器人，添加基于技术指标（如 RSI）的入场过滤器。
2. 实现一个"动态展期"功能：当 Iron Condor 的 Delta 偏移超过阈值时，自动平仓并重新开仓。
3. 为机器人添加邮件或 Telegram 通知功能，当开仓、平仓或触发止损时发送通知。

---

**下一章：** [06-ai-options.md —— AI 驱动的期权交易](section06-ai-options.md)




---
**相关笔记：** [[API与自动化基础]], [[信号生成与策略开发]]
**返回：** [[chapter00-python-ai]] | [[chapter01-主页]]
