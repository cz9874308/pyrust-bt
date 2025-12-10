# 第 2 课：理解核心概念 - 掌握回测框架的基石

## 简介

在上一课中，我们快速运行了第一个回测。现在，让我们深入理解回测框架的核心组件，就像学习开车时，不仅要会开，还要理解发动机、变速箱、方向盘的工作原理一样。

**预计学习时间**：30-40 分钟

## 学习目标

完成本课后，你将能够：

- ✅ 理解 BacktestEngine 的作用和工作原理
- ✅ 理解 Strategy 接口和生命周期
- ✅ 掌握数据格式要求和数据来源
- ✅ 理解回测配置参数的含义和影响

## 前置知识

- 完成第 1 课
- 基本的面向对象概念（类、方法、继承）
- Python 基础（字典、列表）

---

## 1. BacktestEngine：回测引擎核心

### 1.1 什么是 BacktestEngine？

**简单理解**：

BacktestEngine 就像一台"时间机器"的引擎，它负责：
- ⏰ **时间推进**：按时间顺序处理每一根 K 线
- 📊 **订单撮合**：处理买卖订单，计算成交价格
- 💰 **账户管理**：跟踪现金、持仓、成本、盈亏
- 📈 **结果统计**：计算收益率、夏普比率、最大回撤等

就像：
- 🚗 **汽车引擎**：驱动汽车前进
- ⚙️ **回测引擎**：驱动回测执行

### 1.2 BacktestEngine 的主要功能

让我们看看引擎内部做了什么：

```python
from pyrust_bt.api import BacktestEngine, BacktestConfig

# 创建配置
config = BacktestConfig(
    start="2020-01-01",
    end="2020-12-31",
    cash=10000.0,
    commission_rate=0.0005,
    slippage_bps=2.0,
)

# 创建引擎
engine = BacktestEngine(config)
```

**引擎的工作流程**：

1. **初始化**：根据配置创建引擎实例
2. **数据准备**：预提取所有 K 线数据到 Rust 结构（提升性能）
3. **策略启动**：调用策略的 `on_start()` 方法
4. **循环处理**：对每根 K 线：
   - 构造当前 bar 和上下文
   - 调用策略的 `next()` 方法获取交易信号
   - 解析订单动作（字符串或字典格式）
   - 执行订单撮合（市价/限价）
   - 更新持仓和账户状态
   - 触发订单和成交回调
   - 记录净值曲线
5. **策略结束**：调用策略的 `on_stop()` 方法
6. **结果构建**：计算统计指标，构建返回结果

### 1.3 为什么需要 BacktestEngine？

**性能优势**：
- 🚀 **超快速度**：Rust 实现，比纯 Python 快 250 倍
- 📦 **批量处理**：通过 `batch_size` 减少 Python↔Rust 往返
- 💾 **内存优化**：预分配容器，减少内存重分配

**功能完整**：
- ✅ 订单撮合（市价/限价）
- ✅ 成本模型（手续费/滑点）
- ✅ 持仓管理（平均成本/盈亏）
- ✅ 统计计算（收益/风险指标）

---

## 2. Strategy：策略接口

### 2.1 什么是 Strategy？

**简单理解**：

Strategy 就像"交易员"，它负责：
- 🧠 **决策**：根据市场数据决定买卖
- 📝 **记录**：记录交易历史和状态
- 🔔 **响应**：响应订单和成交事件

就像：
- 👨‍💼 **交易员**：观察市场，做出交易决策
- 💻 **Strategy**：观察 K 线，返回交易信号

### 2.2 Strategy 的生命周期

策略有完整的生命周期，就像人的一生：

```python
from pyrust_bt.strategy import Strategy

class MyStrategy(Strategy):
    def on_start(self, ctx):
        """回测开始时调用 - 就像'出生'"""
        # 初始化变量
        self.counter = 0
        print("回测开始了！")
    
    def next(self, bar):
        """每根 K 线调用 - 就像'每天的生活'"""
        self.counter += 1
        # 决策逻辑
        return None
    
    def on_order(self, event):
        """订单事件 - 就像'收到通知'"""
        print(f"订单状态: {event['status']}")
    
    def on_trade(self, event):
        """成交事件 - 就像'交易完成'"""
        print(f"成交: {event['side']} {event['size']} @ {event['price']}")
    
    def on_stop(self):
        """回测结束时调用 - 就像'生命结束'"""
        print(f"回测结束，共处理 {self.counter} 根 K 线")
```

