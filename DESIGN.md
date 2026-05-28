# Binance Strategy - 币安量化交易系统

> 版本: v1.0
> 日期: 2026-05-28
> 状态: 设计阶段
> 仓库: https://github.com/maoxianfei/strategy

---

## 1. 项目概述

Binance Strategy 是一个面向币安交易所的量化交易系统，支持现货和 U 本位合约的实盘交易、回测验证、市场与系统监控。纯命令行操作，分层解耦架构。

### 1.1 核心目标

- **实盘交易** — 通过 Binance API 自动执行策略信号，支持现货和 U 本位合约
- **环境切换** — 先测试网调试跑通，一键切换国际站实盘
- **回测验证** — 用历史 K 线验证策略效果，输出基础统计指标
- **实时监控** — 市场价格波动预警 + 系统运行状态监控

### 1.2 设计原则

| 原则 | 说明 |
|------|------|
| 分层解耦 | 数据层 / 功能模块 / 核心引擎三层，模块间通过接口调用 |
| 策略可插拔 | 所有策略继承统一基类，新策略零改动接入 |
| 现货合约统一接口 | 交易品种差异封装在数据层和交易层，上层无感知 |
| 安全第一 | 默认测试网，API Key 不硬编码，失败不中断 |
| 轻量够用 | 纯命令行，不引入 Web/数据库等重依赖 |

---

## 2. 系统架构

### 2.1 整体架构图

```
┌───────────────────────────────────────────────────────────┐
│                    CLI 入口 (cli.py)                       │
│         python cli.py run / backtest / monitor            │
└──────────────────────┬────────────────────────────────────┘
                       │
                       ▼
┌───────────────────────────────────────────────────────────┐
│                  核心引擎 (engine.py)                      │
│          编排调度：策略→信号→交易→日志 全流程驱动          │
└───┬──────────┬──────────┬──────────┬──────────┬───────────┘
    │          │          │          │          │
    ▼          ▼          ▼          ▼          ▼
┌────────┐ ┌────────┐ ┌────────┐ ┌────────┐ ┌────────┐
│strategy│ │ trader │ │  log   │ │ monitor│ │backtest│
│  策略  │ │  交易  │ │  日志  │ │  监控  │ │  回测  │
│  模块  │ │  模块  │ │  模块  │ │  模块  │ │  模块  │
└───┬────┘ └───┬────┘ └───┬────┘ └───┬────┘ └───┬────┘
    │          │          │          │          │
    └──────────┴──────────┴──────────┴──────────┘
                       │
                       ▼
┌───────────────────────────────────────────────────────────┐
│                 数据层 (data/)                             │
│                                                          │
│  ┌──────────────┐  ┌──────────────┐                      │
│  │ binance_spot │  │binance_futures│                     │
│  │ 现货 API     │  │ 合约 API      │                      │
│  └──────────────┘  └──────────────┘                      │
└───────────────────────────────────────────────────────────┘
```

### 2.2 数据流 — 实盘

```
CLI "run" 命令
      │
      ▼
StrategyEngine.run()
      │
      ├→ MarketMonitor.watch()      ← 后台线程，定时输出市场行情
      ├→ SystemMonitor.report()     ← 后台线程，定时输出系统状态
      │
      ▼ 主循环
data.get_klines(symbol, interval)   ← 拉最新K线
      │
      ▼
strategy.analyze(klines)            ← 计算指标，返回 Signal 或 None
      │
      ▼ 有信号
trader.buy(signal) / trader.sell(signal)  ← 调用币安API下单
      │
      ▼
logger.info(...)                    ← 记录日志
      │
      ▼
sleep → 下一根K线 → 循环
```

### 2.3 数据流 — 回测

```
CLI "backtest" 命令
      │
      ▼
BacktestEngine.run(symbol, interval, start, end)
      │
      ├→ data.get_klines(symbol, interval, start, end)  ← 拉历史K线
      │
      ▼ 逐根K线
strategy.analyze(klines_so_far)     ← 计算信号
      │
      ▼ 有信号
模拟成交（下一根开盘价 + 手续费）
      │
      ▼
计算统计指标（收益率/回撤/夏普/胜率）
      │
      ▼
BacktestReport.format() → 终端输出报告
```

