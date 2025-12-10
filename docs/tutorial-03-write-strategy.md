# 第 3 课：编写你的第一个策略 - 从想法到代码

## 简介

在前两课中，我们理解了回测的基本概念和核心组件。现在，让我们学习如何将交易想法转化为可执行的策略代码。就像学做菜一样，我们不仅要会看菜谱，还要能自己创新菜品。

**预计学习时间**：40-50 分钟

## 学习目标

完成本课后，你将能够：

- ✅ 理解策略的完整生命周期
- ✅ 编写简单移动平均（SMA）策略
- ✅ 掌握订单类型（市价/限价）的使用
- ✅ 使用回调函数监控交易过程

## 前置知识

- 完成第 2 课
- 理解 Strategy 接口
- Python 基础（列表、循环、条件判断）

---

## 1. 策略生命周期详解

### 1.1 完整的生命周期

策略有完整的生命周期，就像人的一生：

```python
from pyrust_bt.strategy import Strategy

class MyStrategy(Strategy):
    def on_start(self, ctx):
        """回测开始时调用 - 就像'出生'"""
        # 初始化变量、准备数据
        self.trade_count = 0
        self.buy_price = None
        print("策略启动，初始资金:", ctx.cash)
    
    def next(self, bar):
        """每根 K 线调用 - 就像'每天的生活'"""
        # 决策逻辑
        return None
    
    def on_order(self, event):
        """订单事件 - 就像'收到通知'"""
        # 记录订单状态
        pass
    
    def on_trade(self, event):
        """成交事件 - 就像'交易完成'"""
        # 记录成交信息
        self.trade_count += 1
        print(f"第 {self.trade_count} 笔交易完成")
    
    def on_stop(self):
        """回测结束时调用 - 就像'生命结束'"""
        # 清理工作、打印总结
        print(f"回测结束，共完成 {self.trade_count} 笔交易")
```

### 1.2 生命周期流程图

```
开始回测
    ↓
on_start(ctx)  ← 初始化（获取初始上下文）
    ↓
next(bar1)     ← 处理第 1 根 K 线
    ↓
on_order()     ← 订单提交（如果有订单）
    ↓
on_trade()     ← 订单成交（如果成交）
    ↓
next(bar2)     ← 处理第 2 根 K 线
    ↓
... (循环处理所有 K 线)
    ↓
next(barN)     ← 处理最后一根 K 线
    ↓
on_stop()      ← 清理和总结
    ↓
结束回测
```

### 1.3 各方法的作用

| 方法 | 调用时机 | 主要用途 | 必需 |
|------|---------|---------|------|
| `on_start(ctx)` | 回测开始时 | 初始化变量、准备数据 | ❌ |
| `next(bar)` | 每根 K 线 | **核心方法**：决策逻辑 | ✅ |
| `on_order(event)` | 订单状态变化 | 监控订单、记录日志 | ❌ |
| `on_trade(event)` | 订单成交时 | 记录成交、更新状态 | ❌ |
| `on_stop()` | 回测结束时 | 清理、打印总结 | ❌ |

**注意**：只有 `next()` 方法是必需的，其他方法都是可选的。

---

## 2. 编写简单移动平均策略

### 2.1 策略思路

**简单移动平均（SMA）策略**是最经典的策略之一：

**策略逻辑**：
- 计算过去 N 天的平均价格（SMA）
- 如果当前价格 > SMA，说明价格上涨趋势，**买入**
- 如果当前价格 < SMA，说明价格下跌趋势，**卖出**

**简单理解**：
- 📈 **价格 > 均线**：趋势向上，买入
- 📉 **价格 < 均线**：趋势向下，卖出

就像：
- 🚗 **超车**：当前速度 > 平均速度，加速
- 🛑 **减速**：当前速度 < 平均速度，减速

### 2.2 代码实现

让我们一步步实现这个策略：

#### 步骤 1：创建策略类

```python
from pyrust_bt.strategy import Strategy
from typing import Any, Dict, List

class SMAStrategy(Strategy):
    def __init__(self, window: int = 5, size: float = 1.0):
        """
        初始化策略
        
        参数：
            window: 均线周期（默认 5 天）
            size: 每次交易数量（默认 1 股）
        """
        self.window = window
        self.size = size
        self._closes: List[float] = []  # 存储历史收盘价
```

#### 步骤 2：实现 next() 方法

```python
    def next(self, bar: Dict[str, Any]):
        """每根 K 线调用，返回交易信号"""
        # 1. 获取当前收盘价
        close = float(bar["close"])
        
        # 2. 保存历史价格
        self._closes.append(close)
        
        # 3. 如果数据不够，不交易
        if len(self._closes) < self.window:
            return None
        
        # 4. 计算移动平均
        sma = sum(self._closes[-self.window:]) / self.window
        
        # 5. 决策逻辑
        if close > sma:
            # 价格高于均线，买入
            return {"action": "BUY", "type": "market", "size": self.size}
        elif close < sma:
            # 价格低于均线，卖出
            return {"action": "SELL", "type": "market", "size": self.size}
        
        # 价格等于均线，不交易
        return None
```

