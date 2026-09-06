# 催化劑與信號追蹤系統

## 概述

投資神人的框架不僅是靜態估值，更包含對「催化劑」的敏銳追蹤——MSCI 納入、台灣 50 成分股調整、外資法人估值框架轉變、QE 資金環境等。這些催化劑不改變 EPS 本身，但會影響市場願意給予的 PE 倍數，從而大幅改變目標價。

本模組負責追蹤、評估並量化這些催化劑對估值的影響。

## 催化劑分類

### 第一類：指數成分股異動

這是投資神人明確提到的催化劑，也是最可量化的。

#### MSCI 成分股調整

```yaml
msci_tracking:
  review_schedule:
    - "2月審核（5月生效）"
    - "5月審核（8月生效）"  # 季度調整
    - "8月審核（11月生效）"  # 半年度調整
    - "11月審核（2月生效）"

  inclusion_criteria:
    # MSCI 台灣指數納入條件
    market_cap_threshold: "台股前 N 大市值"
    liquidity_threshold: "近 12 個月日均成交值"
    free_float_threshold: "自由流通量占比"

  monitoring:
    - metric: "目標股票市值排名變化"
      frequency: "daily"
      alert: "進入可能納入區間"
    - metric: "MSCI 預估調整名單（券商預測）"
      frequency: "每次審核前 2 週"
      alert: "目標股票出現在預測名單中"

  impact_quantification:
    # 納入 MSCI 後的被動資金買盤估算
    estimated_passive_buying: "自由流通市值 × MSCI 權重 × 全球追蹤資金規模"
    historical_price_impact: "+5~15%（納入公告後至生效日）"
    pe_uplift: "+0.5~1.0x"
```

#### 台灣 50 (0050) 成分股調整

```yaml
tw50_tracking:
  review_schedule: "每年 3月、6月、9月、12月（季度審核）"

  inclusion_criteria:
    # 台灣 50 成分股篩選：市值前 50 大
    primary: "上市股票市值排名前 50"
    buffer: "排名 46~55 為觀察名單"

  monitoring:
    - metric: "目標股票市值在台股排名"
      frequency: "daily"
      alert_threshold: "排名進入前 55"
    - metric: "與第 50 名的市值差距"
      frequency: "daily"

  impact_quantification:
    # 0050 ETF 規模約 2000~3000 億
    # 新納入成分股的權重 × ETF 規模 = 被動買盤
    estimated_passive_buying: "權重 × ETF_AUM"
    historical_price_impact: "+3~8%"

  # 投資神人的觀察：2021年5月海運雙雄市值排名
  # 長榮 N.18、陽明 N.34、萬海 N.44
  # 六月大概率長榮陽明進入台灣50
```

### 第二類：法人估值框架轉變

投資神人特別提到 JPM 開始用 PE 10x 來估值航運股，這是一個關鍵信號。

```yaml
analyst_framework_tracking:
  signals:
    - type: "估值方法論轉變"
      description: "外資從 P/B 估值轉為 P/E 估值"
      significance: "very_high"
      example: "JPM 4/19 長榮報告用 PE 10x 估值"
      impact: "當外資以 PE 框架追價，認錯回補買盤驚人"

    - type: "目標價大幅上調"
      description: "法人大幅上調目標價（>30%）"
      significance: "high"
      impact: "帶動其他法人跟進調整"

    - type: "EPS 預估上調"
      description: "法人 consensus EPS 上調"
      significance: "high"
      impact: "收斂估值差距，推升股價"

    - type: "新增覆蓋"
      description: "原本不覆蓋的大型外資開始出報告"
      significance: "medium"
      impact: "增加市場關注度和資金參與度"

  monitoring_sources:
    - "Bloomberg consensus"
    - "各大外資研究報告（JPM/GS/MS/CLSA/UBS...）"
    - "本土法人報告（凱基/元大/富邦/國泰...）"
    - "投顧研究報告"

  tracking_metrics:
    - name: "consensus_eps"
      description: "市場共識 EPS"
      alert: "共識 EPS 與我方估算差距 > 30%（代表市場尚未認知）"

    - name: "consensus_target_price"
      description: "市場共識目標價"
      alert: "共識目標價 < 我方悲觀目標價（嚴重低估）"

    - name: "valuation_method_used"
      description: "法人報告使用的估值方法"
      alert: "開始有法人從 P/B 轉用 P/E"

    - name: "coverage_count"
      description: "覆蓋該股票的法人家數"
      alert: "覆蓋家數突然增加"
```

