# EPS 估算引擎設計

## 概述

本文件定義本業 EPS 估算引擎的核心邏輯。投資神人的關鍵洞見在於：**市場資金追逐的是「本業獲利」而非帳面數字**。一家公司靠賣地賣樓灌出來的高 EPS，市場不會買單；反之，紮紮實實靠本業賺來的 EPS，市場資金至少給予 4 倍以上本益比。因此，精準區分本業 vs. 業外獲利，是整個估值系統的基石。

## 模組一：本業 EPS 過濾器

### 核心公式

投資神人使用的本業 EPS 判斷邏輯：

```
設：
  OPM = 營業利益率 (Operating Profit Margin)
  NPM = 稅後淨利率 (Net Profit Margin)
  TAX = 有效稅率（一般估算為 20%）
  EPS_actual = 當年實際 EPS

判斷：
  IF  OPM × (1 - TAX) >= NPM
  THEN  本業 EPS = EPS_actual
        （本業獲利 ≥ 稅後淨利，代表沒有業外灌水，甚至業外還是虧損的）

  IF  OPM × (1 - TAX) < NPM
  THEN  本業 EPS = EPS_actual × [OPM × (1 - TAX) / NPM]
        （稅後淨利有一部分來自業外，需按比例扣除）
```

### 實際案例

**案例 A — 達爾膚 (6523)，2020 年：**
- EPS_actual = 14.69
- OPM = 31%
- NPM = 86.5%（賣大陸子公司股權，一次性業外收益）
- 本業 EPS = 14.69 × (31% × 0.8 / 86.5%) = 14.69 × 0.2867 = **4.21 元**

**案例 B — 陽明 (2609)，2021 Q1：**
- EPS_actual ≈ 7 元（Q1 自結累計）
- OPM 極高（運價飆漲導致本業利潤極高）
- NPM 接近 OPM × (1-TAX)
- 本業 EPS ≈ 7 元（幾乎全部來自本業）

### 實作細節

```python
def calculate_core_eps(eps_actual: float,
                       operating_margin: float,
                       net_profit_margin: float,
                       tax_rate: float = 0.20) -> dict:
    """
    計算本業 EPS

    Returns:
        {
            "core_eps": float,          # 本業 EPS
            "non_operating_ratio": float, # 業外佔比
            "is_pure_operating": bool,   # 是否純本業
            "confidence": str           # high / medium / low
        }
    """
    operating_after_tax = operating_margin * (1 - tax_rate)

    if net_profit_margin <= 0:
        # 公司虧損，但若營業利益率為正，仍有本業獲利
        if operating_margin > 0:
            return {
                "core_eps": eps_actual * (operating_after_tax / net_profit_margin),
                "non_operating_ratio": 1 - (operating_after_tax / net_profit_margin),
                "is_pure_operating": False,
                "confidence": "medium"
            }
        else:
            return {
                "core_eps": eps_actual,
                "non_operating_ratio": 0,
                "is_pure_operating": True,
                "confidence": "high"
            }

    if operating_after_tax >= net_profit_margin:
        return {
            "core_eps": eps_actual,
            "non_operating_ratio": 0,
            "is_pure_operating": True,
            "confidence": "high"
        }
    else:
        core_ratio = operating_after_tax / net_profit_margin
        return {
            "core_eps": eps_actual * core_ratio,
            "non_operating_ratio": 1 - core_ratio,
            "is_pure_operating": False,
            "confidence": "high" if core_ratio > 0.8 else "medium"
        }
```

### 特殊處理規則

| 情境 | 處理方式 |
|------|---------|
| 營業利益含一次性項目（如日勝生美河市案） | 手動標記，從本業 EPS 中排除 |
| 虧損抵扣額回沖導致稅後淨利異常高 | 調整稅率參數或手動修正 |
| 業外收益來自轉投資（如長期穩定的子公司貢獻） | 可選擇部分納入本業計算 |
| 匯兌損益大幅波動 | 視產業特性決定是否納入 |

## 模組二：季度 EPS 推估模型

### 方法論