#### 步骤 3：完整代码

```python
from pyrust_bt.strategy import Strategy
from typing import Any, Dict, List

class SMAStrategy(Strategy):
    """简单移动平均策略"""
    
    def __init__(self, window: int = 5, size: float = 1.0):
        self.window = window
        self.size = size
        self._closes: List[float] = []
    
    def next(self, bar: Dict[str, Any]):
        close = float(bar["close"])
        self._closes.append(close)
        
        # 数据不够，不交易
        if len(self._closes) < self.window:
            return None
        
        # 计算移动平均
        sma = sum(self._closes[-self.window:]) / self.window
        
        # 决策：价格 > 均线买入，价格 < 均线卖出
        if close > sma:
            return {"action": "BUY", "type": "market", "size": self.size}
        elif close < sma:
            return {"action": "SELL", "type": "market", "size": self.size}
        
        return None
```

### 2.3 运行策略

```python
from pyrust_bt.api import BacktestEngine, BacktestConfig
from pyrust_bt.data import load_csv_to_bars

# 配置
config = BacktestConfig(
    start="2020-01-01",
    end="2020-12-31",
    cash=10000.0,
    commission_rate=0.0005,
    slippage_bps=2.0,
)

# 创建引擎
engine = BacktestEngine(config)

# 加载数据
bars = load_csv_to_bars("examples/data/sample.csv", symbol="SAMPLE")

# 创建策略（5日均线，每次交易1股）
strategy = SMAStrategy(window=5, size=1.0)

# 运行回测
result = engine.run(strategy, bars)

# 查看结果
print(f"最终资金: {result['equity']:,.2f}")
print(f"总收益率: {result['stats']['total_return']:.2%}")
```

---

## 3. 订单类型详解

### 3.1 市价单（Market Order）

**简单理解**：
- 市价单就是"立即成交，按当前市场价格"
- 就像去超市买东西，标价多少就付多少

**特点**：
- ✅ 立即成交
- ✅ 成交价格不确定（可能滑点）
- ✅ 适合快速交易

**代码示例**：
```python
# 方式 1：字符串（简单）
return "BUY"   # 市价买入，默认 size=1
return "SELL"  # 市价卖出，默认 size=1

# 方式 2：字典（详细）
return {
    "action": "BUY",
    "type": "market",
    "size": 1.0
}
```

### 3.2 限价单（Limit Order）

**简单理解**：
- 限价单就是"指定价格，只有达到这个价格才成交"
- 就像在淘宝上设置"价格降到 100 元才买"

**特点**：
- ⏳ 可能不成交（价格没达到）
- ✅ 成交价格确定（不会滑点）
- ✅ 适合精确控制价格

**代码示例**：
```python
return {
    "action": "BUY",
    "type": "limit",
    "size": 1.0,
    "price": 100.0  # 指定价格：100 元
}
```

**注意**：在回测中，限价单会在同 bar 内检查是否成交：
- 买入限价单：如果 bar 的最低价 <= 限价，则成交（按限价）
- 卖出限价单：如果 bar 的最高价 >= 限价，则成交（按限价）

### 3.3 订单格式对比

| 格式 | 优点 | 缺点 | 适用场景 |
|------|------|------|---------|
| 字符串 | 简单快速 | 功能有限 | 快速测试、简单策略 |
| 字典 | 功能完整 | 代码稍长 | 复杂策略、精确控制 |

**建议**：
- 🚀 **快速测试**：使用字符串格式
- 💼 **正式策略**：使用字典格式，更灵活

---

## 4. 回调函数使用

### 4.1 on_order()：订单状态监控

`on_order()` 在订单状态变化时调用：

```python
def on_order(self, event: Dict[str, Any]):
    """订单状态变化时调用"""
    order_id = event.get("order_id")
    status = event.get("status")  # submitted, filled, cancelled
    side = event.get("side")      # BUY or SELL
    size = event.get("size")
    price = event.get("price")
    
    print(f"订单 {order_id}: {status} - {side} {size} @ {price}")
```

**使用场景**：
- 📊 记录所有订单
- 🔔 发送交易通知
- 📈 实时监控策略

### 4.2 on_trade()：成交事件处理

`on_trade()` 在订单成交时调用：

```python
def on_trade(self, event: Dict[str, Any]):
    """订单成交时调用"""
    order_id = event.get("order_id")
    side = event.get("side")      # BUY or SELL
    price = event.get("price")    # 成交价格
    size = event.get("size")      # 成交数量
    
    print(f"成交: {side} {size} 股 @ {price} 元")
    
    # 可以记录到列表
    if not hasattr(self, 'trades'):
        self.trades = []
    self.trades.append({
        "side": side,
        "price": price,
        "size": size
    })
```

**使用场景**：
- 📝 记录交易历史
- 💰 计算盈亏
- 📊 分析交易模式

