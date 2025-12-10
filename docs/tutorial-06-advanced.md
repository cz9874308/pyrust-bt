# 第 6 课：进阶功能探索 - 解锁高级特性

## 简介

在前面的课程中，我们学习了单资产回测、策略编写、数据管理和结果分析。本课将探索更高级的功能：多资产回测、参数优化、因子分析等，帮助你构建更强大的量化交易系统。

**预计学习时间**：50-60 分钟

## 学习目标

完成本课后，你将能够：

- ✅ 掌握多资产/多周期回测
- ✅ 学会参数优化（网格搜索）
- ✅ 理解因子分析和 IC 计算
- ✅ 掌握性能优化技巧

## 前置知识

- 完成前 5 课
- 基本的量化交易概念
- 对策略优化有初步了解

---

## 1. 多资产回测

### 1.1 什么是多资产回测？

**简单理解**：

多资产回测就是同时回测多个股票或资产，就像：
- 🎯 **单资产**：只交易一只股票（如 AAPL）
- 🎯 **多资产**：同时交易多只股票（如 AAPL、TSLA、MSFT）

**应用场景**：
- 组合策略（等权重、市值加权等）
- 对冲策略（做多一只，做空另一只）
- 轮动策略（在不同资产间切换）

### 1.2 使用 run_multi() 方法

```python
from pyrust_bt.api import BacktestEngine, BacktestConfig
from pyrust_bt.data import load_csv_to_bars

# 配置
config = BacktestConfig(
    start="2020-01-01",
    end="2020-12-31",
    cash=100000.0,  # 初始资金
    commission_rate=0.0005,
    slippage_bps=2.0,
)

engine = BacktestEngine(config)

# 准备多资产数据
feeds = {
    "AAPL": load_csv_to_bars("data/AAPL.csv", symbol="AAPL"),
    "TSLA": load_csv_to_bars("data/TSLA.csv", symbol="TSLA"),
    "MSFT": load_csv_to_bars("data/MSFT.csv", symbol="MSFT"),
}

# 多资产策略
class MultiAssetStrategy(Strategy):
    def next(self, bar):
        """bar 中包含 feed_id，表示来自哪个资产"""
        feed_id = bar.get("feed_id")  # "AAPL", "TSLA", "MSFT"
        close = bar["close"]
        
        # 简单的等权重策略：每只股票分配相同资金
        if close > 100:
            return {
                "action": "BUY",
                "type": "market",
                "size": 1.0,
                "feed_id": feed_id  # 指定交易哪个资产
            }
        return None

# 运行多资产回测
strategy = MultiAssetStrategy()
result = engine.run_multi(strategy, feeds)

print(f"最终资产: {result['equity']:,.2f}")
```

### 1.3 联合时间线推进机制

**简单理解**：

多资产回测时，引擎会：
1. 合并所有资产的时间线
2. 按时间顺序处理每根 K 线
3. 每次调用 `next()` 时，传入当前时间点的所有资产数据

**示例**：
```
时间线：
2020-01-02: AAPL bar1, TSLA bar1, MSFT bar1
2020-01-03: AAPL bar2, TSLA bar2, MSFT bar2
...
```

引擎会按时间顺序处理，每次传入一个资产的 bar。

### 1.4 多资产等权重策略示例

```python
class EqualWeightStrategy(Strategy):
    """等权重策略：每只股票分配相同资金"""
    
    def __init__(self, num_assets: int = 3):
        self.num_assets = num_assets
        self.positions = {}  # 记录每只股票的持仓
    
    def on_start(self, ctx):
        self.positions = {}
        # 计算每只股票应该分配的资金
        self.target_value = ctx.cash / self.num_assets
    
    def next(self, bar):
        feed_id = bar.get("feed_id")
        close = bar["close"]
        current_position = self.positions.get(feed_id, 0.0)
        
        # 计算目标持仓
        target_shares = self.target_value / close
        
        # 调整持仓到目标
        if target_shares > current_position:
            # 需要买入
            buy_size = target_shares - current_position
            return {
                "action": "BUY",
                "type": "market",
                "size": buy_size,
                "feed_id": feed_id
            }
        elif target_shares < current_position:
            # 需要卖出
            sell_size = current_position - target_shares
            return {
                "action": "SELL",
                "type": "market",
                "size": sell_size,
                "feed_id": feed_id
            }
        
        return None
```

---

## 2. 参数优化

### 2.1 什么是参数优化？

**简单理解**：

策略通常有参数，例如：
- SMA 策略的均线周期（5 天？10 天？20 天？）
- 止损比例（5%？10%？）

参数优化就是**找到最佳参数组合**，让策略表现最好。

就像：
- 🎯 **调音**：找到最佳音调
- 🎯 **参数优化**：找到最佳参数

### 2.2 网格搜索（Grid Search）