投資神人的推估邏輯是「從產業指標 → 推估營收 → 推估獲利」，核心步驟如下：

```
Step 1: 取得當季產業景氣指標均值
        例：Q2 SCFI 月均值 = (4月均值 + 5月均值 + 6月均值) / 3

Step 2: 透過歷史迴歸建立「指標 → 營收」映射
        月營收 ≈ f(上月產業指標, 長約佔比, 附加費)

Step 3: 透過歷史迴歸建立「營收 → EPS」映射
        季 EPS ≈ g(季營收, 營業成本結構, 稅率)

Step 4: 考慮額外因素調整
        + 長約換約漲價效果（五月生效，六月反映）
        - 燃油成本增加
        ± 虧損抵扣額回沖
        ± 現金增資股本膨脹
```

### 航運業推估範例

```yaml
estimation_model:
  industry: "shipping_container"
  stock: "2609_陽明"

  # Step 1: 運價指標 → 月營收映射
  revenue_model:
    type: "linear_regression"
    features:
      - name: "scfi_last_month_avg"  # 上月 SCFI 月均值（遞延一個月）
        coefficient: 0.35  # 待迴歸校準
      - name: "long_term_contract_ratio"
        value: 0.40  # 長約佔比 40%
      - name: "surcharges"  # 附加費（不在 SCFI 裡）
        estimation: "manual"
    intercept: 50  # 基礎營收（億）

  # Step 2: 季營收 → 季 EPS 映射
  eps_model:
    operating_cost_ratio: 0.65  # 營業成本佔營收比
    sga_quarterly: 15  # 管銷費用（億/季），相對固定
    shares_outstanding: 3484  # 在外流通股數（百萬股）
    tax_rate: 0.20

  # Step 3: 調整項目
  adjustments:
    long_term_contract_renewal:
      effective_month: "2021-05"
      revenue_impact_monthly: "+3.5億"  # 長約價翻倍帶來的增量
      eps_impact_annual: "+3.5~4.5元"
    fuel_cost_increase:
      yoy_change: "+30%"
      eps_impact_annual: "-1.5元"
    tax_loss_carryforward:
      available_amount: "500億"
      if_utilized_eps_boost: "+10~20%"
    capital_increase:
      new_shares: 300000  # 30 萬張
      dilution_rate: 0.083  # 8.3%
      pricing: "市價 × 0.92"
```

### 三情境推估框架

投資神人的估算核心是**永遠做三種情境**，且預設悲觀情境作為底線：

```
┌─────────────────────────────────────────────────────────────────┐
│                    三情境 EPS 推估框架                            │
├─────────────┬─────────────────┬─────────────────────────────────┤
│   情境       │ 產業指標假設     │ EPS 推估邏輯                     │
├─────────────┼─────────────────┼─────────────────────────────────┤
│ 悲觀         │ 旺季不漲，      │ Q2~Q3 指標在淡季區間盤整         │
│ (Bear)      │ Q4 直接腰斬     │ Q4 指標大跌 50%                  │
│             │                 │ 不使用虧損抵扣額                  │
├─────────────┼─────────────────┼─────────────────────────────────┤
│ 中立         │ 旺季溫和上漲，   │ Q2~Q3 指標較 Q1 +5~10%          │
│ (Base)      │ Q4 回檔 30%     │ Q4 指標較旺季 -30%               │
│             │                 │ 長約增益與油價增支互抵             │
├─────────────┼─────────────────┼─────────────────────────────────┤
│ 樂觀         │ 旺季持續創高，   │ Q2~Q3 指標持續突破               │
│ (Bull)      │ Q4 溫和回檔     │ Q4 指標僅較旺季 -15~20%          │
│             │                 │ 使用虧損抵扣額，所得稅回沖         │
└─────────────┴─────────────────┴─────────────────────────────────┘
```

### 推估結果結構

