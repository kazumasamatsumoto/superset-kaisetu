# 時系列データにおける欠損値（値がない場合）の扱い 詳細解説

**対象**: Apache Superset Mixed Timeseries Chart
**調査日**: 2026-04-01
**関連ファイル**:
- `superset-ui-core/src/query/types/QueryResponse.ts`
- `plugin-chart-echarts/src/utils/series.ts`
- `plugin-chart-echarts/src/MixedTimeseries/transformProps.ts`

---

## 1. 問い

**時系列データで値がなかったとしたら0として扱われるのか？それはどこまで扱われるのか？**

---

## 2. 結論（先に要約）

### 値がない場合の扱い

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

## 3. データフロー全体像

### 3.1 バックエンドからフロントエンドへ

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
│   ],                                                        │
│   colnames: ["__timestamp", "series1"],                     │
│   coltypes: [GenericDataType.Temporal, GenericDataType.Numeric] │
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
│   結果: [                                                   │
│     [1609459200000, 100],                                   │
│     [1609545600000, 0],    ← 0に変換                        │
│     [1609632000000, 0]     ← 0に変換                        │
│   ]                                                         │
│                                                             │
│ スタックOFFの場合:                                          │
│   null/undefined → そのまま                                 │
│   結果: [                                                   │
│     [1609459200000, 100],                                   │
│     [1609545600000, null],                                  │
│     [1609632000000, undefined]                              │
│   ]                                                         │
└─────────────────────────────────────────────────────────────┘
                            ↓
┌─────────────────────────────────────────────────────────────┐
│ Step 6: ECharts描画                                         │
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

---

## 4. コードレベルの詳細解説

### 4.1 型定義（バックエンドからのデータ）

**ファイル**: `superset-ui-core/src/query/types/QueryResponse.ts:36-40`

```typescript
export type DataRecordValue = number | string | boolean | Date | null | bigint;

export interface DataRecord {
  [key: string]: DataRecordValue;
}
```

**重要ポイント**:
- `DataRecordValue`に`null`が含まれている
- `undefined`は型定義に含まれていないが、JavaScriptの性質上、キーが存在しない場合は`undefined`になる

**実際のデータ例**:

