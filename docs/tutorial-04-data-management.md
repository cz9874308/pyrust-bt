# 第 4 课：数据管理实战 - 高效管理你的 K 线数据

## 简介

在前面的课程中，我们使用 CSV 文件加载数据。但在实际项目中，我们通常需要管理大量的历史数据，这时就需要更高效的数据管理方式。本课将学习如何使用 DuckDB 数据库来存储和查询 K 线数据，以及如何进行数据导入导出和周期转换。

**预计学习时间**：30-40 分钟

## 学习目标

完成本课后，你将能够：

- ✅ 掌握 CSV 数据加载方法
- ✅ 理解 DuckDB 数据库的使用
- ✅ 学会数据导入导出
- ✅ 掌握 K 线重采样（周期转换）

## 前置知识

- 完成第 3 课
- 基本的数据库概念（可选，本课会简单介绍）

---

## 1. CSV 数据加载

### 1.1 使用 load_csv_to_bars() 函数

最简单的方式是从 CSV 文件加载数据：

```python
from pyrust_bt.data import load_csv_to_bars

# 加载 CSV 文件
bars = load_csv_to_bars("examples/data/sample.csv", symbol="AAPL")

print(f"加载了 {len(bars)} 根 K 线")
print(f"第一根 K 线: {bars[0]}")
```

**CSV 文件格式要求**：

```
datetime,open,high,low,close,volume
2020-01-02,100,101,99,100.5,10000
2020-01-03,100.5,101.5,100,101.0,12000
...
```

**必需字段**：
- `datetime`：时间
- `open`：开盘价
- `high`：最高价
- `low`：最低价
- `close`：收盘价
- `volume`：成交量

### 1.2 CSV 加载的优缺点

**优点**：
- ✅ 简单易用
- ✅ 不需要额外工具
- ✅ 适合小数据量

**缺点**：
- ❌ 每次都要重新读取文件
- ❌ 大数据量时速度慢
- ❌ 不支持快速查询

**适用场景**：
- 快速测试
- 小数据量（< 10 万条）
- 一次性使用

---

## 2. DuckDB 数据库使用

### 2.1 什么是 DuckDB？

**简单理解**：

DuckDB 就像一个"超级 Excel"，但更快更强：
- 📊 **存储数据**：可以存储大量 K 线数据
- ⚡ **快速查询**：比 CSV 快 100-1000 倍
- 🔍 **灵活查询**：可以按时间、标的查询
- 💾 **本地文件**：数据存储在本地文件，不需要服务器

就像：
- 📁 **CSV**：像记事本，简单但慢
- 🗄️ **DuckDB**：像数据库，复杂但快

### 2.2 为什么使用 DuckDB？

**性能对比**：

| 操作 | CSV | DuckDB |
|------|-----|--------|
| 加载 10 万条数据 | ~5 秒 | ~0.1 秒 |
| 查询指定时间范围 | 需要全部加载 | ~0.01 秒 |
| 存储空间 | 较大 | 较小（压缩） |

**使用场景**：
- 📈 大量历史数据（> 10 万条）
- 🔄 频繁查询不同时间范围
- 💾 需要持久化存储
- ⚡ 需要高性能

---

## 3. 数据导入到 DuckDB

### 3.1 从 CSV 导入（最快方式）

**方式一：直接从 CSV 导入（推荐）**

这是最快的方式，DuckDB 直接读取 CSV：

```python
from engine_rust import save_klines_from_csv

# 直接从 CSV 导入到数据库
save_klines_from_csv(
    db_path="data/backtest.db",      # 数据库文件路径
    csv_path="examples/data/sample.csv",  # CSV 文件路径
    symbol="AAPL",                   # 交易标的代码
    period="1d",                     # 周期（1m, 5m, 1h, 1d 等）
    replace=False                    # False=追加，True=替换
)

print("数据导入成功！")
```

**方式二：先加载再保存**

如果需要对数据进行处理，可以先加载再保存：

```python
from pyrust_bt.data import load_csv_to_bars
from engine_rust import save_klines

# 1. 从 CSV 加载数据
bars = load_csv_to_bars("examples/data/sample.csv", symbol="AAPL")

# 2. 可以在这里对数据进行处理
# 例如：过滤、转换等

# 3. 保存到数据库
save_klines(
    db_path="data/backtest.db",
    symbol="AAPL",
    period="1d",
    bars=bars,
    replace=False
)
```

### 3.2 批量导入示例

