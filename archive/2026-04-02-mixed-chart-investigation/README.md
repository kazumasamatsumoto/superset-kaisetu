# Mixed Chart 調査レポート - 2026-04-02

Apache Superset Mixed Timeseries Chart に関する包括的な調査レポート集

---

## 📋 調査概要

**調査日**: 2026-04-02
**対象**: Apache Superset Mixed Timeseries Chart
**調査者**: Claude Code

### 調査のきっかけ

Mixed Chartの以下の問題・疑問について調査：
1. カスタマイズ可能なプロパティの全容
2. シリーズタイプの設定方法
3. X軸ラベルが表示されない問題
4. ツールチップに大量の0が表示される問題
5. 棒グラフの幅が自動で変わる仕組みとカスタマイズ方法
6. Y軸Bounds設定時のauto scale問題
7. Query AとQuery Bの独立性（フィルターの影響範囲）

---

## 📚 レポート一覧

### 1. Mixed Chartカスタマイズプロパティ完全ガイド
**ファイル**: `MIXED_CHART_CUSTOMIZE_PROPERTIES.md`

**内容**:
- Customizeセクションで設定可能な8つのプロパティの詳細
- Series type（グラフの種類）の7つの選択肢
- Stack series、Area chart、Show Valuesなどの設定
- 実践的な設定例5パターン
- 技術的詳細とコード解説

**対象読者**: Mixed Chartを使用する全ユーザー

**重要度**: ⭐⭐⭐⭐⭐

---

### 2. Mixed Chartシリーズタイプ設定ガイド
**ファイル**: `MIXED_CHART_SERIES_TYPE.md`

**内容**:
- Query AとQuery Bのグラフタイプは独立して変更可能
- デフォルト設定（両方ともLine）
- 7種類のグラフタイプ一覧
- 設定の組み合わせ例
- よくある質問

**対象読者**: Mixed Chartの基本を学びたいユーザー

**重要度**: ⭐⭐⭐⭐☆

---

### 3. X軸ラベル表示問題の解決ガイド
**ファイル**: `MIXED_CHART_XAXIS_LABEL_ISSUE.md`

**問題**: 7/1〜7/5のデータがあるのに、7/5のX軸ラベルが表示されない

**内容**:
- 発生原因（EChartsの自動ラベル間引き、X Axis Bounds、Truncate X Axis）
- 6つの解決策（チャート幅を広げる、ラベル回転、フォーマット短縮など）
- 推奨される解決策の組み合わせ
- 技術的詳細（X軸の仕組み、EChartsのデフォルト動作）

**対象読者**: X軸ラベル表示で問題を抱えているユーザー

**重要度**: ⭐⭐⭐⭐☆

---

### 4. ツールチップ0値表示問題の解決ガイド
**ファイル**: `MIXED_CHART_ZERO_VALUES_IN_TOOLTIP.md`

**問題**: ツールチップに大量の「0」が表示され、視認性が低下する

**重要な発見**:
- ✅ **Rich Tooltip（axisモード）が有効な限り、0値は必ず表示される**
- ✅ **Stack series OFF にしても、実際の0は消えない**

**内容**:
- 0が表示される2つの原因（実際に0のデータ vs N/Aが0に変換）
- Rich Tooltipと0表示の関係
- Stack seriesの役割（誤解の訂正）
- 5つの解決策（優先順位付き）
- 技術的詳細（JavaScriptの型チェック、データフロー）

**推奨解決策**:
1. 🥇 Rich Tooltip OFF（最も効果的）
2. 🥈 SQLで0を除外
3. 🥉 tooltipHideZeroValues機能の実装

**対象読者**: ツールチップ表示で問題を抱えているユーザー

**重要度**: ⭐⭐⭐⭐⭐

---

### 5. ツールチップ機能完全ガイド
**ファイル**: `TOOLTIP_COMPREHENSIVE_GUIDE.md`

**内容**:
- ツールチップの基本動作メカニズム
- axisモード vs itemモード
- データ処理：欠損値と0値の扱い
- 0値表示の問題と現状
- tooltipHideZeroValues機能の実装ガイド
- トラブルシューティング