### 第三類：資金環境

```yaml
liquidity_environment:
  # QE 環境對 PE 倍數的系統性影響
  qe_indicators:
    - name: "主要央行資產負債表規模"
      source: "Fed/ECB/BOJ"
      impact: "QE 環境下，同產業的 PE 倍數系統性提升"
      example: "面板股 2017(非QE) PE=4.5x → 2021(QE) PE=7~8x"

    - name: "M1B 年增率"
      source: "央行"
      threshold: "M1B YoY > 10% 為資金寬鬆"
      impact: "資金充沛時市場願意追逐更高 PE"

    - name: "台股融資餘額"
      source: "證交所"
      impact: "散戶槓桿水位，反映市場熱度"

    - name: "台股日均成交量"
      source: "證交所"
      threshold: "日均量 > 3000億為高熱度"
      impact: "高成交量環境有利於大市值股票估值提升"

  regime_classification:
    ultra_loose: "QE + M1B YoY > 15% + 日均量 > 4000億"
    loose: "QE + M1B YoY > 10%"
    neutral: "非 QE，利率平穩"
    tight: "QT + 升息循環"

  pe_adjustment_by_regime:
    ultra_loose: "+2~3x vs 歷史中位數"
    loose: "+1~2x"
    neutral: "使用歷史中位數"
    tight: "-1x"
```

### 第四類：產業供需結構信號

```yaml
supply_demand_signals:
  # 以航運為例
  shipping:
    supply_side:
      - name: "全球貨櫃船訂單量/現有運力比"
        significance: "very_high"
        alert: "訂單/運力比 < 10% = 供給緊縮信號"
      - name: "新船交付時程"
        significance: "high"
        alert: "大量新船交付前 6 個月 = 供給增加預警"
      - name: "船廠產能利用率"
        significance: "medium"

    demand_side:
      - name: "全球貿易量 (WTO 預測)"
        significance: "high"
      - name: "美國零售庫存/銷售比"
        significance: "medium"
        alert: "庫存低 = 補庫存需求"
      - name: "塞港/缺櫃嚴重程度"
        significance: "very_high"
        alert: "有效運力因塞港降低 = 推升運價"

    structural_change:
      - name: "高運價持續時間"
        significance: "very_high"
        # 投資神人的關鍵觀點：
        # 如果高運價不只一年行情而是持續兩年以上
        # PE 9~10x 將不是高點而是常態均價甚至底部支撐
        alert: "高運價持續 > 4 季 = 重新評估 PE 適用區間"

  # 以記憶體為例
  memory:
    supply_side:
      - name: "DRAM 產能擴張計畫（資本支出）"
        significance: "very_high"
      - name: "技術節點轉換進度（1α/1β）"
        significance: "high"
      - name: "寡佔程度（三星/SK海力士/美光市佔）"
        significance: "high"

    demand_side:
      - name: "伺服器 DRAM 含量（AI 推升）"
        significance: "very_high"
      - name: "HBM 需求量"
        significance: "very_high"
      - name: "手機/PC 出貨量"
        significance: "medium"

  # 以 AI 供應鏈為例
  ai_supply_chain:
    supply_side:
      - name: "CoWoS 產能（台積電）"
        significance: "very_high"
      - name: "ABF 載板產能"
        significance: "high"
      - name: "散熱/電源模組產能"
        significance: "medium"

    demand_side:
      - name: "CSP 資本支出計畫"
        significance: "very_high"
      - name: "NVIDIA 營收指引"
        significance: "very_high"
      - name: "AI 訓練 vs 推理需求比"
        significance: "high"
```

### 第五類：公司層面事件

```yaml
company_events:
  capital_structure:
    - type: "現金增資"
      impact_on_eps: "稀釋（股本膨脹）"
      impact_on_bps: "增加（募得現金）"
      monitoring: "股東會議案、承銷公告"
      example: "陽明 30 萬張現增，股本膨脹 9%，EPS 稀釋 8.3%"

    - type: "虧損抵扣額使用"
      impact_on_eps: "增厚（所得稅回沖）"
      monitoring: "季報所得稅費用項目"
      example: "陽明 500 億可抵扣額，使用後 EPS +10~20%"

    - type: "庫藏股"
      impact_on_eps: "增厚（股數減少）"
      monitoring: "董事會決議公告"

  dividend_policy:
    - type: "配息率公告"
      pe_impact: "配息率 ≥ 50% → PE 至少 6x；配息率 < 30% → PE 可能只有 4~5x"
      monitoring: "董事會配息提案、股東會決議"
      timing: "通常 3~4 月公告"

    - type: "官股配息壓力"
      description: "高官股比例的公司，大賺年可能被要求高配息繳庫"
      monitoring: "政府政策風向、財政部態度"
      example: "陽明高官股比例，預期高配息率"

  operational:
    - type: "長約換約"
      impact_on_eps: "增厚（長約價漲價）"
      timing: "通常 5 月生效，6 月反映在營收"
      example: "陽明長約價翻倍，年化 EPS +3.5~4.5 元"

    - type: "重大合約/訂單"
      monitoring: "重大訊息公告"

    - type: "經營層異動"
      monitoring: "人事公告"
      alert: "可能影響配息政策或營運策略"
```

