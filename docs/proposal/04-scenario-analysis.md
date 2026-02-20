# 情境分析與目標價計算模組

## 概述

本文件定義情境分析框架與目標價計算邏輯。投資神人的目標價公式極其簡潔但有力：

> **目標價 = 本業 EPS 估值 × 回測支撐的最低本益比倍數**

關鍵在於：本益比倍數不是主觀拍腦袋的數字，而是 20 年全市場回測得出的統計下限。這讓目標價的計算有了堅實的數據基礎。

## 目標價計算引擎

### 核心公式

```
目標價 = Core_EPS × PE_Multiple

其中：
  Core_EPS    = 本業 EPS 估算值（見 02-eps-estimation.md）
  PE_Multiple = 回測支撐的本益比倍數（見 03-valuation-backtesting.md）
```

### PE 倍數選擇邏輯

```
┌─────────────────────────────────────────────────────────────┐
│                PE 倍數選擇決策樹                              │
│                                                              │
│  配息率 ≥ 50%？                                              │
│    ├── YES ──▶ 本業 EPS > 15？                               │
│    │            ├── YES ──▶ PE 下限 = 9x（高獲利+正常配息）    │
│    │            └── NO  ──▶ PE 下限 = 6x（正常配息）           │
│    └── NO  ──▶ 配息率 ≥ 30%？                                │
│                 ├── YES ──▶ PE 下限 = 4.5~5x（低配息）        │
│                 └── NO  ──▶ PE 下限 = 4x（不配息/極低配息）    │
│                                                              │
│  QE 環境調整：                                                │
│    若處於 QE 寬鬆環境 ──▶ PE 下限可上調 1~2x                   │
│    （依據：2021 面板股 PE 從歷史 4.5x 提升至 7~8x）            │
│                                                              │
│  產業歷史修正：                                                │
│    取該產業過去大賺年的歷史 PE 區間作為參考                      │
│    若產業歷史 PE 下限高於通用下限，採用產業歷史值                  │
└─────────────────────────────────────────────────────────────┘
```

### 各產業歷史 PE 參考區間

| 產業 | 大賺年歷史 PE 區間 | 最低案例 | 建議 PE 下限 |
|------|-------------------|---------|-------------|
| 航運（貨櫃） | 6.5 ~ 9.8x | 2020 陽明 7.1x | 6.5x |
| 面板 | 4.5 ~ 8x | 2017 群創 4.5x | 4.5x（非QE）/ 7x（QE） |
| 記憶體 | 5 ~ 12x | 待回測確認 | 待回測 |
| AI 供應鏈 | 10 ~ 25x | 待回測確認 | 待回測 |
| 口罩/防疫 | 9 ~ 12x | 2020 恆大 9.3x | 9x |
| 鋼鐵 | 5 ~ 8x | 待回測確認 | 待回測 |

## 三情境目標價框架

### 框架定義

```
                    EPS 軸（本業獲利估算）
                         │
         悲觀EPS         │         樂觀EPS
           ◄─────────────┼─────────────►
                         │
    ┌────────────────────┼────────────────────┐
    │                    │                    │
    │   悲觀EPS×低PE     │   悲觀EPS×高PE     │  PE 軸
    │   (最保守目標)      │                    │ (市場估值
    │                    │                    │  意願)
    ├────────────────────┼────────────────────┤
    │                    │                    │
    │   樂觀EPS×低PE     │   樂觀EPS×高PE     │
    │                    │   (最樂觀目標)      │
    │                    │                    │
    └────────────────────┼────────────────────┘
                         │
```

### 計算範例：陽明 (2609) 2021