---

## 3. 目录结构

```
strategy/
├── cli.py                      # CLI 入口（argparse 子命令）
├── engine.py                   # 核心引擎，编排调度
├── config.py                   # 全局配置（API密钥、费率、参数）
├── models.py                   # 统一数据模型
│
├── data/                       # 数据层
│   ├── __init__.py
│   ├── base.py                 # 数据源基类（统一接口）
│   ├── binance_spot.py         # 现货 API 封装
│   ├── binance_futures.py      # U本位合约 API 封装
│   └── cache.py                # K线内存缓存
│
├── strategy/                   # 策略模块
│   ├── __init__.py
│   ├── base.py                 # 策略基类
│   └── rsi.py                  # RSI 超买超卖策略
│
├── trader/                     # 交易模块
│   ├── __init__.py
│   ├── base.py                 # 交易执行器基类
│   ├── spot.py                 # 现货交易执行器
│   └── futures.py              # 合约交易执行器
│
├── logger/                     # 日志模块
│   ├── __init__.py
│   └── setup.py                # 日志配置（文件轮转 + 控制台）
│
├── monitor/                    # 监控模块
│   ├── __init__.py
│   ├── market.py               # 市场监控（价格/波动预警）
│   └── system.py               # 系统监控（API延迟/错误率）
│
├── backtest/                   # 回测模块
│   ├── __init__.py
│   ├── engine.py               # 回测引擎
│   └── report.py               # 回测报告格式化
│
├── config.yaml                 # 配置文件（API Key 等，gitignore）
├── requirements.txt
└── logs/                       # 日志输出目录（gitignore）
```

---

## 4. 核心模块设计

### 4.1 数据模型 (models.py)

```python
from dataclasses import dataclass, field
from datetime import datetime
from enum import Enum
from typing import Optional


class MarketType(Enum):
    SPOT = "spot"
    FUTURES = "futures"


class SignalAction(Enum):
    BUY = "buy"
    SELL = "sell"


class OrderStatus(Enum):
    PENDING = "pending"
    FILLED = "filled"
    PARTIAL = "partial"
    CANCELED = "canceled"
    FAILED = "failed"


@dataclass
class Signal:
    """策略输出的交易信号"""
    symbol: str                          # 交易对，如 "BTCUSDT"
    action: SignalAction                 # 买/卖
    price: float                         # 信号触发价格
    amount: Optional[float] = None       # 建议数量，None 由引擎计算
    market_type: MarketType = MarketType.SPOT
    leverage: int = 1                    # 杠杆倍数（仅合约）
    timestamp: datetime = field(default_factory=datetime.now)


@dataclass
class Order:
    """订单"""
    id: str                              # 币安返回的订单ID
    symbol: str
    side: str                            # BUY / SELL
    price: float
    quantity: float
    status: OrderStatus
    filled_price: float                  # 实际成交均价
    filled_quantity: float
    fee: float                           # 手续费
    fee_asset: str                       # 手续费币种（BNB/USDT）
    timestamp: datetime


@dataclass
class Position:
    """持仓"""
    symbol: str
    market_type: MarketType
    side: str                            # LONG / SHORT / BOTH
    quantity: float
    entry_price: float
    leverage: int
    unrealized_pnl: float
    timestamp: datetime


@dataclass
class BacktestResult:
    """回测结果"""
    total_return: float                  # 总收益率
    max_drawdown: float                  # 最大回撤
    sharpe_ratio: float                  # 夏普比率
    win_rate: float                      # 胜率
    total_trades: int                    # 总交易次数
    profit_trades: int                   # 盈利次数
    loss_trades: int                     # 亏损次数
    signals: list                        # 所有信号
    trades: list                         # 交易明细
```

### 4.2 配置模块 (config.py)

