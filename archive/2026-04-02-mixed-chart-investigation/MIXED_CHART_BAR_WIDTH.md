# Mixed Chart 棒グラフの幅に関する調査レポート

**調査日**: 2026-04-02
**対象**: Apache Superset Mixed Timeseries Chart - 棒グラフの幅の自動調整とカスタマイズ
**調査者**: Claude Code

---

## 📋 調査概要

### 調査のきっかけ

Mixed Chartで棒グラフを使用する際、以下の現象と疑問が発生：

1. **棒グラフが自動で太くなったり細くなったりする**
   - データポイント数やシリーズ数に応じて変化
   - 一貫性のない見た目になることがある

2. **幅をカスタマイズしたい**
   - 特定の幅に固定したい
   - 棒の間隔を調整したい
   - カスタマイズ方法が不明

### 調査結果サマリー

✅ **なぜ自動で太くなったり細くなったりするのか？**
→ EChartsのデフォルト自動計算アルゴリズムによる

✅ **現在カスタマイズできるか？**
→ **できない**（SupersetのUIに設定項目が存在しない）

✅ **カスタマイズを追加する方法は？**
→ コード修正が必要（3つのファイルを編集）

---

## 🔍 棒グラフの幅が自動で変わる理由

### EChartsの自動計算メカニズム

EChartsは以下の要素に基づいて棒グラフの幅を**自動計算**します：

1. **チャートの幅**（描画エリアのピクセル幅）
2. **データポイント数**（X軸のカテゴリ数）
3. **シリーズ数**（表示する系列の数）
4. **barGap**（同じカテゴリ内の異なるシリーズ間の距離）
5. **barCategoryGap**（カテゴリ間の距離）

### 計算式の概要

```
利用可能な幅 = チャート幅 - (カテゴリ間の隙間の合計)
棒の幅 = (利用可能な幅 / データポイント数) - (系列間の隙間)
```

### 具体例

**シナリオ1: データポイントが少ない（5日分）**
```
チャート幅: 800px
データポイント: 5
系列数: 2

→ 各カテゴリの幅: ~160px
→ 棒グラフが太くなる
```

**シナリオ2: データポイントが多い（30日分）**
```
チャート幅: 800px
データポイント: 30
系列数: 2

→ 各カテゴリの幅: ~26px
→ 棒グラフが細くなる
```

**シナリオ3: 系列数が増える（4系列）**
```
チャート幅: 800px
データポイント: 10
系列数: 4

→ 各カテゴリの幅: ~80px
→ 1本あたりの幅: ~15px（系列が多いため細くなる）
```

---

## ⚙️ EChartsのバー幅関連パラメータ

### 1. barWidth

**説明**: 棒グラフの幅を直接指定

**デフォルト値**: `undefined`（自動計算）

**使用例**:
```javascript
series: [{
  type: 'bar',
  barWidth: '60%',  // カテゴリ幅の60%
  // または
  barWidth: 20,     // 固定20px
}]
```

---

### 2. barGap

**説明**: 同じカテゴリ内の異なるシリーズ間の距離

**デフォルト値**: `'20%'`

**動作**:
- `'20%'` = 棒の幅の20%の距離を空ける
- `'0%'` = 隙間なし（棒が隣接）
- `'-100%'` = 完全に重なる

**使用例**:
```javascript
series: [{
  type: 'bar',
  barGap: '30%',  // 30%の隙間
}]
```

**視覚的イメージ**:
```
barGap: '0%'
[■■][■■][■■]

barGap: '20%'
[■■] [■■] [■■]

barGap: '50%'
[■■]  [■■]  [■■]
```

---

### 3. barCategoryGap

**説明**: カテゴリ間の距離

**デフォルト値**: **系列の数に基づいて計算される**（通常20%前後）

**動作**:
- `'40%'` = カテゴリの両側にそれぞれ40%の余白
- 値が大きいほど、カテゴリ間の隙間が広くなる

**使用例**:
```javascript
series: [{
  type: 'bar',
  barCategoryGap: '40%',
}]
```

