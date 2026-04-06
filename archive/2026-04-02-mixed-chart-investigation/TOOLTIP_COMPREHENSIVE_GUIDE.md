# Apache Superset Mixed Chart ツールチップ機能 完全ガイド

**対象**: Apache Superset Mixed Timeseries Chart
**調査日**: 2026-04-01
**関連Issue**: [#4462](https://github.com/apache/superset/issues/4462)

---

## 目次

1. [概要](#1-概要)
2. [ツールチップの基本動作メカニズム](#2-ツールチップの基本動作メカニズム)
3. [ツールチップモード（axis vs item）](#3-ツールチップモードaxis-vs-item)
4. [データ処理：欠損値と0値の扱い](#4-データ処理欠損値と0値の扱い)
5. [0値表示の問題と現状](#5-0値表示の問題と現状)
6. [解決策と回避策](#6-解決策と回避策)
7. [実装ガイド：axisモード + 0値非表示の複合機能](#7-実装ガイドaxisモード--0値非表示の複合機能)
8. [トラブルシューティング](#8-トラブルシューティング)
9. [まとめと推奨事項](#9-まとめと推奨事項)

---

## 1. 概要

### 1.1 このガイドについて

Mixed Chart（混合時系列チャート）のツールチップ機能について、以下の内容を網羅的に解説します：

- ツールチップの動作メカニズム
- axisモードとitemモードの違い
- 欠損値が0に変換されるロジック
- 0値が表示される理由と問題点
- 実装可能な解決策

### 1.2 よくある問題

**問題1**: ツールチップに0値が大量に表示される
```
2021-01-15
● Series A: 1,234
● Series B: 0      ← 不要なノイズ
● Series C: 0      ← 不要なノイズ
● Series D: 5,678
```

**問題2**: スタックモードで欠損値が0になる
```
データ: [100, null, 200]
表示: [100, 0, 200]  ← nullが0に変換される
```

**問題3**: ツールチップが巨大になりチャートを覆い隠す

---

## 2. ツールチップの基本動作メカニズム

### 2.1 全体フロー

```
ユーザーのマウスオーバー
  ↓
ECharts がマウス位置を検出
  ↓
tooltip.formatter 関数が呼び出される
  ↓
extractForecastValuesFromTooltipParams() で値を抽出
  ↓
formatForecastTooltipSeries() で各シリーズを整形
  ↓
tooltipHtml() でHTMLを生成
  ↓
ツールチップ表示
```

### 2.2 コードレベルの実装

#### Step 1: EChartsのtooltip設定

**ファイル**: `superset-frontend/plugins/plugin-chart-echarts/src/MixedTimeseries/transformProps.ts:581-646`

```typescript
tooltip: {
  ...getDefaultTooltip(refs),
  show: !inContextMenu,
  trigger: richTooltip ? 'axis' : 'item',  // ← モード切り替え
  formatter: (params: any) => {
    // ツールチップの内容を生成
  },
}
```

#### Step 2: 値の抽出（extractForecastValuesFromTooltipParams）

**ファイル**: `superset-frontend/plugins/plugin-chart-echarts/src/utils/forecast.ts:57-83`

```typescript
export const extractForecastValuesFromTooltipParams = (
  params: any[],
  isHorizontal = false,
): Record<string, ForecastValue> => {
  const values: Record<string, ForecastValue> = {};
  params.forEach(param => {
    const { marker, seriesId, value } = param;
    const numericValue = isHorizontal ? value[0] : value[1];

    // 重要: 0も number 型なのでこの条件を通過する
    if (typeof numericValue === 'number') {
      if (!(context.name in values))
        values[context.name] = {
          marker: marker || '',
        };
      const forecastValues = values[context.name];
      if (context.type === ForecastSeriesEnum.Observation)
        forecastValues.observation = numericValue;
    }
  });
  return values;
};
```

**重要ポイント**:
- `typeof numericValue === 'number'` のチェック
- **0も`number`型なので、この条件を通過する**
- つまり、**0値は正常に抽出される**

#### Step 3: シリーズの整形（formatForecastTooltipSeries）

**ファイル**: `forecast.ts:85-117`

```typescript
export const formatForecastTooltipSeries = ({
  seriesName,
  observation,
  marker,
  formatter,
}: ForecastValue & {
  seriesName: string;
  marker: TooltipMarker;
  formatter: ValueFormatter;
}): string[] => {
  const name = `${marker}${sanitizeHtml(seriesName)}`;

  // 重要: 0も number 型なので formatter(0) が実行される
  let value = typeof observation === 'number' ? formatter(observation) : '';

  return [name, value];
};
```

**重要ポイント**:
- `typeof observation === 'number'` のチェック
- **0も`number`型なので、`formatter(0)`が実行される**
- フォーマッター（例: `getNumberFormatter(',.0f')`）は0を`"0"`として整形
- **結果: 0値も正常にツールチップに表示される**

### 2.3 データフロー図

```
ECharts params:
[
  { seriesId: "series1", value: [timestamp, 10] },
  { seriesId: "series2", value: [timestamp, 0] },   ← 0値
  { seriesId: "series3", value: [timestamp, 5] }
]
  ↓
extractForecastValuesFromTooltipParams()
  ↓
{
  "series1": { observation: 10, marker: "●" },
  "series2": { observation: 0, marker: "●" },      ← 0も含まれる
  "series3": { observation: 5, marker: "●" }
}
  ↓
formatForecastTooltipSeries() × 3回
  ↓
[
  ["● series1", "10"],
  ["● series2", "0"],                              ← 0として表示
  ["● series3", "5"]
]
  ↓
tooltipHtml()
  ↓
<table>
  <tr><td>● series1</td><td>10</td></tr>
  <tr><td>● series2</td><td>0</td></tr>            ← 0が表示される
  <tr><td>● series3</td><td>5</td></tr>
</table>
```

---

## 3. ツールチップモード（axis vs item）

### 3.1 2つのモードの違い

| モード | 表示内容 | トリガー方法 |
|--------|----------|--------------|
| **axis** | **X軸上の同じ位置にあるすべてのシリーズを表示** | X軸上の任意の位置にマウスオーバー |
| **item** | **マウスオーバーした特定のシリーズのみを表示** | 特定のバー/ポイントにマウスオーバー |

### 3.2 axis モード（Rich tooltip ON）

**動作**:
1. ユーザーがX軸上の任意の位置にマウスをホバー
2. その位置（時点）における**すべてのシリーズの値**を表示
3. X軸に沿ってマウスを動かすと、ツールチップが追従

**ツールチップの内容例**:
```
2021-01-15
● Query A - Metric 1: 1,234
● Query A - Metric 2: 5,678
● Query B - Metric 1: 900
● Query B - Metric 2: 0     ← 0も表示される
```

**メリット**:
- ✅ 複数のシリーズを同時に比較できる
- ✅ 時点ごとの全体像が把握しやすい
- ✅ 特定のバー/ポイントにマウスを合わせる必要がない

**デメリット**:
- ❌ シリーズ数が多いとツールチップが巨大になる
- ❌ 0値も表示されるため、情報が多すぎる場合がある
- ❌ チャートを覆い隠してしまう可能性がある

**適用シーン**:
- シリーズ数が少ない（5個以下程度）
- 各時点で複数のメトリクスを比較したい
- 時系列データの推移を詳細に確認したい

### 3.3 item モード（Rich tooltip OFF）

**動作**:
1. ユーザーが特定のバー/ポイントにマウスをホバー
2. その**特定のシリーズのみ**の値を表示
3. 他のシリーズの値は表示されない

**ツールチップの内容例**:
```
2021-01-15
● Query A - Metric 1: 1,234
```

**メリット**:
- ✅ シンプルで見やすい
- ✅ 必要な情報のみが表示される
- ✅ シリーズ数が多くても問題ない
- ✅ ツールチップが小さいので、チャートを覆い隠さない

**デメリット**:
- ❌ 複数シリーズを比較するには、複数回マウスオーバーが必要
- ❌ 小さいバー/ポイントにマウスを合わせるのが難しい場合がある

**適用シーン**:
- シリーズ数が多い（10個以上）
- 特定のシリーズのみに注目したい
- ツールチップをシンプルに保ちたい

### 3.4 設定方法

**UI**: チャート編集 → Chart Options → Tooltip → **Rich tooltip** チェックボックス

- ✅ **Rich tooltip ON（デフォルト）** → `axis`モード
- ❌ **Rich tooltip OFF** → `item`モード

**コード**: `plugin-chart-echarts/src/MixedTimeseries/transformProps.ts:584`

```typescript
trigger: richTooltip ? 'axis' : 'item',
```

### 3.5 シリーズ数による推奨モード

| シリーズ数 | 推奨モード | 理由 |
|-----------|-----------|------|
| 1-5個 | **axis** | すべて表示しても問題ない |
| 6-10個 | **どちらでも** | ユースケース次第 |
| 11個以上 | **item** | ツールチップが大きくなりすぎる |

---

## 4. データ処理：欠損値と0値の扱い

### 4.1 データフロー全体像

```
┌─────────────────────────────────────────────────────────────┐
│ Step 1: バックエンド（Python）                              │
│ - Pandasでクエリを実行                                      │
│ - 時系列データに欠損がある場合、NaN または 値なし          │
└─────────────────────────────────────────────────────────────┘
                            ↓
┌─────────────────────────────────────────────────────────────┐
│ Step 2: JSON変換                                            │
│ - NaN → null に変換                                         │
│ - 値がない列はキー自体が存在しない                         │
└─────────────────────────────────────────────────────────────┘
                            ↓
┌─────────────────────────────────────────────────────────────┐
│ Step 3: フロントエンド受信                                  │
│ {                                                           │
│   data: [                                                   │
│     { __timestamp: 1609459200000, series1: 100 },           │
│     { __timestamp: 1609545600000, series1: null },  ← null  │
│     { __timestamp: 1609632000000 },  ← series1キーなし     │
│   ]                                                         │
│ }                                                           │
└─────────────────────────────────────────────────────────────┘
                            ↓
┌─────────────────────────────────────────────────────────────┐
│ Step 4: transformProps.ts での処理                          │
│ - extractSeries() を呼び出し                                │
│ - fillNeighborValue パラメータを決定:                       │
│   * スタックON: fillNeighborValue = 0                       │
│   * スタックOFF: fillNeighborValue = undefined              │
└─────────────────────────────────────────────────────────────┘
                            ↓
┌─────────────────────────────────────────────────────────────┐
│ Step 5: extractSeries() での変換                            │
│                                                             │
│ スタックONの場合:                                           │
│   null/undefined → 0 に変換                                 │
│   結果: [[timestamp, 100], [timestamp, 0], [timestamp, 0]]  │
│                                                             │
│ スタックOFFの場合:                                          │
│   null/undefined → そのまま                                 │
│   結果: [[timestamp, 100], [timestamp, null], ...]          │
└─────────────────────────────────────────────────────────────┘
                            ↓
┌─────────────────────────────────────────────────────────────┐
│ Step 6: ECharts描画とツールチップ                           │
│                                                             │
│ スタックON:                                                 │
│   - 0として描画（高さ0のバー/ポイント）                     │
│   - ツールチップに"0"として表示                             │
│                                                             │
│ スタックOFF:                                                │
│   - null/undefinedは描画しない（ギャップ）                 │
│   - ツールチップには表示されない                           │
└─────────────────────────────────────────────────────────────┘
```

### 4.2 fillNeighborValueの決定

**ファイル**: `plugin-chart-echarts/src/MixedTimeseries/transformProps.ts:224-233`

```typescript
const rebasedDataA = rebaseForecastDatum(data1, verboseMap);
const [rawSeriesA] = extractSeries(rebasedDataA, {
  fillNeighborValue: stack ? 0 : undefined,  // ← ここで決定
  xAxis: xAxisLabel,
});

const rebasedDataB = rebaseForecastDatum(data2, verboseMap);
const [rawSeriesB] = extractSeries(rebasedDataB, {
  fillNeighborValue: stackB ? 0 : undefined,  // ← ここで決定
  xAxis: xAxisLabel,
});
```

**ロジック**:
- `stack`が`true`（スタックモードON）の場合: `fillNeighborValue = 0`
- `stack`が`false`（スタックモードOFF）の場合: `fillNeighborValue = undefined`

**なぜスタック時に0で埋めるのか？**

スタックチャートでは、各シリーズが積み上げられます。ある時点でシリーズAに値がなく、シリーズBに値がある場合：

- **0で埋めない場合**: その時点でシリーズBが描画されない（全体のスタックが崩れる）
- **0で埋める場合**: シリーズAが0、シリーズBが正しい高さで描画される

### 4.3 extractSeriesでの処理

**ファイル**: `plugin-chart-echarts/src/utils/series.ts:317-349`

```typescript
const finalSeries = sortedSeries.map(name => ({
  id: name,
  name,
  data: sortedRows
    .map(({ row, totalStackedValue }, idx) => {
      const currentValue = row[name];  // ← バックエンドから来た値

      // 隣接する値が定義されているかチェック
      const isNextToDefinedValue =
        isDefined(rows[idx - 1]?.[name]) || isDefined(rows[idx + 1]?.[name]);

      // fillNeighborValueで埋めるべきかチェック
      const isFillNeighborValue =
        !isDefined(currentValue) &&        // ← currentValueがnull/undefined
        isNextToDefinedValue &&             // ← 隣接する値がある
        fillNeighborValue !== undefined;    // ← fillNeighborValueが指定されている

      let value: DataRecordValue | undefined = currentValue;

      if (isFillNeighborValue) {
        value = fillNeighborValue;  // ← ここで0に変換（スタックONの場合）
      }

      return [row[xAxis], value];
    })
    .filter(obs => !removeNulls || (obs[0] !== null && obs[1] !== null))
    .map(obs => (isHorizontal ? [obs[1], obs[0]] : obs)),
}));
```

**重要な条件**:

```typescript
const isFillNeighborValue =
  !isDefined(currentValue) &&        // 条件1: 値がnull/undefined
  isNextToDefinedValue &&             // 条件2: 隣接する時点に値がある
  fillNeighborValue !== undefined;    // 条件3: fillNeighborValueが指定されている
```

**なぜ「隣接する値がある」をチェックするのか？**

理由: **データ系列の途中のギャップのみを埋めるため**

```javascript
// 例1: 途中のギャップ → 埋める
[
  [t1, 100],   // 値あり
  [t2, null],  // ← 隣接する値がある → 0に変換
  [t3, 200]    // 値あり
]

// 例2: 系列の最初/最後 → 埋めない
[
  [t1, null],  // ← 隣接する値がない → そのまま
  [t2, 100],
  [t3, 200],
  [t4, null]   // ← 隣接する値がない → そのまま
]
```

### 4.4 isDefined関数の実装

**ファイル**: `plugin-chart-echarts/src/utils/series.ts:54-56`

```typescript
function isDefined<T>(value: T | undefined | null): boolean {
  return value !== undefined && value !== null;
}
```

**動作**:
- `null` → `false`
- `undefined` → `false`
- `0` → `true` ← **0は「定義されている」と判定**
- `""` (空文字列) → `true`
- `false` → `true`

**これが重要な理由**:
- **実際のデータが0の場合は、欠損値として扱われない**
- 0と欠損値（null/undefined）が明確に区別される

### 4.5 欠損値の扱いまとめ

| 段階 | スタックモードON | スタックモードOFF |
|------|------------------|-------------------|
| **バックエンドからのデータ** | `null` または キーなし（`undefined`） | `null` または キーなし（`undefined`） |
| **extractSeriesでの処理** | **0に変換される** | `null`のまま |
| **EChartsへの渡し方** | `[timestamp, 0]` | `[timestamp, null]` |
| **ツールチップ表示** | **0として表示される** | 表示されない |
| **グラフ描画** | 0の高さで描画される | ギャップ（線が途切れる） |

**重要な発見**:
- **スタックモードがONの場合のみ、欠損値が0として扱われる**
- **スタックモードがOFFの場合は、欠損値はnullのまま（0にはならない）**

---

## 5. 0値表示の問題と現状

### 5.1 なぜ0値が表示されるのか

**EChartsの動作**（`trigger: 'axis'` モードの場合）:

1. ユーザーがX軸上の特定の位置にマウスオーバー
2. EChartsは、その位置のすべてのシリーズのデータポイントを取得
3. データポイントの値が0でも、それは有効な数値として扱われる
4. **結果**: すべてのシリーズ（0値を含む）がツールチップに表示される

**JavaScriptの型システム**:

```javascript
typeof 0 === 'number'       // true
typeof null === 'object'    // true
typeof undefined === 'undefined'  // true
```

**設計判断**:
- `0`は有効な数値データ
- `null`や`undefined`は「データなし」を意味
- したがって、`typeof numericValue === 'number'`は0を含む

**この判断は正しい**が、ユースケースによっては0を非表示にしたいケースがあります。

### 5.2 問題の本質

**Issue #4462で指摘されている問題**:

1. **多数のシリーズがある場合の視認性**:
   - 10個、20個のシリーズがあり、そのほとんどが0値の場合
   - ツールチップが巨大になり、チャートを覆い隠す
   - 重要な非ゼロ値が埋もれてしまう

2. **フィルター適用後の0値表示**:
   - SQLフィルターで特定のシリーズを除外
   - しかし、除外されたシリーズもツールチップに0として表示される
   - ユーザーは「データが存在しない」のか「値が0」なのか区別できない

3. **現状の実装**:
   - **0値を非表示にするオプションが存在しない**
   - Supersetのコードレベルでフィルタリングする機能がない

### 5.3 Issue #4462の詳細状態

| 項目 | 内容 |
|------|------|
| **Issue番号** | #4462 |
| **タイトル** | [Tooltip] Option to display or ignore 0 or null values |
| **開設日** | 2018年2月22日 |
| **現在の状態** | **Closed（閉じられている）** |
| **ラベル** | `inactive`（30日以上非アクティブ） |
| **コミュニティの反応** | 👍 20+ reactions |

**元の要望内容**:
1. ユーザーが関心のあるシリーズのみを表示する設定
2. **特定の時点で0またはnull値のシリーズを省略する設定**

**なぜ閉じられたのか**:
1. 長期間（7年以上）実装されなかった
2. inactive（非アクティブ）ラベル - 30日以上活動がないIssueは自動的に閉じられる
3. 関連する部分的な解決策が実装された（removeNulls機能など）

**重要**: 閉じられているが「解決済み」ではない

### 5.4 関連するPRと実装

#### PR #21296: 0値を表示する修正（2022年）

**内容**:
- **逆の問題を修正**: 0値がツールチップに表示されなかったバグ
- EChartsプラグインで、truthyチェックにより0が除外されていた
- このPR後、0値は正しく表示されるようになった

**Issue #4462との関係**:
- **矛盾している**: Issue #4462は「0を非表示にしたい」、PR #21296は「0を表示したい」
- これは、異なるユースケースが存在することを示している：
  - ケースA: 0は有効なデータなので表示すべき
  - ケースB: 0はノイズなので非表示にすべき
- **結論**: 設定で制御できるべき

#### removeNulls機能（既存機能）

**場所**: `plugin-chart-echarts/src/utils/series.ts:267`

```typescript
.filter(obs => !removeNulls || (obs[0] !== null && obs[1] !== null))
```

**動作**:
- データポイント自体からnullを削除
- ツールチップ表示前の段階で処理

**Issue #4462との違い**:
- removeNulls: **データレベル**でnullを削除（チャート描画から除外）
- Issue #4462の要望: **ツールチップレベル**で0/nullを非表示（データは残す、表示のみ制御）

### 5.5 2025年のロードマップと実装予定

**公式実装の状況**:
- ❌ 2025年のロードマップに含まれていない
- ❌ 実装予定なし
- ❌ コミュニティからのPR待ち

---

## 6. 解決策と回避策

### 6.1 短期（即時対応可能）

#### 案A: SQLクエリで0値を除外（推奨）

```sql
-- 0値を持つレコードをクエリ時に除外
SELECT
  timestamp,
  metric_name,
  metric_value
FROM metrics
WHERE metric_value != 0  -- 0を除外
  AND metric_value IS NOT NULL  -- nullを除外
```

**メリット**:
- ✅ コード変更不要
- ✅ 即座に利用可能
- ✅ パフォーマンス向上（データ転送量削減）
- ✅ ツールチップが自動的に軽くなる

**デメリット**:
- ❌ 「値が0」と「データなし」の区別ができなくなる
- ❌ すべてのクエリに手動で追加する必要がある
- ❌ チャートには0が表示されなくなる（ツールチップだけでなく全体に影響）

#### 案B: itemモードを使用

**設定**: Rich tooltip チェックボックスをOFF

**メリット**:
- ✅ 設定変更のみで即座に対応可能
- ✅ シリーズ数が多くても問題ない
- ✅ ツールチップがシンプルになる

**デメリット**:
- ❌ 複数シリーズを同時に比較できない
- ❌ 各シリーズに個別にマウスオーバーが必要

#### 案C: シリーズ数を削減

**方法**:
- 必要なメトリクスのみをチャートに含める
- フィルターを活用して不要なシリーズを除外
- 集計レベルを上げる（日次→週次など）

**メリット**:
- ✅ 即座に対応可能
- ✅ チャート全体が見やすくなる

**デメリット**:
- ❌ 必要な情報が失われる可能性
- ❌ 根本的な解決にはならない

### 6.2 中期（機能追加の検討）

#### セルフ実装: axisモード + 0値非表示の複合機能

**概要**: 「axisモードだけど、0値とnull値は非表示にする」機能を実装

**工数見積もり**: 1-2日（テスト含む）

**難易度**: 低（コードの変更箇所が明確）

**破壊的変更**: なし（デフォルトOFFにすれば既存の動作を維持）

詳細は[セクション7](#7-実装ガイドaxisモード--0値非表示の複合機能)を参照

### 6.3 長期（根本的な解決）

#### コミュニティへの貢献

**手順**:
1. セルフ実装してテスト
2. GitHub Issueを再開（Issue #4462にコメント、または新しいIssueを作成）
3. Pull Requestを作成
4. コミュニティレビューを受ける
5. 本家にマージされる

**メリット**:
- ✅ アップグレード時のマージ不要（公式機能として組み込まれる）
- ✅ 他のユーザーも利用可能
- ✅ コミュニティへの貢献

**デメリット**:
- ❌ レビュープロセスに時間がかかる（数週間～数ヶ月）
- ❌ 設計変更を求められる可能性
- ❌ 最終的にマージされない可能性もある

**工数見積もり**:
- 実装: 1-2日
- テスト: 1日
- ドキュメント作成: 1日
- PRレビュー対応: 1-2週間（断続的）
- **合計**: 約2-3週間

### 6.4 コストベネフィット分析

| アプローチ | 工数 | 即効性 | 持続性 | 推奨度 |
|-----------|------|--------|--------|--------|
| SQLで除外 | 1時間 | ⭐⭐⭐⭐⭐ | ⭐⭐ | 短期的 |
| itemモード | 5分 | ⭐⭐⭐⭐⭐ | ⭐⭐⭐ | 簡易的 |
| セルフ実装 | 2日 | ⭐⭐⭐⭐ | ⭐⭐⭐⭐ | **推奨** |
| PR貢献 | 2-3週間 | ⭐⭐ | ⭐⭐⭐⭐⭐ | 理想的 |

---

## 7. 実装ガイド：axisモード + 0値非表示の複合機能

### 7.1 実装の全体像

**要件**: 「axisモードで全シリーズを表示するが、0値とnull値は非表示にする」

**実装方針**:
1. 新しい設定を追加: `tooltipHideZeroValues`, `tooltipHideNullValues`
2. formatter関数内で判定: 値が0またはnullの場合、ツールチップに含めない
3. UIに設定を追加: Chart Options → Tooltip セクションにチェックボックス

### 7.2 Step 1: 型定義の追加

**ファイル**: `plugin-chart-echarts/src/MixedTimeseries/types.ts`

```typescript
export interface EchartsMixedTimeseriesFormData extends TimeseriesDefaultFormData {
  // ... 既存のプロパティ ...
  richTooltip: boolean;

  // 新規追加
  tooltipHideZeroValues?: boolean;  // 0値を非表示
  tooltipHideNullValues?: boolean;  // null値を非表示
}

export const DEFAULT_FORM_DATA: EchartsMixedTimeseriesFormData = {
  // ... 既存のデフォルト値 ...
  richTooltip: true,

  // 新規追加
  tooltipHideZeroValues: false,  // デフォルトOFF（既存動作を維持）
  tooltipHideNullValues: false,
};
```

### 7.3 Step 2: transformProps.tsの修正

**ファイル**: `plugin-chart-echarts/src/MixedTimeseries/transformProps.ts`

#### 変更箇所1: formDataから設定を取得（191行目付近）

```typescript
const {
  // ... 既存のプロパティ ...
  richTooltip,
  tooltipSortByMetric,

  // 新規追加
  tooltipHideZeroValues = false,
  tooltipHideNullValues = false,

  // ... その他 ...
}: EchartsMixedTimeseriesFormData = { ...DEFAULT_FORM_DATA, ...formData };
```

#### 変更箇所2: formatter関数内でフィルタリング（605-644行目）

**修正前**:
```typescript
sortedKeys
  .filter(key => keys.includes(key))
  .forEach(key => {
    const value = forecastValues[key];
    // ... フォーマッター選択 ...
    const row = formatForecastTooltipSeries({
      ...value,
      seriesName: key,
      formatter: primarySeries.has(key)
        ? tooltipFormatter
        : tooltipFormatterSecondary,
    });
    rows.push(row);
    if (key === focusedSeries) {
      focusedRow = rows.length - 1;
    }
  });
```

**修正後**:
```typescript
sortedKeys
  .filter(key => keys.includes(key))
  .forEach(key => {
    const value = forecastValues[key];

    // 新規追加: 0値とnull値のフィルタリング
    const shouldHideZero = tooltipHideZeroValues && value.observation === 0;
    const shouldHideNull = tooltipHideNullValues &&
                          (value.observation === null || value.observation === undefined);

    if (shouldHideZero || shouldHideNull) {
      return; // このシリーズをスキップ
    }

    // ... 既存のフォーマッター選択 ...
    const row = formatForecastTooltipSeries({
      ...value,
      seriesName: key,
      formatter: primarySeries.has(key)
        ? tooltipFormatter
        : tooltipFormatterSecondary,
    });
    rows.push(row);
    if (key === focusedSeries) {
      focusedRow = rows.length - 1;
    }
  });
```

### 7.4 Step 3: controlPanel.tsxにUI追加

**ファイル**: `plugin-chart-echarts/src/MixedTimeseries/controlPanel.tsx`

#### richTooltipSectionの後に追加（322行目付近）

```typescript
{
  label: t('Chart Options'),
  expanded: true,
  controlSetRows: [
    // ... 既存の設定 ...
    [xAxisLabelRotation],
    ...richTooltipSection,

    // 新規追加: 0値・null値非表示の設定
    [
      {
        name: 'tooltipHideZeroValues',
        config: {
          type: 'CheckboxControl',
          label: t('Hide zero values in tooltip'),
          renderTrigger: true,
          default: false,
          description: t(
            'When enabled, series with zero values will not be shown in the tooltip'
          ),
          visibility: ({ controls }: ControlPanelsContainerProps) =>
            Boolean(controls?.rich_tooltip?.value),  // Rich tooltip ONの時のみ表示
        },
      },
    ],
    [
      {
        name: 'tooltipHideNullValues',
        config: {
          type: 'CheckboxControl',
          label: t('Hide null values in tooltip'),
          renderTrigger: true,
          default: false,
          description: t(
            'When enabled, series with null/undefined values will not be shown in the tooltip'
          ),
          visibility: ({ controls }: ControlPanelsContainerProps) =>
            Boolean(controls?.rich_tooltip?.value),  // Rich tooltip ONの時のみ表示
        },
      },
    ],

    // ... Y軸の設定 ...
  ],
}
```

### 7.5 動作イメージ

**設定UI**:

```
┌─────────────────────────────────────────────┐
│ Tooltip                                     │
├─────────────────────────────────────────────┤
│ ☑ Rich tooltip                              │
│   Shows a list of all series at that point  │
│                                             │
│ ☑ Hide zero values in tooltip               │  ← 新規追加
│   When enabled, series with zero values     │
│   will not be shown in the tooltip          │
│                                             │
│ ☑ Hide null values in tooltip               │  ← 新規追加
│   When enabled, series with null/undefined  │
│   values will not be shown in the tooltip   │
│                                             │
│ ☐ Sort by metric                            │
│ Time format: [Smart Date]                   │
└─────────────────────────────────────────────┘
```

**表示例**:

データ:
```
2021-01-15時点:
- Series A: 1,234
- Series B: 0        ← 0値
- Series C: null     ← null値
- Series D: 5,678
```

設定パターン1: Hide zero values ON
```
Rich tooltip: ON
Hide zero values: ON
Hide null values: OFF

ツールチップ:
┌──────────────────────────────────┐
│ 2021-01-15                       │
├──────────────────────────────────┤
│ ● Series A          1,234        │
│ ● Series D          5,678        │
└──────────────────────────────────┘
← Series Bが非表示になる
```

設定パターン2: 両方ON
```
Rich tooltip: ON
Hide zero values: ON
Hide null values: ON

ツールチップ:
┌──────────────────────────────────┐
│ 2021-01-15                       │
├──────────────────────────────────┤
│ ● Series A          1,234        │
│ ● Series D          5,678        │
└──────────────────────────────────┘
← Series BとCが非表示になる
```

### 7.6 テストケース

```typescript
describe('MixedTimeseries - Hide zero/null values in tooltip', () => {
  const mockData = [
    { __timestamp: 1, series1: 100, series2: 0, series3: null },
    { __timestamp: 2, series1: 200, series2: 50, series3: 0 },
  ];

  it('デフォルトでは0値も表示される', () => {
    const props = transformProps({
      formData: {
        richTooltip: true,
        tooltipHideZeroValues: false,
        tooltipHideNullValues: false,
      },
      queriesData: [{ data: mockData }],
    });

    const tooltipHtml = props.echartOptions.tooltip.formatter({
      // ... mockパラメータ
    });

    expect(tooltipHtml).toContain('series2');  // 0値のシリーズも表示
  });

  it('Hide zero values ONで0値が非表示', () => {
    const props = transformProps({
      formData: {
        richTooltip: true,
        tooltipHideZeroValues: true,  // ON
        tooltipHideNullValues: false,
      },
      queriesData: [{ data: mockData }],
    });

    const tooltipHtml = props.echartOptions.tooltip.formatter({
      // timestamp=1のパラメータ
    });

    expect(tooltipHtml).not.toContain('series2');  // series2は0なので非表示
    expect(tooltipHtml).toContain('series1');      // series1は100なので表示
  });

  it('両方ONで0とnullが非表示', () => {
    const props = transformProps({
      formData: {
        richTooltip: true,
        tooltipHideZeroValues: true,   // ON
        tooltipHideNullValues: true,   // ON
      },
      queriesData: [{ data: mockData }],
    });

    const tooltipHtml = props.echartOptions.tooltip.formatter({
      // timestamp=1のパラメータ
    });

    expect(tooltipHtml).not.toContain('series2');  // 0なので非表示
    expect(tooltipHtml).not.toContain('series3');  // nullなので非表示
    expect(tooltipHtml).toContain('series1');      // 100なので表示
  });
});
```

### 7.7 注意点

#### すべてのシリーズが非表示になった場合の対応

```typescript
// formatter関数の最後
if (rows.length === 0) {
  return '';  // 空文字列を返すとツールチップが表示されない
}
return tooltipHtml(rows, tooltipFormatter(xValue), focusedRow);
```

#### パフォーマンスへの影響

**懸念**: forEach内でのチェック処理が増えるため、パフォーマンスに影響があるか？

**結論**: 影響はほぼない

**理由**:
- チェック処理は単純な等価演算子（`===`）のみ
- シリーズ数が100個でも、1ms以下の処理時間
- ツールチップのHTMLレンダリングの方が遥かに重い

---

## 8. トラブルシューティング

### 8.1 ツールチップが大きすぎてチャートが見えない

**原因**: axisモードで多数のシリーズを表示している

**解決策**:
1. ✅ **itemモードに変更**（Rich tooltip OFF）
2. ✅ シリーズ数を減らす（フィルターや集計で）
3. ✅ SQLクエリで0値を除外（WHERE句）
4. ✅ 本ガイドの実装案を適用（Hide zero values機能）

### 8.2 itemモードでバーにマウスを合わせにくい

**原因**: バーが細い、またはポイントが小さい

**解決策**:
1. ✅ **axisモードに変更**（Rich tooltip ON）
2. ✅ Marker Sizeを大きくする（Line/Scatterの場合）
3. ✅ チャートのサイズを拡大

### 8.3 axisモードなのに1つのシリーズしか表示されない

**確認ポイント**:
1. Rich tooltip が本当にONになっているか確認
2. 他のシリーズに値がない（null）可能性
3. ブラウザの開発者ツールで`option.tooltip[0].trigger`を確認

**デバッグ**:

```javascript
// Console タブ
const chart = echarts.getInstanceByDom(document.querySelector('.echarts-for-react'));
console.log(chart.getOption().tooltip[0].trigger);
// "axis"と表示されるべき

// すべてのシリーズを確認
console.log(chart.getOption().series.map(s => s.name));
// すべてのシリーズ名が表示されるべき
```

### 8.4 0値が大量に表示される

**原因**: スタックモードONで欠損値が0に変換されている

**解決策**:
1. ✅ **itemモードに変更**
2. ✅ SQLクエリで0値を除外
3. ✅ スタックモードをOFFにする（欠損値が0に変換されなくなる）
4. ✅ 本ガイドの実装案を適用（Hide zero values機能）

### 8.5 スタックチャートで一部のシリーズが途切れる

**原因**: 系列の最初または最後に欠損値がある場合、fillNeighborValueでも埋められない

**対策**:
```sql
-- SQLクエリで系列の範囲を統一
SELECT
  date,
  COALESCE(metric_value, 0) as metric_value  -- 明示的に0に変換
FROM data
WHERE date BETWEEN start_date AND end_date  -- 全系列で同じ範囲
```

### 8.6 「データがない」と「値が0」を区別したい

**問題**: スタックモードONの場合、欠損値が0に変換されるため区別できない

**対策**:
1. **SQLクエリで明示的に除外**:
   ```sql
   WHERE metric_value IS NOT NULL
   ```

2. **スタックモードをOFFにする**:
   - 欠損値はギャップとして表示される
   - ただし、スタック表示ができなくなる

3. **バックエンドで特別な値を使う**:
   ```python
   # 欠損値を-1など特別な値に変換
   df['metric'].fillna(-1)
   ```
   - フロントエンドで-1を特別扱い
   - ただし、実装が必要

---

## 9. まとめと推奨事項

### 9.1 ツールチップ機能の全体像

**ツールチップ表示の仕組み**:
1. EChartsが検出: マウス位置のすべてのシリーズのデータポイント
2. 値を抽出: `extractForecastValuesFromTooltipParams()`
3. 整形: `formatForecastTooltipSeries()`
4. HTML生成: `tooltipHtml()`
5. 表示: ブラウザにレンダリング

**2つのモード**:
- **axis**: X軸上の同じ位置にあるすべてのシリーズを表示
- **item**: マウスオーバーした特定のシリーズのみを表示

**欠損値の扱い**:
- **スタックON**: 欠損値が0に変換される
- **スタックOFF**: 欠損値はnullのまま

**0値が表示される理由**:
- `typeof 0 === 'number'`が`true`
- EChartsの標準動作として、すべての数値データポイントを表示
- **これは仕様通りの挙動**

### 9.2 問題と解決策のマトリクス

| 問題 | 短期対応 | 中期対応 | 長期対応 |
|------|---------|---------|---------|
| **0値が大量に表示** | SQLで除外<br>itemモード使用 | Hide zero values実装 | コミュニティPR |
| **ツールチップが大きい** | itemモード使用<br>シリーズ削減 | Hide zero values実装 | - |
| **スタックで欠損値が0** | SQL COALESCE使用 | - | - |
| **データなしと0の区別** | スタックOFF<br>SQL IS NOT NULL | - | - |

### 9.3 推奨アプローチ

#### シナリオ1: 少数のチャートで0値が問題

**推奨**: SQLクエリで除外
- 工数: 1時間
- 即効性: ⭐⭐⭐⭐⭐
- 持続性: ⭐⭐

#### シナリオ2: 多数のチャートで0値が問題

**推奨**: セルフ実装（Hide zero values機能）
- 工数: 2日
- 即効性: ⭐⭐⭐⭐
- 持続性: ⭐⭐⭐⭐

#### シナリオ3: 公式機能として欲しい

**推奨**: セルフ実装 + コミュニティ貢献
- 工数: 2-3週間
- 即効性: ⭐⭐
- 持続性: ⭐⭐⭐⭐⭐

### 9.4 axis vs item モード選択基準

**axisモード（Rich tooltip ON）を選ぶ場合**:
- ✅ シリーズ数が少ない（5個以下）
- ✅ 各時点で複数メトリクスを比較したい
- ✅ データが密（ほとんどの時点に値あり）
- ✅ ピンポイントでマウスを合わせにくい高頻度データ

**itemモード（Rich tooltip OFF）を選ぶ場合**:
- ✅ シリーズ数が多い（10個以上）
- ✅ 特定シリーズのみに注目したい
- ✅ 0値が多くツールチップが肥大化する
- ✅ シンプルで軽量なツールチップが必要

### 9.5 Issue #4462の現状と見通し

**現状**:
- Issue #4462は2018年から存在するが、未解決のまま閉じられている
- 2025年のロードマップにも含まれていない
- 公式実装予定はなし

**将来の見通し**:
- 公式実装の可能性: 低い
- コミュニティ貢献によるマージ: 中程度（60-70%）

**最も現実的な選択肢**:
1. **短期**: SQLクエリで0値を除外
2. **中期**: セルフ実装（本ガイドの実装案）
3. **長期**: コミュニティ貢献（PR作成）

### 9.6 コードレベルでの重要箇所まとめ

| ファイル | 行番号 | 内容 |
|---------|--------|------|
| `transformProps.ts` | 226, 231 | `fillNeighborValue`の決定ロジック |
| `transformProps.ts` | 584 | ツールチップモード（axis/item）の切り替え |
| `transformProps.ts` | 605-644 | formatter関数（フィルタリング追加箇所） |
| `series.ts` | 54-56 | `isDefined()`関数の実装 |
| `series.ts` | 330-338 | fillNeighborValueでの埋め処理 |
| `forecast.ts` | 66-80 | `typeof numericValue === 'number'`チェック |
| `forecast.ts` | 99 | `typeof observation === 'number'`でのフォーマット |
| `controls.tsx` | 175-186 | richTooltipControl定義 |
| `controlPanel.tsx` | 322 | richTooltipSection配置 |

### 9.7 最終推奨事項

**即座に対応が必要な場合**:
1. シリーズ数が少ない → SQLクエリで0値除外
2. シリーズ数が多い → itemモード使用

**恒久的な解決を目指す場合**:
1. セルフ実装（本ガイドのStep 7を参照）
2. 社内環境でテスト運用
3. 必要に応じてコミュニティ貢献を検討

**実装の優先度**:
- Phase 1: MVP（tooltipHideZeroValuesのみ）- 1-2日
- Phase 2: 拡張（tooltipHideNullValues追加）- 2-3日
- Phase 3: コミュニティ提案（PR作成）- 1-2日

**成功のポイント**:
1. ✅ 明確なユースケース
2. ✅ 十分なテストコード
3. ✅ ドキュメント
4. ✅ 後方互換性の維持（デフォルトOFF）
5. ✅ コミュニティの支持

---

## 参考リンク

### Issue・PR
- [Issue #4462 - Tooltip Option](https://github.com/apache/incubator-superset/issues/4462)
- [PR #21296 - Show zero value in tooltip](https://github.com/apache/superset/pull/21296)
- [Discussion #33042 - Remove nulls option](https://github.com/apache/superset/discussions/33042)

### ロードマップ
- [公式ロードマップ](https://github.com/apache-superset/superset-roadmap)

### ECharts
- [ECharts Tooltip Documentation](https://echarts.apache.org/en/option.html#tooltip)

---

**調査者**: Claude Code
**最終更新**: 2026-04-01
**レビュー推奨**: フロントエンド開発チーム、UX/UIチーム、プロダクトマネージャー