```python
import os
from engine_rust import save_klines_from_csv

# 准备数据文件列表
data_files = [
    ("data/AAPL_2020.csv", "AAPL", "1d"),
    ("data/AAPL_2021.csv", "AAPL", "1d"),
    ("data/TSLA_2020.csv", "TSLA", "1d"),
]

db_path = "data/backtest.db"

for csv_path, symbol, period in data_files:
    if os.path.exists(csv_path):
        print(f"导入 {symbol} 数据...")
        save_klines_from_csv(
            db_path=db_path,
            csv_path=csv_path,
            symbol=symbol,
            period=period,
            replace=False  # 追加模式
        )
        print(f"  {symbol} 导入完成")
    else:
        print(f"  文件不存在: {csv_path}")

print("所有数据导入完成！")
```

---

## 4. 从 DuckDB 查询数据

### 4.1 使用 get_market_data() 查询

```python
from engine_rust import get_market_data

# 查询指定时间范围的数据
bars = get_market_data(
    db_path="data/backtest.db",
    symbol="AAPL",
    period="1d",
    start="2020-01-01",    # 开始时间（可选）
    end="2020-12-31",      # 结束时间（可选）
    count=-1               # -1 表示查询所有，> 0 表示查询最近 N 条
)

print(f"查询到 {len(bars)} 根 K 线")
```

### 4.2 查询最近 N 条数据

```python
# 查询最近 100 条数据
recent_bars = get_market_data(
    db_path="data/backtest.db",
    symbol="AAPL",
    period="1d",
    count=100  # 查询最近 100 条
)

print(f"最近 100 条数据: {len(recent_bars)} 根")
```

### 4.3 查询多个标的

```python
symbols = ["AAPL", "TSLA", "MSFT"]
all_bars = {}

for symbol in symbols:
    bars = get_market_data(
        db_path="data/backtest.db",
        symbol=symbol,
        period="1d",
        start="2020-01-01",
        end="2020-12-31"
    )
    all_bars[symbol] = bars
    print(f"{symbol}: {len(bars)} 根 K 线")
```

---

## 5. K 线重采样（周期转换）

### 5.1 什么是重采样？

**简单理解**：

重采样就是将 K 线数据从一种周期转换为另一种周期。

例如：
- 1 分钟数据 → 5 分钟数据
- 5 分钟数据 → 1 小时数据
- 1 小时数据 → 1 天数据

就像：
- 🕐 **1 分钟**：每 1 分钟一根 K 线
- 🕐 **5 分钟**：每 5 分钟一根 K 线（将 5 根 1 分钟 K 线合并成 1 根）

### 5.2 重采样规则

重采样按照标准的 OHLCV 规则：

- **Open**：取第一根 K 线的开盘价
- **High**：取所有 K 线的最高价
- **Low**：取所有 K 线的最低价
- **Close**：取最后一根 K 线的收盘价
- **Volume**：取所有 K 线的成交量之和

**示例**：

```
原始数据（1 分钟）：
09:30, 100, 101, 99, 100.5, 1000
09:31, 100.5, 101.5, 100, 101.0, 1200
09:32, 101.0, 102.0, 100.5, 101.5, 1500
09:33, 101.5, 103.0, 101.0, 102.5, 1300
09:34, 102.5, 103.5, 102.0, 103.0, 1100

重采样后（5 分钟）：
09:30, 100, 103.5, 99, 103.0, 6100
```

### 5.3 使用 resample_klines() 函数

```python
from engine_rust import resample_klines

# 加载 1 分钟数据
minute_bars = get_market_data(
    db_path="data/backtest.db",
    symbol="AAPL",
    period="1m",
    start="2020-01-01",
    end="2020-01-02"
)

# 转换为 5 分钟数据
five_min_bars = resample_klines(minute_bars, "5m")

print(f"原始数据: {len(minute_bars)} 根（1 分钟）")
print(f"重采样后: {len(five_min_bars)} 根（5 分钟）")
```

### 5.4 支持的周期格式

| 周期 | 说明 | 示例 |
|------|------|------|
| `1m` | 1 分钟 | `"1m"`, `"5m"`, `"15m"`, `"30m"` |
| `1h` | 1 小时 | `"1h"`, `"4h"` |
| `1d` | 1 天 | `"1d"` |
| `1w` | 1 周 | `"1w"` |
| `1mo` | 1 月 | `"1mo"`, `"1M"` |
| `1y` | 1 年 | `"1y"` |

### 5.5 重采样示例

```python
from engine_rust import get_market_data, resample_klines

# 1. 加载 1 分钟数据
minute_bars = get_market_data(
    db_path="data/backtest.db",
    symbol="AAPL",
    period="1m",
    start="2020-01-01",
    end="2020-01-31"
)

# 2. 转换为不同周期
five_min_bars = resample_klines(minute_bars, "5m")
hourly_bars = resample_klines(minute_bars, "1h")
daily_bars = resample_klines(minute_bars, "1d")

print(f"1 分钟: {len(minute_bars)} 根")
print(f"5 分钟: {len(five_min_bars)} 根")
print(f"1 小时: {len(hourly_bars)} 根")
print(f"1 天: {len(daily_bars)} 根")
```