```python
from dataclasses import dataclass


@dataclass
class Config:
    """全局配置"""
    # API
    api_key: str = ""
    api_secret: str = ""
    testnet: bool = True                 # 默认测试网

    # 交易
    market_type: str = "spot"            # spot / futures
    default_leverage: int = 5            # 合约默认杠杆
    fee_rate: float = 0.001              # 手续费率（默认 0.1%）

    # 策略参数
    rsi_period: int = 14
    rsi_overbought: float = 70.0
    rsi_oversold: float = 30.0

    # 监控
    market_interval: int = 30            # 市场监控刷新间隔（秒）
    system_interval: int = 60            # 系统监控刷新间隔（秒）
    alert_threshold: float = 3.0         # 异常波动阈值（%）

    # K线
    kline_interval: str = "1h"           # 默认K线周期
    kline_limit: int = 200               # 每次拉取K线数量
```

配置文件 `config.yaml`（gitignore，不入库）：

```yaml
api:
  key: "your_api_key_here"
  secret: "your_api_secret_here"
  testnet: true

trading:
  market_type: "spot"           # spot / futures
  default_leverage: 5
  fee_rate: 0.001

strategy:
  name: "rsi"
  rsi_period: 14
  rsi_overbought: 70
  rsi_oversold: 30

monitor:
  market_interval: 30
  system_interval: 60
  alert_threshold: 3.0
```

### 4.3 数据层 (data/)

#### 数据源基类 (data/base.py)

```python
from abc import ABC, abstractmethod
import pandas as pd
from models import Order, Position


class BaseDataSource(ABC):
    """数据源基类 — 现货和合约统一接口"""

    @abstractmethod
    def get_klines(self, symbol: str, interval: str, limit: int = 200) -> pd.DataFrame:
        """获取K线数据，返回标准 DataFrame"""

    @abstractmethod
    def get_klines_history(self, symbol: str, interval: str,
                           start: str, end: str) -> pd.DataFrame:
        """获取历史K线（回测用）"""

    @abstractmethod
    def get_ticker(self, symbol: str) -> dict:
        """获取最新行情（价格/24h涨跌/成交量）"""

    @abstractmethod
    def get_price(self, symbol: str) -> float:
        """获取最新价格"""

    @abstractmethod
    def get_balance(self) -> dict:
        """获取账户余额"""

    @abstractmethod
    def get_positions(self) -> list:
        """获取当前持仓"""

    @abstractmethod
    def place_order(self, symbol: str, side: str, order_type: str,
                    quantity: float, price: float = None) -> Order:
        """下单"""

    @abstractmethod
    def cancel_order(self, symbol: str, order_id: str) -> bool:
        """撤单"""

    @abstractmethod
    def get_order_status(self, symbol: str, order_id: str) -> dict:
        """查询订单状态"""

    @abstractmethod
    def set_leverage(self, symbol: str, leverage: int) -> bool:
        """设置杠杆（仅合约）"""
```

#### K线 DataFrame 标准格式

```python
# 所有 K线数据统一为此格式
# columns: [timestamp, open, high, low, close, volume]
```

#### 现货实现 (data/binance_spot.py)

- SDK: `from binance.client import Client`
- Endpoint: `api.binance.com`（实盘）/ `testnet.binance.vision`（测试网）
- 下单: `client.order_market_buy(symbol, quantity)` / `client.order_market_sell(symbol, quantity)`
- K线: `client.get_klines(symbol, interval, limit)`

#### 合约实现 (data/binance_futures.py)

- SDK: `from binance.um_futures import UMFuturesClient`
- Endpoint: `fapi.binance.com`（实盘）/ `testnet.binancefuture.com`（测试网）
- 下单: `client.create_order(symbol, side, type, quantity)`
- 杠杆: `client.change_leverage(symbol, leverage)`
- 仓位模式: 默认单向持仓模式

#### 缓存 (data/cache.py)

```python
class KlineCache:
    """K线内存缓存，避免重复请求触发频率限制"""

    def __init__(self, ttl: int = 60):
        self._cache: dict = {}     # key: (symbol, interval) → (timestamp, data)
        self._ttl = ttl            # 缓存有效期（秒）

    def get(self, symbol: str, interval: str) -> Optional[pd.DataFrame]:
        """命中缓存返回数据，过期返回 None"""

    def set(self, symbol: str, interval: str, data: pd.DataFrame):
        """写入缓存"""
```

