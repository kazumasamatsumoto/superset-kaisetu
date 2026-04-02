# Mixed Chart 日付フォーマット不具合 - 包括的ガイド

**対象**: Apache Superset Mixed Timeseries Chart
**作成日**: 2026-04-02
**最終更新**: 2026-04-02

---

## はじめに

このドキュメントは、Mixed Timeseries Chartにおける日付フォーマット不具合について、概要から技術的詳細、修正方法まで、すべての情報を統合した包括的なガイドです。

**問題**: Query Aが空の場合、X軸の日時がタイムスタンプ数値（例: `1609459200000`）として表示される

**解決策**: `transformProps.ts`の2行を修正（既存変数の活用）

---

## 目次

### パート1: 概要
1. [問題の概要](#1-問題の概要)
2. [発生条件](#2-発生条件)
3. [症状](#3-症状)
4. [根本原因](#4-根本原因)
5. [解決策のサマリー](#5-解決策のサマリー)

### パート2: 技術詳細
6. [日付フォーマット決定の全フロー](#6-日付フォーマット決定の全フロー)
7. [各ステップの詳細解説](#7-各ステップの詳細解説)
8. [問題発生のメカニズム](#8-問題発生のメカニズム)
9. [データフロー図](#9-データフロー図)
10. [関連する型定義とインターフェース](#10-関連する型定義とインターフェース)

### パート3: 修正ガイド
11. [修正前の準備](#11-修正前の準備)
12. [修正手順](#12-修正手順)
13. [テスト方法](#13-テスト方法)
14. [デプロイ手順](#14-デプロイ手順)
15. [ロールバック手順](#15-ロールバック手順)
16. [トラブルシューティング](#16-トラブルシューティング)

### 付録
17. [よくある質問](#17-よくある質問)
18. [クイックリファレンス](#18-クイックリファレンス)

---

# パート1: 概要

## 1. 問題の概要

Mixed Timeseries Chartにおいて、**Query A（最初のクエリ）が空データを返す場合**、X軸の日時が正しくフォーマットされず、タイムスタンプの数値（例: `1609459200000`）がそのまま表示される不具合が発生しています。

### 視覚的な説明

**正常時（Query Aにデータがある）**:

```
X軸: 2021-01-01  2021-01-02  2021-01-03
```

**不具合発生時（Query Aが空）**:

```
X軸: 1609459200000  1609545600000  1609632000000
     ↑ タイムスタンプが数値のまま表示される
```

---

## 2. 発生条件

以下のすべての条件を満たす場合に発生します：

1. ✅ **Mixed Timeseries Chart を使用**
2. ✅ **Query A（最初のクエリ）が空データを返す**
3. ✅ **Query B（2番目のクエリ）に有効なデータが存在**
4. ✅ **X軸が日時カラム（Temporal型）**

### 具体例

```sql
-- Query A（空になるクエリ）
SELECT
  timestamp_column,
  metric_a
FROM table_a
WHERE condition = 'no_match'  -- データが返らない

-- Query B（データを返すクエリ）
SELECT
  timestamp_column,
  metric_b
FROM table_b
WHERE condition = 'has_data'  -- データが返る
```

**結果**:

- Query A: 0行（空）
- Query B: 100行（データあり）
- **症状**: X軸の日時が数値表示される

---

## 3. 症状

### 3.1 ユーザーから見た症状

**チャート画面**:

```
┌─────────────────────────────────────────┐
│ Mixed Timeseries Chart                  │
├─────────────────────────────────────────┤
│                           ■             │
│                     ■                   │
│               ■                         │
│         ■                               │
└─────────────────────────────────────────┘
  1609459200000  1609545600000  1609632000000
  ↑ 本来は "2021-01-01" などと表示されるべき
```

**ツールチップ**:

```
┌──────────────────────┐
│ 1609459200000        │  ← 日付フォーマットされていない
├──────────────────────┤
│ ● Metric B    100    │
└──────────────────────┘
```

### 3.2 システムログ（該当なし）

- エラーログは出力されません
- JavaScriptコンソールにも警告は表示されません
- **見た目だけの問題として顕在化**するため、気づきにくい

---

## 4. 根本原因

### 4.1 単純な説明

Mixed Chartは内部で2つのクエリ（Query A / Query B）を実行しますが、**X軸のデータ型を判定する際にQuery Aの型情報のみを参照**しているため、Query Aが空の場合に型情報を取得できず、日時フォーマットが適用されません。

### 4.2 コードレベルの原因

**ファイル**: `superset-frontend/plugins/plugin-chart-echarts/src/MixedTimeseries/transformProps.ts`

#### 問題の箇所（235-236行目）

```typescript
const dataTypes = getColtypesMapping(queriesData[0]); // ← Query Aのみ参照
const xAxisDataType = dataTypes?.[xAxisLabel] ?? dataTypes?.[xAxisOrig];
```

**動作**:

1. `queriesData[0]`（Query A）から型情報を取得
2. Query Aが空の場合 → `dataTypes = {}`（空オブジェクト）
3. `xAxisDataType = undefined`
4. 日付フォーマッターが適用されない

#### 既に存在する正しいコード（使われていない）

**147-150行目**:

```typescript
const coltypeMapping = {
  ...getColtypesMapping(queriesData[0]), // Query A
  ...getColtypesMapping(queriesData[1]), // Query B
};
```

**重要**: このコードは**Query AとQuery Bの両方から型情報をマージ**していますが、`xAxisDataType`の判定には使用されていません。

### 4.3 なぜこの問題が発生するのか

**設計の想定**:

- Mixed Chartの開発時、通常は両方のクエリにデータがあることを想定
- Query Aのみから型情報を取得する実装になった

**実際の利用シーン**:

- フィルター条件によってQuery Aが空になるケースがある
- 時系列で特定期間のみデータがあるケース
- A/Bテストで片方のグループにデータがないケース

**結果**:

- 想定外のケースに対応できていなかった

---

## 5. 解決策のサマリー

### 5.1 修正内容（最小限）

**対象ファイル**: `transformProps.ts:235-236`

#### 修正前

```typescript
const dataTypes = getColtypesMapping(queriesData[0]);
const xAxisDataType = dataTypes?.[xAxisLabel] ?? dataTypes?.[xAxisOrig];
```

#### 修正後

```typescript
// const dataTypes = getColtypesMapping(queriesData[0]); // 削除
const xAxisDataType =
  coltypeMapping?.[xAxisLabel] ?? coltypeMapping?.[xAxisOrig];
```

**変更点**:

- `dataTypes`変数を削除
- 既存の`coltypeMapping`（両クエリマージ済み）を使用

### 5.2 修正の効果

**修正前の動作**:

```
Query A: 空（型情報なし）
   ↓
dataTypes = {}
   ↓
xAxisDataType = undefined
   ↓
フォーマッター = String
   ↓
タイムスタンプが数値表示
```

**修正後の動作**:

```
Query A: 空（型情報なし）
Query B: データあり（型情報あり）
   ↓
coltypeMapping = { __timestamp: GenericDataType.Temporal, ... }
   ↓
xAxisDataType = GenericDataType.Temporal
   ↓
フォーマッター = getXAxisFormatter()
   ↓
日時フォーマット適用 ✅
```

### 5.3 リスク評価

| 項目                 | 評価        | 理由                     |
| -------------------- | ----------- | ------------------------ |
| **破壊的変更**       | ❌ なし     | 既存変数の再利用のため   |
| **パフォーマンス**   | ✅ 影響なし | 計算量は同じ             |
| **テストカバレッジ** | ⚠️ 追加推奨 | 新しいテストケースが必要 |
| **後方互換性**       | ✅ 保持     | 既存の動作を維持         |

**総合評価**: 🟢 **低リスク**

---

# パート2: 技術詳細

## 6. 日付フォーマット決定の全フロー

### 6.1 フロー概要

```
┌─────────────────────────────────────────────────────────────────┐
│ Step 1: クエリ実行とデータ取得                                    │
│ - Query A実行 → queriesData[0]                                  │
│ - Query B実行 → queriesData[1]                                  │
└─────────────┬───────────────────────────────────────────────────┘
              ↓
┌─────────────────────────────────────────────────────────────────┐
│ Step 2: transformProps関数の開始                                 │
│ - formData, queriesData, datasource等を受け取る                  │
└─────────────┬───────────────────────────────────────────────────┘
              ↓
┌─────────────────────────────────────────────────────────────────┐
│ Step 3: coltypeMapping作成（147-150行目）                        │
│ - Query AとQuery Bの型情報をマージ                              │
│ - 正しく両方を考慮している                                      │
└─────────────┬───────────────────────────────────────────────────┘
              ↓
┌─────────────────────────────────────────────────────────────────┐
│ Step 4: xAxisDataType取得（235-236行目）⚠️ 問題箇所              │
│ - Query Aのみから型情報を取得                                    │
│ - coltypeMappingを使用していない                                 │
└─────────────┬───────────────────────────────────────────────────┘
              ↓
┌─────────────────────────────────────────────────────────────────┐
│ Step 5: フォーマッター決定（476-483行目）                         │
│ - xAxisDataType === GenericDataType.Temporal ?                  │
│   YES → getXAxisFormatter() / getTooltipTimeFormatter()        │
│   NO  → String                                                  │
└─────────────┬───────────────────────────────────────────────────┘
              ↓
┌─────────────────────────────────────────────────────────────────┐
│ Step 6: EChartsオプション設定（509-535行目）                      │
│ - xAxis.axisLabel.formatter = xAxisFormatter                    │
│ - tooltip.formatter内でtooltipFormatterを使用                   │
└─────────────┬───────────────────────────────────────────────────┘
              ↓
┌─────────────────────────────────────────────────────────────────┐
│ Step 7: チャート描画                                             │
│ - EChartsがフォーマッターを使用してX軸ラベルを描画              │
└─────────────────────────────────────────────────────────────────┘
```

---

## 7. 各ステップの詳細解説

### Step 1: クエリ実行とデータ取得

**役割**: バックエンドからデータと型情報を取得

**データ構造**:
```typescript
queriesData = [
  // Query A
  {
    data: [{ __timestamp: 1609459200000, metric_a: 100 }, ...],
    colnames: ['__timestamp', 'metric_a'],
    coltypes: [GenericDataType.Temporal, GenericDataType.Numeric],
    label_map: { ... }
  },
  // Query B
  {
    data: [{ __timestamp: 1609459200000, metric_b: 50 }, ...],
    colnames: ['__timestamp', 'metric_b'],
    coltypes: [GenericDataType.Temporal, GenericDataType.Numeric],
    label_map: { ... }
  }
]
```

**Query Aが空の場合**:
```typescript
queriesData = [
  // Query A（空）
  {
    data: [],           // ← 空配列
    colnames: [],       // ← 空配列
    coltypes: [],       // ← 空配列
    label_map: {}
  },
  // Query B（データあり）
  {
    data: [{ __timestamp: 1609459200000, metric_b: 50 }, ...],
    colnames: ['__timestamp', 'metric_b'],
    coltypes: [GenericDataType.Temporal, GenericDataType.Numeric],
    label_map: { ... }
  }
]
```

---

### Step 2: transformProps関数の開始

**ファイル**: `superset-frontend/plugins/plugin-chart-echarts/src/MixedTimeseries/transformProps.ts:117-132`

```typescript
export default function transformProps(
  chartProps: EchartsMixedTimeseriesProps,
): EchartsMixedTimeseriesChartTransformedProps {
  const {
    width,
    height,
    formData,
    queriesData,    // ← Step 1から受け取ったデータ
    hooks,
    filterState,
    datasource,
    theme,
    inContextMenu,
    emitCrossFilters,
  } = chartProps;

  // ... 処理が続く
}
```

**入力パラメータ**:
- `formData`: チャート設定（ユーザーがUIで設定した値）
- `queriesData`: クエリ結果（Step 1で取得したデータ）
- `datasource`: データソース情報

---

### Step 3: coltypeMapping作成（正しい実装）

**ファイル**: `transformProps.ts:144-150`

```typescript
const data1 = (queriesData[0].data || []) as TimeseriesDataRecord[];
const data2 = (queriesData[1].data || []) as TimeseriesDataRecord[];
const annotationData = getAnnotationData(chartProps);

// ✅ 両方のクエリから型情報を取得してマージ
const coltypeMapping = {
  ...getColtypesMapping(queriesData[0]),  // Query A
  ...getColtypesMapping(queriesData[1]),  // Query B
};
```

**getColtypesMapping関数の実装**:

**ファイル**: `utils/series.ts:391-401`

```typescript
export const getColtypesMapping = ({
  coltypes = [],
  colnames = [],
}: Pick<ChartDataResponseResult, 'coltypes' | 'colnames'>): Record<
  string,
  GenericDataType
> =>
  colnames.reduce(
    (accumulator, item, index) => ({ ...accumulator, [item]: coltypes[index] }),
    {},
  );
```

**動作例**:

```typescript
// Query Aが空の場合
getColtypesMapping(queriesData[0])
// Input: { colnames: [], coltypes: [] }
// Output: {}

// Query Bにデータがある場合
getColtypesMapping(queriesData[1])
// Input: {
//   colnames: ['__timestamp', 'metric_b'],
//   coltypes: [GenericDataType.Temporal, GenericDataType.Numeric]
// }
// Output: {
//   __timestamp: GenericDataType.Temporal,  // = 2
//   metric_b: GenericDataType.Numeric       // = 0
// }

// マージ結果
const coltypeMapping = {
  ...{},  // Query A（空）
  ...{ __timestamp: 2, metric_b: 0 }  // Query B
}
// = { __timestamp: 2, metric_b: 0 }
```

**重要**: この時点で、`coltypeMapping`は正しくQuery Bの型情報を持っています！

---

### Step 4: xAxisDataType取得（⚠️ 問題箇所）

**ファイル**: `transformProps.ts:235-236`

```typescript
// ⚠️ 問題: Query Aのみから型情報を取得
const dataTypes = getColtypesMapping(queriesData[0]);
const xAxisDataType = dataTypes?.[xAxisLabel] ?? dataTypes?.[xAxisOrig];
```

**Query Aが空の場合の動作**:

```typescript
// Step 1
const dataTypes = getColtypesMapping(queriesData[0]);
// queriesData[0] = { colnames: [], coltypes: [] }
// dataTypes = {}

// Step 2
const xAxisDataType = dataTypes?.[xAxisLabel] ?? dataTypes?.[xAxisOrig];
// xAxisLabel = '__timestamp'
// dataTypes['__timestamp'] = undefined  （空オブジェクトなので）
// xAxisOrig = '__timestamp'
// dataTypes['__timestamp'] = undefined
// 結果: xAxisDataType = undefined
```

**なぜcoltypeMappingを使わないのか？**

→ **おそらく実装ミス**。coltypeMappingが存在することを見落としたか、後から追加された可能性があります。

**修正案**:

```typescript
// dataTypes変数を削除し、既存のcoltypeMappingを使用
const xAxisDataType = coltypeMapping?.[xAxisLabel] ?? coltypeMapping?.[xAxisOrig];
```

**修正後の動作**:

```typescript
// coltypeMapping = { __timestamp: 2, metric_b: 0 }
const xAxisDataType = coltypeMapping['__timestamp'] ?? coltypeMapping['__timestamp'];
// coltypeMapping['__timestamp'] = GenericDataType.Temporal = 2
// 結果: xAxisDataType = 2  ✅ 正しく取得できる
```

---

### Step 5: フォーマッター決定

**ファイル**: `transformProps.ts:476-483`

```typescript
const tooltipFormatter =
  xAxisDataType === GenericDataType.Temporal
    ? getTooltipTimeFormatter(tooltipTimeFormat)
    : String;

const xAxisFormatter =
  xAxisDataType === GenericDataType.Temporal
    ? getXAxisFormatter(xAxisTimeFormat)
    : String;
```

**GenericDataType.Temporalの定義**:

**ファイル**: `superset-frontend/packages/superset-ui-core/src/query/types/QueryResponse.ts:26-31`

```typescript
export enum GenericDataType {
  Numeric = 0,
  String = 1,
  Temporal = 2,  // ← 時刻型
  Boolean = 3,
}
```

**条件分岐の詳細**:

```typescript
// ケース1: xAxisDataType = 2（Temporal）の場合
xAxisDataType === GenericDataType.Temporal
// 2 === 2 → true
// → tooltipFormatter = getTooltipTimeFormatter(tooltipTimeFormat)
// → xAxisFormatter = getXAxisFormatter(xAxisTimeFormat)

// ケース2: xAxisDataType = undefined の場合（Query A空のバグケース）
xAxisDataType === GenericDataType.Temporal
// undefined === 2 → false
// → tooltipFormatter = String
// → xAxisFormatter = String
```

**フォーマッター関数の実装**:

**ファイル**: `utils/formatters.ts:77-99`

```typescript
export function getTooltipTimeFormatter(
  format?: string,
): TimeFormatter | StringConstructor {
  if (format === SMART_DATE_ID) {
    return getSmartDateVerboseFormatter();
  }
  if (format) {
    return getTimeFormatter(format);
  }
  return String;  // ← フォーマット指定がない場合
}

export function getXAxisFormatter(
  format?: string,
): TimeFormatter | StringConstructor | undefined {
  if (format === SMART_DATE_ID || !format) {
    return undefined;  // ← EChartsのデフォルトフォーマットを使用
  }
  if (format) {
    return getTimeFormatter(format);
  }
  return String;
}
```

**String関数の動作**:

```typescript
String(1609459200000)  // → "1609459200000"
String(new Date(1609459200000))  // → "Fri Jan 01 2021 09:00:00 GMT+0900 (日本標準時)"
```

**getTimeFormatter()の動作**:

```typescript
const formatter = getTimeFormatter('%Y-%m-%d');
formatter(1609459200000)  // → "2021-01-01"
```

**問題の動作**:

```typescript
// xAxisDataType = undefined の場合
const xAxisFormatter = String;

// X軸ラベルの生成
xAxisFormatter(1609459200000)  // → "1609459200000"
// ユーザーには数値が表示される
```

---

### Step 6: EChartsオプション設定

**ファイル**: `transformProps.ts:509-535`

```typescript
const echartOptions: EChartsCoreOption = {
  useUTC: true,
  grid: {
    ...defaultGrid,
    ...chartPadding,
  },
  xAxis: {
    type: xAxisType,
    name: xAxisTitle,
    nameGap: convertInteger(xAxisTitleMargin),
    nameLocation: 'middle',
    axisLabel: {
      formatter: xAxisFormatter,  // ← Step 5で決定したフォーマッター
      rotate: xAxisLabelRotation,
    },
    minorTick: { show: minorTicks },
    minInterval:
      xAxisType === AxisType.Time && timeGrainSqla
        ? TIMEGRAIN_TO_TIMESTAMP[
            timeGrainSqla as keyof typeof TIMEGRAIN_TO_TIMESTAMP
          ]
        : 0,
    ...getMinAndMaxFromBounds(...),
  },
  // ... yAxis, tooltip等の設定
}
```

**tooltip設定（585-646行目）**:

```typescript
tooltip: {
  ...getDefaultTooltip(refs),
  show: !inContextMenu,
  trigger: richTooltip ? 'axis' : 'item',
  formatter: (params: any) => {
    const xValue: number = richTooltip
      ? params[0].value[0]
      : params.value[0];

    // ... 中略 ...

    // tooltipFormatterを使用してX軸値をフォーマット
    return tooltipHtml(rows, tooltipFormatter(xValue), focusedRow);
    //                         ↑ Step 5で決定したtooltipFormatter
  },
}
```

---

### Step 7: チャート描画

**EChartsの内部処理**:

1. `echartOptions`を受け取る
2. `xAxis.axisLabel.formatter`を使用してX軸ラベルを生成
3. `tooltip.formatter`を使用してツールチップを生成
4. Canvas上に描画

**問題が顕在化するタイミング**:

```
ECharts内部処理:
  ↓
X軸ラベル生成: xAxisFormatter(1609459200000)
  ↓
String(1609459200000) = "1609459200000"  ← 数値として表示
  ↓
Canvas描画
```

---

## 8. 問題発生のメカニズム

### 8.1 正常ケース（Query Aにデータがある）

```
┌─────────────────────────────────────────┐
│ Query A実行                             │
│ - colnames: ['__timestamp', 'metric_a'] │
│ - coltypes: [2, 0]                      │
└────────────┬────────────────────────────┘
             ↓
┌────────────────────────────────────────────┐
│ Step 3: coltypeMapping作成                │
│ {                                          │
│   __timestamp: 2,                          │
│   metric_a: 0,                             │
│   metric_b: 0  // Query Bからマージ        │
│ }                                          │
└────────────┬───────────────────────────────┘
             ↓
┌────────────────────────────────────────────┐
│ Step 4: xAxisDataType取得                  │
│ dataTypes = getColtypesMapping(A)          │
│ = { __timestamp: 2, metric_a: 0 }          │
│                                            │
│ xAxisDataType = dataTypes['__timestamp']   │
│ = 2  ✅                                    │
└────────────┬───────────────────────────────┘
             ↓
┌────────────────────────────────────────────┐
│ Step 5: フォーマッター決定                  │
│ 2 === GenericDataType.Temporal → true     │
│                                            │
│ xAxisFormatter = getXAxisFormatter(...)    │
│ ✅ 日付フォーマッターが設定される           │
└────────────┬───────────────────────────────┘
             ↓
         正しく表示
```

### 8.2 バグケース（Query Aが空）

```
┌─────────────────────────────────────────┐
│ Query A実行（空）                        │
│ - colnames: []                           │
│ - coltypes: []                           │
└────────────┬────────────────────────────┘
             ↓
┌────────────────────────────────────────────┐
│ Step 3: coltypeMapping作成                │
│ {                                          │
│   __timestamp: 2,  // Query Bから取得 ✅   │
│   metric_b: 0                              │
│ }                                          │
└────────────┬───────────────────────────────┘
             ↓
┌────────────────────────────────────────────┐
│ Step 4: xAxisDataType取得 ⚠️ 問題          │
│ dataTypes = getColtypesMapping(A)          │
│ = {}  ← 空オブジェクト                     │
│                                            │
│ xAxisDataType = dataTypes['__timestamp']   │
│ = undefined  ❌                            │
└────────────┬───────────────────────────────┘
             ↓
┌────────────────────────────────────────────┐
│ Step 5: フォーマッター決定                  │
│ undefined === GenericDataType.Temporal     │
│ → false                                    │
│                                            │
│ xAxisFormatter = String                    │
│ ❌ 数値変換関数が設定される                 │
└────────────┬───────────────────────────────┘
             ↓
    タイムスタンプが数値表示
```

### 8.3 修正後の動作（Query Aが空でも正常）

```
┌─────────────────────────────────────────┐
│ Query A実行（空）                        │
│ - colnames: []                           │
│ - coltypes: []                           │
└────────────┬────────────────────────────┘
             ↓
┌────────────────────────────────────────────┐
│ Step 3: coltypeMapping作成                │
│ {                                          │
│   __timestamp: 2,  // Query Bから取得 ✅   │
│   metric_b: 0                              │
│ }                                          │
└────────────┬───────────────────────────────┘
             ↓
┌────────────────────────────────────────────┐
│ Step 4: xAxisDataType取得（修正後）         │
│ // dataTypesは削除                         │
│ xAxisDataType = coltypeMapping['__timestamp']│
│ = 2  ✅                                    │
└────────────┬───────────────────────────────┘
             ↓
┌────────────────────────────────────────────┐
│ Step 5: フォーマッター決定                  │
│ 2 === GenericDataType.Temporal → true     │
│                                            │
│ xAxisFormatter = getXAxisFormatter(...)    │
│ ✅ 日付フォーマッターが設定される           │
└────────────┬───────────────────────────────┘
             ↓
         正しく表示 ✅
```

---

## 9. データフロー図

### 9.1 現在の実装（バグあり）

```
┌──────────────┐
│ Backend      │
│              │
│ Query A: 空  │────┐
│ Query B: データ│    │
└──────────────┘    │
                    ↓
    ┌───────────────────────────────────────┐
    │ transformProps.ts                     │
    │                                       │
    │ ┌─────────────────────┐               │
    │ │ coltypeMapping作成  │               │
    │ │ (147-150行目)       │               │
    │ │                     │               │
    │ │ Query A + Query B   │               │
    │ │ ✅ 正しくマージ      │               │
    │ └─────────────────────┘               │
    │           │                           │
    │           │ ⚠️ 使用されていない        │
    │           │                           │
    │ ┌─────────────────────┐               │
    │ │ xAxisDataType取得   │               │
    │ │ (235-236行目)       │               │
    │ │                     │               │
    │ │ Query Aのみ参照     │               │
    │ │ ❌ 型情報が取得できない│               │
    │ └─────────────────────┘               │
    │           ↓                           │
    │    xAxisDataType                      │
    │    = undefined                        │
    │           ↓                           │
    │ ┌─────────────────────┐               │
    │ │ フォーマッター決定   │               │
    │ │ (476-483行目)       │               │
    │ │                     │               │
    │ │ undefined === 2?    │               │
    │ │ → false            │               │
    │ │ → String           │               │
    │ └─────────────────────┘               │
    └───────────────┬───────────────────────┘
                    ↓
              ┌──────────┐
              │ ECharts  │
              │          │
              │ 1609...  │ ← 数値表示
              └──────────┘
```

### 9.2 修正後の実装

```
┌──────────────┐
│ Backend      │
│              │
│ Query A: 空  │────┐
│ Query B: データ│    │
└──────────────┘    │
                    ↓
    ┌───────────────────────────────────────┐
    │ transformProps.ts                     │
    │                                       │
    │ ┌─────────────────────┐               │
    │ │ coltypeMapping作成  │               │
    │ │ (147-150行目)       │               │
    │ │                     │               │
    │ │ Query A + Query B   │               │
    │ │ ✅ 正しくマージ      │               │
    │ └─────────────────────┘               │
    │           ↓                           │
    │           │ ✅ 修正後は使用される      │
    │           ↓                           │
    │ ┌─────────────────────┐               │
    │ │ xAxisDataType取得   │               │
    │ │ (235-236行目)       │               │
    │ │                     │               │
    │ │ coltypeMappingを使用│               │
    │ │ ✅ 型情報が取得できる│               │
    │ └─────────────────────┘               │
    │           ↓                           │
    │    xAxisDataType                      │
    │    = 2 (Temporal)                     │
    │           ↓                           │
    │ ┌─────────────────────┐               │
    │ │ フォーマッター決定   │               │
    │ │ (476-483行目)       │               │
    │ │                     │               │
    │ │ 2 === 2?           │               │
    │ │ → true             │               │
    │ │ → getXAxisFormatter │               │
    │ └─────────────────────┘               │
    └───────────────┬───────────────────────┘
                    ↓
              ┌──────────┐
              │ ECharts  │
              │          │
              │ 2021-01-01│ ← 日付表示 ✅
              └──────────┘
```

---

## 10. 関連する型定義とインターフェース

### 10.1 ChartDataResponseResult

**ファイル**: `superset-ui-core/src/query/types/QueryResponse.ts`

```typescript
export interface ChartDataResponseResult {
  /**
   * Metadata about the columns in the result
   */
  colnames?: string[];
  coltypes?: GenericDataType[];

  /**
   * Data records returned from the query
   */
  data: DataRecord[];

  /**
   * Mapping of verbose names
   */
  label_map?: Record<string, string[]>;

  // ... other properties
}
```

### 10.2 GenericDataType

```typescript
export enum GenericDataType {
  Numeric = 0,
  String = 1,
  Temporal = 2,
  Boolean = 3,
}
```

### 10.3 TimeseriesDataRecord

**ファイル**: `superset-ui-core/src/chart/types/Base.ts`

```typescript
export interface TimeseriesDataRecord {
  __timestamp: number | string | Date | null;
  [key: string]: DataRecordValue;
}
```

### 10.4 完全なコールスタック

```
1. transformProps() - transformProps.ts:117
   ↓
2. coltypeMapping作成 - transformProps.ts:147
   ├→ getColtypesMapping(queriesData[0])
   └→ getColtypesMapping(queriesData[1])
   ↓
3. ⚠️ xAxisDataType取得 - transformProps.ts:235
   └→ getColtypesMapping(queriesData[0]) ← 問題
   ↓
4. フォーマッター決定 - transformProps.ts:476
   ├→ xAxisDataType === GenericDataType.Temporal ?
   ├→ getTooltipTimeFormatter() または String
   └→ getXAxisFormatter() または String
   ↓
5. echartOptions設定 - transformProps.ts:509
   ├→ xAxis.axisLabel.formatter = xAxisFormatter
   └→ tooltip.formatter内でtooltipFormatterを使用
   ↓
6. ECharts描画
   └→ フォーマッターを使用してラベル生成
```

### 10.5 関連ファイルマップ

```
superset-frontend/
└── plugins/
    └── plugin-chart-echarts/
        └── src/
            ├── MixedTimeseries/
            │   ├── transformProps.ts      ← メインファイル（問題箇所）
            │   ├── controlPanel.tsx
            │   └── types.ts
            ├── utils/
            │   ├── series.ts              ← getColtypesMapping()
            │   ├── formatters.ts          ← getXAxisFormatter()等
            │   └── axis.ts
            └── types.ts

packages/
└── superset-ui-core/
    └── src/
        └── query/
            └── types/
                └── QueryResponse.ts       ← GenericDataType定義
```

---

# パート3: 修正ガイド

## 11. 修正前の準備

### 11.1 前提条件の確認

以下が揃っていることを確認してください：

- ✅ Supersetのソースコードへのアクセス権
- ✅ フロントエンドのビルド環境（Node.js, npm/yarn）
- ✅ テスト環境
- ✅ バックアップ（念のため）

### 11.2 影響範囲の確認

**影響を受けるファイル**: **1ファイル**のみ
```
superset-frontend/plugins/plugin-chart-echarts/src/MixedTimeseries/transformProps.ts
```

**変更行数**: **2行**
- 削除: 1行
- 変更: 1行

### 11.3 バックアップ

```bash
# 対象ファイルのバックアップを作成
cd superset/superset-frontend/plugins/plugin-chart-echarts/src/MixedTimeseries
cp transformProps.ts transformProps.ts.backup

# または、Git commitで保存
cd /path/to/superset-kenshou/superset
git add .
git commit -m "Backup before fixing date format bug"
```

---

## 12. 修正手順

### 12.1 ファイルの場所を確認

```bash
cd /Users/kazu/kenshou/superset-kenshou/superset/superset-frontend/plugins/plugin-chart-echarts/src/MixedTimeseries
ls -la transformProps.ts
```

### 12.2 ファイルを開く

お好みのエディタで開いてください：

```bash
# VS Code
code transformProps.ts

# vi/vim
vi transformProps.ts

# nano
nano transformProps.ts
```

### 12.3 235-236行目を見つける

**検索方法**:
- VS Code: `Ctrl+G` (Mac: `Cmd+G`) → `235` を入力
- vi/vim: `:235` を入力
- nano: `Ctrl+_` → `235` を入力

**現在のコード**（235-236行目）:
```typescript
const dataTypes = getColtypesMapping(queriesData[0]);
const xAxisDataType = dataTypes?.[xAxisLabel] ?? dataTypes?.[xAxisOrig];
```

### 12.4 コードを修正

#### 修正前
```typescript
const dataTypes = getColtypesMapping(queriesData[0]);
const xAxisDataType = dataTypes?.[xAxisLabel] ?? dataTypes?.[xAxisOrig];
```

#### 修正後
```typescript
// const dataTypes = getColtypesMapping(queriesData[0]);  // 削除（コメントアウト）
const xAxisDataType = coltypeMapping?.[xAxisLabel] ?? coltypeMapping?.[xAxisOrig];
```

**変更点**:
1. **235行目**: `const dataTypes = ...`の行をコメントアウト
2. **236行目**: `dataTypes` を `coltypeMapping` に変更

### 12.5 変更内容の確認

**diffで確認**:
```bash
diff transformProps.ts.backup transformProps.ts
```

**期待される出力**:
```diff
235c235
< const dataTypes = getColtypesMapping(queriesData[0]);
---
> // const dataTypes = getColtypesMapping(queriesData[0]);  // 削除（コメントアウト）
236c236
< const xAxisDataType = dataTypes?.[xAxisLabel] ?? dataTypes?.[xAxisOrig];
---
> const xAxisDataType = coltypeMapping?.[xAxisLabel] ?? coltypeMapping?.[xAxisOrig];
```

### 12.6 ファイルを保存

保存して閉じてください。

---

## 13. テスト方法

### 13.1 フロントエンドのビルド

```bash
cd /Users/kazu/kenshou/superset-kenshou/superset/superset-frontend

# 依存関係のインストール（初回のみ）
npm install
# または
yarn install

# ビルド
npm run build
# または
yarn build
```

### 13.2 Supersetの起動

```bash
cd /Users/kazu/kenshou/superset-kenshou/superset

# 開発モードで起動
superset run -p 8088 --with-threads --reload --debugger
```

### 13.3 テストケース

以下の4パターンでテストしてください：

#### テストケース1: Query A空・Query B有（バグ修正対象）

**設定**:
- Query A: `SELECT timestamp, metric_a FROM table WHERE 1=0` （空）
- Query B: `SELECT timestamp, metric_b FROM table WHERE 1=1` （データあり）

**期待結果**:
- ✅ X軸が日付フォーマット（例: `2021-01-01`）で表示される
- ✅ ツールチップも日付フォーマットで表示される

**修正前の動作**:
- ❌ X軸が数値（例: `1609459200000`）で表示される

#### テストケース2: Query A有・Query B空

**設定**:
- Query A: `SELECT timestamp, metric_a FROM table WHERE 1=1` （データあり）
- Query B: `SELECT timestamp, metric_b FROM table WHERE 1=0` （空）

**期待結果**:
- ✅ X軸が日付フォーマットで表示される（修正前から正常）

#### テストケース3: 両方有（通常ケース）

**設定**:
- Query A: `SELECT timestamp, metric_a FROM table WHERE 1=1` （データあり）
- Query B: `SELECT timestamp, metric_b FROM table WHERE 1=1` （データあり）

**期待結果**:
- ✅ X軸が日付フォーマットで表示される（修正前から正常）

#### テストケース4: 両方空

**設定**:
- Query A: `SELECT timestamp, metric_a FROM table WHERE 1=0` （空）
- Query B: `SELECT timestamp, metric_b FROM table WHERE 1=0` （空）

**期待結果**:
- ✅ チャートが空または「No data」メッセージ
- ✅ エラーが発生しない

### 13.4 動作確認チェックリスト

```
□ Query A空・Query B有で日付フォーマット表示
□ Query A有・Query B空で日付フォーマット表示
□ 両方有で日付フォーマット表示
□ 両方空でエラーなし
□ X軸ラベルの回転が正しく動作
□ ツールチップの日付フォーマットが正しい
□ ズーム機能が正常に動作
□ 凡例の表示が正常
```

### 13.5 自動テストの追加（推奨）

**ファイル**: `superset-frontend/plugins/plugin-chart-echarts/test/MixedTimeseries/transformProps.test.ts`

```typescript
import transformProps from '../../src/MixedTimeseries/transformProps';
import { GenericDataType } from '@superset-ui/core';

describe('Mixed Chart X-axis date formatting', () => {
  it('should format dates when Query A is empty but Query B has data', () => {
    const chartProps = {
      formData: {
        // ... 必要な設定
      },
      queriesData: [
        // Query A empty
        {
          data: [],
          colnames: [],
          coltypes: [],
          label_map: {},
        },
        // Query B has data
        {
          data: [{ __timestamp: 1609459200000, metric_b: 100 }],
          colnames: ['__timestamp', 'metric_b'],
          coltypes: [GenericDataType.Temporal, GenericDataType.Numeric],
          label_map: {},
        },
      ],
      // ... その他の必須プロパティ
    };

    const result = transformProps(chartProps);

    // xAxisFormatterが日付フォーマッター（Stringではない）であることを確認
    expect(typeof result.echartOptions.xAxis.axisLabel.formatter).toBe('function');
    // または
    expect(result.echartOptions.xAxis.axisLabel.formatter).not.toBe(String);
  });
});
```

**テストの実行**:
```bash
cd superset-frontend
npm test -- --testPathPattern=MixedTimeseries/transformProps.test.ts
```

---

## 14. デプロイ手順

### 14.1 開発環境での確認完了後

```bash
# Git commitで変更を記録
cd /Users/kazu/kenshou/superset-kenshou/superset
git add superset-frontend/plugins/plugin-chart-echarts/src/MixedTimeseries/transformProps.ts
git commit -m "Fix: X-axis date formatting in Mixed Chart when Query A is empty

- Modified transformProps.ts to use coltypeMapping instead of queriesData[0]
- Fixes issue where timestamps displayed as numbers when first query is empty
- No breaking changes, backward compatible"
```

### 14.2 本番環境へのデプロイ

**手順は環境に依存しますが、一般的な流れ**:

```bash
# 1. プルリクエスト作成（チーム開発の場合）
git push origin fix-mixed-chart-date-format

# 2. レビュー承認後、本番ブランチにマージ

# 3. 本番環境でビルド
ssh production-server
cd /path/to/superset
git pull
cd superset-frontend
npm install
npm run build

# 4. Supersetサービスの再起動
supervisorctl restart superset
# または
systemctl restart superset
```

---

## 15. ロールバック手順

### 15.1 問題が発生した場合

**バックアップから復元**:
```bash
cd /Users/kazu/kenshou/superset-kenshou/superset/superset-frontend/plugins/plugin-chart-echarts/src/MixedTimeseries
cp transformProps.ts.backup transformProps.ts
```

**Gitから復元**:
```bash
cd /Users/kazu/kenshou/superset-kenshou/superset
git checkout HEAD -- superset-frontend/plugins/plugin-chart-echarts/src/MixedTimeseries/transformProps.ts
```

### 15.2 再ビルド

```bash
cd superset-frontend
npm run build
```

---

## 16. トラブルシューティング

### 問題1: ビルドエラーが発生する

**エラー例**:
```
ERROR in ./src/MixedTimeseries/transformProps.ts
Module not found: Error: Can't resolve 'coltypeMapping'
```

**原因**: 変数名のタイプミス

**解決策**:
- 236行目の変数名が正しく `coltypeMapping` になっているか確認
- スペルミスがないか確認

### 問題2: X軸がundefinedと表示される

**原因**: 両方のクエリが空、またはX軸カラムが見つからない

**確認ポイント**:
- Query AまたはQuery Bに `__timestamp` カラムが存在するか
- `xAxisLabel` または `xAxisOrig` が正しく設定されているか

### 問題3: テストケース1でまだ数値表示される

**原因**: ブラウザキャッシュ

**解決策**:
```bash
# ハードリロード
# Chrome/Firefox: Ctrl+Shift+R (Mac: Cmd+Shift+R)

# または、キャッシュをクリア
# Chrome: DevTools → Application → Clear storage
```

### 問題4: TypeScriptの型エラー

**エラー例**:
```
Property 'coltypeMapping' does not exist
```

**原因**: coltypeMappingがスコープ外

**確認**: 235行目が147-150行目より後にあることを確認

```typescript
// 147-150行目でcoltypeMappingが定義されている必要がある
const coltypeMapping = {
  ...getColtypesMapping(queriesData[0]),
  ...getColtypesMapping(queriesData[1]),
};

// ... その他のコード ...

// 235-236行目で使用
const xAxisDataType = coltypeMapping?.[xAxisLabel] ?? coltypeMapping?.[xAxisOrig];
```

### 問題5: ビルドが遅い

**原因**: フルビルドしている

**解決策**:
```bash
# 開発モードで起動（変更を自動検知）
cd superset-frontend
npm run dev
```

### 問題6: 既存のチャートが壊れた

**原因**: 予期しない副作用

**確認ポイント**:
- Query Aにデータがある通常ケース（テストケース2, 3）が正常か確認
- ロールバック手順でバックアップから復元

**報告**:
- 問題の詳細を開発チームに報告
- スクリーンショットとエラーログを添付

---

# 付録

## 17. よくある質問

### Q1: どのドキュメントから読めばいいですか？

**A**: 役割によって異なります：

- **PM/説明が必要な方** → パート1（概要）のみ
- **すぐ修正したい開発者** → パート1（概要） + パート3（修正ガイド）
- **技術的詳細を知りたい開発者** → すべて順番に

### Q2: 修正にどれくらい時間がかかりますか？

**A**: 約30分（準備、修正、テスト含む）

- 準備: 5分
- 修正: 5分
- ビルド: 10分
- テスト: 10分

### Q3: リスクはありますか？

**A**: 低リスクです

- 既存変数の再利用のため破壊的変更なし
- パフォーマンスへの影響なし
- ロールバック手順も用意

### Q4: 他のチャートタイプにも影響しますか？

**A**: いいえ、Mixed Timeseries Chartのみです

### Q5: コミュニティに報告すべきですか？

**A**: 推奨します

- 他のユーザーも同じ問題に遭遇している可能性が高い
- 本家に取り込まれれば、将来のアップグレードで修正の維持が不要
- GitHub IssueまたはPull Requestで報告可能

### Q6: この修正は本家Supersetにも適用されますか？

**A**: 未確認ですが、同じ問題が存在する可能性が高いです

- 本家へのPR提出を推奨
- コミュニティ貢献として歓迎される可能性が高い

### Q7: 修正後のパフォーマンスは？

**A**: 影響ありません

- `coltypeMapping`は既に計算済み
- 新しい計算は発生しない
- メモリ使用量も変わらない

### Q8: 古いバージョンのSupersetでも発生しますか？

**A**: 可能性があります

- Mixed Chartが導入されたバージョン以降で発生する可能性
- バージョンごとに実装を確認する必要あり

---

## 18. クイックリファレンス

### 修正内容（2行）

**ファイル**: `superset-frontend/plugins/plugin-chart-echarts/src/MixedTimeseries/transformProps.ts`

**行番号**: 235-236

**修正前**:
```typescript
const dataTypes = getColtypesMapping(queriesData[0]);
const xAxisDataType = dataTypes?.[xAxisLabel] ?? dataTypes?.[xAxisOrig];
```

**修正後**:
```typescript
// const dataTypes = getColtypesMapping(queriesData[0]);  // Query Aのみ参照していた問題箇所
const xAxisDataType = coltypeMapping?.[xAxisLabel] ?? coltypeMapping?.[xAxisOrig];
```

### テストケース（4パターン）

| ケース | Query A | Query B | 期待結果 | 修正前 |
|--------|---------|---------|----------|--------|
| 1 | 空 | データあり | ✅ 日付表示 | ❌ 数値表示 |
| 2 | データあり | 空 | ✅ 日付表示 | ✅ 日付表示 |
| 3 | データあり | データあり | ✅ 日付表示 | ✅ 日付表示 |
| 4 | 空 | 空 | ✅ エラーなし | ✅ エラーなし |

### コマンド集

```bash
# バックアップ作成
cd superset/superset-frontend/plugins/plugin-chart-echarts/src/MixedTimeseries
cp transformProps.ts transformProps.ts.backup

# ビルド
cd superset-frontend
npm run build

# 開発モード起動
npm run dev

# テスト実行
npm test -- --testPathPattern=MixedTimeseries

# Superset起動
cd /path/to/superset
superset run -p 8088 --with-threads --reload --debugger

# Git commit
git add superset-frontend/plugins/plugin-chart-echarts/src/MixedTimeseries/transformProps.ts
git commit -m "Fix: X-axis date formatting in Mixed Chart when Query A is empty"

# ロールバック
git checkout HEAD -- superset-frontend/plugins/plugin-chart-echarts/src/MixedTimeseries/transformProps.ts
```

### 関連ファイル

| ファイル | 役割 |
|---------|------|
| `MixedTimeseries/transformProps.ts:147-150` | coltypeMapping作成（正しい実装） |
| `MixedTimeseries/transformProps.ts:235-236` | xAxisDataType取得（問題箇所） |
| `MixedTimeseries/transformProps.ts:476-483` | フォーマッター決定 |
| `utils/series.ts:391-401` | getColtypesMapping関数 |
| `utils/formatters.ts:77-99` | フォーマッター関数 |
| `QueryResponse.ts:26-31` | GenericDataType定義 |

---

## まとめ

### 問題の本質

1. **coltypeMappingは正しく作成されている**（両クエリをマージ）
2. **xAxisDataTypeの取得時に使用されていない**（Query Aのみ参照）
3. **Query Aが空の場合、型情報が取得できない**
4. **結果、日付フォーマッターが設定されない**

### 修正のポイント

- **既存の変数を活用**することで、最小限の変更で解決
- **破壊的変更なし**
- **パフォーマンスへの影響なし**

### 次のステップ

1. ✅ 修正を適用（パート3の手順に従う）
2. ✅ 4パターンでテスト
3. ✅ デプロイ
4. 🔲 コミュニティへの報告検討
5. 🔲 ドキュメント更新

---

**作成者**: Claude Code
**作成日**: 2026-04-02
**最終更新**: 2026-04-02
**レビュー推奨**: フロントエンド開発チーム、プロダクトマネージャー

---

**フィードバック**: このドキュメントに関するフィードバックや質問がありましたら、開発チームまでお願いします。