网格搜索是最简单的参数优化方法：

```python
from pyrust_bt.optimize import grid_search
from pyrust_bt.api import BacktestEngine, BacktestConfig

# 定义参数范围
param_grid = {
    "window": [5, 10, 20, 30],      # 均线周期
    "size": [1.0, 2.0, 3.0]         # 交易数量
}

# 定义策略类
class SMAStrategy(Strategy):
    def __init__(self, window: int = 5, size: float = 1.0):
        self.window = window
        self.size = size
        self._closes = []
    
    def next(self, bar):
        # ... SMA 策略逻辑
        pass

# 运行网格搜索
results = grid_search(
    strategy_class=SMAStrategy,
    param_grid=param_grid,
    bars=bars,
    config=config,
    scoring="sharpe"  # 优化目标：夏普比率
)

# 查看最佳参数
best_params = results["best_params"]
best_score = results["best_score"]

print(f"最佳参数: {best_params}")
print(f"最佳得分: {best_score:.4f}")
```

### 2.3 优化结果分析

```python
# 查看所有参数组合的结果
for params, score in results["all_results"]:
    print(f"参数 {params}: 得分 {score:.4f}")

# 绘制参数热力图
import matplotlib.pyplot as plt
import numpy as np

# 准备数据
windows = [5, 10, 20, 30]
sizes = [1.0, 2.0, 3.0]
scores = np.zeros((len(windows), len(sizes)))

for params, score in results["all_results"]:
    w_idx = windows.index(params["window"])
    s_idx = sizes.index(params["size"])
    scores[w_idx, s_idx] = score

# 绘制热力图
plt.figure(figsize=(10, 6))
plt.imshow(scores, cmap='viridis', aspect='auto')
plt.colorbar(label='Sharpe Ratio')
plt.xlabel('Size')
plt.ylabel('Window')
plt.xticks(range(len(sizes)), sizes)
plt.yticks(range(len(windows)), windows)
plt.title('参数优化热力图')
plt.tight_layout()
plt.savefig("optimization_heatmap.png", dpi=300)
plt.show()
```

### 2.4 自定义评分函数

```python
def custom_scoring(result):
    """自定义评分函数"""
    stats = result["stats"]
    
    # 综合考虑多个指标
    score = (
        stats["sharpe"] * 0.4 +      # 夏普比率权重 40%
        stats["total_return"] * 0.3 + # 总收益权重 30%
        (1 - stats["max_drawdown"]) * 0.3  # 回撤权重 30%（回撤越小越好）
    )
    
    return score

# 使用自定义评分
results = grid_search(
    strategy_class=SMAStrategy,
    param_grid=param_grid,
    bars=bars,
    config=config,
    scoring=custom_scoring
)
```

---

## 3. 因子分析

### 3.1 什么是因子？

**简单理解**：

因子就是"影响股价的指标"，例如：
- 📊 **动量因子**：过去 N 天的收益率
- 📊 **价值因子**：市盈率、市净率
- 📊 **技术因子**：RSI、MACD

**因子分析**就是评估因子是否有效：
- 因子值高的股票，未来收益是否更高？
- 因子的预测能力如何？

### 3.2 因子回测

```python
from pyrust_bt.analyzers import factor_backtest

# 准备数据（需要包含因子值）
bars = [
    {
        "datetime": "2020-01-02",
        "close": 100.0,
        "factor": 0.5,  # 因子值
        ...
    },
    ...
]

# 运行因子回测
result = factor_backtest(
    bars=bars,
    factor_key="factor",      # 因子字段名
    quantiles=5,              # 分成 5 组
    forward=1                 # 未来 1 期收益
)

print("因子回测结果:")
print(f"  分位数收益: {result['quantile_returns']}")
print(f"  IC: {result['ic']:.4f}")
print(f"  ICIR: {result['icir']:.4f}")
```

### 3.3 IC 和 ICIR

**IC（Information Coefficient）**：
- 衡量因子与未来收益的相关性
- 范围：-1 到 1
- > 0：正相关（因子值高，未来收益高）
- < 0：负相关（因子值高，未来收益低）

**ICIR（IC Information Ratio）**：
- IC 的稳定性
- 值越大，因子越稳定

```python
# 查看 IC 时间序列
ic_series = result["ic_series"]
print(f"平均 IC: {np.mean(ic_series):.4f}")
print(f"IC 标准差: {np.std(ic_series):.4f}")
print(f"ICIR: {result['icir']:.4f}")

# 绘制 IC 时间序列
plt.figure(figsize=(12, 6))
plt.plot(ic_series)
plt.axhline(y=0, color='r', linestyle='--', label='IC=0')
plt.xlabel("时间")
plt.ylabel("IC")
plt.title("IC 时间序列")
plt.legend()
plt.grid(True, alpha=0.3)
plt.tight_layout()
plt.savefig("ic_series.png", dpi=300)
plt.show()
```