**視覚的イメージ**:
```
barCategoryGap: '20%'
  [■■■■]    [■■■■]    [■■■■]
   cat1       cat2       cat3

barCategoryGap: '50%'
  [■■■■]        [■■■■]        [■■■■]
   cat1           cat2           cat3
```

---

### 4. barMinWidth / barMaxWidth

**説明**: 棒の最小幅/最大幅を制限

**デフォルト値**:
- `barMinWidth`: カテゴリ軸では `1`、値軸では `null`
- `barMaxWidth`: `undefined`

**使用例**:
```javascript
series: [{
  type: 'bar',
  barMinWidth: 5,   // 最小5px
  barMaxWidth: 50,  // 最大50px
}]
```

---

## 🛠️ Supersetの現状

### 調査結果

**ファイル**: `controlPanel.tsx`（行132-262）

```typescript
function createCustomizeSection(
  label: string,
  controlSuffix: string,
): ControlSetRow[] {
  return [
    // ... 他の設定
    {
      name: `seriesType${controlSuffix}`,
      config: {
        type: 'SelectControl',
        label: t('Series type'),
        choices: [
          [EchartsTimeseriesSeriesType.Bar, t('Bar')],
          // ...
        ],
      },
    },
    // 棒グラフの幅に関する設定は存在しない ❌
  ];
}
```

**発見**:
- ✅ Series type（Line、Bar、Scatterなど）は選択可能
- ✅ Stack series、Opacity、Marker sizeなどの設定は存在
- ❌ **barWidth、barGap、barCategoryGapの設定は存在しない**

---

**ファイル**: `transformers.ts`（行236-309）

```typescript
let plotType;
if (seriesType === 'bar') {
  plotType = 'bar';
}

return {
  ...series,
  type: plotType,
  itemStyle,
  stack: stackId,
  // barWidth、barGap、barCategoryGap の設定なし ❌
};
```

**発見**:
- シリーズオブジェクトを作成する際、`type: 'bar'` は設定される
- **barWidth、barGap、barCategoryGapは一切設定されていない**
- 完全にEChartsのデフォルト動作に依存

---

**コード検索結果**:

```bash
grep -r "barWidth\|barGap\|barCategoryGap" superset-frontend/plugins/plugin-chart-echarts/
# → 結果: 0件
```

**結論**: Superset Mixed Chartのコードには、棒グラフの幅をカスタマイズする機能が**一切実装されていない**

---

## 💡 カスタマイズする方法

### 現状: カスタマイズ不可

**理由**:
1. UIに設定項目がない
2. コードに実装がない
3. EChartsのデフォルト動作のみ使用

### 解決策: コード修正による実装

棒グラフの幅をカスタマイズするには、以下の3つのファイルを修正する必要があります。

---

### ステップ1: types.ts にデフォルト値を追加

**ファイル**: `src/MixedTimeseries/types.ts`

```typescript
export type EchartsMixedTimeseriesFormData = QueryFormData & {
  // ... 既存のプロパティ
  seriesType: EchartsTimeseriesSeriesType;
  stack: boolean;

  // 新規追加 ✨
  barWidth?: string | number;        // 例: '60%' or 20
  barGap?: string;                   // 例: '20%'
  barCategoryGap?: string;           // 例: '40%'
  barMinWidth?: number;              // 例: 5
  barMaxWidth?: number;              // 例: 50
};

export const DEFAULT_FORM_DATA: EchartsMixedTimeseriesFormData = {
  // ... 既存のデフォルト値
  seriesType: EchartsTimeseriesSeriesType.Line,
  stack: false,

  // 新規追加 ✨
  barWidth: undefined,               // 自動計算
  barGap: '20%',                     // EChartsのデフォルト
  barCategoryGap: undefined,         // 系列数に基づく自動計算
  barMinWidth: undefined,
  barMaxWidth: undefined,
};
```

---

### ステップ2: controlPanel.tsx にUI設定を追加

**ファイル**: `src/MixedTimeseries/controlPanel.tsx`

**追加場所**: `createCustomizeSection` 関数内（行158の後）