### 4.4 策略模块 (strategy/)

#### 策略基类 (strategy/base.py)

```python
from abc import ABC, abstractmethod
from typing import Optional
import pandas as pd
from models import Signal


class BaseStrategy(ABC):
    """策略基类"""

    @abstractmethod
    def name(self) -> str:
        """策略名称"""

    @abstractmethod
    def analyze(self, klines: pd.DataFrame) -> Optional[Signal]:
        """
        分析K线，返回交易信号

        Args:
            klines: 标准K线 DataFrame
                    columns: [timestamp, open, high, low, close, volume]

        Returns:
            Signal 或 None（无信号）
        """
```

#### RSI 策略 (strategy/rsi.py)

```python
class RSIStrategy(BaseStrategy):
    """RSI 超买超卖策略"""

    def __init__(self, period: int = 14, overbought: float = 70.0,
                 oversold: float = 30.0, symbol: str = "BTCUSDT",
                 market_type: MarketType = MarketType.SPOT,
                 leverage: int = 1):
        self.period = period
        self.overbought = overbought
        self.oversold = oversold
        self.symbol = symbol
        self.market_type = market_type
        self.leverage = leverage

    def name(self) -> str:
        return f"RSI({self.period})"

    def analyze(self, klines: pd.DataFrame) -> Optional[Signal]:
        """
        逻辑：
        - RSI 计算使用 talib.RSI 或 pandas-ta
        - RSI 下穿 oversold（30）→ BUY
        - RSI 上穿 overbought（70）→ SELL
        - 其他情况 → None
        """
```

RSI 策略默认参数：

| 参数 | 默认值 | 说明 |
|------|--------|------|
| period | 14 | RSI 计算周期 |
| overbought | 70 | 超买阈值 |
| oversold | 30 | 超卖阈值 |

### 4.5 交易模块 (trader/)

#### 交易执行器基类 (trader/base.py)

```python
from abc import ABC, abstractmethod
from models import Signal, Order, Position


class BaseTrader(ABC):
    """交易执行器基类"""

    @abstractmethod
    def buy(self, signal: Signal) -> Order:
        """执行买入"""

    @abstractmethod
    def sell(self, signal: Signal) -> Order:
        """执行卖出"""

    @abstractmethod
    def get_positions(self) -> list:
        """获取当前持仓"""

    @abstractmethod
    def has_position(self, symbol: str) -> bool:
        """检查是否有持仓"""
```

#### 现货交易执行器 (trader/spot.py)

```python
class SpotTrader(BaseTrader):
    """现货交易执行器"""

    def buy(self, signal: Signal) -> Order:
        """
        1. 查询 USDT 可用余额
        2. 计算买入数量 = 可用金额 / 价格（取币安精度）
        3. data.place_order(symbol, BUY, MARKET, quantity)
        4. 返回 Order
        """

    def sell(self, signal: Signal) -> Order:
        """
        1. 查询币种可用余额
        2. 全部卖出
        3. data.place_order(symbol, SELL, MARKET, quantity)
        4. 返回 Order
        """

    def get_positions(self) -> list:
        """现货持仓 = 非零余额币种列表"""
```

#### 合约交易执行器 (trader/futures.py)

```python
class FuturesTrader(BaseTrader):
    """合约交易执行器"""

    def __init__(self, data_source, default_leverage: int = 5):
        self._data = data_source
        self.default_leverage = default_leverage

    def buy(self, signal: Signal) -> Order:
        """
        1. 设置杠杆 data.set_leverage(symbol, signal.leverage)
        2. 检查是否有空仓 → 有则先平空
        3. data.place_order(symbol, BUY, MARKET, quantity) → 开多
        4. 返回 Order
        """

    def sell(self, signal: Signal) -> Order:
        """
        1. 检查持仓状态
           - 有多仓 → 平多
           - 无持仓 → 开空
        2. data.place_order(...)
        3. 返回 Order
        """

    def get_positions(self) -> list:
        """合约持仓列表"""
```

交易模块安全设计：