**対象読者**: ツールチップ機能を深く理解したいユーザー、機能実装を検討するユーザー

**重要度**: ⭐⭐⭐⭐⭐

---

### 6. スタックモード詳細解説
**ファイル**: `STACK_MODE_EXPLAINED.md`

**内容**:
- Stack seriesの役割と動作
- N/A（null）が0に変換される仕組み
- fillNeighborValueの決定ロジック
- extractSeries関数の処理フロー
- スタックモードと非スタックモードの違い

**対象読者**: Stack seriesの挙動を理解したいユーザー

**重要度**: ⭐⭐⭐⭐☆

---

### 7. 棒グラフの幅に関する調査レポート
**ファイル**: `MIXED_CHART_BAR_WIDTH.md`

**問題**: 棒グラフが太くなったり細くなったり自動で変化する

**重要な発見**:
- ✅ **EChartsのデフォルト自動計算アルゴリズムによる**
- ✅ **SupersetのUIにカスタマイズ設定は存在しない**

**内容**:
- 棒グラフの幅が自動で変わる理由（EChartsの計算式）
- barWidth、barGap、barCategoryGapの詳細解説
- Supersetの現状（設定なし）
- カスタマイズする方法（4つのファイルを修正）
- 実装手順の詳細ガイド

**推奨解決策**:
1. 🥇 チャート幅を調整（即座に対応可能）
2. 🥈 コード修正による実装（1-2日）
3. 🥉 コミュニティ貢献（2-3週間）

**対象読者**: 棒グラフの幅をカスタマイズしたいユーザー、機能実装を検討するユーザー

**重要度**: ⭐⭐⭐⭐☆

---

### 8. Y軸Bounds設定時のAuto Scale問題
**ファイル**: `MIXED_CHART_YAXIS_BOUNDS_AUTOSCALE.md`

**問題**: Y軸Boundsに0-100を設定すると、auto scaleが機能せず常に0-100に固定される

**重要な発見**:
- ✅ **`scale: true`がデフォルトで有効だが、固定値を設定すると無効化される**
- ✅ **EChartsの関数コールバックを使えば制約付きauto scaleを実現可能**

**内容**:
- auto scaleが機能しない原因（固定値とscale: trueの競合）
- EChartsの関数コールバック機能の解説
- 制約付きauto scaleの実装方法
- 動作確認とテストケース
- 代替案の比較

**推奨解決策**:
1. 🥇 関数コールバックで実装（1-2時間）- 根本解決
2. 🥈 Boundsを空にする（1分）- 即座に対応可能
3. 🥉 SQLでクリッピング（1時間）- データレベルで制限

**対象読者**: Y軸のBoundsとauto scaleを両立させたいユーザー、機能実装を検討するユーザー

**重要度**: ⭐⭐⭐⭐⭐

---

### 9. Query AとQuery Bの独立性調査レポート
**ファイル**: `MIXED_CHART_QUERY_INDEPENDENCE.md`

**質問**: Mixed Chartにおいて、片方のWHERE条件（Filters）はもう一つのチャートに影響を与えるか？

**結論**: ✅ **完全に独立している**

**重要な発見**:
- ✅ **Query Aの `adhoc_filters` は Query Bに影響を与えない**
- ✅ **Query Bの `adhoc_filters_b` は Query Aに影響を与えない**
- ✅ **各クエリは独立したSQL文として実行される**

**内容**:
- UI設定レベルでの分離（`adhoc_filters` vs `adhoc_filters_b`）
- FormData分離処理（`removeFormDataSuffix` vs `retainFormDataSuffix`）
- クエリ構築の独立性（独立した `buildQueryContext` 呼び出し）
- データ取得の独立性（`queriesData[0]` vs `queriesData[1]`）
- 検証方法とデバッグTips
- ベストプラクティスと避けるべき使い方

**対象読者**: Mixed Chartで複数のクエリを設定するユーザー、フィルターの影響範囲を理解したいユーザー

