# 估值回測系統設計

## 概述

本文件定義歷史估值回測系統的方法論與實作規格。這是整套系統最核心的差異化模組——投資神人的核心論點正是建立在 **20 年台股全市場回測** 的統計結果之上：「台灣股市 20 年來所有資金對所有股票交易出來的結果，本業獲利 + 正常配息的股票，本益比至少都有 6 倍。」

這不是主觀判斷，而是數據事實。本系統要做的就是將這個回測機制產品化、自動化、可跨產業套用。

## 回測方法論

### 原始方法論還原

投資神人的回測步驟：

```
Step 1: 取得台股所有上市櫃股票的年度財務數據（20 年）

Step 2: 篩選 — 排除日均量 < 200 張的「市場棄嬰」

Step 3: 計算每檔股票每年的「本業 EPS」
        使用本業 EPS 過濾公式（見 02-eps-estimation.md）

Step 4: 取得每檔股票每年的「股價最高點」

Step 5: 計算本業本益比 = 當年股價最高點 / 本業 EPS

Step 6: 標記配息資訊 — 配息率是否達 50% 以上

Step 7: 排名、分群、統計
        • 全市場本業 P/E 最低排名
        • 本業 EPS > 15 元以上高獲利股的 P/E 排名
        • 分群：正常配息 vs. 不配息/低配息
        • 分群：景氣循環 vs. 穩定成長

Step 8: 歸納統計規律
        → 本業獲利 + 正常配息 → P/E ≥ 6x
        → 本業 EPS > 15 + 正常配息 → P/E ≥ 9x
        → 此規律跨牛熊市、跨產業成立
```

### 系統化回測流程

```
┌──────────────────────────────────────────────────────────────┐
│                    回測引擎主流程                              │
│                                                              │
│  ┌─────────┐    ┌──────────┐    ┌──────────┐    ┌─────────┐ │
│  │ 數據載入  │───▶│ 篩選過濾  │───▶│ 指標計算  │───▶│ 統計分析 │ │
│  └─────────┘    └──────────┘    └──────────┘    └─────────┘ │
│       │                                              │       │
│       ▼                                              ▼       │
│  ┌─────────┐                                   ┌─────────┐  │
│  │ 資料驗證  │                                   │ 報告產出  │  │
│  └─────────┘                                   └─────────┘  │
└──────────────────────────────────────────────────────────────┘
```

## 回測參數規格

### 基本參數

```yaml
backtest_config:
  # 回測範圍
  start_year: 2001
  end_year: 2025  # 每年自動延伸
  market: "TWSE+OTC"  # 上市 + 上櫃

  # 篩選條件
  filters:
    min_daily_avg_volume: 200  # 張，排除市場棄嬰
    min_positive_eps: 0.01     # 排除虧損年度
    exclude_etf: true
    exclude_tdr: true
    exclude_special: true      # 排除 KY 股等特殊股

  # 本業 EPS 計算
  core_eps:
    tax_rate: 0.20
    method: "operating_margin_filter"  # 見 02-eps-estimation.md

  # 本益比計算
  pe_calculation:
    price_metric: "annual_high"    # 使用當年股價最高點
    eps_metric: "core_eps"         # 使用本業 EPS
    formula: "annual_high / core_eps"

  # 配息分類
  dividend_classification:
    normal_payout_threshold: 0.50   # 配息率 ≥ 50% 為正常配息
    low_payout_threshold: 0.30      # 配息率 30~50% 為低配息
    # 配息率 < 30% 或不配息為「不配息/爛配息」

  # 高獲利門檻
  high_eps_threshold: 15  # 本業 EPS > 15 元為高獲利股
```

### 進階分析參數

```yaml
advanced_analysis:
  # P/E vs P/B 比較分析
  valuation_comparison:
    metrics: ["PE", "PB"]
    measure: "coefficient_of_variation"  # 變異係數，越小越穩定
    # 投資神人的發現：同一群股票的 P/E 區間穩定（CV 小），
    # P/B 區間極度發散（CV 大），證明市場用 P/E 而非 P/B 定價

  # 牛熊市分群
  market_regime:
    bull_years: [2003, 2004, 2005, 2006, 2007, 2009, 2010, 2013, 2014, 2017, 2019, 2020, 2021]
    bear_years: [2001, 2002, 2008, 2011, 2015, 2018, 2022]
    # 驗證 P/E 下限是否在空頭年顯著下降

  # 產業分群
  industry_groups:
    cyclical: ["航運", "面板", "DRAM", "鋼鐵", "石化", "塑膠"]
    growth: ["半導體", "IC 設計", "軟體", "生技"]
    stable: ["電信", "食品", "金融"]
    one_time: ["口罩", "防疫"]
    # 驗證景氣循環股的 P/E 下限是否與其他類別有系統性差異

  # 異常值處理
  outlier_handling:
    method: "manual_review"
    # 投資神人的做法：不自動排除，而是逐一檢視異常值的原因
    # 例：2013 日勝生美河市案 → 有具體原因可排除
    # 例：2017 群創友達 4.5x → 不排除，視為「最差案例」
    document_reason: true  # 每個異常值都要記錄原因
```

