# Mixed Chart 日付フォーマット不具合 - 技術詳細

**対象**: Apache Superset Mixed Timeseries Chart
**作成日**: 2026-04-02

---

## 目次

1. [日付フォーマット決定の全フロー](#1-日付フォーマット決定の全フロー)
2. [各ステップの詳細解説](#2-各ステップの詳細解説)
3. [問題発生のメカニズム](#3-問題発生のメカニズム)
4. [データフロー図](#4-データフロー図)
5. [関連する型定義とインターフェース](#5-関連する型定義とインターフェース)
6. [コードトレース](#6-コードトレース)

---

## 1. 日付フォーマット決定の全フロー

### 1.1 フロー概要

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

## 2. 各ステップの詳細解説

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

## 3. 問題発生のメカニズム

### 3.1 正常ケース（Query Aにデータがある）

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

### 3.2 バグケース（Query Aが空）

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

### 3.3 修正後の動作（Query Aが空でも正常）

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

## 4. データフロー図

### 4.1 現在の実装（バグあり）

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

### 4.2 修正後の実装

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

## 5. 関連する型定義とインターフェース

### 5.1 ChartDataResponseResult

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

### 5.2 GenericDataType

```typescript
export enum GenericDataType {
  Numeric = 0,
  String = 1,
  Temporal = 2,
  Boolean = 3,
}
```

### 5.3 TimeseriesDataRecord

**ファイル**: `superset-ui-core/src/chart/types/Base.ts`

```typescript
export interface TimeseriesDataRecord {
  __timestamp: number | string | Date | null;
  [key: string]: DataRecordValue;
}
```

---

## 6. コードトレース

### 6.1 完全なコールスタック

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

### 6.2 関連ファイルマップ

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

**[DATE_FORMAT_BUG_FIX_GUIDE.md](./DATE_FORMAT_BUG_FIX_GUIDE.md)** で具体的な修正手順を確認してください。

---

**作成者**: Claude Code
**作成日**: 2026-04-02
**レビュー推奨**: フロントエンド開発チーム