```javascript
// パターン1: 値が明示的にnull
{ __timestamp: 1609459200000, metric1: null }

// パターン2: キー自体が存在しない（JavaScriptではundefinedとして扱われる）
{ __timestamp: 1609459200000 }  // metric1キーがない

// パターン3: 値が0（実際のデータ）
{ __timestamp: 1609459200000, metric1: 0 }
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
- `stack`または`stackB`が`true`（スタックモードON）の場合:
  - `fillNeighborValue = 0`
  - **欠損値を0で埋める**
- `stack`または`stackB`が`false`（スタックモードOFF）の場合:
  - `fillNeighborValue = undefined`
  - **欠損値を埋めない**

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

      // minPositiveValueの計算（省略）

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
      } else if (
        stack === StackControlsValue.Expand &&
        totalStackedValue !== undefined
      ) {
        value = ((value || 0) as number) / totalStackedValue;  // ← 割合計算時も0に
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

---

## 5. 具体例で見る処理の流れ

### 例1: スタックモードON、途中に欠損値がある場合

**バックエンドからのデータ**:
```javascript
[
  { __timestamp: "2021-01-01", series1: 100, series2: 50 },
  { __timestamp: "2021-01-02", series1: null, series2: 60 },  // series1が欠損
  { __timestamp: "2021-01-03", series1: 120, series2: 70 }
]
```

**extractSeries呼び出し**:
```typescript
extractSeries(data, {
  fillNeighborValue: 0,  // スタックONなので0
  xAxis: "__timestamp",
})
```

**処理**:

| 時点 | series1の値 | isDefined? | 隣接値? | 埋める? | 最終値 |
|------|-------------|------------|---------|---------|--------|
| 2021-01-01 | 100 | true | - | - | 100 |
| 2021-01-02 | null | **false** | **true** (前後に値あり) | **yes** | **0** |
| 2021-01-03 | 120 | true | - | - | 120 |

**EChartsへ渡されるデータ**:
```javascript
{
  id: "series1",
  name: "series1",
  data: [
    [Date("2021-01-01"), 100],
    [Date("2021-01-02"), 0],    // ← 0に変換された
    [Date("2021-01-03"), 120]
  ]
}
```

**ツールチップ表示**:
```
2021-01-02
● series1: 0      ← 0として表示される
● series2: 60
```

### 例2: スタックモードOFF、途中に欠損値がある場合

**同じデータ**:
```javascript
[
  { __timestamp: "2021-01-01", series1: 100, series2: 50 },
  { __timestamp: "2021-01-02", series1: null, series2: 60 },
  { __timestamp: "2021-01-03", series1: 120, series2: 70 }
]
```

**extractSeries呼び出し**:
```typescript
extractSeries(data, {
  fillNeighborValue: undefined,  // スタックOFFなのでundefined
  xAxis: "__timestamp",
})
```

**処理**:

| 時点 | series1の値 | isDefined? | fillNeighborValue | 埋める? | 最終値 |
|------|-------------|------------|-------------------|---------|--------|
| 2021-01-01 | 100 | true | - | - | 100 |
| 2021-01-02 | null | false | **undefined** | **no** | **null** |
| 2021-01-03 | 120 | true | - | - | 120 |

**EChartsへ渡されるデータ**:
```javascript
{
  id: "series1",
  name: "series1",
  data: [
    [Date("2021-01-01"), 100],
    [Date("2021-01-02"), null],   // ← nullのまま
    [Date("2021-01-03"), 120]
  ]
}
```

**EChartsの描画**:
- 2021-01-01から2021-01-02への線は描かれない（ギャップ）
- 2021-01-02から2021-01-03への線も描かれない（ギャップ）

**ツールチップ表示**:
```
2021-01-02
● series2: 60      ← series1は表示されない
```

### 例3: 系列の最初に欠損値がある場合（スタックON）

**バックエンドからのデータ**:
```javascript
[
  { __timestamp: "2021-01-01", series1: null },  // 最初が欠損
  { __timestamp: "2021-01-02", series1: 100 },
  { __timestamp: "2021-01-03", series1: 120 }
]
```

**処理**:

| 時点 | series1の値 | isDefined? | 隣接値? | 埋める? | 最終値 |
|------|-------------|------------|---------|---------|--------|
| 2021-01-01 | null | false | **false** (前に値なし) | **no** | **null** |
| 2021-01-02 | 100 | true | - | - | 100 |
| 2021-01-03 | 120 | true | - | - | 120 |

**重要**:
- **スタックONでも、系列の最初/最後の欠損値は0に変換されない**
- 「隣接する値がある」条件がfalseになるため

**EChartsへ渡されるデータ**:
```javascript
data: [
  [Date("2021-01-01"), null],   // ← 埋められない
  [Date("2021-01-02"), 100],
  [Date("2021-01-03"), 120]
]
```

### 例4: 実際のデータが0の場合

**バックエンドからのデータ**:
```javascript
[
  { __timestamp: "2021-01-01", series1: 100 },
  { __timestamp: "2021-01-02", series1: 0 },     // ← 実際に0
  { __timestamp: "2021-01-03", series1: 120 }
]
```

**処理**:

| 時点 | series1の値 | isDefined? | 処理 | 最終値 |
|------|-------------|------------|------|--------|
| 2021-01-01 | 100 | true | そのまま | 100 |
| 2021-01-02 | **0** | **true** | **そのまま** | **0** |
| 2021-01-03 | 120 | true | そのまま | 120 |

**重要**:
- **0は`isDefined(0) === true`なので、欠損値として扱われない**
- fillNeighborValueの処理をスキップ
- そのまま0として渡される

**ツールチップ表示**:
```
2021-01-02
● series1: 0      ← 実際のデータとして表示
```

---

## 6. まとめ

### 6.1 欠損値の定義

Supersetでは、以下を「欠損値」として扱います：

| 値 | 欠損値? | 理由 |
|----|---------|------|
| `null` | ✅ はい | バックエンドから明示的にnull |
| `undefined` | ✅ はい | JavaScriptでキーが存在しない |
| `0` | ❌ いいえ | 有効な数値データ |
| `""` (空文字列) | ❌ いいえ | 有効な文字列データ |
| `false` | ❌ いいえ | 有効なブールデータ |

### 6.2 欠損値が0として扱われる条件

**すべての条件を満たす必要があります**:

1. ✅ **スタックモードがON**である
2. ✅ **値がnullまたはundefined**である
3. ✅ **前後の時点に有効な値がある**（系列の途中である）

**条件を満たさない場合**:
- スタックモードOFF → 欠損値はnullのまま
- 系列の最初/最後 → 0に変換されない
- 実際のデータが0 → そのまま0（欠損値ではない）

### 6.3 各段階での扱い

| 段階 | 値の形式 | 備考 |
|------|----------|------|
| **1. バックエンド（Python）** | `NaN` または 値なし | Pandasのデータフレーム |
| **2. JSON変換** | `null` または キーなし | JSON.stringify()で変換 |
| **3. フロントエンド受信** | `null` または `undefined` | TypeScript型: `DataRecordValue` |
| **4. extractSeries処理** | 条件により`0`に変換、または`null`のまま | fillNeighborValueによる |
| **5. ECharts渡し** | `[timestamp, 0]` または `[timestamp, null]` | 配列形式 |
| **6. ツールチップ** | `"0"`として表示、または非表示 | formatForecastTooltipSeriesで処理 |
| **7. グラフ描画** | 0の高さで描画、またはギャップ | EChartsのレンダリング |

### 6.4 実務上の注意点

#### ケース1: 「データがない」と「値が0」を区別したい

**問題**:
スタックモードONの場合、欠損値が0に変換されるため、「実際に0」と「データがない」を区別できません。

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

#### ケース2: スタックチャートで一部のシリーズが途切れる

**問題**:
スタックチャートなのに、特定の時点で特定のシリーズが表示されない。

**原因**:
系列の最初または最後に欠損値がある場合、fillNeighborValueでも埋められません。

**対策**:
```sql
-- SQLクエリで系列の範囲を統一
SELECT
  date,
  COALESCE(metric_value, 0) as metric_value  -- 明示的に0に変換
