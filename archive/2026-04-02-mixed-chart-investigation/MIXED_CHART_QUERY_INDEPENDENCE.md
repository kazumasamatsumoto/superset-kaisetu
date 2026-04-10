# Mixed Chart - Query AとQuery Bの独立性調査レポート

## 📋 目次

1. [概要](#概要)
2. [調査結果サマリー](#調査結果サマリー)
3. [技術的な独立性の証明](#技術的な独立性の証明)
4. [実装の詳細](#実装の詳細)
5. [検証方法](#検証方法)
6. [結論](#結論)

---

## 概要

### 調査の目的

**質問**: Mixed Chartにおいて、片方のWHERE条件（Filters）はもう一つのチャートに影響を与えるか？

### 調査対象

- Query A（通常のクエリ）の `adhoc_filters`
- Query B（サフィックス付きクエリ）の `adhoc_filters_b`
- 両者の独立性の検証

### 結論（先に）

**No - Query AとQuery Bは完全に独立している**

Query Aの `adhoc_filters` は Query Bに影響を与えません。
Query Bの `adhoc_filters_b` は Query Aに影響を与えません。

---

## 調査結果サマリー

### ✅ 完全に独立している理由

| 観点 | Query A | Query B | 関係性 |
|------|---------|---------|--------|
| **フォームコントロール** | `adhoc_filters` | `adhoc_filters_b` | 完全に別のキー |
| **FormData分離** | サフィックスなし | `_b` サフィックスのみ | `removeFormDataSuffix` と `retainFormDataSuffix` で完全分離 |
| **クエリ構築** | `buildQueryContext(formData1)` | `buildQueryContext(formData2)` | 独立したクエリコンテキスト |
| **データ取得** | `queriesData[0]` | `queriesData[1]` | 独立したクエリ結果 |
| **SQL生成** | Query A用のSQL | Query B用のSQL | 完全に別のSQL文 |

### 📊 データの流れ

```
┌─────────────────────────────────────────────────────────────┐
│ 1. ユーザー設定（UI）                                          │
├─────────────────────────────────────────────────────────────┤
│ Query A:                                                    │
│   - Metrics: ['metric1']                                    │
│   - Filters: [WHERE date > '2024-01-01']                   │
│                                                             │
│ Query B:                                                    │
│   - Metrics B: ['metric2']                                  │
│   - Filters B: [WHERE category = 'A']                      │
└─────────────────────────────────────────────────────────────┘
                          ↓
┌─────────────────────────────────────────────────────────────┐
│ 2. FormData（統合オブジェクト）                                 │
├─────────────────────────────────────────────────────────────┤
│ {                                                           │
│   metrics: ['metric1'],                                     │
│   adhoc_filters: [WHERE date > '2024-01-01'],             │
│   metrics_b: ['metric2'],                                   │
│   adhoc_filters_b: [WHERE category = 'A'],                │
│   // ... other settings                                    │
│ }                                                           │
└─────────────────────────────────────────────────────────────┘
                          ↓
┌─────────────────────────────────────────────────────────────┐
│ 3. buildQuery.ts - FormData分離                             │
├─────────────────────────────────────────────────────────────┤
│ formData1 = removeFormDataSuffix(baseFormData, '_b')       │
│ {                                                           │
│   metrics: ['metric1'],                                     │
│   adhoc_filters: [WHERE date > '2024-01-01'],  ← Query A  │
│   // _b suffix keys removed                                │
│ }                                                           │
│                                                             │
│ formData2 = retainFormDataSuffix(baseFormData, '_b')       │
│ {                                                           │
│   metrics: ['metric2'],      ← _b suffix removed           │
│   adhoc_filters: [WHERE category = 'A'],  ← Query B       │
│   // base keys replaced with _b suffix values             │
│ }                                                           │
└─────────────────────────────────────────────────────────────┘
                          ↓
┌─────────────────────────────────────────────────────────────┐
│ 4. クエリ構築（独立実行）                                        │
├─────────────────────────────────────────────────────────────┤
│ Query 1:                                                    │
│   SELECT metric1                                            │
│   FROM table                                                │
│   WHERE date > '2024-01-01'                                │
│   GROUP BY time                                             │
│                                                             │
│ Query 2:                                                    │
│   SELECT metric2                                            │
│   FROM table                                                │
│   WHERE category = 'A'                                     │
│   GROUP BY time                                             │
└─────────────────────────────────────────────────────────────┘
                          ↓
┌─────────────────────────────────────────────────────────────┐
│ 5. データベース実行（独立）                                      │
├─────────────────────────────────────────────────────────────┤
│ queriesData[0]: [{date: '2024-01-01', metric1: 100}, ...]  │
│ queriesData[1]: [{date: '2024-01-01', metric2: 50}, ...]   │
└─────────────────────────────────────────────────────────────┘
                          ↓
┌─────────────────────────────────────────────────────────────┐
│ 6. ECharts描画（統合表示）                                     │
├─────────────────────────────────────────────────────────────┤
│ 同じX軸上に両方のデータを表示                                   │
│ （データは独立、表示のみ統合）                                  │
└─────────────────────────────────────────────────────────────┘
```

---

## 技術的な独立性の証明

### 1. コントロールの分離

**ファイル**: `controlPanel.tsx:61-86`

```typescript
function createQuerySection(
  label: string,
  controlSuffix: string,
): ControlPanelSectionConfig {
  return {
    label,
    expanded: true,
    controlSetRows: [
      [{ name: `metrics${controlSuffix}`, config: sharedControls.metrics }],
      [{ name: `groupby${controlSuffix}`, config: sharedControls.groupby }],
      [{ name: `adhoc_filters${controlSuffix}`, config: sharedControls.adhoc_filters }],
      // ...
    ],
  };
}

// Query A: no suffix
const queryA = createQuerySection(t('Query A'), '');
// → adhoc_filters

// Query B: '_b' suffix
const queryB = createQuerySection(t('Query B'), '_b');
// → adhoc_filters_b
```

**重要ポイント**:
- Query Aは `adhoc_filters` キーを使用
- Query Bは `adhoc_filters_b` キーを使用
- **完全に別のFormDataキー** → UI設定の段階で既に分離

---

### 2. FormDataの分離処理

**ファイル**: `buildQuery.ts:44-93`

```typescript
export default function buildQuery(formData: QueryFormData) {
  const baseFormData = { ...formData };

  // Query A用: _b サフィックスを持つキーを削除
  const formData1 = removeFormDataSuffix(baseFormData, '_b');

  // Query B用: _b サフィックスのキーのみを保持し、サフィックスを削除
  const formData2 = retainFormDataSuffix(baseFormData, '_b');

  // 独立したクエリコンテキストを構築
  const queryContexts = [formData1, formData2].map(fd =>
    buildQueryContext(fd, baseQueryObject => {
      // 各クエリは完全に独立して構築される
      return [
        {
          ...baseQueryObject,
          // ... クエリ設定
        },
      ];
    }),
  );

  return {
    ...queryContexts[0],
    queries: [...queryContexts[0].queries, ...queryContexts[1].queries],
  };
}
```

**重要ポイント**:
- `removeFormDataSuffix`: Query A用に `_b` サフィックス付きキーを**完全に削除**
- `retainFormDataSuffix`: Query B用に `_b` サフィックスのキーのみを保持
- 2つのクエリコンテキストは**完全に独立**して構築される

---

### 3. FormData分離関数の詳細

**ファイル**: `formDataSuffix.ts`

#### `removeFormDataSuffix` 関数

```typescript
export const removeFormDataSuffix = (
  formData: QueryFormData,
  controlSuffix: string,
): QueryFormData => {
  const formDataCopy = { ...formData };

  // _b サフィックスを持つすべてのキーを削除
  Object.keys(formDataCopy).forEach(key => {
    if (key.endsWith(controlSuffix)) {
      delete formDataCopy[key];
    }
  });

  return formDataCopy;
};
```

**実行例**:

```javascript
// Input
{
  metrics: ['sales'],
  adhoc_filters: [WHERE date > '2024-01-01'],
  metrics_b: ['revenue'],
  adhoc_filters_b: [WHERE category = 'A'],
}

// Output (Query A用)
{
  metrics: ['sales'],
  adhoc_filters: [WHERE date > '2024-01-01'],
  // metrics_b と adhoc_filters_b は削除された
}
```

---

#### `retainFormDataSuffix` 関数

```typescript
export const retainFormDataSuffix = (
  formData: QueryFormData,
  controlSuffix: string,
): QueryFormData => {
  const formDataCopy = { ...formData };

  // _b サフィックスなしのキーを削除し、
  // _b サフィックス付きのキーをサフィックスなしのキーに変換
  const suffixLength = controlSuffix.length;

  Object.keys(formDataCopy).forEach(key => {
    if (key.endsWith(controlSuffix)) {
      // metrics_b → metrics に変換
      const baseKey = key.slice(0, -suffixLength);
      formDataCopy[baseKey] = formDataCopy[key];
      delete formDataCopy[key];
    } else if (formDataCopy.hasOwnProperty(`${key}${controlSuffix}`)) {
      // サフィックス付きバージョンが存在する場合、ベースキーを削除
      delete formDataCopy[key];
    }
  });

  return formDataCopy;
};
```

**実行例**:

```javascript
// Input
{
  metrics: ['sales'],
  adhoc_filters: [WHERE date > '2024-01-01'],
  metrics_b: ['revenue'],
  adhoc_filters_b: [WHERE category = 'A'],
}

// Output (Query B用)
{
  metrics: ['revenue'],           // metrics_b から変換
  adhoc_filters: [WHERE category = 'A'],  // adhoc_filters_b から変換
  // 元の metrics と adhoc_filters は削除された
}
```

---

### 4. クエリ構築の独立性

**ファイル**: `buildQuery.ts:57-82`

```typescript
const queryContexts = [formData1, formData2].map(fd =>
  buildQueryContext(fd, baseQueryObject => {
    return [
      {
        ...baseQueryObject,
        // columns は各クエリで独立
        columns: fd.groupby,
        // metrics は各クエリで独立
        metrics: fd.metrics,
        // filters は各クエリで独立
        filters: fd.adhoc_filters,  // ← 完全に独立したフィルター
        // ...
      },
    ];
  }),
);
```

**重要ポイント**:
- `formData1.adhoc_filters` と `formData2.adhoc_filters` は**完全に別の配列**
- 各クエリは独立した `baseQueryObject` を持つ
- フィルターのマージや共有は一切発生しない

---

### 5. データ取得の独立性

**ファイル**: `transformProps.ts:138-150`

```typescript
export default function transformProps(
  chartProps: EchartsTimeseriesChartProps,
): TimeseriesChartTransformedProps {
  const { queriesData } = chartProps;

  // Query Aのデータ
  const dataA = queriesData[0].data || [];

  // Query Bのデータ
  const dataB = queriesData[1].data || [];

  // それぞれ独立して処理される
  const seriesA = extractSeries(dataA, { fillNeighborValue: ... });
  const seriesB = extractSeries(dataB, { fillNeighborValue: ... });

  // ...
}
```

**重要ポイント**:
- `queriesData[0]` と `queriesData[1]` は**完全に独立したクエリ結果**
- 各データは独立してシリーズ抽出される
- フィルターの影響は各クエリのデータにのみ反映される

---

## 実装の詳細

### 独立性を保証する3つのレイヤー

#### Layer 1: UI設定の分離

```typescript
// controlPanel.tsx
export const sections = [
  {
    label: t('Query A'),
    controlSetRows: [
      [{ name: 'adhoc_filters', ... }],  // ← Query A専用
    ],
  },
  {
    label: t('Query B'),
    controlSetRows: [
      [{ name: 'adhoc_filters_b', ... }],  // ← Query B専用
    ],
  },
];
```

→ **異なるキー名で保存** → UI設定の段階で完全に分離

---

#### Layer 2: FormData分離処理

```typescript
// buildQuery.ts
const formData1 = removeFormDataSuffix(baseFormData, '_b');
// → { adhoc_filters: [...] } のみ保持

const formData2 = retainFormDataSuffix(baseFormData, '_b');
// → { adhoc_filters: [...] } (元は adhoc_filters_b) のみ保持
```

→ **サフィックスパターンで完全分離** → クエリ構築前に独立したFormDataを生成

---

#### Layer 3: クエリ実行の独立

```typescript
// Superset Backend
Query 1:
SELECT
  DATE_TRUNC('day', time_column) as __timestamp,
  metric1
FROM table
WHERE date > '2024-01-01'  ← Query Aのフィルターのみ
GROUP BY __timestamp

Query 2:
SELECT
  DATE_TRUNC('day', time_column) as __timestamp,
  metric2
FROM table
WHERE category = 'A'  ← Query Bのフィルターのみ
GROUP BY __timestamp
```

→ **完全に別のSQL文が実行される** → データベースレベルで独立

---

## 検証方法

### 実験的検証の手順

#### ステップ1: 異なるフィルターを設定

**Query A**:
- Metrics: `COUNT(*)`
- Filters: `WHERE category = 'Electronics'`

**Query B**:
- Metrics B: `SUM(amount)`
- Filters B: `WHERE category = 'Books'`

---

#### ステップ2: DATAタブで確認

**Query A (queriesData[0])**:
```json
[
  {"__timestamp": "2024-01-01", "COUNT(*)": 150},
  {"__timestamp": "2024-01-02", "COUNT(*)": 180},
  // Electronics カテゴリーのみ
]
```

**Query B (queriesData[1])**:
```json
[
  {"__timestamp": "2024-01-01", "SUM(amount)": 5000},
  {"__timestamp": "2024-01-02", "SUM(amount)": 6200},
  // Books カテゴリーのみ
]
```

→ **データの行数や値が完全に異なる** → フィルターが独立して適用されている証明

---

#### ステップ3: チャート上で視覚的に確認

- Query Aのライン: Electronics のデータのみを反映
- Query Bのライン: Books のデータのみを反映

→ **完全に独立したトレンドが表示される**

---

### コード的検証

#### 検証1: FormData分離の確認

```javascript
// Developer Console (Chrome DevTools)

// 元のFormData
const formData = {
  metrics: ['sales'],
  adhoc_filters: [{clause: 'WHERE', expressionType: 'SIMPLE', ...}],
  metrics_b: ['revenue'],
  adhoc_filters_b: [{clause: 'WHERE', expressionType: 'SIMPLE', ...}],
};

// Query A用
const formData1 = removeFormDataSuffix(formData, '_b');
console.log(formData1.adhoc_filters);  // Query Aのフィルターのみ
console.log(formData1.adhoc_filters_b);  // undefined

// Query B用
const formData2 = retainFormDataSuffix(formData, '_b');
console.log(formData2.adhoc_filters);  // Query Bのフィルター（元は _b）
console.log(formData2.adhoc_filters_b);  // undefined
```

---

#### 検証2: クエリ構築の確認

```javascript
// buildQuery.ts にconsole.log追加

export default function buildQuery(formData: QueryFormData) {
  const formData1 = removeFormDataSuffix(baseFormData, '_b');
  const formData2 = retainFormDataSuffix(baseFormData, '_b');

  console.log('Query A filters:', formData1.adhoc_filters);
  console.log('Query B filters:', formData2.adhoc_filters);
  // → 完全に異なる配列が出力される

  // ...
}
```

---

#### 検証3: SQL文の確認

Superset UIの「View query」ボタンをクリック:

```sql
-- Query A
SELECT
  DATE_TRUNC('day', time_column) as __timestamp,
  COUNT(*) as "COUNT(*)"
FROM sales_table
WHERE category = 'Electronics'
GROUP BY __timestamp
ORDER BY __timestamp

-- Query B
SELECT
  DATE_TRUNC('day', time_column) as __timestamp,
  SUM(amount) as "SUM(amount)"
FROM sales_table
WHERE category = 'Books'
GROUP BY __timestamp
ORDER BY __timestamp
```

→ **WHERE句が完全に異なる** → 独立性の最終証明

---

## 結論

### ✅ Query AとQuery Bは完全に独立している

#### 独立性の保証メカニズム

1. **UI設計**: 異なるコントロール名（`adhoc_filters` vs `adhoc_filters_b`）
2. **FormData分離**: サフィックスパターンによる完全分離
3. **クエリ構築**: 独立した `buildQueryContext` 呼び出し
4. **SQL生成**: 完全に別のSQL文
5. **データ取得**: 独立したクエリ結果 (`queriesData[0]` vs `queriesData[1]`)

---

### 📊 影響範囲のまとめ

| 設定項目 | Query Aに影響 | Query Bに影響 | 共有される |
|---------|--------------|--------------|-----------|
| **adhoc_filters** | ✅ | ❌ | ❌ |
| **adhoc_filters_b** | ❌ | ✅ | ❌ |
| **metrics** | ✅ | ❌ | ❌ |
| **metrics_b** | ❌ | ✅ | ❌ |
| **groupby** | ✅ | ❌ | ❌ |
| **groupby_b** | ❌ | ✅ | ❌ |
| **x_axis** | ✅ | ✅ | ✅ (時間軸のみ) |
| **time_grain_sqla** | ✅ | ✅ | ✅ (時間グレインのみ) |

**重要**: `x_axis` と `time_grain_sqla` は共有されるが、これは**表示設定のみ**。WHERE条件（フィルター）には影響しない。

---

### 💡 実用的な意味

#### ✅ 可能なこと

1. **完全に異なる期間のデータを比較**
   - Query A: `WHERE date >= '2024-01-01'`
   - Query B: `WHERE date >= '2023-01-01'`

2. **完全に異なるカテゴリーを比較**
   - Query A: `WHERE region = 'East'`
   - Query B: `WHERE region = 'West'`

3. **複雑な条件の独立設定**
   - Query A: `WHERE status = 'completed' AND amount > 1000`
   - Query B: `WHERE status = 'pending' AND priority = 'high'`

---

#### ❌ できないこと（制約）

1. **Query Aのフィルターをベースに Query Bを相対的にフィルター**
   - 例: Query Aの結果に含まれる顧客IDのみを Query Bで使う
   - 理由: クエリは完全に独立して実行される

2. **一方のクエリ結果を他方の WHERE条件に使用**
   - 例: Query Aの最大値を Query Bのフィルター閾値に使う
   - 理由: クエリは並行して実行される（依存関係なし）

---

### 🎯 ベストプラクティス

#### ✅ 推奨される使い方

1. **同じテーブルで異なるセグメントを比較**
   ```
   Query A: 今年の売上（WHERE year = 2024）
   Query B: 昨年の売上（WHERE year = 2023）
   ```

2. **異なるテーブルのデータを同じグラフに表示**
   ```
   Query A: 実績データ（FROM actual_sales）
   Query B: 予測データ（FROM forecast_sales）
   ```

3. **異なる集計方法を同時に表示**
   ```
   Query A: 合計（SUM(amount)）
   Query B: 平均（AVG(amount)）
   ```

---

#### ⚠️ 避けるべき使い方

1. **依存関係のあるクエリの設定**
   ```
   Query A: ユーザー抽出
   Query B: そのユーザーの購入履歴（依存関係あり）
   → 代わりに JOIN を使う
   ```

2. **一方のフィルターが他方を制約すると期待する**
   ```
   Query A: WHERE region = 'East'
   Query B: Filters Bなし（Query Aのフィルターを継承すると期待）
   → 継承されない！Query Bは全データを取得
   ```

---

## 参考情報

### 主要ファイル

| ファイル | 役割 | 重要度 |
|---------|------|--------|
| `controlPanel.tsx` | UI設定、サフィックスパターン定義 | ⭐⭐⭐ |
| `buildQuery.ts` | FormData分離、独立クエリ構築 | ⭐⭐⭐ |
| `formDataSuffix.ts` | サフィックス処理ユーティリティ | ⭐⭐⭐ |
| `transformProps.ts` | クエリ結果の独立処理 | ⭐⭐ |

---

### 関連する概念

- **Form Data**: ユーザー設定を保持するオブジェクト
- **Query Context**: クエリ構築のコンテキスト情報
- **Suffix Pattern**: `_b` サフィックスによる設定の区別
- **adhoc_filters**: WHERE条件を定義するフィルター配列

---

### デバッグ Tips

#### 1. DATAタブでクエリ結果を確認

- `queriesData[0]` と `queriesData[1]` のデータ件数を比較
- フィルターが正しく適用されているか確認

#### 2. View Query で SQL文を確認

- 各クエリのWHERE句を確認
- 期待通りのフィルターが適用されているか確認

#### 3. Developer Console でFormDataを確認

```javascript
// Explore画面でFormDataを確認
console.log('FormData:', chartProps.formData);
console.log('Query A filters:', chartProps.formData.adhoc_filters);
console.log('Query B filters:', chartProps.formData.adhoc_filters_b);
```

---

## 結論（再掲）

**Mixed ChartのQuery AとQuery Bは完全に独立しています。**

- Query Aの `adhoc_filters` は Query Bに影響を与えません
- Query Bの `adhoc_filters_b` は Query Aに影響を与えません
- 各クエリは独立したSQL文として実行されます
- データ取得、処理、表示のすべての段階で独立性が保たれます

**安心して異なるフィルター条件を設定できます！** 🎉

---

**作成者**: Claude Code
**作成日**: 2026-04-06
**調査対象**: Apache Superset Mixed Timeseries Chart (ECharts)
**関連レポート**:
- [MIXED_CHART_CUSTOMIZE_PROPERTIES.md](./MIXED_CHART_CUSTOMIZE_PROPERTIES.md)
- [MIXED_CHART_SERIES_TYPE.md](./MIXED_CHART_SERIES_TYPE.md)