| 规则 | 说明 |
|------|------|
| 失败不中断 | 下单失败记日志，主循环继续运行 |
| 精度处理 | 按币安规则取小数位（LOT_SIZE / STEP_SIZE） |
| 测试网默认 | `testnet=True` 为默认值，切实盘需显式指定 |
| 预览日志 | 每次下单前打印 `[预览] BTCUSDT BUY 0.001 @ 68000` |

### 4.6 日志模块 (logger/setup.py)

```python
import logging
from logging.handlers import TimedRotatingFileHandler


def setup_logger(name: str = "strategy", log_dir: str = "logs") -> logging.Logger:
    """
    配置日志

    格式: [2026-05-28 22:35:00] [INFO] [trader.spot] BTCUSDT BUY 0.001 @ 68000.00

    特性:
    - 双输出：文件 + 控制台
    - 按天轮转，保留 30 天
    - 日志文件: logs/strategy_2026-05-28.log
    """
```

日志分级：

| 级别 | 用途 |
|------|------|
| DEBUG | K线数据详情、API 请求参数 |
| INFO | 交易信号、下单结果、正常监控输出 |
| WARNING | API 延迟过高、波动预警 |
| ERROR | 下单失败、API 连接异常 |

### 4.7 监控模块 (monitor/)

#### 市场监控 (monitor/market.py)

```python
class MarketMonitor:
    """市场监控 — 定时输出行情 + 异常预警"""

    def __init__(self, data_source, interval: int = 30,
                 alert_threshold: float = 3.0):
        self._data = data_source
        self.interval = interval
        self.alert_threshold = alert_threshold

    def watch(self, symbols: list[str]):
        """
        后台线程运行，每 interval 秒输出一次：

        ┌─────────────────────────────────────────────┐
        │ [22:35:00] 市场监控                         │
        ├──────────┬────────┬──────────┬─────────────┤
        │ 交易对    │  价格   │  24h涨跌  │  异常标记    │
        ├──────────┼────────┼──────────┼─────────────┤
        │ BTCUSDT  │ 68000  │ +2.3%    │             │
        │ ETHUSDT  │ 3800   │ -5.1%    │ ⚠ 波动异常  │
        └──────────┴────────┴──────────┴─────────────┘
        """
```

预警规则：

| 规则 | 阈值 | 输出 |
|------|------|------|
| 价格急涨 | 1分钟内涨幅 > alert_threshold | `⚠ BTCUSDT 急涨 +4.2%` |
| 价格急跌 | 1分钟内跌幅 > alert_threshold | `⚠ ETHUSDT 急跌 -3.8%` |

#### 系统监控 (monitor/system.py)

```python
class SystemMonitor:
    """系统监控 — API 状态 + 策略运行统计"""

    def __init__(self, interval: int = 60):
        self.interval = interval
        self.start_time = datetime.now()
        self.api_latencies: list = []       # API 延迟记录
        self.api_errors: int = 0            # API 错误计数
        self.signal_count = {"buy": 0, "sell": 0}
        self.trade_count = {"success": 0, "failed": 0}

    def report(self):
        """
        后台线程运行，每 interval 秒输出一次：

        ┌─────────────────────────────────────────────┐
        │ [22:35:00] 系统监控                         │
        ├──────────┬─────────────────────────────────┤
        │ 运行时间  │ 3h 25m                          │
        │ API延迟   │ 120ms (avg) / 350ms (max)      │
        │ API错误   │ 2 次 (最近1小时)                 │
        │ 策略状态  │ RSI(14) 运行中                   │
        │ 信号数    │ 买入 3 / 卖出 2                  │
        │ 交易成功  │ 4 / 失败 1                       │
        └──────────┴─────────────────────────────────┘
        """
```

预警规则：

| 规则 | 阈值 | 输出 |
|------|------|------|
| API 连续失败 | 连续 3 次 | `⚠ API 连接异常，暂停交易 30s` |
| API 延迟过高 | 平均延迟 > 500ms | `⚠ API 延迟过高: 620ms` |

### 4.8 回测模块 (backtest/)

#### 回测引擎 (backtest/engine.py)