```yaml
target_price_calculation:
  stock: "2609_陽明"
  date: "2021-05-06"

  # EPS 情境
  eps_scenarios:
    bear: { annual_eps: 22.0, range: [22.0, 23.5] }
    base: { annual_eps: 28.0, range: [27.0, 29.0] }
    bull: { annual_eps: 31.75, range: [30.0, 33.5] }

  # PE 倍數情境
  pe_scenarios:
    # 下限：20年回測，正常配息股至少6x
    floor: 6.0
    # 產業歷史：航運大賺年 6.5~9.8x
    industry_low: 6.5
    industry_high: 9.8
    # QE 加成：參考面板股 2017→2021 的 PE 提升
    qe_adjusted_low: 7.0
    qe_adjusted_high: 10.0

  # 配息假設
  dividend_assumption:
    payout_ratio: "60~70%"  # 陽明過去大賺年的配息紀錄
    supports_pe_floor: 6.0  # 正常配息 → 至少 6x

  # 目標價矩陣
  target_price_matrix:
    #                   PE=6x    PE=7x    PE=8x    PE=9x
    bear_eps_22:      [ 132,     154,     176,     198   ]
    base_eps_28:      [ 168,     196,     224,     252   ]
    bull_eps_32:      [ 192,     224,     256,     288   ]

  # 核心結論
  primary_target:
    conservative: 162  # 悲觀EPS下限 × PE下限 = 27×6
    base:         196  # 中立EPS × 產業歷史PE低端 = 28×7
    optimistic:   288  # 樂觀EPS上限 × 產業歷史PE高端 = 32×9

  # 投資神人原文結論
  author_stated_target: "至少 162~174"
  author_logic: "中立EPS 27~29 × 回測支撐最低PE 6x"
```

## 動態調整機制

### 觸發條件與調整規則

目標價不是靜態數字，而是隨最新資訊動態更新的：

| 觸發事件 | 調整方向 | 具體動作 |
|---------|---------|---------|
| 產業指標持續創高 | ↑ EPS | 上調後續季度 EPS 假設 |
| 產業指標淡季回檔幅度小於預期 | ↑ EPS | 縮小悲觀情境的衰退假設 |
| 公司自結 EPS 超出預估 | ↑ EPS | 以實際值校準，上調後續估算 |
| 公司宣布高配息 | ↑ PE | PE 倍數可從 6x 上調至 7x+ |
| 公司宣布不配息/低配息 | ↓ PE | PE 倍數可能降至 4~5x |
| 確認使用虧損抵扣額 | ↑ EPS | EPS 估算 +10~20% |
| 確認現增稀釋 | ↓ EPS | EPS 估算 -8~9% |
| 納入 MSCI/台灣50 | ↑ PE | 新增法人資金推升估值 |
| QE 環境確認/加強 | ↑ PE | 參考同期其他產業 PE 膨脹幅度 |
| 高運價持續超過一年（非一年行情） | ↑↑ PE | PE 可能從 6~9x 提升至 10x+ |
| 外資法人開始用 PE 框架追價 | ↑ PE | 認錯回補的買盤推升 PE |

### 動態調整日誌

系統應記錄每次目標價調整的完整日誌：

```json
{
  "adjustment_log": [
    {
      "date": "2021-05-06",
      "trigger": "SCFI 連續兩週創歷史新高至 3100",
      "previous_target": { "bear": 132, "base": 168, "bull": 198 },
      "new_target": { "bear": 162, "base": 196, "bull": 252 },
      "eps_change": "上調 Q2~Q3 EPS 假設",
      "pe_change": "維持不變",
      "reasoning": "運價淡季回檔幅度遠低於預期，止跌回升又創高..."
    },
    {
      "date": "2021-05-15",
      "trigger": "Q1 季報公布，確認使用虧損抵扣額",
      "eps_change": "全年 EPS +15%",
      "pe_change": "維持不變",
      "reasoning": "所得稅回沖使 EPS 增厚"
    }
  ]
}
```

## 風險管理框架

### 下行風險情境

投資神人雖然極度看多，但其估值框架的核心是「保守的 PE 下限」。系統也需要定義下行風險：

```yaml
downside_risks:
  # 估值崩壞風險
  - name: "配息政策反轉"
    trigger: "公司宣布不配息或配息率<30%"
    impact: "PE 從 6x 降至 4~5x"
    target_revision: "-15~30%"

  # 獲利崩壞風險
  - name: "產業指標崩盤"
    trigger: "運價/報價較高點腰斬"
    impact: "下調後續季度 EPS 至悲觀情境以下"
    target_revision: "重估 EPS，可能 -30~50%"

  # 系統性風險
  - name: "大盤系統性崩盤"
    trigger: "加權指數跌破年線 20%+"
    impact: "歷史回測顯示 PE 下限仍成立，但高點 PE 會壓縮"
    target_revision: "維持 PE 下限估值，但降低樂觀 PE 假設"

  # 結構性風險
  - name: "產業結構永久改變"
    trigger: "新產能大量投產、替代品出現"
    impact: "市場重新定價為衰退股"
    target_revision: "需重新評估 PE 適用區間"
```

