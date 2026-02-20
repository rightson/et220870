# 技術系統架構設計

## 概述

本文件定義 CARS（Cyclical Alpha Research System）的技術架構。系統設計目標是將投資神人的手動研究流程（Excel 回測、PTT 發文、手動追蹤運價）轉化為自動化、可擴展的研究平台。

## 系統架構總覽

```
┌─────────────────────────────────────────────────────────────────────┐
│                         使用者介面層                                  │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌───────────────────┐   │
│  │ Web 儀表板 │  │ 告警通知  │  │ 報告產出  │  │ 互動式回測工具     │   │
│  └──────────┘  └──────────┘  └──────────┘  └───────────────────┘   │
├─────────────────────────────────────────────────────────────────────┤
│                         應用服務層                                    │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌───────────────────┐   │
│  │ EPS 估算  │  │ 回測引擎  │  │ 情境分析  │  │ 信號追蹤           │   │
│  │   引擎    │  │          │  │ & 目標價  │  │ & 催化劑           │   │
│  └──────────┘  └──────────┘  └──────────┘  └───────────────────┘   │
├─────────────────────────────────────────────────────────────────────┤
│                         數據處理層                                    │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌───────────────────┐   │
│  │ ETL 管線  │  │ 數據清洗  │  │ 排程管理  │  │ 數據品質監控        │   │
│  └──────────┘  └──────────┘  └──────────┘  └───────────────────┘   │
├─────────────────────────────────────────────────────────────────────┤
│                         數據收集層                                    │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌───────────────────┐   │
│  │ 網頁爬蟲  │  │ API 介接  │  │ 檔案匯入  │  │ 手動輸入介面        │   │
│  └──────────┘  └──────────┘  └──────────┘  └───────────────────┘   │
├─────────────────────────────────────────────────────────────────────┤
│                         數據儲存層                                    │
│  ┌────────────────┐  ┌────────────────┐  ┌──────────────────────┐  │
│  │ PostgreSQL      │  │ Redis          │  │ Object Storage       │  │
│  │ (結構化數據)     │  │ (快取/即時)    │  │ (原始檔案/報告)       │  │
│  └────────────────┘  └────────────────┘  └──────────────────────┘  │
└─────────────────────────────────────────────────────────────────────┘
```

## 技術選型

### 核心技術堆疊

| 層級 | 技術選擇 | 選擇理由 |
|------|---------|---------|
| **語言** | Python 3.11+ | 金融數據分析生態系最豐富 |
| **Web 框架** | FastAPI | 高效能 API、自動文件產生 |
| **任務排程** | Celery + Redis | 分散式排程、支援定時任務 |
| **資料庫** | PostgreSQL 15 | 時序數據支援、JSON 欄位 |
| **快取** | Redis | 即時數據快取、任務佇列 |
| **前端** | React + Recharts | 互動式圖表、即時更新 |
| **爬蟲** | Scrapy + Playwright | 靜態/動態頁面兼容 |
| **數據分析** | pandas + numpy | 金融數據分析標準工具 |
| **容器化** | Docker + Docker Compose | 開發/部署一致性 |

### 外部服務整合

| 服務 | 用途 | 整合方式 |
|------|------|---------|
| 證交所 OpenAPI | 股價、營收、財報 | REST API |
| TrendForce | 記憶體/面板報價 | 訂閱 + 爬蟲 |
| 上海航運交易所 | SCFI 指數 | 爬蟲 |
| Goodinfo / CMoney | 歷史財務數據 | 爬蟲 |
| Telegram / LINE | 告警通知 | Bot API |

## 資料庫設計

### 核心資料表