```typescript
function createCustomizeSection(
  label: string,
  controlSuffix: string,
): ControlSetRow[] {
  return [
    // ... 既存の設定（seriesType、stackなど）

    // 新規追加: 棒グラフの幅設定 ✨
    [
      {
        name: `barWidth${controlSuffix}`,
        config: {
          type: 'TextControl',
          label: t('Bar Width'),
          renderTrigger: true,
          default: undefined,
          description: t(
            'Width of bars. Can be absolute value (e.g., 20) or percentage (e.g., "60%"). ' +
            'Leave empty for automatic calculation.'
          ),
          visibility: ({ controls }) =>
            controls?.[`seriesType${controlSuffix}`]?.value === 'bar',
        },
      },
    ],
    [
      {
        name: `barGap${controlSuffix}`,
        config: {
          type: 'TextControl',
          label: t('Bar Gap'),
          renderTrigger: true,
          default: '20%',
          description: t(
            'Gap between bars of different series in the same category. ' +
            'Default: "20%"'
          ),
          visibility: ({ controls }) =>
            controls?.[`seriesType${controlSuffix}`]?.value === 'bar',
        },
      },
    ],
    [
      {
        name: `barCategoryGap${controlSuffix}`,
        config: {
          type: 'TextControl',
          label: t('Bar Category Gap'),
          renderTrigger: true,
          default: undefined,
          description: t(
            'Gap between different categories. Default: auto-calculated based on series count.'
          ),
          visibility: ({ controls }) =>
            controls?.[`seriesType${controlSuffix}`]?.value === 'bar',
        },
      },
    ],

    // ... 既存の設定（opacity、markerEnabledなど）
  ];
}
```

**重要ポイント**:
- `visibility` を使用して、`seriesType` が `'bar'` の場合のみ表示
- Query A と Query B で独立して設定可能（`controlSuffix` により区別）

---

### ステップ3: transformers.ts で設定を適用

**ファイル**: `src/Timeseries/transformers.ts`

**修正場所**: `transformSeries` 関数（行290-309付近）

**現在のコード**:
```typescript
return {
  ...series,
  type: plotType,
  itemStyle,
  stack: stackId,
  // ...
};
```

**修正後のコード**:
```typescript
// バー幅設定の抽出
const {
  barWidth,
  barGap,
  barCategoryGap,
  barMinWidth,
  barMaxWidth,
} = opts;

// barオプションの構築
const barOptions = plotType === 'bar' ? {
  barWidth,
  barGap,
  barCategoryGap,
  barMinWidth,
  barMaxWidth,
} : {};

// undefined の値を除外（EChartsのデフォルトを使用）
Object.keys(barOptions).forEach(key => {
  if (barOptions[key] === undefined) {
    delete barOptions[key];
  }
});

return {
  ...series,
  type: plotType,
  itemStyle,
  stack: stackId,
  ...barOptions,  // ✨ バー設定を追加
  // ...
};
```

**重要ポイント**:
- `plotType === 'bar'` の場合のみ適用
- `undefined` の値は除外（EChartsのデフォルト動作を維持）

---

### ステップ4: transformProps.ts で設定を渡す

**ファイル**: `src/MixedTimeseries/transformProps.ts`

**修正場所**: `transformSeries` を呼び出す箇所

**追加が必要な箇所（例: Query A の処理）**:

```typescript
const transformedSeriesA = transformSeries(rebasedDataA, colorScale, {
  // ... 既存のオプション
  seriesType: seriesTypeA,
  stack: stackA,

  // 新規追加 ✨
  barWidth: formData.barWidth,
  barGap: formData.barGap,
  barCategoryGap: formData.barCategoryGap,
  barMinWidth: formData.barMinWidth,
  barMaxWidth: formData.barMaxWidth,
});
```

---

## 📊 実装後の使用例

### 例1: 棒の幅を固定（20px）

**設定**:
```
Chart Options > Query A > Bar Width: 20
```

**結果**: データポイント数に関わらず、常に20pxの幅

---

### 例2: カテゴリ幅の60%

**設定**:
```
Chart Options > Query A > Bar Width: 60%
```

**結果**: 各カテゴリの60%を棒の幅として使用