### 止損與部位管理建議

```
投資神人的持倉特徵分析：
  • 極度集中（23,590 張單一標的）
  • 多帳戶分散（7 個帳戶）
  • 波段持有為主 + 短線價差操作
  • 部位只增不減（直到目標價達成）

系統建議的部位管理框架：
  ┌─────────────────────────────────────────────┐
  │  進場條件：                                    │
  │  • 本業 EPS 估算完成                           │
  │  • 當前股價 < 悲觀目標價的 60%                   │
  │  • 至少有一項產業指標確認景氣上行                  │
  │                                               │
  │  加碼條件：                                    │
  │  • 新數據上調 EPS 估算                          │
  │  • 股價仍低於中立目標價                          │
  │  • 確認配息政策正常                              │
  │                                               │
  │  警戒條件：                                    │
  │  • 產業指標連續 N 週下滑                         │
  │  • 公司下調營運展望                              │
  │  • 配息政策低於預期                              │
  │                                               │
  │  出場條件：                                    │
  │  • 股價觸及樂觀目標價                            │
  │  • 產業基本面反轉（非短期回檔）                    │
  │  • 出現更高預期回報的替代標的                      │
  └─────────────────────────────────────────────┘
```

## 跨產業套用範例

### 記憶體產業目標價計算

```yaml
target_price_example:
  stock: "2408_南亞科"
  year: 2025  # 假設範例

  eps_scenarios:
    bear: { annual_eps: 12.0, assumption: "DRAM 合約價 Q3 起下跌 20%" }
    base: { annual_eps: 16.0, assumption: "DRAM 合約價全年持穩" }
    bull: { annual_eps: 22.0, assumption: "HBM 需求推動 DRAM 合約價再漲 15%" }

  pe_reference:
    historical_memory_boom_pe: "5~12x"  # 待回測確認
    dividend_payout_expected: "50~60%"
    pe_floor: 6.0
    pe_ceiling: 10.0

  target_price_matrix:
    bear_pe6: 72     # 12 × 6
    base_pe7: 112    # 16 × 7
    bull_pe9: 198    # 22 × 9

  primary_target: "至少 72（悲觀），中立 112"
```

### AI 供應鏈目標價計算

```yaml
target_price_example:
  stock: "3037_欣興"
  year: 2025  # 假設範例

  eps_scenarios:
    bear: { annual_eps: 18.0, assumption: "AI 伺服器出貨量低於指引 10%" }
    base: { annual_eps: 24.0, assumption: "AI 伺服器出貨量符合指引" }
    bull: { annual_eps: 30.0, assumption: "AI 推理需求超預期，載板供不應求" }

  pe_reference:
    # AI 供應鏈不同於傳統景氣循環股
    # 市場可能給予更高 PE（成長股溢價）
    historical_pe: "10~20x"  # 待回測確認
    growth_premium: true
    pe_floor: 9.0
    pe_ceiling: 15.0

  target_price_matrix:
    bear_pe9: 162    # 18 × 9
    base_pe12: 288   # 24 × 12
    bull_pe15: 450   # 30 × 15

  note: "AI 供應鏈可能不適用傳統景氣循環股的低 PE 框架，需回測驗證"
```

## 回溯驗證機制

### 事後驗證

每年結束後，回溯驗證目標價計算的準確度：

```
FOR EACH 當年有設定目標價的股票:
  actual_high = 實際年度股價最高點
  target_bear = 悲觀目標價
  target_base = 中立目標價
  target_bull = 樂觀目標價

  評估：
  • actual_high vs target_bear → 是否至少觸及悲觀目標？
  • actual_high vs target_base → 中立目標的準確度？
  • 哪個情境最接近實際結果？
  • 目標價偏差的主因：EPS 估算偏差 or PE 倍數偏差？
```

這個回溯驗證的結果將用於持續優化 EPS 估算模型和 PE 倍數選擇邏輯。