```sql
-- 產業配置表
CREATE TABLE industries (
    id SERIAL PRIMARY KEY,
    code VARCHAR(50) UNIQUE NOT NULL,      -- e.g. "shipping_container"
    name VARCHAR(100) NOT NULL,             -- e.g. "航運（貨櫃）"
    config JSONB NOT NULL,                  -- 產業參數配置
    created_at TIMESTAMP DEFAULT NOW()
);

-- 股票基本資料
CREATE TABLE stocks (
    id SERIAL PRIMARY KEY,
    code VARCHAR(10) UNIQUE NOT NULL,       -- e.g. "2609"
    name VARCHAR(100) NOT NULL,             -- e.g. "陽明"
    industry_id INTEGER REFERENCES industries(id),
    market VARCHAR(10) NOT NULL,            -- "TWSE" or "OTC"
    shares_outstanding BIGINT,              -- 在外流通股數
    is_active BOOLEAN DEFAULT TRUE,
    metadata JSONB                          -- 官股比例等額外資訊
);

-- 產業景氣指標
CREATE TABLE industry_indicators (
    id SERIAL PRIMARY KEY,
    industry_id INTEGER REFERENCES industries(id),
    indicator_name VARCHAR(100) NOT NULL,    -- e.g. "SCFI"
    date DATE NOT NULL,
    value DECIMAL(15,4) NOT NULL,
    metadata JSONB,                          -- 子指標、航線別數據
    source VARCHAR(100),
    created_at TIMESTAMP DEFAULT NOW(),
    UNIQUE(industry_id, indicator_name, date)
);

-- 財務數據（季度）
CREATE TABLE financials_quarterly (
    id SERIAL PRIMARY KEY,
    stock_id INTEGER REFERENCES stocks(id),
    year INTEGER NOT NULL,
    quarter INTEGER NOT NULL,               -- 1~4
    revenue DECIMAL(15,2),                  -- 營收（百萬）
    operating_income DECIMAL(15,2),         -- 營業利益
    operating_margin DECIMAL(8,4),          -- 營業利益率
    net_income DECIMAL(15,2),               -- 稅後淨利
    net_profit_margin DECIMAL(8,4),         -- 稅後淨利率
    eps DECIMAL(10,4),                      -- 每股盈餘
    bps DECIMAL(10,4),                      -- 每股淨值
    source VARCHAR(50),                     -- "quarterly_report" / "self_reported"
    created_at TIMESTAMP DEFAULT NOW(),
    UNIQUE(stock_id, year, quarter)
);

-- 月營收
CREATE TABLE monthly_revenue (
    id SERIAL PRIMARY KEY,
    stock_id INTEGER REFERENCES stocks(id),
    year INTEGER NOT NULL,
    month INTEGER NOT NULL,
    revenue DECIMAL(15,2) NOT NULL,          -- 月營收（千元）
    yoy_change DECIMAL(8,4),                -- 年增率
    mom_change DECIMAL(8,4),                -- 月增率
    created_at TIMESTAMP DEFAULT NOW(),
    UNIQUE(stock_id, year, month)
);

-- 股價日資料
CREATE TABLE daily_prices (
    id SERIAL PRIMARY KEY,
    stock_id INTEGER REFERENCES stocks(id),
    date DATE NOT NULL,
    open_price DECIMAL(10,2),
    high_price DECIMAL(10,2),
    low_price DECIMAL(10,2),
    close_price DECIMAL(10,2),
    volume BIGINT,                          -- 成交量（張）
    turnover DECIMAL(15,2),                 -- 成交金額
    created_at TIMESTAMP DEFAULT NOW(),
    UNIQUE(stock_id, date)
);

-- 配息紀錄
CREATE TABLE dividends (
    id SERIAL PRIMARY KEY,
    stock_id INTEGER REFERENCES stocks(id),
    year INTEGER NOT NULL,                  -- 配息所屬年度
    cash_dividend DECIMAL(10,4),            -- 現金股利
    stock_dividend DECIMAL(10,4),           -- 股票股利
    payout_ratio DECIMAL(8,4),              -- 配息率
    ex_dividend_date DATE,
    created_at TIMESTAMP DEFAULT NOW(),
    UNIQUE(stock_id, year)
);

-- 本業 EPS 計算結果
CREATE TABLE core_eps (
    id SERIAL PRIMARY KEY,
    stock_id INTEGER REFERENCES stocks(id),
    year INTEGER NOT NULL,
    actual_eps DECIMAL(10,4),               -- 實際 EPS
    core_eps DECIMAL(10,4),                 -- 本業 EPS
    non_operating_ratio DECIMAL(8,4),       -- 業外佔比
    is_pure_operating BOOLEAN,
    confidence VARCHAR(10),                 -- high/medium/low
    calculation_detail JSONB,               -- 計算過程記錄
    created_at TIMESTAMP DEFAULT NOW(),
    UNIQUE(stock_id, year)
);

-- 回測結果
CREATE TABLE backtest_results (
    id SERIAL PRIMARY KEY,
    stock_id INTEGER REFERENCES stocks(id),
    year INTEGER NOT NULL,
    annual_high DECIMAL(10,2),              -- 年度股價最高點
    core_eps DECIMAL(10,4),                 -- 本業 EPS
    core_pe DECIMAL(10,4),                  -- 本業 P/E = annual_high / core_eps
    pb_at_high DECIMAL(10,4),               -- 股價高點的 P/B
    payout_ratio DECIMAL(8,4),              -- 配息率
    avg_daily_volume DECIMAL(15,2),         -- 近一年日均量
    dividend_class VARCHAR(20),             -- "normal"/"low"/"none"
    eps_class VARCHAR(20),                  -- "high_eps_15"/"normal"
    notes TEXT,                             -- 異常值備註
    created_at TIMESTAMP DEFAULT NOW(),
    UNIQUE(stock_id, year)
);

-- EPS 預估
CREATE TABLE eps_estimates (
    id SERIAL PRIMARY KEY,
    stock_id INTEGER REFERENCES stocks(id),
    year INTEGER NOT NULL,
    quarter INTEGER,                        -- NULL = 全年
    scenario VARCHAR(10) NOT NULL,          -- "bear"/"base"/"bull"
    estimated_eps DECIMAL(10,4),
    eps_range_low DECIMAL(10,4),
    eps_range_high DECIMAL(10,4),
    key_assumptions JSONB,
    estimation_date DATE NOT NULL,
    is_latest BOOLEAN DEFAULT TRUE,         -- 最新一版估計
    created_at TIMESTAMP DEFAULT NOW()
);

-- 目標價
CREATE TABLE target_prices (
    id SERIAL PRIMARY KEY,
    stock_id INTEGER REFERENCES stocks(id),
    scenario VARCHAR(10) NOT NULL,          -- "bear"/"base"/"bull"
    eps_used DECIMAL(10,4),
    pe_used DECIMAL(10,4),
    target_price DECIMAL(10,2),
    calculation_date DATE NOT NULL,
    rationale TEXT,
    is_latest BOOLEAN DEFAULT TRUE,
    created_at TIMESTAMP DEFAULT NOW()
);

-- 催化劑/事件追蹤
CREATE TABLE catalysts (
    id SERIAL PRIMARY KEY,
    stock_id INTEGER REFERENCES stocks(id),
    catalyst_type VARCHAR(50) NOT NULL,     -- "msci"/"tw50"/"analyst"/"dividend"
    event_date DATE,
    description TEXT,
    impact_on_eps VARCHAR(10),              -- "positive"/"negative"/"neutral"
    impact_on_pe VARCHAR(10),
    quantified_impact JSONB,
    status VARCHAR(20) DEFAULT 'pending',   -- "pending"/"confirmed"/"expired"
    created_at TIMESTAMP DEFAULT NOW()
);

-- 調整日誌
CREATE TABLE adjustment_logs (
    id SERIAL PRIMARY KEY,
    stock_id INTEGER REFERENCES stocks(id),
    adjustment_date DATE NOT NULL,
    trigger_event TEXT NOT NULL,
    previous_target JSONB,
    new_target JSONB,
    eps_change TEXT,
    pe_change TEXT,
    reasoning TEXT,
    created_at TIMESTAMP DEFAULT NOW()
);
```

