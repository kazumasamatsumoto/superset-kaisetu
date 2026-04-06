# superset-kenshou

Apache Superset Mixed Timeseries Chart の調査・解説リポジトリです。

---

## 📚 アーカイブ

すべての調査レポートは日付別のアーカイブディレクトリに整理されています。

### 📂 [archive/2026-04-02-mixed-chart-investigation/](./archive/2026-04-02-mixed-chart-investigation/)

**Mixed Chart 包括的調査**（最新）

**調査内容**:
- ✅ カスタマイズプロパティ完全ガイド
- ✅ シリーズタイプ設定方法
- ✅ X軸ラベル表示問題の解決
- ✅ **ツールチップ0値表示問題の解決**（重要）
- ✅ **棒グラフの幅の自動調整とカスタマイズ方法**
- ✅ **Y軸Boundsとauto scaleの両立方法**（重要）

**重要な発見**:
- **Rich Tooltip ON では0が必ず表示される**
- **Stack series OFF でも実際の0は消えない**
- **SupersetのUIに棒グラフ幅の設定は存在しない**
- **固定値のmin/maxはscale: trueを無効化する**

**レポート**:
1. `MIXED_CHART_CUSTOMIZE_PROPERTIES.md` - カスタマイズプロパティ完全ガイド
2. `MIXED_CHART_SERIES_TYPE.md` - シリーズタイプ設定ガイド
3. `MIXED_CHART_XAXIS_LABEL_ISSUE.md` - X軸ラベル表示問題
4. `MIXED_CHART_ZERO_VALUES_IN_TOOLTIP.md` - ツールチップ0値表示問題 ⭐
5. `TOOLTIP_COMPREHENSIVE_GUIDE.md` - ツールチップ機能完全ガイド
6. `STACK_MODE_EXPLAINED.md` - スタックモード詳細解説
7. `MIXED_CHART_BAR_WIDTH.md` - 棒グラフの幅に関する調査 ⭐
8. `MIXED_CHART_YAXIS_BOUNDS_AUTOSCALE.md` - Y軸Bounds auto scale問題 ⭐

[詳細はこちら](./archive/2026-04-02-mixed-chart-investigation/README.md)

---

### 📂 [archive/2026-04-02-date-format-bug/](./archive/2026-04-02-date-format-bug/)

**日付フォーマット不具合調査**

**問題**: Query Aが空の場合、X軸が `1609459200000` のような数値表示

**修正**: `transformProps.ts:235-236` の2行を修正

**レポート**:
1. `DATE_FORMAT_BUG_COMPREHENSIVE.md` - 包括的ガイド
2. `DATE_FORMAT_BUG_INDEX.md` - インデックス
3. `DATE_FORMAT_BUG_OVERVIEW.md` - 概要
4. `DATE_FORMAT_BUG_TECHNICAL.md` - 技術詳細
5. `DATE_FORMAT_BUG_FIX_GUIDE.md` - 修正ガイド

[詳細はこちら](./archive/2026-04-02-date-format-bug/README.md)

---

### 📂 [archive/2026-04-01-tooltip-investigation/](./archive/2026-04-01-tooltip-investigation/)

**ツールチップ初期調査**

**レポート**:
1. `TOOLTIP_MODE_REPORT.md` - axisモード vs itemモード
2. その他のツールチップ関連ドキュメント

[詳細はこちら](./archive/2026-04-01-tooltip-investigation/README.md)

---

## 🔥 よくある問題と解決策

### 問題1: ツールチップに大量の0が表示される

**原因**: Rich Tooltip（axisモード）では、`typeof 0 === 'number'` が true のため、0も表示される

**解決策**:
1. 🥇 **Rich Tooltip OFF にする**（1分で完了）
   - Chart Options → Tooltip → Rich tooltip のチェックを外す
2. 🥈 SQLで0を除外（1時間）
   - `WHERE metric_value != 0`
3. 🥉 tooltipHideZeroValues機能の実装（2-3日）

**詳細**: [archive/2026-04-02-mixed-chart-investigation/MIXED_CHART_ZERO_VALUES_IN_TOOLTIP.md](./archive/2026-04-02-mixed-chart-investigation/MIXED_CHART_ZERO_VALUES_IN_TOOLTIP.md)

---

### 問題2: X軸の最終日ラベルが表示されない

**例**: 7/1〜7/5のデータがあるのに、7/5が表示されない

**原因**: EChartsの自動ラベル間引き、X Axis Bounds、Truncate X Axis

**解決策**:
1. X Axis Time Format: `%-m/%-d`（短いフォーマット）
2. X Axis Label Rotation: 45度
3. チャート幅を広げる

**詳細**: [archive/2026-04-02-mixed-chart-investigation/MIXED_CHART_XAXIS_LABEL_ISSUE.md](./archive/2026-04-02-mixed-chart-investigation/MIXED_CHART_XAXIS_LABEL_ISSUE.md)

---

### 問題3: DATAタブで「N/A」なのにツールチップで「0」と表示される

**原因**: Stack series が ON の場合、N/A（null）が自動的に0に変換される