## 回測輸出規格

### 年度報告表格

每年產出兩張核心表格：

#### 表格 A：當年本業本益比最低排名（前 20~25 名）

| 排名 | 股票代號 | 股票名稱 | 產業 | 本業EPS | 年度股價高點 | 本業P/E | 配息率 | P/B | 備註 |
|------|---------|---------|------|---------|------------|---------|-------|-----|------|
| 1 | XXXX | OOO | 面板 | 8.5 | 38.2 | 4.49 | 42% | 1.8 | 低配息 |
| 2 | XXXX | OOO | 航運 | 5.0 | 32.5 | 6.50 | 65% | 2.1 | 正常配息 |
| ... | ... | ... | ... | ... | ... | ... | ... | ... | ... |

#### 表格 B：當年本業 EPS > 15 元的股票 P/E 排名

| 排名 | 股票代號 | 股票名稱 | 產業 | 本業EPS | 年度股價高點 | 本業P/E | 配息率 | P/B | 備註 |
|------|---------|---------|------|---------|------------|---------|-------|-----|------|
| 1 | XXXX | OOO | 防疫 | 23.0 | 215 | 9.35 | 80% | 4.9 | 一年行情 |
| 2 | XXXX | OOO | 傳產 | 16.6 | 102 | 6.14 | 35% | 5.2 | 低配息 |
| ... | ... | ... | ... | ... | ... | ... | ... | ... | ... |

### 統計摘要

```json
{
  "year": 2020,
  "total_stocks_analyzed": 1650,
  "after_volume_filter": 1200,
  "profitable_stocks": 850,

  "table_a_summary": {
    "lowest_20_core_pe": {
      "min_pe": 4.12,
      "max_pe": 8.50,
      "median_pe": 6.25,
      "min_pe_with_normal_dividend": 6.02,
      "count_below_6x_with_normal_dividend": 0
    }
  },

  "table_b_summary": {
    "eps_above_15": {
      "count": 35,
      "min_pe": 9.26,
      "max_pe": 20.69,
      "median_pe": 14.5,
      "pe_cv": 0.28,
      "pb_cv": 1.05,
      "pe_range_ratio": 2.23,
      "pb_range_ratio": 10.76
    }
  },

  "key_finding": "本業獲利+正常配息的股票，最低P/E為6.02倍，符合≥6x規律"
}
```

## 核心統計驗證

### 驗證一：P/E 下限穩定性

**假說**：本業獲利 + 正常配息 (≥50%) 的股票，當年股價高點本益比 ≥ 6 倍。

```
驗證方法：
  FOR EACH year IN [2001..2025]:
    stocks = 篩選(日均量≥200, 本業EPS>0, 配息率≥50%)
    pe_list = [stock.annual_high / stock.core_eps FOR stock IN stocks]
    min_pe = MIN(pe_list)
    record(year, min_pe, stock_with_min_pe)

  統計：
    • 有多少年的 min_pe < 6？（預期：0 或極少數可解釋的例外）
    • min_pe 的分布（均值、中位數、標準差）
    • 例外案例的具體原因分析
```

### 驗證二：高獲利股 P/E 下限

**假說**：本業 EPS > 15 元 + 正常配息的股票，P/E ≥ 9 倍。

```
驗證方法：
  FOR EACH year IN [2001..2025]:
    stocks = 篩選(日均量≥200, 本業EPS>15, 配息率≥50%)
    pe_list = [stock.annual_high / stock.core_eps FOR stock IN stocks]
    min_pe = MIN(pe_list)
    record(year, min_pe, stock_with_min_pe)
```

### 驗證三：P/E vs P/B 穩定性比較

**假說**：同一群高獲利股票，P/E 的離散程度遠小於 P/B。

```
驗證方法：
  FOR EACH year IN [2001..2025]:
    stocks = 篩選(本業EPS>15)
    pe_cv = CV(pe_list)  # P/E 的變異係數
    pb_cv = CV(pb_list)  # P/B 的變異係數
    pe_range_ratio = MAX(pe) / MIN(pe)
    pb_range_ratio = MAX(pb) / MIN(pb)
    record(year, pe_cv, pb_cv, pe_range_ratio, pb_range_ratio)

  預期結果：
    • pe_cv 顯著 < pb_cv（每年皆然）
    • pe_range_ratio 通常 < 3x
    • pb_range_ratio 通常 > 5x
```