### 索引策略

```sql
-- 高頻查詢索引
CREATE INDEX idx_indicators_industry_date ON industry_indicators(industry_id, date DESC);
CREATE INDEX idx_financials_stock_year ON financials_quarterly(stock_id, year, quarter);
CREATE INDEX idx_prices_stock_date ON daily_prices(stock_id, date DESC);
CREATE INDEX idx_backtest_year ON backtest_results(year, core_pe);
CREATE INDEX idx_estimates_stock_latest ON eps_estimates(stock_id, is_latest) WHERE is_latest = TRUE;
CREATE INDEX idx_target_latest ON target_prices(stock_id, is_latest) WHERE is_latest = TRUE;
```

## 核心模組詳細設計

### 模組一：數據收集引擎

```
data_collectors/
├── base.py              # 基礎爬蟲類別
├── twse_collector.py    # 證交所數據（股價、營收、財報）
├── scfi_collector.py    # SCFI 運價指數
├── trendforce_collector.py  # 記憶體/面板報價
├── goodinfo_collector.py    # 歷史財務數據
├── analyst_report_collector.py  # 法人報告追蹤
└── config/
    ├── shipping.yaml    # 航運產業數據源配置
    ├── memory.yaml      # 記憶體產業數據源配置
    └── ai_supply.yaml   # AI 供應鏈數據源配置
```

### 模組二：EPS 估算引擎

```
eps_engine/
├── core_eps_filter.py       # 本業 EPS 過濾器
├── quarterly_estimator.py   # 季度 EPS 推估
├── scenario_builder.py      # 三情境框架
├── adjustment_handler.py    # 動態調整處理
├── accuracy_tracker.py      # 預估準確度追蹤
└── templates/
    ├── shipping.py          # 航運 EPS 推估模板
    ├── memory.py            # 記憶體 EPS 推估模板
    └── ai_supply.py         # AI 供應鏈 EPS 推估模板
```

### 模組三：回測引擎