FROM data
WHERE date BETWEEN start_date AND end_date  -- 全系列で同じ範囲
```

#### ケース3: ツールチップに0が大量に表示される

**問題**:
多数のシリーズがあり、ほとんどが0（欠損値）の場合、ツールチップが巨大になる。

**対策**:
前述のTOOLTIP_REPORT.mdを参照（Issue #4462の回避策）。

---

## 7. コードレベルでの検証方法

### 7.1 ブラウザ開発者ツールでの確認

**Step 1: バックエンドからのデータを確認**

Chrome DevTools → Network タブ → `chart/data` リクエスト → Response

```json
{
  "result": [{
    "data": [
      { "__timestamp": 1609459200000, "metric1": 100 },
      { "__timestamp": 1609545600000, "metric1": null },  // ← 欠損値
      { "__timestamp": 1609632000000, "metric1": 120 }
    ]
  }]
}
```

**Step 2: extractSeries後のデータを確認**

Console タブで以下を実行:

```javascript
// EChartsインスタンスを取得
const chart = echarts.getInstanceByDom(document.querySelector('.echarts-for-react'));

// シリーズデータを確認
console.log(chart.getOption().series[0].data);
// 出力例（スタックON）:
// [[1609459200000, 100], [1609545600000, 0], [1609632000000, 120]]
//                                         ↑ 0に変換されている

// 出力例（スタックOFF）:
// [[1609459200000, 100], [1609545600000, null], [1609632000000, 120]]
//                                         ↑ nullのまま
```

### 7.2 単体テストでの検証

```typescript
import { extractSeries } from '../utils/series';

describe('extractSeries - 欠損値の処理', () => {
  const data = [
    { __timestamp: 1, series1: 100 },
    { __timestamp: 2, series1: null },  // 欠損値
    { __timestamp: 3, series1: 120 },
  ];

  it('スタックONの場合、途中の欠損値を0に変換', () => {
    const [series] = extractSeries(data, {
      fillNeighborValue: 0,
      xAxis: '__timestamp',
    });

    expect(series[0].data).toEqual([
      [1, 100],
      [2, 0],    // ← 0に変換
      [3, 120],
    ]);
  });

  it('スタックOFFの場合、欠損値をnullのまま保持', () => {
    const [series] = extractSeries(data, {
      fillNeighborValue: undefined,
      xAxis: '__timestamp',
    });

    expect(series[0].data).toEqual([
      [1, 100],
      [2, null],  // ← nullのまま
      [3, 120],
    ]);
  });

  it('実際のデータが0の場合、0として扱う', () => {
    const dataWithZero = [
      { __timestamp: 1, series1: 100 },
      { __timestamp: 2, series1: 0 },    // 実際に0
      { __timestamp: 3, series1: 120 },
    ];

    const [series] = extractSeries(dataWithZero, {
      fillNeighborValue: 0,
      xAxis: '__timestamp',
    });

    expect(series[0].data).toEqual([
      [1, 100],
      [2, 0],    // ← 0のまま（欠損値として扱われない）
      [3, 120],
    ]);
  });
});
```

---

## 8. 関連するコード箇所まとめ

| ファイル | 行番号 | 内容 |
|---------|--------|------|
| `QueryResponse.ts` | 36-40 | `DataRecordValue`と`DataRecord`の型定義 |
| `transformProps.ts` | 226, 231 | `fillNeighborValue`の決定ロジック |
| `series.ts` | 54-56 | `isDefined()`関数の実装 |
| `series.ts` | 330-338 | fillNeighborValueでの埋め処理 |
| `series.ts` | 339-343 | スタック展開時の`value \|\| 0`処理 |
| `forecast.ts` | 66-80 | ツールチップでの`typeof numericValue === 'number'`チェック |
| `forecast.ts` | 99 | `typeof observation === 'number'`でのフォーマット |

---

**調査者**: Claude Code
**レビュー推奨**: データエンジニアリングチーム、フロントエンド開発チーム
