# 第 5 课：结果分析与可视化 - 读懂你的回测结果

## 简介

运行回测后，我们得到了结果数据。但这些数据意味着什么？如何评估策略的表现？本课将学习如何分析和理解回测结果，使用 Analyzers 工具进行深度分析，并可视化展示结果。

**预计学习时间**：40-50 分钟

## 学习目标

完成本课后，你将能够：

- ✅ 理解回测结果的结构和含义
- ✅ 掌握性能指标的含义和计算方法
- ✅ 使用 Analyzers 进行深度分析
- ✅ 可视化展示回测结果

## 前置知识

- 完成第 4 课
- 基本的统计概念（可选，本课会简单介绍）

---

## 1. 理解回测结果结构

### 1.1 基本结果字段

回测结果是一个字典，包含以下主要字段：

```python
result = {
    # 账户信息
    'cash': 9950.0,           # 当前现金
    'position': 1.0,          # 当前持仓（股数）
    'avg_cost': 100.5,        # 平均成本（元/股）
    'equity': 10050.0,        # 总资产（现金 + 持仓市值）
    'realized_pnl': 50.0,     # 已实现盈亏
    
    # 统计指标
    'stats': {
        'start_equity': 10000.0,
        'end_equity': 10050.0,
        'total_return': 0.005,
        'annualized_return': 0.012,
        'volatility': 0.15,
        'sharpe': 0.8,
        'calmar': 0.5,
        'max_drawdown': 0.1,
        'max_dd_duration': 10,
        'total_trades': 5,
        'winning_trades': 3,
        'losing_trades': 2,
        'win_rate': 0.6,
        'total_pnl': 50.0
    },
    
    # 净值曲线
    'equity_curve': [
        ('2020-01-02', 10000.0),
        ('2020-01-03', 10020.0),
        ...
    ],
    
    # 交易记录
    'trades': [
        (1, 'BUY', 100.0, 1.0),
        (2, 'SELL', 102.0, 1.0),
        ...
    ]
}
```

### 1.2 账户信息解读

```python
# 查看账户信息
print(f"当前现金: {result['cash']:,.2f} 元")
print(f"当前持仓: {result['position']:,.2f} 股")
print(f"平均成本: {result['avg_cost']:,.2f} 元/股")
print(f"总资产: {result['equity']:,.2f} 元")
print(f"已实现盈亏: {result['realized_pnl']:,.2f} 元")
```

**字段说明**：

- **cash**：当前可用现金
- **position**：当前持有的股票数量
- **avg_cost**：持仓的平均成本（用于计算盈亏）
- **equity**：总资产 = 现金 + 持仓市值
- **realized_pnl**：已平仓的盈亏（卖出时实现）

---

## 2. 性能指标详解

### 2.1 收益指标

#### 总收益率（Total Return）

```python
total_return = result['stats']['total_return']
print(f"总收益率: {total_return:.2%}")
```

**计算公式**：
```
总收益率 = (期末资产 - 期初资产) / 期初资产
```

**简单理解**：
- 0.05 = 5%：赚了 5%
- -0.1 = -10%：亏了 10%

#### 年化收益率（Annualized Return）

```python
annualized_return = result['stats']['annualized_return']
print(f"年化收益率: {annualized_return:.2%}")
```

**简单理解**：
- 如果回测 1 年，年化收益率 = 总收益率
- 如果回测 6 个月，年化收益率 ≈ 总收益率 × 2

**为什么需要年化？**
- 不同回测周期可以比较
- 例如：3 个月赚 5% vs 1 年赚 10%，哪个更好？

### 2.2 风险指标

#### 波动率（Volatility）

```python
volatility = result['stats']['volatility']
print(f"波动率: {volatility:.4f}")
```

**简单理解**：
- 波动率衡量收益的"稳定性"
- 0.15 = 15%：收益波动较大
- 0.05 = 5%：收益波动较小

**就像**：
- 🎢 **过山车**：波动率大，刺激但风险高
- 🚗 **平稳行驶**：波动率小，稳定但可能收益低

#### 最大回撤（Max Drawdown）

```python
max_drawdown = result['stats']['max_drawdown']
print(f"最大回撤: {max_drawdown:.2%}")
```

**简单理解**：
- 最大回撤 = 从最高点到最低点的最大跌幅
- 0.2 = 20%：最多亏过 20%

**示例**：
```
净值曲线：
10000 → 12000 → 10000 → 11000
         ↑        ↓
      最高点    最低点
      
最大回撤 = (12000 - 10000) / 12000 = 16.7%
```

**为什么重要？**
- 衡量策略的"最坏情况"
- 帮助评估风险承受能力

### 2.3 风险调整收益指标

#### 夏普比率（Sharpe Ratio）