**生命周期流程**：

```
开始回测
    ↓
on_start()  ← 初始化
    ↓
next(bar1)  ← 处理第 1 根 K 线
    ↓
on_order()  ← 订单提交
    ↓
on_trade()  ← 订单成交
    ↓
next(bar2)  ← 处理第 2 根 K 线
    ↓
... (循环)
    ↓
next(barN)  ← 处理最后一根 K 线
    ↓
on_stop()   ← 清理
    ↓
结束回测
```

### 2.3 next() 方法：策略的核心

`next()` 方法是策略的核心，它决定了交易逻辑：

```python
def next(self, bar):
    """
    每根 K 线调用一次
    
    参数：
        bar: 当前 K 线数据，包含：
            - datetime: 时间
            - open: 开盘价
            - high: 最高价
            - low: 最低价
            - close: 收盘价
            - volume: 成交量
            - symbol: 交易标的
    
    返回值：
        - 字符串: "BUY" 或 "SELL"（市价单，默认 size=1）
        - 字典: {"action": "BUY"|"SELL", "type": "market"|"limit", "size": float, "price"?: float}
        - None: 不下单
    """
    close = bar["close"]
    
    # 简单策略：价格 > 100 买入，< 100 卖出
    if close > 100:
        return "BUY"  # 返回字符串
    elif close < 100:
        return {"action": "SELL", "type": "market", "size": 1.0}  # 返回字典
    return None  # 不交易
```

**返回值说明**：

1. **字符串格式**（简单）：
   ```python
   return "BUY"   # 市价买入，默认 size=1
   return "SELL"  # 市价卖出，默认 size=1
   ```

2. **字典格式**（详细）：
   ```python
   return {
       "action": "BUY",      # 买入或卖出
       "type": "market",     # 市价单或限价单
       "size": 1.0,          # 交易数量
       "price": 100.5        # 限价单的价格（可选）
   }
   ```

### 2.4 回调函数：监控交易过程

策略可以通过回调函数监控交易过程：

```python
def on_order(self, event):
    """订单状态变化时调用"""
    # event 包含：
    # - order_id: 订单 ID
    # - status: 订单状态（submitted, filled, cancelled）
    # - side: 买卖方向（BUY/SELL）
    # - type: 订单类型（market/limit）
    # - size: 交易数量
    # - price: 订单价格
    pass

def on_trade(self, event):
    """订单成交时调用"""
    # event 包含：
    # - order_id: 订单 ID
    # - side: 买卖方向
    # - price: 成交价格
    # - size: 成交数量
    pass
```

**使用场景**：
- 📊 记录交易历史
- 🔔 发送交易通知
- 📈 实时监控策略表现

---

## 3. 数据格式：K 线数据

### 3.1 数据格式要求

回测需要 K 线数据，格式必须符合要求：

```python
bar = {
    "datetime": "2020-01-02 09:30:00",  # 时间（必需）
    "open": 100.0,                       # 开盘价（必需）
    "high": 101.0,                       # 最高价（必需）
    "low": 99.0,                         # 最低价（必需）
    "close": 100.5,                      # 收盘价（必需）
    "volume": 10000.0,                   # 成交量（必需）
    "symbol": "AAPL"                     # 交易标的（可选）
}
```

**字段说明**：

| 字段 | 类型 | 必需 | 说明 |
|------|------|------|------|
| datetime | str | ✅ | 时间，格式：YYYY-MM-DD 或 YYYY-MM-DD HH:MM:SS |
| open | float | ✅ | 开盘价 |
| high | float | ✅ | 最高价，必须 >= max(open, close) |
| low | float | ✅ | 最低价，必须 <= min(open, close) |
| close | float | ✅ | 收盘价 |
| volume | float | ✅ | 成交量 |
| symbol | str | ❌ | 交易标的代码（可选） |

### 3.2 数据来源

数据可以来自多个来源：

1. **CSV 文件**（最简单）：
   ```python
   from pyrust_bt.data import load_csv_to_bars
   
   bars = load_csv_to_bars("data.csv", symbol="AAPL")
   ```

2. **DuckDB 数据库**（高性能）：
   ```python
   from engine_rust import get_market_data
   
   bars = get_market_data(
       db_path="data/backtest.db",
       symbol="AAPL",
       period="1d",
       start="2020-01-01",
       end="2020-12-31"
   )
   ```