```json
{
  "stock": "2609",
  "year": 2021,
  "estimation_date": "2021-05-06",
  "scenarios": {
    "bear": {
      "q1_eps": 7.0,
      "q2_eps": 6.5,
      "q3_eps": 7.0,
      "q4_eps": 1.5,
      "annual_eps": 22.0,
      "annual_eps_range": [22.0, 23.5],
      "key_assumptions": [
        "SCFI Q2~Q3 均值 2600~2900（淡季區間盤整）",
        "SCFI Q4 腰斬至 1300~1400",
        "不使用虧損抵扣額"
      ]
    },
    "base": {
      "q1_eps": 7.0,
      "q2_eps": 7.75,
      "q3_eps": 9.0,
      "q4_eps": 4.25,
      "annual_eps": 28.0,
      "annual_eps_range": [27.0, 29.0],
      "key_assumptions": [
        "SCFI Q2~Q3 均值 2900~3000",
        "SCFI Q4 回檔至 2000 左右",
        "長約增益與油價增支互抵"
      ]
    },
    "bull": {
      "q1_eps": 7.0,
      "q2_eps": 8.25,
      "q3_eps": 10.0,
      "q4_eps": 6.5,
      "annual_eps": 31.75,
      "annual_eps_range": [30.0, 33.5],
      "key_assumptions": [
        "SCFI Q2~Q3 持續創高",
        "SCFI Q4 僅溫和回檔至 2500",
        "使用虧損抵扣額，EPS +10~20%"
      ]
    }
  },
  "adjustments_applied": {
    "capital_increase_dilution": -0.083,
    "tax_carryforward_boost": "not_applied_in_bear_base"
  },
  "confidence_level": "medium",
  "next_update_trigger": "Q1 季報公布 / SCFI 下週數據"
}
```

## 模組三：跨產業 EPS 推估範本

### 記憶體產業推估結構

```
DRAM 合約價 (月) ──遞延60天──▶ 月營收估算
                                    │
DRAM 現貨價 (日) ──領先指標──▶ 合約價走勢判斷
                                    │
庫存天數 (月) ──供需判斷──▶ 價格趨勢預測
                                    │
                              ┌─────▼─────┐
                              │ 季 EPS 推估 │
                              └─────┬─────┘
                                    │
                    ┌───────────┬────┴────┬───────────┐
                    ▼           ▼         ▼           ▼
              南亞科 EPS   旺宏 EPS   群聯 EPS   十銓 EPS
```

### AI 供應鏈推估結構

```
NVIDIA 營收指引 (季) ──▶ CoWoS 需求量推估
                              │
CSP 資本支出 (季) ──▶ AI 伺服器需求推估
                              │
                        ┌─────▼─────┐
                        │ 供應鏈拆解  │
                        └─────┬─────┘
                              │
          ┌──────────┬────────┼────────┬──────────┐
          ▼          ▼        ▼        ▼          ▼
       IC 載板    ABF 基板   散熱     電源      組裝代工
       (欣興)    (景碩)    (雙鴻)   (台達電)   (鴻海)
```

## 模組四：EPS 追蹤與修正

### 即時修正觸發條件

| 觸發事件 | 修正動作 |
|---------|---------|
| 月營收公布，偏離預估 >10% | 調整當季 EPS 估算 |
| 產業指標突破新高/新低 | 重估後續季度假設 |
| 公司自結月 EPS 公布 | 以實際值取代估算值 |
| 季報公布 | 全面校準模型參數 |
| 長約價格確認 | 調整長約效應估算 |
| 公司宣布資本結構事件 | 調整稀釋/增厚效果 |
| 油價/原物料大幅波動 | 調整成本面假設 |

### 預估準確度回饋迴路

```
預估 EPS ──vs──▶ 實際 EPS ──▶ 誤差分析 ──▶ 模型參數校準
                                   │
                                   ▼
                            找出誤差來源：
                            • 產業指標映射係數偏差？
                            • 成本結構假設錯誤？
                            • 漏算特定調整項目？
                            • 遞延天數估計不準？
```

每季結束後，系統自動比較預估值 vs. 實際值，計算 MAPE（平均絕對百分比誤差），並回饋調整模型參數，持續提升推估精準度。
