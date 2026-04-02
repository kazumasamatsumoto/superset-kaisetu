# ツールチップモード（axis vs item）の違いと設定方法

**対象**: Apache Superset Mixed Timeseries Chart
**調査日**: 2026-04-01
**関連ファイル**:
- `plugin-chart-echarts/src/MixedTimeseries/transformProps.ts:584`
- `plugin-chart-echarts/src/controls.tsx:175-186`
- `plugin-chart-echarts/src/MixedTimeseries/controlPanel.tsx:322`

---

## 1. 質問の答え（要約）

### Q: 時系列で全てのデータが見えるのがaxisモードでそのタイミングの値だけ見えるのがitemモードというわけ？

**A: はい、その通りです！**

| モード | 表示内容 | トリガー方法 |
|--------|----------|--------------|
| **axis** | **X軸上の同じ位置にあるすべてのシリーズを表示** | X軸上の任意の位置にマウスオーバー |
| **item** | **マウスオーバーした特定のシリーズのみを表示** | 特定のバー/ポイントにマウスオーバー |

### Q: それはどこで設定の変更ができるの？

**A: チャート編集画面の「Rich tooltip」チェックボックスで変更できます**

- ✅ **Rich tooltip ON（デフォルト）** → `axis`モード
- ❌ **Rich tooltip OFF** → `item`モード

**設定場所**: チャート編集 → Chart Options → Tooltip セクション → **Rich tooltip**

---

## 2. 2つのモードの詳細な違い

### 2.1 axis モード（Rich tooltip ON）

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
- 複数のシリーズを同時に比較できる
- 時点ごとの全体像が把握しやすい
- 特定のバー/ポイントにマウスを合わせる必要がない（X軸上ならどこでもOK）

**デメリット**:
- シリーズ数が多いとツールチップが巨大になる
- 0値も表示されるため、情報が多すぎる場合がある
- チャートを覆い隠してしまう可能性がある

**適用シーン**:
- シリーズ数が少ない（5個以下程度）
- 各時点で複数のメトリクスを比較したい
- 時系列データの推移を詳細に確認したい

### 2.2 item モード（Rich tooltip OFF）

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
- シンプルで見やすい
- 必要な情報のみが表示される
- シリーズ数が多くても問題ない
- ツールチップが小さいので、チャートを覆い隠さない

**デメリット**:
- 複数シリーズを比較するには、複数回マウスオーバーが必要
- 小さいバー/ポイントにマウスを合わせるのが難しい場合がある

**適用シーン**:
- シリーズ数が多い（10個以上）
- 特定のシリーズのみに注目したい
- ツールチップをシンプルに保ちたい

---

## 3. コードレベルでの実装

### 3.1 transformProps.tsでの設定

**ファイル**: `plugin-chart-echarts/src/MixedTimeseries/transformProps.ts:581-646`

```typescript
tooltip: {
  ...getDefaultTooltip(refs),
  show: !inContextMenu,
  trigger: richTooltip ? 'axis' : 'item',  // ← ここで決定
  formatter: (params: any) => {
    // richTooltipの値に応じて処理が変わる
    const xValue: number = richTooltip
      ? params[0].value[0]      // axis: 最初のパラメータから取得
      : params.value[0];         // item: 単一パラメータから取得

    const forecastValue: any[] = richTooltip
      ? params                   // axis: すべてのシリーズの配列
      : [params];                // item: 単一シリーズを配列化

    // ... ツールチップの内容を生成
  },
}
```

**重要なロジック**:

| richTooltip | trigger | params の型 | 説明 |
|-------------|---------|-------------|------|
| `true` | `'axis'` | `Array<ParamObject>` | すべてのシリーズの配列 |
| `false` | `'item'` | `ParamObject` | 単一のシリーズオブジェクト |

### 3.2 controls.tsxでの定義

**ファイル**: `plugin-chart-echarts/src/controls.tsx:175-186`

```typescript
const richTooltipControl: ControlSetItem = {
  name: 'rich_tooltip',
  config: {
    type: 'CheckboxControl',
    label: t('Rich tooltip'),
    renderTrigger: true,
    default: true,                    // ← デフォルトはON（axisモード）
    description: t(
      'Shows a list of all series available at that point in time',
    ),
  },
};
```

**設定内容**:
- **name**: `'rich_tooltip'`（formDataのキー）
- **type**: `'CheckboxControl'`（チェックボックス）
- **label**: `'Rich tooltip'`（UI表示名）
- **default**: `true`（デフォルトでON）
- **description**: `'Shows a list of all series available at that point in time'`

### 3.3 controlPanel.tsxでの配置

**ファイル**: `plugin-chart-echarts/src/MixedTimeseries/controlPanel.tsx:322`