3. **Python 列表**（灵活）：
   ```python
   bars = [
       {"datetime": "2020-01-02", "open": 100, "high": 101, 
        "low": 99, "close": 100.5, "volume": 10000},
       # ... 更多数据
   ]
   ```

### 3.3 数据验证

引擎会自动验证数据格式：

- ✅ 检查必需字段是否存在
- ✅ 检查价格合理性（high >= max(open, close)）
- ✅ 检查时间顺序（必须按时间排序）

如果数据格式不正确，会抛出异常。

---

## 4. BacktestConfig：回测配置

### 4.1 配置参数详解

BacktestConfig 定义了回测的所有参数：

```python
from pyrust_bt.api import BacktestConfig

config = BacktestConfig(
    start="2020-01-01",        # 回测开始时间
    end="2020-12-31",          # 回测结束时间
    cash=10000.0,              # 初始资金
    commission_rate=0.0005,    # 手续费率
    slippage_bps=2.0,          # 滑点（基点）
    batch_size=1000            # 批处理大小
)
```

### 4.2 参数详细说明

#### start / end：时间范围

```python
start="2020-01-01"  # 回测开始时间
end="2020-12-31"    # 回测结束时间
```

**说明**：
- 格式：YYYY-MM-DD 或 YYYY-MM-DD HH:MM:SS
- 引擎会过滤这个时间范围之外的数据
- 如果数据不包含这个范围，会使用实际数据范围

#### cash：初始资金

```python
cash=10000.0  # 初始资金：1 万元
```

**说明**：
- 单位：元（或其他货币单位）
- 这是回测开始时的现金
- 随着交易，现金会变化

#### commission_rate：手续费率

```python
commission_rate=0.0005  # 手续费率：0.05%（万五）
```

**简单理解**：
- 每次交易都会收取手续费
- 0.0005 = 0.05% = 万五（每 1 万元交易收取 5 元）
- 买入和卖出都会收取

**计算示例**：
- 买入 100 股，价格 100 元/股
- 交易金额：100 × 100 = 10,000 元
- 手续费：10,000 × 0.0005 = 5 元

#### slippage_bps：滑点

```python
slippage_bps=2.0  # 滑点：2 个基点（0.02%）
```

**简单理解**：
- 滑点就是"实际成交价格与预期价格的偏差"
- 1 个基点（bps）= 0.01%
- 2 bps = 0.02%

**为什么有滑点？**
- 市场波动：下单时价格可能已经变化
- 流动性：大单可能影响价格
- 延迟：网络延迟导致价格变化

**计算示例**：
- 预期买入价格：100 元
- 滑点：2 bps = 0.02%
- 实际买入价格：100 × (1 + 0.0002) = 100.02 元

#### batch_size：批处理大小

```python
batch_size=1000  # 批处理大小：1000 根 K 线
```

**简单理解**：
- 引擎会批量处理 K 线，减少 Python↔Rust 往返
- 数值越大，性能越好，但内存占用更多
- 建议值：1000-5000

**性能影响**：
- batch_size=100：较慢，但内存占用少
- batch_size=1000：平衡（推荐）
- batch_size=5000：最快，但内存占用多

### 4.3 配置参数的影响

不同配置会影响回测结果：

```python
# 配置 1：低手续费、无滑点（理想情况）
config1 = BacktestConfig(
    cash=10000.0,
    commission_rate=0.0001,  # 万1
    slippage_bps=0.0,        # 无滑点
)

# 配置 2：高手续费、有滑点（实际情况）
config2 = BacktestConfig(
    cash=10000.0,
    commission_rate=0.0005,  # 万5
    slippage_bps=2.0,        # 2 bps
)

# 同样的策略，config2 的收益会更低（更接近实际情况）
```

**建议**：
- 📊 **研究阶段**：使用较低的手续费和滑点，快速验证策略
- 💼 **实盘准备**：使用实际的手续费和滑点，评估真实表现

---

## 5. EngineContext：执行上下文

### 5.1 什么是 EngineContext？

**简单理解**：

EngineContext 就像"账户状态快照"，它告诉策略当前的账户情况：

