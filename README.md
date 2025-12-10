# pyrust-bt

<div  align="center">
<img src="images/logo.jpeg" width = "200" height = "200" alt="" align=center />
</div>

A hybrid backtesting framework: Python for strategy and data, Rust for the high-performance core via PyO3 bindings. Designed to balance researcher productivity with engine throughput, suitable from research to small-team production.

[中文文档 | Chinese README](README.zh-CN.md)

## Features

-   Rust Engine

    -   Time advancement over bars/ticks
    -   Order matching: market/limit (same-bar simplified execution)
    -   Cost model: commission `commission_rate`, slippage `slippage_bps`
    -   Portfolio & ledger: `position / avg_cost / cash / equity / realized_pnl`
    -   Vectorized indicators: `SMA / RSI` (sliding window optimized)
    -   Statistics: total return, annualized return, volatility, Sharpe, Calmar, max drawdown & duration
    -   Performance: batch processing (`batch_size`), pre-extracted data, preallocated buffers, inlined hot paths

-   Python API

    -   Strategy lifecycle: `on_start` → `next(bar)` → `on_stop` with `on_order / on_trade` callbacks
    -   Action format:
        -   String: `"BUY" | "SELL"`
        -   Dict: `{ "action": "BUY"|"SELL", "type": "market"|"limit", "size": float, "price"?: float }`
    -   Data loader: CSV → list[dict] (MVP; pluggable for Parquet/Arrow)
    -   Analyzers: drawdown segments, round-trips, enhanced performance metrics, factor backtests (quantiles/IC/monotonicity), unified report
    -   **Cross-sectional Factor Analysis**: multi-factor evaluation system, quantile portfolio backtesting, IC/ICIR analysis, factor ranking
    -   Optimizer: naive grid search (customizable scoring key)

-   API & Frontend
    -   FastAPI: `POST /runs`, `GET /runs`, `GET /runs/{id}`
    -   Streamlit: submit runs, list & visualize results (equity curve + stats)

## Install & Build

Prereqs: Python 3.8+, Rust (`rustup`), maturin

```powershell
pip install maturin
cd rust/engine_rust

# Option A: Install directly into the active Python environment (best for local dev)
maturin develop --release

# Option B: Build wheel only, install manually afterwards
python -m maturin build --release
pip install --force-reinstall (Get-ChildItem target/wheels/engine_rust-*.whl | Select-Object -First 1).FullName
```

## Quick Start

-   Minimal backtest

    ```powershell
    cd ../..
    python examples/run_mvp.py
    ```

-   Analyzer demo

    ```powershell
    python examples/run_analyzers.py
    ```

-   Grid search

    ```powershell
    python examples/run_grid_search.py
    ```

-   Cross-sectional factor backtesting

    ```powershell
    python examples/run_cs_momentum_sample.py
    ```

-   Quantile portfolio backtesting with trading simulation

    ```powershell
    python examples/run_cs_quantile_portfolios.py
    ```

-   Performance test & batch-size comparison

    ```powershell
    python examples/run_performance_test.py
    ```

Sample data: `examples/data/sample.csv` (headers: `datetime,open,high,low,close,volume`).

## Market Data: DuckDB + QMT

### DuckDB Local Store

-   The default database lives at `data/backtest.db`, but you can point scripts to any DuckDB file via `--db`.
-   Bulk-import historical data:

    ```powershell
    # Write CSV directly into DuckDB (requires the compiled engine_rust extension)
    python examples/import_csv_to_db.py data/513500_d.csv --symbol 513500.SH --period 1d --db data/backtest.db
    ```

-   Use `--no-direct-csv` to parse the CSV in Python first (useful for inspection) before saving through Rust.
-   Internally `save_klines` / `save_klines_from_csv` persist to the canonical schema; feel free to inspect the DB with `duckdb` CLI or any DuckDB-compatible tool.

### Zero-Maintenance QMT / XtData Backfill

-   When the local DuckDB store misses date ranges, `MarketDataService` can call QMT Mini (xtdata) to download the gap and write it back, achieving a “DB first → auto backfill” workflow.
-   Preparation checklist:
    1. Install the `XtQuant` Python package by copying it into your interpreter’s `site-packages`, e.g. `D:\ProgramData\miniconda3\Lib\site-packages` (adjust to your environment).
    2. Verify `import XtQuant.XtData` works in Python.
    3. Set the `XTDATA_DIR` environment variable to the MiniQmt `userdata_mini` directory (default in examples: `D:\国金证券QMT交易端\userdata_mini`).
-   Run the multi-asset equal-weight example to smoke-test the data pipeline:

    ```powershell
    python examples/run_multi_asset_rebalance_strategy.py
    ```

-   See `docs/xtdata_market_data_plan.md` for architecture details and operational recommendations.

## In Code