```typescript
{
  label: t('Chart Options'),
  expanded: true,
  controlSetRows: [
    ['color_scheme'],
    ['time_shift_color'],
    ...createCustomizeSection(t('Query A'), ''),
    ...createCustomizeSection(t('Query B'), 'B'),
    // ... その他の設定 ...
    [xAxisLabelRotation],
    ...richTooltipSection,  // ← ここでツールチップセクションを追加
    // ... Y軸の設定 ...
  ],
}
```

**richTooltipSectionの内容**（controls.tsx:242-249）:

```typescript
export const richTooltipSection: ControlSetRow[] = [
  [<ControlSubSectionHeader>{t('Tooltip')}</ControlSubSectionHeader>],
  [richTooltipControl],              // ← Rich tooltip チェックボックス
  [tooltipTotalControl],             // ← Show total（Mixed Chartでは非表示）
  [tooltipPercentageControl],        // ← Show percentage（Mixed Chartでは非表示）
  [tooltipSortByMetricControl],      // ← Sort by metric
  [tooltipTimeFormatControl],        // ← Time format
];
```

**注意**: Mixed Chartでは、`tooltipTotalControl`と`tooltipPercentageControl`は非表示になります（223行目、238行目の`form_data.viz_type !== VizType.MixedTimeseries`条件）。

---

## 4. UI上での設定方法（ステップバイステップ）

### 4.1 チャート編集画面での設定

**Step 1: チャート編集画面を開く**
1. ダッシュボードまたはチャート一覧から対象のMixed Chartを選択
2. 「Edit Chart」ボタンをクリック

**Step 2: Chart Options セクションを開く**
1. 左側のパネルで「Chart Options」セクションを探す
2. セクションが閉じている場合は、クリックして展開

**Step 3: Tooltip サブセクションを見つける**
1. Chart Optionsセクション内で下にスクロール
2. 「Tooltip」というサブセクションヘッダーを見つける
   - X Axis セクションの下
   - Y Axis セクションの上

**Step 4: Rich tooltip を設定**
1. ✅ **チェックON** → axisモード（すべてのシリーズを表示）
2. ❌ **チェックOFF** → itemモード（マウスオーバーしたシリーズのみ）

**Step 5: 設定を適用**
1. 「Run」ボタンをクリックして変更を確認
2. 問題なければ「Save」ボタンで保存

### 4.2 設定の確認方法

**ブラウザ開発者ツールでの確認**:

```javascript
// Console タブで実行
const chart = echarts.getInstanceByDom(document.querySelector('.echarts-for-react'));
const option = chart.getOption();

// tooltipの設定を確認
console.log(option.tooltip[0].trigger);
// 出力: "axis" または "item"
```

---

## 5. 実際の動作の違い（具体例）

### 例: 4つのシリーズがあるMixed Chart

**データ**:
```
2021-01-15時点:
- Query A - Metric 1: 1,234
- Query A - Metric 2: 5,678
- Query B - Metric 1: 900
- Query B - Metric 2: 0
```

### 5.1 axis モード（Rich tooltip ON）の動作

**ユーザーアクション**:
- X軸の「2021-01-15」付近にマウスをホバー

**ツールチップ表示**:
```
┌──────────────────────────────────┐
│ 2021-01-15                       │
├──────────────────────────────────┤
│ ● Query A - Metric 1    1,234    │
│ ● Query A - Metric 2    5,678    │
│ ● Query B - Metric 1      900    │
│ ● Query B - Metric 2        0    │
└──────────────────────────────────┘
```

**特徴**:
- 4つすべてのシリーズが表示される
- 0値も表示される
- X軸上のどこでも同じツールチップが表示される（その時点のデータ）

### 5.2 item モード（Rich tooltip OFF）の動作

**ユーザーアクション**:
- 「Query A - Metric 1」のバーにマウスをホバー

**ツールチップ表示**:
```
┌──────────────────────────────────┐
│ 2021-01-15                       │
├──────────────────────────────────┤
│ ● Query A - Metric 1    1,234    │
└──────────────────────────────────┘
```

**特徴**:
- マウスオーバーしたシリーズのみ表示
- 他のシリーズを見るには、個別にマウスオーバーが必要
- ツールチップが小さくてシンプル

---

## 6. params パラメータの違い

### 6.1 axis モード（Rich tooltip ON）

**EChartsから渡されるparamsの構造**:

```typescript
// params は配列（複数シリーズ）
params = [
  {
    seriesName: "Query A - Metric 1",
    seriesId: "Query A - Metric 1",
    value: [1609459200000, 1234],  // [timestamp, value]
    marker: '<span style="...">●</span>',
    // ... その他のプロパティ
  },
  {
    seriesName: "Query A - Metric 2",
    seriesId: "Query A - Metric 2",
    value: [1609459200000, 5678],
    marker: '<span style="...">●</span>',
  },
  {
    seriesName: "Query B - Metric 1",
    seriesId: "Query B - Metric 1",
    value: [1609459200000, 900],
    marker: '<span style="...">●</span>',
  },
  {
    seriesName: "Query B - Metric 2",
    seriesId: "Query B - Metric 2",
    value: [1609459200000, 0],
    marker: '<span style="...">●</span>',
  }
]
```