## 信號評分系統

### 多因子信號評分

```
信號強度 = Σ (各催化劑權重 × 催化劑評分)

┌──────────────────────┬────────┬────────────────────────┐
│ 催化劑                │ 權重    │ 評分 (0~10)             │
├──────────────────────┼────────┼────────────────────────┤
│ 產業指標趨勢          │ 30%    │ 創高=10, 盤整=5, 下跌=2 │
│ EPS 上修動能          │ 25%    │ 上修>20%=10, 持平=5     │
│ 配息政策確認          │ 15%    │ ≥60%=10, ≥50%=7, <30%=2│
│ 指數納入催化          │ 10%    │ 確定納入=10, 可能=5     │
│ 法人估值轉變          │ 10%    │ 多家轉用PE=10, 少數=5   │
│ 資金環境              │ 10%    │ 超寬鬆=10, 中性=5      │
├──────────────────────┼────────┼────────────────────────┤
│ 信號總分              │ 100%   │ 0~10                   │
└──────────────────────┴────────┴────────────────────────┘

信號解讀：
  8~10: 多重催化劑共振，目標價有上調可能，可考慮加碼
  5~7:  基本面穩健，維持目標價
  3~4:  部分風險浮現，需密切監控
  0~2:  多重利空共振，目標價可能需下修
```

## 信號儀表板

### 即時監控面板

```
╔══════════════════════════════════════════════════════╗
║  CARS 信號儀表板 — 2609 陽明                          ║
╠══════════════════════════════════════════════════════╣
║                                                      ║
║  [產業指標]  SCFI: 3,100 ▲ 創歷史新高    ██████████  ║
║  [EPS 動能]  Q2 EPS 估: 7.5 ▲ 上修中     ████████░░  ║
║  [配息預期]  預估配息率: 65% ✓            ████████░░  ║
║  [指數催化]  市值排名: #34 → 0050 候選    ██████░░░░  ║
║  [法人動態]  JPM 首次用 PE 10x 估值       ██████░░░░  ║
║  [資金環境]  QE 持續中，M1B YoY +18%      ██████████  ║
║                                                      ║
║  綜合信號: 8.5 / 10  [Strong Buy Signal]             ║
║                                                      ║
║  目標價: 保守 162 ─── 中立 196 ─── 樂觀 288          ║
║  現價: 84 │ 上行空間: +93% ~ +243%                   ║
║                                                      ║
║  下次關鍵事件:                                        ║
║  • 5/14 Q1 季報公布（確認虧損抵扣額使用？）            ║
║  • 5/28 MSCI 季度調整公告                             ║
║  • 6月初 SCFI 旺季走勢確認                            ║
╚══════════════════════════════════════════════════════╝
```

## 跨產業事件日曆

系統應維護一個覆蓋所有追蹤產業的事件日曆：

| 日期 | 事件 | 影響產業 | 重要度 |
|------|------|---------|-------|
| 每月 1~10 日 | 上月營收公告 | 全產業 | ★★★ |
| 每季 | 季報公布 | 全產業 | ★★★★ |
| 3/6/9/12 月 | 台灣 50 成分股審核 | 大市值股 | ★★★ |
| 2/5/8/11 月 | MSCI 季度調整 | 大市值股 | ★★★ |
| 每週五 | SCFI 公布 | 航運 | ★★★★ |
| 每月初 | DRAM 合約價公布 | 記憶體 | ★★★★ |
| 每季 | NVIDIA 財報 | AI 供應鏈 | ★★★★★ |
| 每季 | CSP 財報（資本支出指引） | AI 供應鏈 | ★★★★ |
| 3~4 月 | 配息公告 | 全產業 | ★★★★ |
| 5~6 月 | 股東會 | 全產業 | ★★★ |