---

### 例3: 棒の間隔を広げる

**設定**:
```
Chart Options > Query A > Bar Gap: 50%
Chart Options > Query A > Bar Category Gap: 40%
```

**結果**: シリーズ間とカテゴリ間の隙間が広がる

---

### 例4: 幅の範囲を制限

**設定**:
```
Chart Options > Query A > Bar Min Width: 5
Chart Options > Query A > Bar Max Width: 50
```

**結果**: 棒の幅が5px〜50pxの範囲に制限される

---

## 🎯 推奨アクション

### 短期対応（即座に対応可能）

現状では**カスタマイズ不可**のため、以下の方法で対処：

1. **チャートの幅を調整**
   - Dashboard編集 → チャートのサイズ変更
   - 幅を広げれば棒が細くなり、狭めれば太くなる

2. **データの期間を調整**
   - 表示する日数を変更
   - 少ない日数 → 太い棒
   - 多い日数 → 細い棒

3. **系列数を調整**
   - 表示するメトリクス数を減らす
   - 系列が少ないほど、1本あたりが太くなる

---

### 中期対応（機能実装）

**工数**: 1-2日

**手順**:
1. types.ts にデフォルト値を追加
2. controlPanel.tsx にUI設定を追加
3. transformers.ts で設定を適用
4. transformProps.ts で設定を渡す
5. ローカル環境でテスト
6. 本番環境にデプロイ

**メリット**:
- ✅ UIから簡単にカスタマイズ可能
- ✅ Query AとQuery Bで独立して設定
- ✅ デフォルト動作は維持（後方互換性）

---

### 長期対応（コミュニティ貢献）

**工数**: 2-3週間

**手順**:
1. 上記の機能を実装
2. テストケースを追加
3. ドキュメントを作成
4. Pull Requestを作成
5. コミュニティレビュー
6. 本家にマージ

**メリット**:
- ✅ 公式機能として他のユーザーも利用可能
- ✅ 長期的なメンテナンスが容易
- ✅ コミュニティへの貢献

---

## 📌 まとめ

### 重要な発見

1. **棒グラフの幅が自動で変わる理由**
   - ✅ EChartsのデフォルト自動計算アルゴリズム
   - ✅ チャート幅、データポイント数、系列数に基づく
   - ✅ barGap（デフォルト20%）とbarCategoryGap（系列数ベース）で制御

2. **現在のカスタマイズ可否**
   - ❌ SupersetのUIには設定項目なし
   - ❌ コードにも実装なし
   - ✅ EChartsのデフォルト動作のみ使用

3. **カスタマイズを追加する方法**
   - ✅ 3つのファイル（types.ts、controlPanel.tsx、transformers.ts）を修正
   - ✅ barWidth、barGap、barCategoryGap、barMinWidth、barMaxWidthを追加
   - ✅ 1-2日の工数で実装可能

---

### データフロー

```
UI設定（controlPanel.tsx）
    ↓
FormData（types.ts）
    ↓
transformProps.ts → transformSeries の opts として渡す
    ↓
transformers.ts → ECharts series オブジェクトに設定
    ↓
ECharts描画
```

---

### 関連ファイル

| ファイル | 役割 | 修正の必要性 |
|---------|------|------------|
| `controlPanel.tsx` | UI設定の定義 | ✅ 必要 |
| `types.ts` | 型定義、デフォルト値 | ✅ 必要 |
| `transformers.ts` | データ変換、series作成 | ✅ 必要 |
| `transformProps.ts` | propsの変換 | ✅ 必要 |

---

### 参考リソース

- [ECharts Bar Series Documentation](https://echarts.apache.org/en/option.html#series-bar)
- [ECharts Handbook - Bar Chart](https://apache.github.io/echarts-handbook/en/how-to/chart-types/bar/basic-bar/)
- [GitHub Issue #20148 - Default value of barCategoryGap and barGap](https://github.com/apache/echarts/issues/20148)

---

**調査実施**: Claude Code
**作成日**: 2026-04-02
**レビュー推奨**: フロントエンド開発チーム、データ可視化チーム