### 3.4 分位数分析

```python
# 查看各分位数的收益
quantile_returns = result["quantile_returns"]
print("分位数收益:")
for i, ret in enumerate(quantile_returns, 1):
    print(f"  第 {i} 分位数: {ret:.2%}")

# 绘制分位数收益图
plt.figure(figsize=(10, 6))
plt.bar(range(1, len(quantile_returns) + 1), quantile_returns)
plt.xlabel("分位数（1=最低，5=最高）")
plt.ylabel("收益率")
plt.title("分位数收益分析")
plt.grid(True, alpha=0.3, axis='y')
plt.tight_layout()
plt.savefig("quantile_returns.png", dpi=300)
plt.show()
```

---

## 4. 性能优化技巧

### 4.1 batch_size 优化

```python
# 小 batch_size（慢，但内存占用少）
config1 = BacktestConfig(
    batch_size=100,  # 每次处理 100 根 K 线
    ...
)

# 大 batch_size（快，但内存占用多）
config2 = BacktestConfig(
    batch_size=5000,  # 每次处理 5000 根 K 线
    ...
)

# 建议：
# - 小数据量（< 10k bars）: 1000
# - 中等数据量（10k-100k bars）: 2000-3000
# - 大数据量（> 100k bars）: 3000-5000
```

### 4.2 使用向量化指标

```python
# ❌ 慢：Python 循环计算
def compute_sma_py(closes, window):
    sma = []
    for i in range(len(closes)):
        if i < window:
            sma.append(None)
        else:
            sma.append(sum(closes[i-window:i]) / window)
    return sma

# ✅ 快：Rust 向量化计算
from engine_rust import compute_sma

sma = compute_sma(closes, window)  # 快 10-50 倍
```

### 4.3 数据预提取

```python
# ❌ 慢：每次从数据库查询
for date in dates:
    bars = get_market_data(db_path, symbol, period, start=date, end=date)
    result = engine.run(strategy, bars)

# ✅ 快：一次性加载所有数据
all_bars = get_market_data(db_path, symbol, period, start=start, end=end)
result = engine.run(strategy, all_bars)
```

### 4.4 使用 DuckDB 存储

```python
# ❌ 慢：每次从 CSV 读取
bars = load_csv_to_bars("data.csv")

# ✅ 快：从数据库查询
bars = get_market_data("data/backtest.db", symbol, period)
```

---

## 5. 实战练习

### 练习 1：多资产等权重策略

实现一个多资产等权重策略，同时交易 3 只股票。

### 练习 2：参数优化

对 SMA 策略进行参数优化，找到最佳均线周期。

### 练习 3：因子分析

计算动量因子（过去 5 天收益率），并进行因子回测。

---

## 本节要点

✅ **多资产回测支持组合策略**：可以同时交易多个资产  
✅ **参数优化帮助找到最佳参数**：使用网格搜索等方法  
✅ **因子分析评估因子有效性**：IC、ICIR、分位数分析  
✅ **性能优化很重要**：batch_size、向量化、数据预提取  
✅ **高级功能需要更多实践**：多尝试、多实验

---

## 下一步学习

恭喜你完成了所有教程！🎉🎉🎉

现在你已经：
- ✅ 掌握了回测框架的完整使用
- ✅ 能够编写和优化策略
- ✅ 学会了数据管理和结果分析
- ✅ 了解了高级功能

**下一步**：
- 📚 阅读项目 README，了解更多功能
- 💻 探索 examples 目录下的更多示例
- 🚀 开始构建你自己的量化交易系统
- 🤝 参与社区讨论，分享经验

---

## 扩展阅读

- 多资产回测示例：[../examples/run_multi_assets.py](../examples/run_multi_assets.py)
- 参数优化示例：[../examples/run_grid_search.py](../examples/run_grid_search.py)
- 因子分析示例：[../examples/run_cs_momentum_sample.py](../examples/run_cs_momentum_sample.py)
- 项目 README：[../README.md](../README.md)

---

## 总结

通过这 6 课的学习，你已经掌握了：

1. **快速上手**：环境安装、第一个回测
2. **核心概念**：引擎、策略、数据、配置
3. **策略编写**：从想法到代码
4. **数据管理**：CSV、DuckDB、重采样
5. **结果分析**：性能指标、可视化
6. **进阶功能**：多资产、优化、因子分析

**记住**：
- 💡 回测不是预测未来，而是验证策略
- 💡 好的策略需要清晰的逻辑和充分的测试
- 💡 性能优化很重要，但不要过早优化
- 💡 多实践、多实验、多学习

**祝你量化交易之路顺利！** 🚀📈

---

**教程完成！** 如有问题，欢迎查阅项目文档或参与社区讨论。

