# 情報收集系統設計

## 概述

本文件定義交易研究系統的數據收集層，涵蓋產業景氣指標、財務數據、法人動態與市場資金流向。此層是整個系統的基礎——投資神人的核心優勢之一就是對產業即時數據的掌握（如 SCFI 運價指數的週度追蹤、月度營收自結數的即時跟進）。

## 數據源分類

### 一、產業景氣領先指標

這是系統最關鍵的數據層。投資神人透過追蹤 SCFI 指數的週度變化，能在季報公布前就精準預估當季獲利。每個產業都有其對應的領先指標。

#### 產業指標配置表

| 產業 | 指標名稱 | 頻率 | 數據源 | 領先營收天數 |
|------|---------|------|--------|-------------|
| **航運（貨櫃）** | SCFI 上海出口集裝箱運價指數 | 週 | 上海航運交易所 sse.net.cn | ~30 天 |
| | BDI 波羅的海乾散貨指數 | 日 | Baltic Exchange | ~30 天 |
| | CCFI 中國出口集裝箱運價指數 | 週 | 上海航運交易所 | ~30 天 |
| | FBX Freightos 全球貨櫃運價指數 | 日 | Freightos | ~30 天 |
| **記憶體** | DRAM 合約價（DDR4/DDR5） | 月 | TrendForce / DRAMeXchange | ~60 天 |
| | NAND Flash 合約價 | 月 | TrendForce | ~60 天 |
| | DRAM 現貨價 | 日 | inSpectrum / DRAMeXchange | ~30 天 |
| | 模組廠庫存天數 | 月 | 研調機構報告 | ~90 天 |
| **AI 供應鏈** | NVIDIA GPU 出貨量/營收指引 | 季 | NVIDIA 財報/法說會 | ~90 天 |
| | CSP 資本支出計畫 | 季 | MSFT/GOOG/AMZN/META 財報 | ~180 天 |
| | CoWoS 先進封裝產能利用率 | 月 | 供應鏈訪查 | ~60 天 |
| | AI 伺服器 ODM 接單量 | 月 | 供應鏈訪查/法說會 | ~90 天 |
| **面板** | 大尺寸面板報價（32"/43"/55"/65"） | 週 | WitsView / TrendForce | ~30 天 |
| | 面板廠稼動率 | 月 | DSCC / TrendForce | ~60 天 |
| | TV 面板庫存週數 | 月 | 研調機構 | ~60 天 |
| **被動元件** | MLCC 標準品報價 | 月 | 通路商報價 | ~30 天 |
| | 交期（Lead Time） | 月 | 通路商/原廠 | ~60 天 |

#### 指標收集規格

```yaml
indicator_config:
  name: "SCFI"
  industry: "shipping_container"
  frequency: "weekly"
  source_url: "https://www.sse.net.cn/index/singleIndex?indexType=scfi"
  scraping_method: "html_parser"  # or "api", "pdf_parser", "manual"
  lag_days: 30  # 營收通常比運價遞延一個月
  history_start: "2003-01-01"
  fields:
    - name: "composite_index"
      type: "float"
      unit: "points"
    - name: "route_uswc"  # 美西航線
      type: "float"
      unit: "usd_per_feu"
    - name: "route_europe"  # 歐洲航線
      type: "float"
      unit: "usd_per_teu"
  alerts:
    - condition: "new_all_time_high"
      severity: "high"
    - condition: "week_over_week_change > 10%"
      severity: "medium"
    - condition: "below_20_week_ma"
      severity: "warning"
```

### 二、公司財務數據

投資神人重視的核心財務指標，用於計算本業 EPS 和判斷配息能力。

#### 必要財務欄位

| 欄位 | 來源 | 頻率 | 用途 |
|------|------|------|------|
| 營業收入 | 月營收公告 | 月 | 營收趨勢追蹤 |
| 營業利益 | 季報 | 季 | 計算本業獲利 |
| 營業利益率 | 季報 | 季 | 本業 EPS 過濾公式 |
| 稅後淨利 | 季報 | 季 | 實際 EPS 計算 |
| 稅後淨利率 | 季報 | 季 | 本業 EPS 過濾公式 |
| 每股盈餘 (EPS) | 季報/自結 | 季/月 | 核心估值依據 |
| 每股淨值 (BPS) | 季報 | 季 | P/B 參考（下限判斷） |
| 股本 | 季報 | 季 | 稀釋計算 |
| 累積虧損抵扣額 | 年報 | 年 | 所得稅回沖預估 |
| 現金股利 | 股東會決議 | 年 | 配息率計算 |
| 配息率 | 計算值 | 年 | P/E 倍數區間判斷 |

#### 數據源優先級

1. **證交所公開資訊觀測站** (mops.twse.com.tw) — 最權威
2. **公司自結數** — 最即時（月度 EPS 自結）
3. **CMoney / Goodinfo / TEJ** — 歷史數據彙整最方便
4. **公司法說會/年報** — 長約價格、資本支出計畫等質化資訊

### 三、市場交易數據