```python
ctx = {
    "position": 10.0,      # 当前持仓（股数）
    "avg_cost": 100.5,     # 平均成本（元/股）
    "cash": 9000.0,        # 当前现金（元）
    "equity": 10050.0,     # 总资产（现金 + 持仓市值）
    "bar_index": 100       # 当前 K 线索引
}
```

### 5.2 如何使用 EngineContext？

在 `on_start()` 方法中，引擎会传入初始上下文：

```python
def on_start(self, ctx):
    """回测开始时，可以获取初始账户状态"""
    print(f"初始资金: {ctx.cash}")
    print(f"初始持仓: {ctx.position}")
```

在 `next()` 方法中，可以通过策略属性访问当前状态（引擎会更新）：

```python
class MyStrategy(Strategy):
    def next(self, bar):
        # 注意：next() 方法中不能直接访问 ctx
        # 但可以通过其他方式获取状态（后续课程会讲解）
        close = bar["close"]
        return "BUY" if close > 100 else None
```

---

## 6. 核心概念总结

### 6.1 组件关系图

```
┌─────────────────┐
│  BacktestConfig │  ← 配置参数（规则）
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│ BacktestEngine  │  ← 回测引擎（执行者）
└────────┬────────┘
         │
    ┌────┴────┐
    │         │
    ▼         ▼
┌─────────┐ ┌──────────┐
│ Strategy│ │   Data   │  ← 策略（决策）+ 数据（材料）
└─────────┘ └──────────┘
    │
    ▼
┌─────────┐
│ Results │  ← 回测结果（产出）
└─────────┘
```

### 6.2 工作流程总结

1. **准备阶段**：
   - 创建 BacktestConfig（设置规则）
   - 创建 BacktestEngine（启动引擎）
   - 加载数据（准备材料）
   - 创建 Strategy（准备策略）

2. **执行阶段**：
   - 引擎按时间顺序处理每根 K 线
   - 调用策略的 next() 方法获取交易信号
   - 执行订单撮合和账户更新
   - 触发回调函数

3. **结束阶段**：
   - 调用策略的 on_stop() 方法
   - 计算统计指标
   - 返回回测结果

---

## 7. 实战练习

### 练习 1：理解配置参数

创建一个配置，设置：
- 初始资金：50,000 元
- 手续费率：万 3（0.0003）
- 滑点：1.5 bps
- 批处理大小：2000

```python
# 你的代码
config = BacktestConfig(
    # 填写参数
)
```

### 练习 2：理解策略生命周期

创建一个策略，在生命周期方法中打印信息：

```python
class LifecycleStrategy(Strategy):
    def on_start(self, ctx):
        print("=== 回测开始 ===")
        print(f"初始资金: {ctx.cash}")
    
    def next(self, bar):
        print(f"处理 K 线: {bar['datetime']}")
        return None
    
    def on_stop(self):
        print("=== 回测结束 ===")
```

### 练习 3：理解数据格式

创建一个包含 3 根 K 线的数据列表：

```python
bars = [
    # 填写数据
]
```

---

## 本节要点

✅ **BacktestEngine 是回测的核心**：负责时间推进、订单撮合、账户管理  
✅ **Strategy 定义交易逻辑**：通过 next() 方法返回交易信号  
✅ **数据格式必须符合要求**：包含 datetime、open、high、low、close、volume  
✅ **配置参数影响回测结果**：手续费和滑点会降低收益  
✅ **策略有完整的生命周期**：on_start → next → on_stop  
✅ **回调函数用于监控交易**：on_order 和 on_trade 可以记录交易过程

---

## 下一步学习

恭喜你完成了第 2 课！🎉

现在你已经：
- ✅ 理解了回测引擎的工作原理
- ✅ 掌握了策略接口和生命周期
- ✅ 了解了数据格式和配置参数

**下一步**：继续学习 [第 3 课：编写你的第一个策略](./tutorial-03-write-strategy.md)，学习如何将交易想法转化为可执行的策略代码。

---

## 扩展阅读

- 查看 BacktestEngine 源码：[../rust/engine_rust/src/lib.rs](../rust/engine_rust/src/lib.rs)
- 查看 Strategy 基类：[../python/pyrust_bt/strategy.py](../python/pyrust_bt/strategy.py)
- 了解更多配置选项：查看 BacktestConfig 的定义

---

**记住**：理解核心概念是编写好策略的基础。就像盖房子需要打好地基一样，理解这些概念能帮助你写出更好的策略！

**继续加油！** 🚀