**重要度**: ⭐⭐⭐⭐☆

---

## 🔍 主要な発見と結論

### 1. Rich Tooltipと0表示の関係

**発見**: Rich Tooltip ON（axisモード）の場合、**0は必ず表示される**

**理由**:
```javascript
typeof 0 === 'number'  // true
```
→ 0は有効な数値として扱われるため、ツールチップに表示される

**対策**: Rich Tooltip OFF、またはtooltipHideZeroValues機能の実装

---

### 2. Stack seriesの役割

**誤解**: Stack series OFF にすれば0が消える

**正解**: Stack seriesは**N/A → 0 変換のみ**を制御

| データの状態 | Stack ON | Stack OFF |
|-------------|----------|-----------|
| 実際に0のデータ | 0として表示 | 0として表示（変わらない） |
| N/A（null）のデータ | 0に変換→表示 | 表示されない（変わる） |

---

### 3. X軸ラベル表示の仕組み

**発見**: 最終日のラベルが表示されない原因は主に3つ

1. EChartsの自動ラベル間引き
2. X Axis Boundsの設定
3. 棒グラフのboundaryGap

**推奨解決策**:
- X Axis Time Format: `%-m/%-d`（短いフォーマット）
- X Axis Label Rotation: 45度
- チャート幅を広げる

---

### 4. 棒グラフの幅の自動調整

**発見**: 棒グラフの幅はEChartsのデフォルト自動計算による

**自動計算の要素**:
1. チャート幅
2. データポイント数
3. シリーズ数
4. barGap（デフォルト20%）
5. barCategoryGap（系列数ベース）

**現状**: SupersetのUIにカスタマイズ設定なし

**カスタマイズ方法**: 4つのファイル（types.ts、controlPanel.tsx、transformers.ts、transformProps.ts）を修正

---

### 5. Y軸Boundsとauto scaleの競合

**発見**: Y軸Boundsに固定値を設定すると、`scale: true`が無効化される

**原因**:
```typescript
// defaults.ts
scale: true  // auto scale有効

// transformProps.ts
min: 0,      // 固定値を設定
max: 100,    // → scale: trueが無効化される
```

**解決策**: EChartsの関数コールバックを使用

```typescript
min: (value) => Math.max(value.min, 0),   // 制約付きauto scale
max: (value) => Math.min(value.max, 100),
```

**効果**:
- データが20-30 → 軸も20-30（auto scale有効）
- データが-10-150 → 軸は0-100（制約適用）

**詳細**: [MIXED_CHART_YAXIS_BOUNDS_AUTOSCALE.md](./MIXED_CHART_YAXIS_BOUNDS_AUTOSCALE.md)

---

### 6. Query AとQuery Bの完全独立性

**発見**: Query AとQuery Bのフィルター（WHERE条件）は完全に独立している

**独立性の保証メカニズム**:
1. **UI設定**: 異なるコントロール名（`adhoc_filters` vs `adhoc_filters_b`）
2. **FormData分離**: サフィックスパターンによる完全分離
3. **クエリ構築**: 独立した `buildQueryContext` 呼び出し
4. **SQL生成**: 完全に別のSQL文
5. **データ取得**: 独立したクエリ結果（`queriesData[0]` vs `queriesData[1]`）

**影響範囲**:
- Query Aの `adhoc_filters` → Query Aにのみ影響
- Query Bの `adhoc_filters_b` → Query Bにのみ影響
- 交差影響なし

**実用的な意味**:
- ✅ 完全に異なる期間のデータを比較可能
- ✅ 完全に異なるカテゴリーを比較可能
- ✅ 複雑な条件を独立して設定可能
- ❌ 一方のクエリ結果を他方の条件に使用不可（クエリは並行実行）

**詳細**: [MIXED_CHART_QUERY_INDEPENDENCE.md](./MIXED_CHART_QUERY_INDEPENDENCE.md)

---

## 🛠️ 技術的ポイント

### データフロー

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

### 関連ファイル