| 欄位 | 頻率 | 用途 |
|------|------|------|
| 每日股價 (OHLCV) | 日 | 本益比計算、技術面參考 |
| 日均量 (近一年) | 日 | 流動性篩選（排除日均量 < 200 張） |
| 三大法人買賣超 | 日 | 資金流向追蹤 |
| 融資融券餘額 | 日 | 散戶槓桿水位 |
| 市值排名 | 日 | MSCI/台灣50 成分股納入判斷 |
| 外資持股比例 | 週 | 外資參與度變化 |

### 四、總體經濟與資金環境

| 指標 | 頻率 | 用途 |
|------|------|------|
| 央行利率決策 | 不定期 | QE/QT 環境判斷 |
| M1B / M2 年增率 | 月 | 資金動能 |
| 國際油價 (Brent/WTI) | 日 | 成本面影響（航運燃油、石化） |
| 美元指數 (DXY) | 日 | 匯率影響 |
| MSCI 季度調整公告 | 季 | 成分股異動催化劑 |
| 台灣 50 成分股審核 | 季 | 指數納入催化劑 |

## 數據收集架構

### 自動化爬蟲流程

```
┌────────────┐     ┌──────────────┐     ┌────────────┐     ┌──────────┐
│  數據源     │────▶│  爬蟲/API    │────▶│  ETL 管線   │────▶│  資料庫   │
│            │     │  排程器      │     │  清洗/標準化 │     │          │
└────────────┘     └──────────────┘     └────────────┘     └──────────┘
                          │                                       │
                          ▼                                       ▼
                   ┌──────────────┐                        ┌──────────┐
                   │  異常偵測     │                        │  告警系統 │
                   │  數據品質檢查 │───────────────────────▶│  通知推送 │
                   └──────────────┘                        └──────────┘
```

### 排程策略

| 數據類型 | 排程頻率 | 排程時間 |
|---------|---------|---------|
| 產業指標（日頻） | 每日 | 每日 18:00 收盤後 |
| 產業指標（週頻） | 每週 | 週五 20:00 |
| 月營收 | 每月 | 每月 1~10 日檢查 |
| 季報數據 | 每季 | 季報公告日 +1 日 |
| 股價/籌碼 | 每日 | 每日 14:30 收盤後 |
| 總經數據 | 依公告 | 公告日當天 |

### 數據品質規則

1. **完整性檢查**：每日排程完成後驗證當日所有預期數據是否到齊
2. **合理性檢查**：數值是否在歷史合理範圍內（如 SCFI 不可能為負）
3. **一致性檢查**：交叉驗證不同數據源的同一指標
4. **時效性檢查**：數據延遲超過預期時觸發告警

## 產業配置範本

系統設計為模組化，新增產業時只需填寫配置檔：

```yaml
# industry_config/memory.yaml
industry:
  name: "記憶體"
  code: "memory"
  sub_sectors:
    - "DRAM"
    - "NAND Flash"
    - "Specialty DRAM"

  target_stocks:
    - { code: "2408", name: "南亞科", sub_sector: "DRAM" }
    - { code: "8150", name: "南茂", sub_sector: "DRAM" }
    - { code: "3450", name: "聯鈞", sub_sector: "DRAM" }
    - { code: "4967", name: "十銓", sub_sector: "DRAM" }
    - { code: "8299", name: "群聯", sub_sector: "NAND Flash" }
    - { code: "2337", name: "旺宏", sub_sector: "Specialty" }

  leading_indicators:
    - name: "DRAM_contract_price_DDR5_8Gb"
      source: "trendforce"
      frequency: "monthly"
      lag_to_revenue_days: 60

    - name: "NAND_contract_price_256Gb"
      source: "trendforce"
      frequency: "monthly"
      lag_to_revenue_days: 60

    - name: "DRAM_spot_price"
      source: "dramexchange"
      frequency: "daily"
      lag_to_revenue_days: 30

    - name: "module_inventory_weeks"
      source: "research_report"
      frequency: "monthly"
      lag_to_revenue_days: 90

  cost_factors:
    - name: "矽晶圓價格"
      impact: "negative"  # 漲價對獲利為負面
    - name: "電費"
      impact: "negative"

  seasonal_pattern:
    peak_quarters: ["Q3", "Q4"]  # 傳統旺季
    trough_quarters: ["Q1"]       # 傳統淡季

  supply_cycle:
    capacity_expansion_lead_time_months: 24  # 新廠從動工到量產
    current_phase: "upcycle"  # upcycle / downcycle / trough / peak
```

## 告警機制

### 告警等級

| 等級 | 觸發條件範例 | 通知方式 |
|------|-------------|---------|
| **Critical** | 產業指標創歷史新高/新低 | 即時推送 + 簡訊 |
| **High** | 週/月變動超過設定閾值 | 即時推送 |
| **Medium** | 公司自結 EPS 公布 | 每日摘要 |
| **Low** | 法人報告更新 | 週報彙整 |

### 關鍵告警場景（以航運為例）

- SCFI 連續 N 週創新高 → 觸發樂觀情境重估
- SCFI 淡季回檔幅度小於歷史平均 → 上調全年 EPS 估計
- 公司自結月 EPS 超出預估 → 調整季度估算
- 市值進入台灣前 50 大 → 追蹤 MSCI/0050 納入可能性
- 公司宣布配息政策 → 重估本益比適用區間