```python
sharpe = result['stats']['sharpe']
print(f"夏普比率: {sharpe:.4f}")
```

**简单理解**：
- 夏普比率 = (收益 - 无风险收益) / 波动率
- 衡量"每承担一单位风险，获得多少收益"

**评判标准**：
- > 1：不错
- > 2：很好
- > 3：优秀

**就像**：
- 🏃 **跑步**：速度快但累（高收益高波动）
- 🚶 **走路**：速度慢但轻松（低收益低波动）
- 夏普比率：衡量"效率"

#### Calmar 比率

```python
calmar = result['stats']['calmar']
print(f"Calmar 比率: {calmar:.4f}")
```

**简单理解**：
- Calmar 比率 = 年化收益率 / 最大回撤
- 衡量"收益与最大风险的比值"

**评判标准**：
- > 1：不错
- > 2：很好

### 2.4 交易统计

```python
stats = result['stats']
print(f"总交易次数: {stats['total_trades']}")
print(f"盈利交易: {stats['winning_trades']}")
print(f"亏损交易: {stats['losing_trades']}")
print(f"胜率: {stats['win_rate']:.2%}")
print(f"总盈亏: {stats['total_pnl']:,.2f} 元")
```

**指标说明**：
- **总交易次数**：买入和卖出的总次数
- **胜率**：盈利交易 / 总交易次数
- **总盈亏**：所有交易的盈亏总和

---

## 3. 使用 Analyzers 进行深度分析

### 3.1 回撤分析

#### compute_drawdown_segments()

分析回撤的各个阶段：

```python
from pyrust_bt.analyzers import compute_drawdown_segments

# 获取净值曲线
equity_curve = result['equity_curve']

# 计算回撤段落
drawdown_segments = compute_drawdown_segments(equity_curve)

print(f"共有 {len(drawdown_segments)} 个回撤阶段")
for i, segment in enumerate(drawdown_segments, 1):
    print(f"\n回撤 {i}:")
    print(f"  开始时间: {segment['start_time']}")
    print(f"  结束时间: {segment['end_time']}")
    print(f"  最大回撤: {segment['max_drawdown']:.2%}")
    print(f"  持续时间: {segment['duration']} 根 K 线")
```

**输出示例**：
```
共有 3 个回撤阶段

回撤 1:
  开始时间: 2020-03-01
  结束时间: 2020-03-15
  最大回撤: -15.2%
  持续时间: 15 根 K 线
```

### 3.2 回合交易分析

#### round_trips_from_trades()

分析完整的买卖回合：

```python
from pyrust_bt.analyzers import round_trips_from_trades

# 获取交易记录和 K 线数据
trades = result['trades']
bars = [...]  # 你的 K 线数据

# 计算回合交易
round_trips = round_trips_from_trades(trades, bars)

print(f"共有 {len(round_trips)} 个完整回合")
for i, trip in enumerate(round_trips[:5], 1):  # 显示前 5 个
    print(f"\n回合 {i}:")
    print(f"  买入时间: {trip['entry_time']}")
    print(f"  卖出时间: {trip['exit_time']}")
    print(f"  买入价格: {trip['entry_price']:.2f}")
    print(f"  卖出价格: {trip['exit_price']:.2f}")
    print(f"  盈亏: {trip['pnl']:.2f} 元")
    print(f"  收益率: {trip['return']:.2%}")
```

### 3.3 性能指标计算

#### compute_performance_metrics()

计算完整的性能指标：

```python
from pyrust_bt.analyzers import compute_performance_metrics

# 获取净值曲线
equity_curve = result['equity_curve']

# 计算性能指标
metrics = compute_performance_metrics(equity_curve)

print("性能指标:")
print(f"  总收益率: {metrics['total_return']:.2%}")
print(f"  年化收益率: {metrics['annualized_return']:.2%}")
print(f"  波动率: {metrics['volatility']:.4f}")
print(f"  夏普比率: {metrics['sharpe']:.4f}")
print(f"  Sortino 比率: {metrics['sortino']:.4f}")
print(f"  Calmar 比率: {metrics['calmar']:.4f}")
print(f"  最大回撤: {metrics['max_drawdown']:.2%}")
print(f"  VaR (95%): {metrics['var_95']:.2%}")
```

### 3.4 综合报告生成

#### generate_analysis_report()

生成完整的分析报告：

```python
from pyrust_bt.analyzers import generate_analysis_report

# 生成报告
report = generate_analysis_report(
    equity_curve=result['equity_curve'],
    trades=result['trades'],
    bars=bars,
    initial_cash=10000.0
)

# 打印报告
print(report)

# 或者保存到文件
with open("backtest_report.txt", "w", encoding="utf-8") as f:
    f.write(report)
```