```python
class BacktestEngine:
    """回测引擎"""

    def __init__(self, strategy: BaseStrategy, data_source: BaseDataSource):
        self.strategy = strategy
        self.data_source = data_source

    def run(self, symbol: str, interval: str,
            start: str, end: str) -> BacktestResult:
        """
        回测流程：

        1. 拉取历史K线 data_source.get_klines_history(symbol, interval, start, end)
        2. 逐根K线喂给 strategy.analyze(klines_so_far)
        3. 有信号 → 模拟成交（按下一根K线开盘价，扣除手续费）
        4. 维护虚拟持仓（有持仓时不重复买入）
        5. 计算统计指标
        6. 返回 BacktestResult
        """
```

模拟成交规则：

| 项目 | 规则 |
|------|------|
| 成交价格 | 信号触发后下一根 K 线开盘价 |
| 手续费 | 统一按 0.1%（Taker） |
| 持仓管理 | 有持仓时忽略 BUY 信号，无持仓时忽略 SELL 信号 |
| 合约回测 | 支持多空双向，杠杆按配置计算盈亏 |

#### 回测报告 (backtest/report.py)

```python
class BacktestReport:
    """回测报告格式化"""

    def format(self, result: BacktestResult) -> str:
        """
        终端输出：

        ╔══════════════════════════════════════════╗
        ║          回测报告 - RSI(14)              ║
        ║          BTCUSDT · 1h · 2025-01~2025-06 ║
        ╠══════════════════════════════════════════╣
        ║ 总收益率:       +12.35%                  ║
        ║ 最大回撤:       -4.82%                   ║
        ║ 夏普比率:       1.43                     ║
        ║ 总交易:         28 笔                    ║
        ║ 盈利/亏损:      18 / 10                  ║
        ║ 胜率:           64.3%                    ║
        ╠══════════════════════════════════════════╣
        ║ 【最近5笔交易】                           ║
        ║  2025-05-20 BUY  @ 68000  →              ║
        ║  2025-05-22 SELL @ 69500  → +2.2%       ║
        ║  2025-05-25 BUY  @ 67200  →              ║
        ║  2025-05-27 SELL @ 66800  → -0.6%       ║
        ║  2025-05-28 BUY  @ 68500  → (持仓中)     ║
        ╚══════════════════════════════════════════╝
        """
```

统计指标：

| 指标 | 计算方式 |
|------|----------|
| 总收益率 | (期末资金 - 期初资金) / 期初资金 |
| 最大回撤 | 峰谷最大跌幅 |
| 夏普比率 | (日均收益率 - 0) / 日收益率标准差 × sqrt(365) |
| 胜率 | 盈利次数 / 总交易次数 |

### 4.9 核心引擎 (engine.py)

```python
class StrategyEngine:
    """核心引擎 — 编排所有模块"""

    def __init__(self, config: Config):
        self.config = config
        self.data = self._create_data_source()      # 根据 market_type 创建
        self.strategy = self._create_strategy()      # 创建策略实例
        self.trader = self._create_trader()          # 根据 market_type 创建
        self.logger = setup_logger()
        self.market_monitor = MarketMonitor(self.data, config.market_interval)
        self.system_monitor = SystemMonitor(config.system_interval)

    def run(self, symbol: str, interval: str):
        """实盘主循环"""
        # 1. 启动监控线程
        # 2. 主循环：获取K线 → 策略分析 → 下单 → 日志 → sleep
        # 3. Ctrl+C 优雅退出

    def backtest(self, symbol: str, interval: str,
                 start: str, end: str):
        """回测入口"""
        # 复用 BacktestEngine + BacktestReport

    def _create_data_source(self):
        """根据 config.market_type 创建现货或合约数据源"""

    def _create_strategy(self):
        """根据 config 创建策略"""

    def _create_trader(self):
        """根据 config.market_type 创建交易执行器"""
```

### 4.10 CLI 入口 (cli.py)