### 4.3 完整示例：带回调的策略

```python
class SMAStrategyWithCallbacks(Strategy):
    def __init__(self, window: int = 5, size: float = 1.0):
        self.window = window
        self.size = size
        self._closes: List[float] = []
        self.trades = []  # 记录交易历史
    
    def on_start(self, ctx):
        """回测开始"""
        print(f"策略启动，初始资金: {ctx.cash:,.2f}")
        self.trades = []
    
    def next(self, bar):
        """决策逻辑"""
        close = float(bar["close"])
        self._closes.append(close)
        
        if len(self._closes) < self.window:
            return None
        
        sma = sum(self._closes[-self.window:]) / self.window
        
        if close > sma:
            return {"action": "BUY", "type": "market", "size": self.size}
        elif close < sma:
            return {"action": "SELL", "type": "market", "size": self.size}
        
        return None
    
    def on_order(self, event):
        """订单状态变化"""
        status = event.get("status")
        if status == "submitted":
            print(f"订单提交: {event.get('side')} {event.get('size')} 股")
    
    def on_trade(self, event):
        """订单成交"""
        side = event.get("side")
        price = event.get("price")
        size = event.get("size")
        print(f"成交: {side} {size} 股 @ {price:.2f} 元")
        
        # 记录交易
        self.trades.append({
            "side": side,
            "price": price,
            "size": size
        })
    
    def on_stop(self):
        """回测结束"""
        print(f"\n回测结束，共完成 {len(self.trades)} 笔交易")
        if self.trades:
            print("交易记录:")
            for i, trade in enumerate(self.trades, 1):
                print(f"  {i}. {trade['side']} {trade['size']} 股 @ {trade['price']:.2f} 元")
```

---

## 5. 实战练习

### 练习 1：修改均线周期

修改 SMA 策略，使用 10 日均线而不是 5 日均线：

```python
# 你的代码
strategy = SMAStrategy(window=10, size=1.0)
```

运行回测，观察结果变化。

### 练习 2：添加止损逻辑

在 SMA 策略中添加止损：
- 如果持仓亏损超过 5%，立即卖出

```python
class SMAStrategyWithStopLoss(Strategy):
    def __init__(self, window: int = 5, size: float = 1.0, stop_loss: float = 0.05):
        self.window = window
        self.size = size
        self.stop_loss = stop_loss  # 止损比例：5%
        self._closes: List[float] = []
        self.buy_price = None  # 记录买入价格
    
    def next(self, bar):
        close = float(bar["close"])
        self._closes.append(close)
        
        # 如果有持仓，检查止损
        if self.buy_price is not None:
            loss = (self.buy_price - close) / self.buy_price
            if loss >= self.stop_loss:
                # 触发止损，卖出
                self.buy_price = None
                return {"action": "SELL", "type": "market", "size": self.size}
        
        # 原有的 SMA 逻辑
        if len(self._closes) < self.window:
            return None
        
        sma = sum(self._closes[-self.window:]) / self.window
        
        if close > sma and self.buy_price is None:
            # 买入并记录价格
            self.buy_price = close
            return {"action": "BUY", "type": "market", "size": self.size}
        elif close < sma and self.buy_price is not None:
            # 卖出并清除记录
            self.buy_price = None
            return {"action": "SELL", "type": "market", "size": self.size}
        
        return None
```

### 练习 3：使用限价单

修改策略，使用限价单而不是市价单：

```python
# 买入时，使用当前价格 - 0.5% 作为限价
limit_price = close * 0.995  # 当前价格的 99.5%
return {
    "action": "BUY",
    "type": "limit",
    "size": self.size,
    "price": limit_price
}
```

---

## 本节要点

✅ **策略生命周期**：on_start → next → on_stop，next() 是核心  
✅ **SMA 策略**：价格 > 均线买入，价格 < 均线卖出  
✅ **订单类型**：市价单立即成交，限价单指定价格  
✅ **回调函数**：on_order 和 on_trade 用于监控交易过程  
✅ **策略就是实现 next() 方法**：根据 K 线数据返回交易信号

---

## 下一步学习

恭喜你完成了第 3 课！🎉

现在你已经：
- ✅ 理解了策略的完整生命周期
- ✅ 能够编写简单的交易策略
- ✅ 掌握了订单类型和回调函数

**下一步**：继续学习 [第 4 课：数据管理实战](./tutorial-04-data-management.md)，学习如何高效管理你的 K 线数据。

---

## 扩展阅读

- 查看完整示例：[../examples/run_mvp.py](../examples/run_mvp.py)
- 了解更多策略模式：查看 examples 目录下的其他示例
- 学习指标计算：查看 [../python/pyrust_bt/indicators.py](../python/pyrust_bt/indicators.py)

---

**记住**：策略的核心是 `next()` 方法，它决定了"什么时候买、什么时候卖"。好的策略需要清晰的逻辑和充分的测试！

**继续加油！** 🚀