**解決策**:
1. Stack series を OFF にする（N/Aは表示されなくなる）
2. Rich Tooltip を OFF にする（itemモードに切り替え）

**注意**: 実際に0のデータが存在する場合、Stack series OFF にしても0は消えません

**詳細**: [archive/2026-04-02-mixed-chart-investigation/MIXED_CHART_ZERO_VALUES_IN_TOOLTIP.md](./archive/2026-04-02-mixed-chart-investigation/MIXED_CHART_ZERO_VALUES_IN_TOOLTIP.md)

---

### 問題4: X軸の日時がタイムスタンプ数値で表示される

**例**: `1609459200000` のような数値表示

**原因**: Query Aが空の場合、xAxisDataType が正しく取得されない

**修正**: `transformProps.ts:235-236` の2行を修正（既存変数の活用）

**詳細**: [archive/2026-04-02-date-format-bug/DATE_FORMAT_BUG_COMPREHENSIVE.md](./archive/2026-04-02-date-format-bug/DATE_FORMAT_BUG_COMPREHENSIVE.md)

---

### 問題5: 棒グラフが太くなったり細くなったり自動で変化する

**原因**: EChartsのデフォルト自動計算アルゴリズムによる（チャート幅、データポイント数、系列数に基づく）

**解決策**:
1. 🥇 **チャート幅を調整**（即座に対応可能）
   - Dashboard編集 → チャートのサイズ変更
2. 🥈 barWidth設定の実装（1-2日）
   - types.ts、controlPanel.tsx、transformers.ts、transformProps.tsを修正
3. 🥉 コミュニティ貢献（2-3週間）

**現状**: SupersetのUIにカスタマイズ設定なし

**詳細**: [archive/2026-04-02-mixed-chart-investigation/MIXED_CHART_BAR_WIDTH.md](./archive/2026-04-02-mixed-chart-investigation/MIXED_CHART_BAR_WIDTH.md)

---

### 問題6: Y軸Boundsを0-100に設定するとauto scaleが効かない

**例**: データが20-30の範囲なのに、軸が0-100に固定される

**原因**: 固定値のmin/maxが`scale: true`を無効化する

**解決策**:
1. 🥇 **関数コールバックで実装**（1-2時間）- 根本解決
   - `min: (value) => Math.max(value.min, 0)`
   - `max: (value) => Math.min(value.max, 100)`
2. 🥈 Y-axis Boundsを空にする（1分）
3. 🥉 SQLでデータをクリッピング（1時間）

**詳細**: [archive/2026-04-02-mixed-chart-investigation/MIXED_CHART_YAXIS_BOUNDS_AUTOSCALE.md](./archive/2026-04-02-mixed-chart-investigation/MIXED_CHART_YAXIS_BOUNDS_AUTOSCALE.md)

---

## 📖 クイックスタート

### Mixed Chartのカスタマイズ方法を知りたい

→ [MIXED_CHART_CUSTOMIZE_PROPERTIES.md](./archive/2026-04-02-mixed-chart-investigation/MIXED_CHART_CUSTOMIZE_PROPERTIES.md)

8つのプロパティ（Series type、Stack series、Area chartなど）の詳細と実践例

---

### ツールチップの仕組みを理解したい

→ [TOOLTIP_COMPREHENSIVE_GUIDE.md](./archive/2026-04-02-mixed-chart-investigation/TOOLTIP_COMPREHENSIVE_GUIDE.md)

axisモード vs itemモード、0値表示の問題、実装ガイド

---

### ツールチップの0を消したい

→ [MIXED_CHART_ZERO_VALUES_IN_TOOLTIP.md](./archive/2026-04-02-mixed-chart-investigation/MIXED_CHART_ZERO_VALUES_IN_TOOLTIP.md)

**最も簡単な方法**: Rich Tooltip OFF（1分で完了）

---

### 棒グラフの幅をカスタマイズしたい

→ [MIXED_CHART_BAR_WIDTH.md](./archive/2026-04-02-mixed-chart-investigation/MIXED_CHART_BAR_WIDTH.md)

**最も簡単な方法**: チャート幅を調整（即座に対応可能）

**恒久的な解決**: barWidth設定の実装（詳細な実装手順付き）

---

### Y軸のBoundsとauto scaleを両立させたい

→ [MIXED_CHART_YAXIS_BOUNDS_AUTOSCALE.md](./archive/2026-04-02-mixed-chart-investigation/MIXED_CHART_YAXIS_BOUNDS_AUTOSCALE.md)

**最も簡単な方法**: Boundsを空にする（1分で完了）

**恒久的な解決**: 関数コールバックで実装（制約付きauto scaleを実現）

---

## 🎯 重要な発見まとめ

### 1. Rich Tooltip と 0表示の関係

```javascript
typeof 0 === 'number'  // true
```

→ **Rich Tooltip ON（axisモード）では、0は必ず表示される**

解決策: Rich Tooltip OFF、または tooltipHideZeroValues機能の実装

---

### 2. Stack series の役割

**誤解**: Stack series OFF にすれば0が消える

**正解**: Stack seriesは「N/A → 0 変換」のみを制御