```
backtest_engine/
├── data_loader.py           # 歷史數據載入
├── filter_pipeline.py       # 篩選管線（日均量、本業EPS）
├── pe_calculator.py         # 本業本益比計算
├── statistical_analyzer.py  # 統計分析（PE 下限驗證）
├── pe_vs_pb_comparator.py   # PE vs PB 離散度比較
├── report_generator.py      # 年度報告產出
└── validators/
    ├── pe_floor_validator.py     # 驗證 PE≥6 規律
    ├── high_eps_validator.py     # 驗證 EPS>15 → PE≥9
    └── bull_bear_validator.py    # 牛熊市穩定性驗證
```

### 模組四：目標價計算

```
target_price/
├── pe_selector.py           # PE 倍數選擇決策樹
├── target_calculator.py     # 目標價計算引擎
├── dynamic_adjuster.py      # 動態調整機制
├── risk_manager.py          # 風險情境管理
└── position_advisor.py      # 持倉建議（可選）
```

### 模組五：信號追蹤

```
signal_tracker/
├── msci_tracker.py          # MSCI 成分股追蹤
├── tw50_tracker.py          # 台灣50 成分股追蹤
├── analyst_monitor.py       # 法人估值框架監控
├── liquidity_monitor.py     # 資金環境監控
├── event_calendar.py        # 跨產業事件日曆
├── signal_scorer.py         # 多因子信號評分
└── alert_dispatcher.py      # 告警分發
```

## API 設計

### 核心 API 端點

```
# 產業指標
GET  /api/v1/indicators/{industry}/latest         # 最新產業指標
GET  /api/v1/indicators/{industry}/history         # 歷史走勢

# EPS 估算
GET  /api/v1/eps/{stock_code}/estimate             # 當前 EPS 估算
POST /api/v1/eps/{stock_code}/scenario             # 自訂情境估算
GET  /api/v1/eps/{stock_code}/history              # 歷史估算 vs 實際

# 回測
GET  /api/v1/backtest/annual/{year}                # 年度回測結果
GET  /api/v1/backtest/summary                      # 20年彙總
GET  /api/v1/backtest/industry/{industry}           # 產業專項回測
POST /api/v1/backtest/custom                       # 自訂參數回測

# 目標價
GET  /api/v1/target/{stock_code}                   # 當前目標價
GET  /api/v1/target/{stock_code}/matrix            # 目標價矩陣
GET  /api/v1/target/{stock_code}/adjustments       # 調整日誌

# 信號
GET  /api/v1/signals/{stock_code}/score            # 信號評分
GET  /api/v1/signals/{stock_code}/catalysts        # 催化劑列表
GET  /api/v1/signals/calendar                      # 事件日曆

# 儀表板
GET  /api/v1/dashboard/{stock_code}                # 個股研究儀表板
GET  /api/v1/dashboard/watchlist                   # 觀察清單總覽
```

## 部署架構

```yaml
# docker-compose.yml 概念
services:
  # 應用服務
  api:
    build: ./api
    ports: ["8000:8000"]
    depends_on: [db, redis]

  # 排程任務
  worker:
    build: ./worker
    depends_on: [db, redis]

  scheduler:
    build: ./scheduler
    depends_on: [redis]

  # 數據收集
  collector:
    build: ./collector
    depends_on: [db]

  # 資料庫
  db:
    image: postgres:15
    volumes: ["pgdata:/var/lib/postgresql/data"]

  redis:
    image: redis:7

  # 前端
  frontend:
    build: ./frontend
    ports: ["3000:3000"]
```

## 開發路線圖

### Phase 1：基礎建設（MVP）
- [ ] 資料庫建置與基礎數據匯入
- [ ] 證交所數據爬蟲（股價、營收、財報）
- [ ] 本業 EPS 過濾器實作
- [ ] 20 年歷史回測引擎（核心）
- [ ] 基本 CLI 工具輸出回測報告

### Phase 2：估值引擎
- [ ] 產業指標爬蟲（SCFI、DRAM 報價）
- [ ] 季度 EPS 推估模型
- [ ] 三情境目標價計算
- [ ] 動態調整機制
- [ ] 調整日誌記錄

### Phase 3：信號系統
- [ ] MSCI / 台灣50 追蹤
- [ ] 法人報告監控
- [ ] 資金環境指標
- [ ] 多因子信號評分
- [ ] Telegram/LINE 告警推送

### Phase 4：使用者介面
- [ ] Web 儀表板（個股研究頁）
- [ ] 互動式回測工具
- [ ] 觀察清單管理
- [ ] 事件日曆視覺化
- [ ] 目標價矩陣圖表

### Phase 5：進階功能
- [ ] 跨產業自動比較
- [ ] 預估準確度追蹤與模型優化
- [ ] 歷史情境複盤工具
- [ ] 多使用者支援
- [ ] 行動端推送優化