| ファイル | 主な内容 |
|---------|---------|
| `transformProps.ts` | fillNeighborValue決定、Rich tooltip設定 |
| `utils/series.ts` | extractSeries、isDefined関数 |
| `utils/forecast.ts` | ツールチップの値抽出、フォーマット |
| `controlPanel.tsx` | UI設定の定義 |
| `types.ts` | 型定義、デフォルト値 |

---

## 📊 問題解決マトリクス

| 問題 | 短期対応 | 中期対応 | 長期対応 |
|------|---------|---------|---------|
| **ツールチップに0が大量表示** | Rich Tooltip OFF<br>SQLで除外 | tooltipHideZeroValues実装 | コミュニティPR |
| **X軸ラベルが表示されない** | チャート幅を広げる<br>ラベル回転 | - | - |
| **N/Aが0になる** | Stack series OFF | - | - |
| **データなしと0の区別** | Stack OFF<br>SQL IS NOT NULL | tooltipHideNullValues実装 | - |
| **棒グラフの幅が自動で変わる** | チャート幅を調整<br>データ期間を調整 | barWidth設定の実装 | コミュニティPR |
| **Y軸Boundsでauto scaleが無効** | Boundsを空にする<br>SQLでクリッピング | 関数コールバック実装 | コミュニティPR |

---

## 🎯 推奨アクション

### すぐに対応が必要な場合

1. **ツールチップの0表示**: Rich Tooltip OFF（1分で完了）
2. **X軸ラベル**: チャート幅を広げる + ラベル回転（5分で完了）
3. **N/Aの0変換**: Stack series OFF（1分で完了）

### 恒久的な解決を目指す場合

1. **tooltipHideZeroValues機能の実装**（工数: 2-3日）
   - Rich Tooltip ONのまま使える
   - 設定で柔軟に制御可能
   - 詳細は `TOOLTIP_COMPREHENSIVE_GUIDE.md` セクション7参照

2. **コミュニティ貢献**（工数: 2-3週間）
   - 実装してテスト
   - Pull Request作成
   - 本家にマージ

---

## 🔗 関連リソース

### 内部ドキュメント

- `../2026-04-01-tooltip-investigation/` - 初期のツールチップ調査
- `../2026-04-02-date-format-bug/` - 日付フォーマット不具合調査

### 外部リソース

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

## 📌 まとめ

### 調査の成果

1. ✅ Mixed Chartのカスタマイズプロパティを完全に文書化
2. ✅ ツールチップの0表示問題の原因を特定
3. ✅ **重要な発見**: Rich Tooltip ON では0が必ず表示される
4. ✅ **誤解の訂正**: Stack series OFF でも実際の0は消えない
5. ✅ X軸ラベル表示問題の解決策を提示
6. ✅ 実装可能な機能（tooltipHideZeroValues）の設計
7. ✅ 棒グラフの幅の自動調整メカニズムを解明
8. ✅ **重要な発見**: SupersetのUIに棒グラフ幅の設定は存在しない
9. ✅ barWidth カスタマイズ機能の実装方法を設計
10. ✅ Y軸Boundsとauto scaleの競合問題を特定
11. ✅ **重要な発見**: 固定値のmin/maxは`scale: true`を無効化する
12. ✅ EChartsの関数コールバックで制約付きauto scaleを実現
13. ✅ Query AとQuery Bの独立性を検証
14. ✅ **重要な発見**: フィルターは完全に独立、交差影響なし
15. ✅ サフィックスパターンによる完全分離メカニズムを解明

### 今後のアクション

**即座に対応可能**:
- Rich Tooltip OFF
- SQLで0を除外
- チャート幅の調整

**中期的な対応**:
- tooltipHideZeroValues機能の実装
- 社内環境でテスト運用

**長期的な対応**:
- コミュニティ貢献（PR作成）
- 公式機能としての組み込み

---

**調査実施**: Claude Code
**作成日**: 2026-04-02
**レビュー推奨**: フロントエンド開発チーム、データ可視化チーム、プロダクトマネージャー