| データの状態 | Stack ON | Stack OFF |
|-------------|----------|-----------|
| 実際に0のデータ | 0として表示 | 0として表示（変わらない） |
| N/A（null）のデータ | 0に変換→表示 | 表示されない（変わる） |

---

### 3. データの流れ

```
データベース（NULL/0）
    ↓
DATAタブ（N/A/0）
    ↓
extractSeries（Stack ONの場合、N/A→0変換）
    ↓
ECharts描画
    ↓
ツールチップ（Rich Tooltip ONの場合、すべて表示）
```

---

### 4. 棒グラフの幅の自動調整

**発見**: EChartsのデフォルト自動計算アルゴリズムによる

**計算要素**:
```
棒の幅 = f(チャート幅, データポイント数, 系列数, barGap, barCategoryGap)
```

**現状**: SupersetのUIに設定なし

**カスタマイズ**: 4つのファイル（types.ts、controlPanel.tsx、transformers.ts、transformProps.ts）を修正

**詳細**: [MIXED_CHART_BAR_WIDTH.md](./archive/2026-04-02-mixed-chart-investigation/MIXED_CHART_BAR_WIDTH.md)

---

### 5. Y軸Boundsとauto scaleの競合

**発見**: 固定値のmin/maxが`scale: true`を無効化する

**問題**:
```typescript
// defaults.ts
scale: true  // auto scale有効

// transformProps.ts
min: 0,      // 固定値
max: 100,    // → scale無効化
```

**解決策**: EChartsの関数コールバック

```typescript
min: (value) => Math.max(value.min, 0),
max: (value) => Math.min(value.max, 100),
```

**効果**:
- データ20-30 → 軸20-30（auto scale）
- データ-10-150 → 軸0-100（制約）

**詳細**: [MIXED_CHART_YAXIS_BOUNDS_AUTOSCALE.md](./archive/2026-04-02-mixed-chart-investigation/MIXED_CHART_YAXIS_BOUNDS_AUTOSCALE.md)

---

## 🛠️ 技術情報

### 主要ファイル

| ファイル | 役割 |
|---------|------|
| `transformProps.ts` | fillNeighborValue決定、Rich tooltip設定 |
| `utils/series.ts` | extractSeries、isDefined関数 |
| `utils/forecast.ts` | ツールチップの値抽出、フォーマット |
| `controlPanel.tsx` | UI設定の定義 |
| `types.ts` | 型定義、デフォルト値 |

### 主要関数

- `extractSeries()`: データを変換してECharts形式にする
- `isDefined()`: null/undefinedでないかチェック（`isDefined(0) === true`）
- `extractForecastValuesFromTooltipParams()`: ツールチップ用に値を抽出
- `formatForecastTooltipSeries()`: 各シリーズを整形

---

## 🔗 外部リソース

- [Issue #4462 - Tooltip Option](https://github.com/apache/incubator-superset/issues/4462)
- [PR #21296 - Show zero value in tooltip](https://github.com/apache/superset/pull/21296)
- [ECharts Tooltip Documentation](https://echarts.apache.org/en/option.html#tooltip)

---

## 📝 用語集

| 用語 | 説明 |
|------|------|
| **Rich Tooltip** | axisモード。X軸上のすべてのシリーズを表示 |
| **item mode** | マウスを置いたシリーズのみを表示 |
| **Stack series** | 積み上げグラフ表示の設定 |
| **fillNeighborValue** | 欠損値を埋める値（Stack ON時は0） |
| **N/A** | "Not Available"。データなし、null、undefined |
| **extractSeries** | データを変換してECharts形式にする関数 |
| **isDefined** | null/undefinedでないかチェックする関数 |

---

## 📊 問題解決マトリクス

| 問題 | 短期対応（即効） | 中期対応（実装） | 長期対応（根本） |
|------|----------------|----------------|----------------|
| **ツールチップに0が大量表示** | Rich Tooltip OFF<br>SQLで除外 | tooltipHideZeroValues実装 | コミュニティPR |
| **X軸ラベルが表示されない** | チャート幅を広げる<br>ラベル回転 | - | - |
| **N/Aが0になる** | Stack series OFF | - | - |
| **データなしと0の区別** | Stack OFF<br>SQL IS NOT NULL | tooltipHideNullValues実装 | - |
| **棒グラフの幅が自動で変わる** | チャート幅を調整<br>データ期間を調整 | barWidth設定の実装 | コミュニティPR |
| **Y軸Boundsでauto scale無効** | Boundsを空にする<br>SQLでクリッピング | 関数コールバック実装 | コミュニティPR |

---

## 🚀 今後のアクション

### すぐに対応可能

- Rich Tooltip OFF（1分）
- SQLで0を除外（1時間）
- チャート幅の調整（5分）

### 中期的な対応

- tooltipHideZeroValues機能の実装（2-3日）
- barWidth設定機能の実装（1-2日）
- Y軸Bounds関数コールバック実装（1-2時間）
- 社内環境でテスト運用

### 長期的な対応

- コミュニティ貢献（PR作成）
- 公式機能としての組み込み

---

**作成者**: Claude Code
**最終更新**: 2026-04-02
**リポジトリ**: 個人調査用