**処理**:

```typescript
const xValue: number = params[0].value[0];  // 最初のシリーズからX値を取得
const forecastValue: any[] = params;        // 配列をそのまま使用
```

### 6.2 item モード（Rich tooltip OFF）

**EChartsから渡されるparamsの構造**:

```typescript
// params は単一オブジェクト
params = {
  seriesName: "Query A - Metric 1",
  seriesId: "Query A - Metric 1",
  value: [1609459200000, 1234],
  marker: '<span style="...">●</span>',
  // ... その他のプロパティ
}
```

**処理**:

```typescript
const xValue: number = params.value[0];   // オブジェクトから直接X値を取得
const forecastValue: any[] = [params];    // 配列化して統一的に処理
```

---

## 7. パフォーマンスとUXの考慮

### 7.1 シリーズ数によるモード選択の推奨

| シリーズ数 | 推奨モード | 理由 |
|-----------|-----------|------|
| 1-5個 | **axis** | すべて表示しても問題ない |
| 6-10個 | **どちらでも** | ユースケース次第 |
| 11個以上 | **item** | ツールチップが大きくなりすぎる |

### 7.2 データの特性による選択

| データ特性 | 推奨モード | 理由 |
|-----------|-----------|------|
| 密なデータ（ほとんどの時点に値あり） | **axis** | すべて比較しやすい |
| 疎なデータ（多くが0やnull） | **item** | 0値が大量に表示されるのを避ける |
| 高頻度の時系列（秒単位など） | **axis** | ピンポイントでホバーしにくい |
| 低頻度の時系列（月単位など） | **どちらでも** | - |

### 7.3 ツールチップのパフォーマンス

**axis モード**:
- ツールチップのレンダリングコストが高い（多くのHTMLを生成）
- 多数のシリーズがある場合、マウス移動時の再描画が遅延する可能性
- ブラウザのメモリ使用量が増加

**item モード**:
- ツールチップが小さいため、レンダリングが高速
- マウス移動時の負荷が低い
- ブラウザのメモリ効率が良い

**推奨**:
- **100シリーズ以上**: 必ずitemモードを使用
- **リアルタイムダッシュボード**: itemモードを推奨
- **高解像度モニター**: axisモードでも大きなツールチップを表示可能

---

## 8. 他のツールチップ設定との関係

### 8.1 Tooltip Time Format

**設定**: `tooltipTimeFormat`

**影響**:
- axisモード: ツールチップのタイトル（時点）のフォーマット
- itemモード: 同様にタイトルのフォーマット

**例**:
```typescript
// 設定: "%Y-%m-%d %H:%M"
// 表示: 2021-01-15 14:30
```

### 8.2 Sort by Metric

**設定**: `tooltipSortByMetric`

**影響**:
- axisモード: シリーズを値の降順でソート（デフォルトは定義順）
- itemモード: **無効**（単一シリーズなので意味がない）

**visibility条件**（controls.tsx:209）:

```typescript
visibility: ({ controls }: ControlPanelsContainerProps) =>
  Boolean(controls?.rich_tooltip?.value),
```

→ **Rich tooltip OFFの場合、この設定は非表示になる**

---

## 9. よくある問題とトラブルシューティング

### 問題1: ツールチップが大きすぎてチャートが見えない

**原因**: axisモードで多数のシリーズを表示している

**解決策**:
1. **itemモードに変更**（Rich tooltip OFF）
2. シリーズ数を減らす（フィルターや集計で）
3. SQLクエリで0値を除外（WHERE句）

### 問題2: itemモードでバーにマウスを合わせにくい

**原因**: バーが細い、またはポイントが小さい

**解決策**:
1. **axisモードに変更**（Rich tooltip ON）
2. Marker Sizeを大きくする（Line/Scatterの場合）
3. チャートのサイズを拡大

### 問題3: axisモードなのに1つのシリーズしか表示されない

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

### 問題4: 0値が大量に表示される

**原因**: スタックモードONで欠損値が0に変換されている

**解決策**:
1. **itemモードに変更**
2. SQLクエリで0値を除外
3. スタックモードをOFFにする（欠損値が0に変換されなくなる）

**参考**: MISSING_VALUES_REPORT.md

---

## 10. まとめ

### 10.1 axis vs item の選択基準

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

### 10.2 設定場所の再確認

**UI**: Chart Options → Tooltip → **Rich tooltip** チェックボックス

**コード**:
- **定義**: `plugin-chart-echarts/src/controls.tsx:175-186`
- **使用**: `plugin-chart-echarts/src/MixedTimeseries/transformProps.ts:584`
- **配置**: `plugin-chart-echarts/src/MixedTimeseries/controlPanel.tsx:322`

**formDataキー**: `rich_tooltip`

**デフォルト値**: `true`（axisモード）

---

**調査者**: Claude Code
**レビュー推奨**: UX/UIチーム、フロントエンド開発チーム