---

## 4. 可视化展示

### 4.1 绘制净值曲线

使用 matplotlib 绘制净值曲线：

```python
import matplotlib.pyplot as plt
from datetime import datetime

# 准备数据
equity_curve = result['equity_curve']
dates = [datetime.strptime(item[0], "%Y-%m-%d") for item in equity_curve]
equities = [item[1] for item in equity_curve]

# 绘制图表
plt.figure(figsize=(12, 6))
plt.plot(dates, equities, linewidth=2, label="净值曲线")
plt.xlabel("日期")
plt.ylabel("资产（元）")
plt.title("回测净值曲线")
plt.legend()
plt.grid(True, alpha=0.3)
plt.tight_layout()
plt.savefig("equity_curve.png", dpi=300)
plt.show()
```

### 4.2 绘制回撤图

```python
# 计算回撤
drawdowns = []
peak = equity_curve[0][1]
for date, equity in equity_curve:
    if equity > peak:
        peak = equity
    drawdown = (equity - peak) / peak
    drawdowns.append((date, drawdown))

# 绘制回撤图
dates = [datetime.strptime(item[0], "%Y-%m-%d") for item in drawdowns]
dd_values = [item[1] * 100 for item in drawdowns]  # 转换为百分比

plt.figure(figsize=(12, 6))
plt.fill_between(dates, dd_values, 0, alpha=0.3, color='red', label="回撤")
plt.plot(dates, dd_values, linewidth=1, color='red')
plt.xlabel("日期")
plt.ylabel("回撤（%）")
plt.title("回测回撤图")
plt.legend()
plt.grid(True, alpha=0.3)
plt.tight_layout()
plt.savefig("drawdown.png", dpi=300)
plt.show()
```

### 4.3 绘制交易分布

```python
# 分析交易盈亏分布
round_trips = round_trips_from_trades(result['trades'], bars)
profits = [trip['pnl'] for trip in round_trips if trip['pnl'] > 0]
losses = [trip['pnl'] for trip in round_trips if trip['pnl'] < 0]

# 绘制分布图
plt.figure(figsize=(10, 6))
plt.hist(profits, bins=20, alpha=0.7, label="盈利交易", color='green')
plt.hist(losses, bins=20, alpha=0.7, label="亏损交易", color='red')
plt.xlabel("盈亏（元）")
plt.ylabel("交易次数")
plt.title("交易盈亏分布")
plt.legend()
plt.grid(True, alpha=0.3)
plt.tight_layout()
plt.savefig("trade_distribution.png", dpi=300)
plt.show()
```

---

## 5. 实战练习

### 练习 1：分析回测结果

运行一个回测，然后分析结果：

```python
# 1. 运行回测
result = engine.run(strategy, bars)

# 2. 打印基本统计
stats = result['stats']
print("=== 回测结果 ===")
print(f"总收益率: {stats['total_return']:.2%}")
print(f"年化收益率: {stats['annualized_return']:.2%}")
print(f"夏普比率: {stats['sharpe']:.4f}")
print(f"最大回撤: {stats['max_drawdown']:.2%}")
print(f"胜率: {stats['win_rate']:.2%}")
```

### 练习 2：绘制净值曲线

使用 matplotlib 绘制净值曲线和回撤图。

### 练习 3：生成完整报告

使用 `generate_analysis_report()` 生成完整的分析报告并保存到文件。

---

## 本节要点

✅ **回测结果包含账户、统计、曲线、交易**：全面了解策略表现  
✅ **性能指标帮助评估策略**：收益、风险、风险调整收益  
✅ **Analyzers 提供深度分析**：回撤、回合交易、综合报告  
✅ **可视化帮助理解结果**：净值曲线、回撤图、交易分布  
✅ **好的策略需要多维度评估**：不能只看收益率

---

## 下一步学习

恭喜你完成了第 5 课！🎉

现在你已经：
- ✅ 理解了回测结果的结构
- ✅ 掌握了性能指标的含义
- ✅ 学会了使用 Analyzers 分析工具
- ✅ 能够可视化展示结果

**下一步**：继续学习 [第 6 课：进阶功能探索](./tutorial-06-advanced.md)，学习多资产回测、参数优化、因子分析等高级功能。

---

## 扩展阅读

- 查看分析器示例：[../examples/run_analyzers.py](../examples/run_analyzers.py)
- 了解更多分析函数：查看 [../python/pyrust_bt/analyzers.py](../python/pyrust_bt/analyzers.py)
- 学习更多可视化技巧：matplotlib 官方文档

---

**记住**：回测结果只是历史表现，不代表未来。但通过深入分析，我们可以更好地理解策略的特征和风险，为实盘交易做好准备！

**继续加油！** 🚀