-   Config & engine

    ```python
    from pyrust_bt.api import BacktestEngine, BacktestConfig
    cfg = BacktestConfig(start="2020-01-01", end="2020-12-31", cash=100000,
                         commission_rate=0.0005, slippage_bps=2.0, batch_size=1000)
    engine = BacktestEngine(cfg)
    ```

-   Strategy (minimal)

    ```python
    from pyrust_bt.strategy import Strategy
    class MyStrategy(Strategy):
        def next(self, bar):
            if bar["close"] > 100:
                return {"action": "BUY", "type": "market", "size": 1.0}
            return None
    ```

-   Run

    ```python
    from pyrust_bt.data import load_csv_to_bars
    bars = load_csv_to_bars("examples/data/sample.csv", symbol="SAMPLE")
    result = engine.run(MyStrategy(), bars)
    print(result["stats"], result["equity"])  # stats & equity
    ```

## Analysis & Reports

-   Drawdown segments: `compute_drawdown_segments(equity_curve)`
-   Round trips: `round_trips_from_trades(trades, bars)` / export CSV
-   Performance metrics: `compute_performance_metrics(equity_curve)` (Sharpe/Sortino/Calmar/VAR)
-   Factor backtest: `factor_backtest(bars, factor_key, quantiles, forward)`
-   Unified report: `generate_analysis_report(...)`
-   **Cross-sectional Factor Evaluation**:
    -   Multi-factor analyzer: `MultiFactorAnalyzer` with time-series/cross-sectional methods
    -   Factor ranking: IC, ICIR, monotonicity, stability, turnover analysis
    -   Quantile portfolio backtesting: `CrossSectionFactorBacktester` for large-scale factor evaluation
    -   Export: detailed reports, factor rankings, correlation matrices

## API & Frontend

-   Start API (FastAPI)

    ```powershell
    pip install fastapi uvicorn pydantic requests streamlit
    python -m uvicorn python.server_main:app --reload
    ```

-   Start frontend (Streamlit)

    ```powershell
    set PYRUST_BT_API=http://127.0.0.1:8000
    streamlit run frontend/streamlit_app.py
    ```

## Performance Notes

-   Prefer larger `batch_size` (e.g., 1000–5000) to reduce Python round-trips
-   Prefer dict actions over strings
-   Use Rust vectorized indicators (`compute_sma/compute_rsi`) when possible
-   For large data, prefer Parquet/Arrow and partitioned reads (by symbol/time)

## Architecture

### System Architecture

The following diagram illustrates the overall architecture of pyrust-bt:

```mermaid
graph TB
    subgraph "Frontend Layer"
        UI[Streamlit UI<br/>可视化界面]
    end

    subgraph "API Layer"
        API[FastAPI Server<br/>RESTful API]
    end

    subgraph "Python Application Layer"
        Strategy[Strategy<br/>策略定义]
        Wrapper[API Wrapper<br/>Python接口封装]
        DataLoader[Data Loader<br/>数据加载器]
        Analyzers[Analyzers<br/>性能分析器]
        MarketData[Market Data Service<br/>市场数据服务]
    end

    subgraph "Rust Engine Layer (PyO3)"
        Engine[BacktestEngine<br/>回测引擎核心]
        Config[BacktestConfig<br/>回测配置]
        Context[EngineContext<br/>执行上下文]
        Indicators[Vectorized Indicators<br/>向量化指标计算]
        Stats[Statistics<br/>统计计算]
        DBModule[Database Module<br/>数据库操作模块]
    end

    subgraph "Data Storage Layer"
        DuckDB[(DuckDB<br/>K线数据存储)]
        CSV[CSV Files<br/>CSV数据文件]
    end

    UI -->|HTTP Request| API
    API -->|调用| Wrapper
    Wrapper -->|PyO3绑定| Engine
    Strategy -->|策略逻辑| Engine
    DataLoader -->|加载数据| Wrapper
    MarketData -->|查询数据| DuckDB
    Engine -->|读取数据| DuckDB
    Engine -->|保存数据| DuckDB
    CSV -->|导入| DuckDB
    Engine -->|返回结果| Analyzers
    Analyzers -->|分析报告| API
    API -->|JSON响应| UI
    Engine -->|使用| Config
    Engine -->|提供| Context
    Engine -->|调用| Indicators
    Engine -->|调用| Stats
    Engine -->|使用| DBModule
```

### Data Flow

The following diagram shows the complete data flow from input to output:

```mermaid
sequenceDiagram
    participant User as 用户/前端
    participant API as FastAPI
    participant Python as Python Layer
    participant Rust as Rust Engine
    participant DB as DuckDB
    participant CSV as CSV Files

    Note over User,CSV: 数据准备阶段
    User->>CSV: 提供CSV数据文件
    CSV->>Python: 读取CSV文件
    Python->>DB: 导入数据到DuckDB<br/>(save_klines_from_csv)
    DB-->>Python: 数据保存成功

    Note over User,CSV: 回测执行阶段
    User->>API: POST /runs (提交回测任务)
    API->>Python: 创建BacktestEngine
    Python->>Rust: 初始化引擎(PyO3)
    Rust-->>Python: 引擎就绪

    Python->>DB: 查询K线数据<br/>(get_market_data)
    DB-->>Python: 返回K线列表
    Python->>Rust: 预提取数据到Rust结构<br/>(extract_bars_data)
    Rust-->>Python: 数据准备完成

    loop 每根K线处理
        Rust->>Python: 调用策略next()方法<br/>(传入bar和context)
        Python->>Python: 策略逻辑计算
        Python-->>Rust: 返回交易信号<br/>(BUY/SELL/None)
        Rust->>Rust: 订单撮合<br/>(市价/限价)
        Rust->>Rust: 更新持仓状态<br/>(position/cash/equity)
        Rust->>Python: 触发回调<br/>(on_order/on_trade)
        Rust->>Rust: 记录净值曲线
    end

    Rust->>Rust: 计算统计指标<br/>(收益/夏普/回撤等)
    Rust-->>Python: 返回回测结果
    Python->>Python: 性能分析<br/>(Analyzers)
    Python-->>API: 返回完整结果
    API-->>User: JSON响应<br/>(stats/equity/trades)

    Note over User,CSV: 结果展示阶段
    User->>API: GET /runs/{id}
    API-->>User: 返回回测结果
    User->>User: 可视化展示<br/>(净值曲线/统计指标)
```

### Key Components

-   **Frontend Layer**: Streamlit-based web UI for submitting backtests and visualizing results
-   **API Layer**: FastAPI RESTful API for managing backtest runs
-   **Python Layer**: Strategy definitions, data loading, analysis tools, and API wrappers
-   **Rust Engine**: High-performance core engine with PyO3 bindings for order matching, position management, and statistics
-   **Data Storage**: DuckDB for efficient K-line data storage and retrieval

### Performance Optimization Points

1. **Data Pre-extraction**: All bar data is extracted to Rust structures upfront, reducing Python↔Rust round-trips
2. **Batch Processing**: Configurable `batch_size` to reduce GIL contention when calling Python strategies
3. **Vectorized Operations**: Optimized indicator calculations using sliding window algorithms
4. **Direct Database Access**: Rust functions directly query DuckDB, bypassing Python overhead
5. **Temporary Table Strategy**: Ultra-fast batch inserts using temporary tables for data persistence

## Project Structure

-   `rust/engine_rust`: Rust engine (PyO3), indicators & stats
-   `python/pyrust_bt`: Python API/strategy/data/analyzers/optimizer
-   `python/pyrust_bt/multi_factor_analyzer.py`: Multi-factor evaluation system
-   `python/pyrust_bt/cs_factor_backtester.py`: Cross-sectional factor backtesting
-   `examples`: MVP, analyzers, grid search, performance tests
-   `examples/run_cs_momentum_sample.py`: Cross-sectional momentum factor demo
-   `examples/run_cs_quantile_portfolios.py`: Quantile portfolio trading simulation
-   `frontend`: Streamlit UI

## TODO / Roadmap

-   Engine/Matching
    -   Partial fills, order book, stop/take-profit, OCO, conditional orders
    -   Multi-asset/multi-timeframe alignment, calendar/timezone
    -   Liquidity/impact models
-   Data
    -   Parquet/Arrow zero-copy pipelines, columnar batching
    -   DataFeed abstraction (DB/object storage) & caching
-   Analysis/Reports
    -   Rich analyzers (group stats, drawdown visualization, trade distributions)
    -   Report export (PDF/HTML) & multi-run comparison
    -   Advanced factor analysis (industry/market cap neutralization, rolling quantiles)
-   Optimization/Parallelism
    -   Random/Bayesian search, cross-validation
    -   Multi-process/distributed runs (Ray/Celery/k8s Jobs)
-   Frontend/UX
    -   React + ECharts/Plotly (task mgmt, playback, filters, annotations)
    -   WebSocket live logs/progress/equity
-   Quality/Eng
    -   Unit/integration/regression tests, benchmarks
    -   CI (wheel build/artifacts), releases

## 🚀 Performance Highlights

-   Backtest speed: from 1,682 bars/s to **419,552 bars/s** (≈250×)
-   Dataset: 550k bars in ~1.3s
-   Memory: preallocated buffers to reduce reallocations
-   Batching: configurable `batch_size` to reduce GIL contention

## Community

Pull requests are welcome!

![Join](images/yzbjs1.png)

## License

MIT

## Disclaimer

This tool is for research and education only and does not constitute investment advice. You are solely responsible for your trading decisions and associated risks.