---

## 6. 数据管理最佳实践

### 6.1 数据组织建议

**目录结构**：

```
project/
├── data/
│   ├── backtest.db          # DuckDB 数据库文件
│   ├── raw/                 # 原始 CSV 文件
│   │   ├── AAPL_2020.csv
│   │   └── TSLA_2020.csv
│   └── processed/           # 处理后的数据
└── scripts/
    └── import_data.py       # 数据导入脚本
```

### 6.2 数据导入流程

```python
# import_data.py
from engine_rust import save_klines_from_csv
import os

def import_all_data():
    """导入所有数据到数据库"""
    db_path = "data/backtest.db"
    raw_data_dir = "data/raw"
    
    # 遍历所有 CSV 文件
    for filename in os.listdir(raw_data_dir):
        if filename.endswith(".csv"):
            # 从文件名提取信息（例如：AAPL_2020_1d.csv）
            parts = filename.replace(".csv", "").split("_")
            symbol = parts[0]
            period = parts[-1]  # 假设最后一部分是周期
            
            csv_path = os.path.join(raw_data_dir, filename)
            
            print(f"导入 {symbol} ({period})...")
            save_klines_from_csv(
                db_path=db_path,
                csv_path=csv_path,
                symbol=symbol,
                period=period,
                replace=False
            )
            print(f"  {symbol} 导入完成")

if __name__ == "__main__":
    import_all_data()
```

### 6.3 数据验证

导入数据后，建议验证数据完整性：

```python
from engine_rust import get_market_data

def validate_data(db_path, symbol, period):
    """验证数据完整性"""
    bars = get_market_data(
        db_path=db_path,
        symbol=symbol,
        period=period,
        count=-1
    )
    
    if not bars:
        print(f"❌ {symbol} ({period}): 无数据")
        return False
    
    # 检查数据连续性
    print(f"✅ {symbol} ({period}):")
    print(f"   数据量: {len(bars)} 根")
    print(f"   时间范围: {bars[0]['datetime']} ~ {bars[-1]['datetime']}")
    
    return True

# 验证所有数据
symbols = ["AAPL", "TSLA", "MSFT"]
periods = ["1d", "1h"]

for symbol in symbols:
    for period in periods:
        validate_data("data/backtest.db", symbol, period)
```

---

## 7. 实战练习

### 练习 1：导入 CSV 到数据库

将 `examples/data/sample.csv` 导入到数据库：

```python
# 你的代码
from engine_rust import save_klines_from_csv

save_klines_from_csv(
    db_path="data/backtest.db",
    csv_path="examples/data/sample.csv",
    symbol="SAMPLE",
    period="1d",
    replace=False
)
```

### 练习 2：查询并重采样

从数据库查询 1 分钟数据，转换为 5 分钟数据：

```python
# 你的代码
from engine_rust import get_market_data, resample_klines

# 1. 查询 1 分钟数据
# 2. 转换为 5 分钟数据
# 3. 打印结果
```

### 练习 3：批量导入

创建一个脚本，批量导入多个 CSV 文件：

```python
# 你的代码
# 遍历目录中的所有 CSV 文件
# 自动提取 symbol 和 period
# 批量导入
```

---

## 本节要点

✅ **CSV 适合小数据量**：简单易用，但速度较慢  
✅ **DuckDB 适合大数据量**：快速查询，支持复杂操作  
✅ **数据导入两种方式**：直接从 CSV 导入（最快）或先加载再保存  
✅ **重采样转换周期**：1 分钟 → 5 分钟 → 1 小时 → 1 天  
✅ **数据组织很重要**：合理的目录结构便于管理

---

## 下一步学习

恭喜你完成了第 4 课！🎉

现在你已经：
- ✅ 掌握了 CSV 数据加载
- ✅ 理解了 DuckDB 数据库使用
- ✅ 学会了数据导入导出和重采样

**下一步**：继续学习 [第 5 课：结果分析与可视化](./tutorial-05-analysis.md)，学习如何分析和理解回测结果。

---

## 扩展阅读

- 查看数据导入示例：[../examples/import_csv_to_db.py](../examples/import_csv_to_db.py)
- 了解更多数据库操作：查看 `rust/engine_rust/src/database.rs` 的注释
- DuckDB 官方文档：了解 SQL 查询语法

---

**记住**：好的数据管理是高效回测的基础。合理组织数据，使用数据库存储，可以大大提升回测效率！

**继续加油！** 🚀

