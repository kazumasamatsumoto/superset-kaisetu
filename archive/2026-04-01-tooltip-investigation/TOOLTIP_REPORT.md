# Mixed Chart ツールチップ表示メカニズム 詳細解説

**対象**: Apache Superset Mixed Timeseries Chart
**調査日**: 2026-04-01
**関連Issue**: [#4462](https://github.com/apache/superset/issues/4462)

---

## 1. 概要

Mixed Chart（混合時系列チャート）において、棒グラフをマウスオーバーした際にツールチップに値が表示される仕組みと、**0値が表示される問題**について詳細に解説します。

---

## 2. ツールチップ表示の仕組み

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

### 2.2 各処理ステップの詳細

#### Step 1: EChartsのtooltip設定

**ファイル**: `superset-frontend/plugins/plugin-chart-echarts/src/MixedTimeseries/transformProps.ts:581-646`

```typescript
tooltip: {
  ...getDefaultTooltip(refs),
  show: !inContextMenu,
  trigger: richTooltip ? 'axis' : 'item',
  formatter: (params: any) => {
    // ツールチップの内容を生成
  },
}
```

**重要なパラメータ**:
- `trigger`: `'axis'`（リッチツールチップ）または`'item'`（通常ツールチップ）
  - **`'axis'`モード**: X軸上の同じ位置にあるすべてのシリーズの値を表示
  - **`'item'`モード**: マウスオーバーした特定のシリーズのみを表示

#### Step 2: tooltip.formatter 関数の実装

**ファイル**: `transformProps.ts:585-646`

```typescript
formatter: (params: any) => {
  // X軸の値を取得
  const xValue: number = richTooltip
    ? params[0].value[0]
    : params.value[0];

  const forecastValue: any[] = richTooltip ? params : [params];

  // ツールチップに表示する系列をソート
  const sortedKeys = extractTooltipKeys(
    forecastValue,
    1,
    richTooltip,
    tooltipSortByMetric,
  );

  const rows: string[][] = [];

  // 各シリーズの値を抽出
  const forecastValues =
    extractForecastValuesFromTooltipParams(forecastValue);

  const keys = Object.keys(forecastValues);
  let focusedRow;

  // 各シリーズを行として追加
  sortedKeys
    .filter(key => keys.includes(key))
    .forEach(key => {
      const value = forecastValues[key];
      // フォーマッター選択ロジック
      const tooltipFormatter = getFormatter(...);

      // シリーズを整形
      const row = formatForecastTooltipSeries({
        ...value,
        seriesName: key,
        formatter: tooltipFormatter,
      });
      rows.push(row);

      if (key === focusedSeries) {
        focusedRow = rows.length - 1;
      }
    });

  return tooltipHtml(rows, tooltipFormatter(xValue), focusedRow);
},
```

#### Step 3: 値の抽出（extractForecastValuesFromTooltipParams）

**ファイル**: `superset-frontend/plugins/plugin-chart-echarts/src/utils/forecast.ts:57-83`

```typescript
export const extractForecastValuesFromTooltipParams = (
  params: any[],
  isHorizontal = false,
): Record<string, ForecastValue> => {
  const values: Record<string, ForecastValue> = {};
  params.forEach(param => {
    const { marker, seriesId, value } = param;
    const context = extractForecastSeriesContext(seriesId);
    const numericValue = isHorizontal ? value[0] : value[1];

    // 重要: numericValue が number 型かチェック
    if (typeof numericValue === 'number') {
      if (!(context.name in values))
        values[context.name] = {
          marker: marker || '',
        };
      const forecastValues = values[context.name];
      if (context.type === ForecastSeriesEnum.Observation)
        forecastValues.observation = numericValue;
      // ... その他の予測値の処理
    }
  });
  return values;
};
```

**重要ポイント**:
- `typeof numericValue === 'number'` のチェック
- **0も`number`型なので、この条件を通過する**
- つまり、**0値は正常に抽出される**

#### Step 4: シリーズの整形（formatForecastTooltipSeries）

**ファイル**: `forecast.ts:85-117`

```typescript
export const formatForecastTooltipSeries = ({
  seriesName,
  observation,
  forecastTrend,
  forecastLower,
  forecastUpper,
  marker,
  formatter,
}: ForecastValue & {
  seriesName: string;
  marker: TooltipMarker;
  formatter: ValueFormatter;
}): string[] => {
  const name = `${marker}${sanitizeHtml(seriesName)}`;

  // 重要: observation が number 型かチェック
  let value = typeof observation === 'number' ? formatter(observation) : '';

  if (forecastTrend || forecastLower || forecastUpper) {
    // 予測値の処理
    // ...
  }
  return [name, value];
};
```

**重要ポイント**:
- `typeof observation === 'number'` のチェック
- **0も`number`型なので、`formatter(0)`が実行される**
- フォーマッター（例: `getNumberFormatter(',.0f')`）は0を`"0"`として整形
- **結果: 0値も正常にツールチップに表示される**

#### Step 5: HTML生成（tooltipHtml）

**ファイル**: `superset-frontend/packages/superset-ui-core/src/utils/tooltip.ts:28-58`

```typescript
export function tooltipHtml(
  data: string[][],
  title?: string,
  focusedRow?: number,
) {
  const titleRow = title
    ? `<span style="font-weight: 700;${TRUNCATION_STYLE}">${title}</span>`
    : '';
  return sanitizeHtml(`
    <div>
      ${titleRow}
      <table>
          ${data.length === 0 ? `<tr><td>${t('No data')}</td></tr>` : ''}
          ${data
            .map((row, i) => {
              const rowStyle =
                i === focusedRow ? 'font-weight: 700;' : 'opacity: 0.8;';
              const cells = row.map((cell, j) => {
                const cellStyle = `
                  text-align: ${j > 0 ? 'right' : 'left'};
                  padding-left: ${j === 0 ? 0 : 16}px;
                  ${TRUNCATION_STYLE}
                `;
                return `<td style="${cellStyle}">${cell}</td>`;
              });
              return `<tr style="${rowStyle}">${cells.join('')}</tr>`;
            })
            .join('')}
      </table>
    </div>`);
}
```

**動作**:
- `data`配列の各要素（`[name, value]`）をテーブル行として表示
- `value`が`"0"`の場合も、セルに`"0"`として表示される

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

## 3. 0値が表示される理由

### 3.1 EChartsの動作

**`trigger: 'axis'` モードの場合**:

1. ユーザーがX軸上の特定の位置にマウスオーバー
2. EChartsは、その位置のすべてのシリーズのデータポイントを取得
3. データポイントの値が0でも、それは有効な数値として扱われる
4. **結果**: すべてのシリーズ（0値を含む）がツールチップに表示される

**これはEChartsの標準動作**であり、仕様通りの挙動です。

### 3.2 問題の本質

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

### 3.3 なぜ0を`number`として扱うのか

JavaScriptの型システムでは：

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

---

## 4. 既知の問題と回避策

### 4.1 Issue #4462の詳細

**タイトル**: "[Tooltip] Option to display or ignore 0 or null values"

**報告日**: 2018年2月

**状態**: 未解決（2026年4月時点）

**内容**:
> Charts with many series (particularly time series) are overwhelming the chart (blocking it entirely) with tooltip. Two solutions can be:
> 1. Allow option to configure tooltip with only series user is interested in
> 2. Omit 0 or null series for particular time points

**追加コメント**:
> This is also an issue when a filter is applied to a query - even though the filter excludes values, the tooltip and legend still show these series as 0 values.

**コミュニティの反応**:
- 👍 20+ reactions
- 多数のユーザーが同じ問題に遭遇
- しかし、実装の難しさから未解決

### 4.2 技術的な課題

#### 課題1: データの意味の曖昧性

```typescript
// 以下の3つのケースをどう区別するか？
const case1 = { value: 0 };         // 実際のデータが0
const case2 = { value: null };      // データが存在しない
const case3 = undefined;            // シリーズ自体が存在しない
```

現在の実装では：
- `case1`のみがツールチップに表示される
- しかし、ユーザーは「実際に0」なのか「フィルター除外」なのかわからない

#### 課題2: 設定の複雑さ

0値を非表示にするオプションを追加する場合：

1. **チャート全体の設定**:
   - すべてのシリーズに対して0を非表示
   - 問題: 実際に0が重要なデータの場合に困る

2. **シリーズごとの設定**:
   - シリーズAは0を表示、シリーズBは非表示
   - 問題: 設定UIが複雑になる

3. **動的な判断**:
   - フィルター適用時のみ0を非表示
   - 問題: フィルター情報をフロントエンドで判断する必要がある

#### 課題3: 後方互換性

- 既存のダッシュボードの挙動が変わる
- デフォルト設定をどうするか
- 既存ユーザーへの影響

### 4.3 回避策

#### 案A: SQLレベルでの除外（推奨）

```sql
-- 0値を持つレコードをクエリ時に除外
SELECT
  timestamp,
  metric_name,
  metric_value
FROM metrics
WHERE metric_value != 0  -- 0を除外
```

**メリット**:
- データソースで制御
- パフォーマンス向上（データ転送量削減）
- ツールチップが自動的に軽くなる

**デメリット**:
- 「値が0」と「データなし」の区別ができなくなる

#### 案B: カスタムフォーマッターの使用

```python
# superset_config.py
def custom_tooltip_formatter(value):
    if value == 0:
        return ''  # 0を空文字列に
    return str(value)
```

**メリット**:
- 柔軟な制御

**デメリット**:
- 現在のSupersetではサポートされていない（要実装）

#### 案C: EChartsテーマのカスタマイズ

```json
{
  "tooltip": {
    "formatter": function(params) {
      // カスタムロジック
      return params.filter(p => p.value[1] !== 0);
    }
  }
}
```

**メリット**:
- EChartsレベルで制御

**デメリット**:
- Supersetのテーマシステムでは関数を含むJSONを扱えない

#### 案D: UI設定の追加（将来の実装）

```typescript
// 理想的な設定
{
  tooltipHideZeroValues: true,  // 0値を非表示
  tooltipHideNullValues: true,  // null値を非表示
  tooltipMaxSeries: 10,         // 最大表示シリーズ数
}
```

**メリット**:
- ユーザーが制御可能
- 柔軟性が高い

**デメリット**:
- 実装コストが高い
- UIの複雑化

---

## 5. コードレベルの修正案（参考）

### 5.1 formatter関数の拡張

**対象ファイル**: `transformProps.ts:585-646`

```typescript
formatter: (params: any) => {
  // ... 既存のコード ...

  sortedKeys
    .filter(key => keys.includes(key))
    .forEach(key => {
      const value = forecastValues[key];

      // 新規: 0値を非表示にするオプション
      if (formData.tooltipHideZeroValues && value.observation === 0) {
        return; // この系列をスキップ
      }

      const tooltipFormatter = getFormatter(...);
      const row = formatForecastTooltipSeries({
        ...value,
        seriesName: key,
        formatter: tooltipFormatter,
      });
      rows.push(row);
    });

  return tooltipHtml(rows, tooltipFormatter(xValue), focusedRow);
},
```

### 5.2 フォームコントロールの追加

**対象ファイル**: `MixedTimeseries/controlPanel.tsx`

```typescript
{
  name: 'tooltipHideZeroValues',
  config: {
    type: 'CheckboxControl',
    label: t('Hide zero values in tooltip'),
    default: false,
    description: t('When enabled, series with zero values will not be shown in the tooltip'),
  },
}
```

### 5.3 型定義の追加

**対象ファイル**: `MixedTimeseries/types.ts`

```typescript
export interface EchartsMixedTimeseriesFormData extends TimeseriesDefaultFormData {
  // ... 既存のプロパティ ...
  tooltipHideZeroValues?: boolean;
}
```

---

## 6. 関連する他のIssue

### Issue #30137: 混合チャートでツールチップが不適切に結合される

**Pull Request**: https://github.com/apache/superset/pull/30137

**内容**:
- 新しいツールチップが混合チャートで複数のシリーズを不適切に結合
- この修正により、各シリーズが正しく分離されて表示されるようになった

**影響**:
- ツールチップの表示精度が向上
- しかし、0値表示の問題は未解決

### Issue #14943: セカンダリY軸のフォーマットがツールチップに反映されない

**内容**:
- Mixed Timeseries Chartのセカンダリ Y軸（サイドバー）のフォーマット設定
- ツールチップには反映されない

**現状**:
- transformProps.ts:619-632で部分的に対応
- しかし、完全な解決には至っていない

---

## 7. まとめ

### 7.1 ツールチップ表示の仕組み

1. **EChartsが検出**: マウス位置のすべてのシリーズのデータポイント
2. **値を抽出**: `extractForecastValuesFromTooltipParams()`
3. **整形**: `formatForecastTooltipSeries()`
4. **HTML生成**: `tooltipHtml()`
5. **表示**: ブラウザにレンダリング

### 7.2 0値が表示される理由

- JavaScriptの型システムで`typeof 0 === 'number'`が`true`
- EChartsの標準動作として、すべての数値データポイントを表示
- **これは仕様通りの挙動**

### 7.3 問題点

- 多数のシリーズがある場合、ツールチップが巨大になる
- フィルター除外されたデータも0として表示される
- 0値を非表示にするオプションが存在しない

### 7.4 現実的な対応

#### 短期（即時対応可能）
1. **SQLクエリで0値を除外**:
   ```sql
   WHERE metric_value != 0
   ```

2. **シリーズ数を削減**:
   - 必要なメトリクスのみをチャートに含める
   - フィルターを活用して不要なシリーズを除外

#### 中期（機能追加の検討）
3. **設定オプションの追加**:
   - `tooltipHideZeroValues`: 0値を非表示
   - `tooltipMaxSeries`: 最大表示シリーズ数
   - コミュニティへの提案・実装

#### 長期（根本的な解決）
4. **Issue #4462への貢献**:
   - GitHub Issue での議論参加
   - 実装案の提示
   - Pull Request の作成

---

## 8. 参考リンク

- [Issue #4462: Tooltip option to display or ignore 0 or null values](https://github.com/apache/superset/issues/4462)
- [PR #30137: Fix tooltip inappropriately combines series on mixed chart](https://github.com/apache/superset/pull/30137)
- [Issue #14943: Mixed Timeseries Chart formatting for secondary Y axis](https://github.com/apache/superset/issues/14943)
- [ECharts Tooltip Documentation](https://echarts.apache.org/en/option.html#tooltip)

---

**調査者**: Claude Code
**レビュー推奨**: フロントエンド開発チーム、UX/UIチーム