### 驗證四：牛熊市穩定性

**假說**：P/E 下限規律在空頭年（如 2008）依然成立。

```
驗證方法：
  比較 bull_years vs bear_years 的:
    • 平均 min_pe
    • 高獲利股平均 min_pe
    • 是否有系統性差異

  投資神人的發現：2008 金融海嘯年，
  市場給予本業獲利爆發股的 P/E 並沒有明顯比多頭年低。
```

### 驗證五：產業別 P/E 差異

**假說**：不同產業的 P/E 有差異，但下限規律依然成立。

```
驗證方法：
  FOR EACH industry IN [航運, 面板, DRAM, ...]:
    FOR EACH year WHERE industry HAS 獲利爆發股:
      record(year, industry, min_pe, max_pe, median_pe)

  分析：
    • 哪些產業市場資金最不愛？（面板：歷史 P/E 下限約 4.5x）
    • 哪些產業市場資金相對慷慨？（航運：歷史 P/E 6.5~9.8x）
    • QE 時代 vs 非 QE 時代，同產業 P/E 是否有系統性提升？
```

## 目標產業專項回測

### 記憶體產業回測

```yaml
memory_backtest:
  target_stocks:
    - "2408_南亞科"
    - "8150_南茂"
    - "2337_旺宏"
    - "4967_十銓"

  historical_boom_years:
    - year: 2006
      driver: "DRAM 供不應求"
    - year: 2010
      driver: "景氣復甦需求爆發"
    - year: 2017
      driver: "DRAM 寡佔格局形成"
    - year: 2021
      driver: "遠距需求 + 車用缺貨"
    - year: 2024
      driver: "AI/HBM 需求爆發"

  questions_to_answer:
    - "記憶體大賺年，市場給予的 P/E 區間為何？"
    - "相比航運/面板，市場對記憶體的估值偏好有何差異？"
    - "HBM 概念是否提升了市場對記憶體股的 P/E 容忍度？"
```

### AI 供應鏈回測

```yaml
ai_supply_chain_backtest:
  target_stocks:
    - "3661_世芯"
    - "2379_瑞昱"
    - "3034_聯詠"
    - "3037_欣興"
    - "6669_緯穎"
    - "2317_鴻海"

  historical_boom_years:
    - year: 2023
      driver: "ChatGPT 引爆 AI 投資潮"
    - year: 2024
      driver: "CSP 資本支出全面擴張"
    - year: 2025
      driver: "AI 推理需求爆發"

  questions_to_answer:
    - "AI 供應鏈的 P/E 估值與傳統景氣循環股有何差異？"
    - "市場是否因 AI 長期趨勢而給予更高 P/E？"
    - "上游 IC 設計 vs 中游零組件 vs 下游組裝，P/E 梯度如何？"
```

## 回測報告格式

### 20 年彙總報告

```
══════════════════════════════════════════════
     台股 20 年本業本益比回測彙總報告
     回測期間：2001 ~ 2025
══════════════════════════════════════════════

■ 核心發現

  1. 本業獲利 + 配息率≥50% 的股票
     → 當年股價高點本益比最低值：____倍 (____年 ____股票)
     → 20 年平均最低值：____倍
     → 例外次數：____次（均有可解釋原因）

  2. 本業 EPS > 15 元 + 配息率≥50% 的股票
     → 當年股價高點本益比最低值：____倍
     → 20 年平均最低值：____倍

  3. P/E vs P/B 離散度比較
     → P/E 平均變異係數：____
     → P/B 平均變異係數：____
     → P/E 平均區間比：____
     → P/B 平均區間比：____

  4. 牛熊市差異
     → 多頭年平均 min P/E：____
     → 空頭年平均 min P/E：____
     → 差異是否顯著：____

■ 各年度明細（見附表）

■ 產業別 P/E 分布（見附表）

■ 異常值清單與原因分析（見附表）
══════════════════════════════════════════════
```

## 回測自動化排程

| 任務 | 頻率 | 說明 |
|------|------|------|
| 年度全量回測 | 每年 4 月（年報全部公布後） | 完整重跑 20+ 年回測 |
| 即時年度回測 | 每日收盤後 | 更新當年度的即時 P/E 排名 |
| 產業專項回測 | 按需 / 季 | 針對特定產業深入回測 |
| 模型參數校準 | 每年 | 以最新一年數據校準篩選參數 |