```python
import argparse


def main():
    parser = argparse.ArgumentParser(description="Binance 量化交易系统")
    subparsers = parser.add_subparsers(dest="command")

    # run — 实盘交易
    run_parser = subparsers.add_parser("run", help="启动实盘交易")
    run_parser.add_argument("--symbol", default="BTCUSDT")
    run_parser.add_argument("--type", choices=["spot", "futures"], default="spot")
    run_parser.add_argument("--interval", default="1h")
    run_parser.add_argument("--leverage", type=int, default=5)
    run_parser.add_argument("--testnet", action="store_true", default=True)

    # backtest — 回测
    bt_parser = subparsers.add_parser("backtest", help="运行回测")
    bt_parser.add_argument("--symbol", default="BTCUSDT")
    bt_parser.add_argument("--interval", default="1h")
    bt_parser.add_argument("--start", required=True)
    bt_parser.add_argument("--end", required=True)

    # monitor — 仅监控
    mon_parser = subparsers.add_parser("monitor", help="仅启动监控")
    mon_parser.add_argument("--symbols", default="BTCUSDT,ETHUSDT")
    mon_parser.add_argument("--interval", type=int, default=30)
```

CLI 用法：

```bash
# 实盘（测试网）
python cli.py run --symbol BTCUSDT --type spot --interval 1h
python cli.py run --symbol ETHUSDT --type futures --leverage 5 --interval 15m

# 切国际站实盘
python cli.py run --symbol BTCUSDT --type spot --testnet false

# 回测
python cli.py backtest --symbol BTCUSDT --interval 1h --start 2025-01-01 --end 2025-06-01

# 仅监控
python cli.py monitor --symbols BTCUSDT,ETHUSDT --interval 30
```

---

## 5. 技术选型

| 项目 | 选择 | 说明 |
|------|------|------|
| 语言 | Python 3.13 | managed runtime |
| 币安 SDK | python-binance | 官方维护，封装 REST + WebSocket |
| 技术指标 | pandas-ta | 纯 Python，无需编译 C 库（ta-lib 需编译） |
| 数据处理 | pandas | DataFrame 统一格式 |
| CLI | argparse | 标准库，轻量 |
| 配置管理 | PyYAML | 读写 config.yaml |
| 日志 | logging + TimedRotatingFileHandler | 标准库 |
| 并发 | threading | 监控用后台线程，主循环在主线程 |

---

## 6. 币安交易规则约束

| 规则 | 说明 | 系统处理 |
|------|------|----------|
| 精度 | 不同币种数量/价格精度不同 | 按交易所规则取小数位 |
| 最小下单量 | 有 LOT_SIZE 限制 | 下单前查询过滤 |
| 频率限制 | 请求权重机制（1200 weight/min） | SDK 内置 + 内存缓存兜底 |
| 手续费 | 现货 0.1%，合约 Maker 0.02%/Taker 0.04% | 回测统一按 0.1% |
| BNB 抵扣 | 用 BNB 付手续费可打折 | 可配置，默认用 USDT |

---

## 7. 错误处理策略

| 场景 | 处理方式 |
|------|----------|
| API 调用失败 | 重试 2 次，间隔 5s，仍失败记日志跳过 |
| API 频率限制触发 | sleep 等待至权重恢复 |
| 余额不足 | 记 WARNING 日志，跳过本次交易 |
| 网络超时 | 设置 10s 超时，超时重试 |
| 配置文件缺失 | 使用默认配置 + 提示用户 |
| Ctrl+C | 优雅退出：关闭监控线程、打印运行统计 |
| 数据格式异常 | 跳过异常K线，记 WARNING |

---

## 8. 实施顺序

| 阶段 | 内容 | 预估文件 |
|------|------|----------|
| P0 | 数据层 + 数据模型 + 配置 + 日志 | models.py, config.py, data/, logger/ |
| P1 | 策略模块 + RSI 策略 | strategy/ |
| P2 | 交易模块（现货 + 合约） | trader/ |
| P3 | 核心引擎 + CLI 入口 | engine.py, cli.py |
| P4 | 监控模块 | monitor/ |
| P5 | 回测模块 | backtest/ |
| P6 | 测试网联调 + 国际站切换 | config.yaml 调整 |

依赖关系：P0 → P1 → P2 → P3（核心链路），P4/P5 可并行。
