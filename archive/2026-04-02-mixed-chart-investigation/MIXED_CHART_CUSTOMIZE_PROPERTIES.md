# Mixed Chart - Customizeプロパティ完全ガイド

**対象**: Apache Superset Mixed Timeseries Chart
**作成日**: 2026-04-02

---

## 目次

1. [概要](#概要)
2. [プロパティ一覧](#プロパティ一覧)
3. [各プロパティの詳細](#各プロパティの詳細)
4. [実践的な設定例](#実践的な設定例)
5. [技術的詳細](#技術的詳細)
6. [よくある質問](#よくある質問)

---

## 概要

Mixed Chartでは、**Query A**と**Query B**のそれぞれに対して、独立したカスタマイズ設定が可能です。

### 設定画面の構成

```
┌─────────────────────────────────────────┐
│ Customize Query A                       │
├─────────────────────────────────────────┤
│ ・Series type                           │
│ ・Stack series                          │
│ ・Area chart                            │
│ ・Show Values                           │
│ ・Opacity                               │
│ ・Marker                                │
│ ・Marker size                           │
│ ・Y Axis                                │
└─────────────────────────────────────────┘

┌─────────────────────────────────────────┐
│ Customize Query B                       │
├─────────────────────────────────────────┤
│ ・Series type                           │
│ ・Stack series                          │
│ ・Area chart                            │
│ ・Show Values                           │
│ ・Opacity                               │
│ ・Marker                                │
│ ・Marker size                           │
│ ・Y Axis                                │
└─────────────────────────────────────────┘
```

### 重要な特徴

- ✅ **完全な独立性**: Query AとQuery Bは完全に独立して設定可能
- ✅ **8つのプロパティ**: 各クエリに対して8つのカスタマイズ項目が利用可能
- ✅ **リアルタイムプレビュー**: 設定変更がすぐにチャートに反映される
- ✅ **柔軟な組み合わせ**: グラフタイプや表示スタイルを自由に組み合わせ可能

---

## プロパティ一覧

| # | プロパティ名 | 型 | デフォルト値 | 説明 |
|---|------------|-----|------------|------|
| 1 | **Series type** | Select | Line | グラフの種類（折れ線、棒グラフなど） |
| 2 | **Stack series** | Checkbox | false | シリーズを積み上げ表示 |
| 3 | **Area chart** | Checkbox | false | 折れ線の下を塗りつぶす |
| 4 | **Show Values** | Checkbox | false | データポイントに数値を表示 |
| 5 | **Opacity** | Slider | 0.2 | 透明度（0.0〜1.0） |
| 6 | **Marker** | Checkbox | false | データポイントにマーカーを表示 |
| 7 | **Marker size** | Slider | 6 | マーカーのサイズ（0〜100） |
| 8 | **Y Axis** | Select | Primary | 使用するY軸（Primary/Secondary） |

---

## 各プロパティの詳細

### 1. Series type（グラフの種類）

**パラメータ名**: `seriesType` (Query A) / `seriesTypeB` (Query B)

#### 説明
グラフの表示形式を選択します。

#### 選択肢

| 値 | 表示名 | 説明 | 用途 |
|---|--------|------|------|
| `line` | **Line** | 標準的な折れ線グラフ | トレンドの可視化 |
| `scatter` | **Scatter** | 散布図（点のみ、線なし） | データポイントの分布確認 |
| `smooth` | **Smooth Line** | 曲線で滑らかに接続 | 視覚的に滑らかなトレンド表示 |
| `bar` | **Bar** | 縦棒グラフ | 値の比較、カテゴリ別の表示 |
| `start` | **Step - start** | 階段状（変化点が左） | 段階的な変化の表示（開始時点） |
| `middle` | **Step - middle** | 階段状（変化点が中央） | 段階的な変化の表示（中央時点） |
| `end` | **Step - end** | 階段状（変化点が右） | 段階的な変化の表示（終了時点） |

#### デフォルト値
```typescript
seriesType: EchartsTimeseriesSeriesType.Line  // 'line'
```

#### 視覚的な違い

```
Line（折れ線）:          Scatter（散布図）:
    ●───●                   ●   ●
   /     \
  ●       ●                ●       ●

Smooth Line:            Bar（棒グラフ）:
    ●───●                  │ ││ │
   ╱     ╲                 │ ││ │
  ●       ●                ┴─┴┴─┴

Step - start:           Step - middle:
    ●───┐                     ●─┐
  ┌─┘   │                   ┌─┘ │
  ●     └─●                ●─┘   └─●

Step - end:
        ●
    ┌───┘
  ┌─┘
  ●
```

#### 使用上の注意
- 折れ線タイプ（Line, Scatter, Smooth, Step）では、AreaやMarkerの設定が有効
- 棒グラフ（Bar）では、Markerの設定は無視される

---

### 2. Stack series（積み上げ表示）

**パラメータ名**: `stack` (Query A) / `stackB` (Query B)

#### 説明
複数のシリーズを積み上げて表示します。

#### 設定値
- **false** (デフォルト): 通常表示（重ねて表示）
- **true**: 積み上げ表示

#### デフォルト値
```typescript
stack: false
```

#### 視覚的な違い

```
Stack OFF（重ねて表示）:
  ●───●
 /     \
●       ●    ▲───▲
            /     \
           ▲       ▲

Stack ON（積み上げ表示）:
      ▲───▲
     /     \
    /       \
   ●─────────●
  /           \
 ●             ●
```

#### 使用例
- **棒グラフで積み上げ**: Query A = Bar + Stack ON、Query B = Bar + Stack ON
- **エリアチャートで積み上げ**: Query A = Line + Area ON + Stack ON、Query B = Line + Area ON + Stack ON

#### 注意点
- Stackを有効にする場合、両方のクエリで同じグラフタイプを使用することが推奨されます
- 散布図（Scatter）では積み上げ表示は意味をなしません

---

### 3. Area chart（エリアチャート）

**パラメータ名**: `area` (Query A) / `areaB` (Query B)

#### 説明
折れ線グラフの下の領域を塗りつぶします。

#### 設定値
- **false** (デフォルト): 通常の折れ線のみ
- **true**: 折れ線の下を塗りつぶす

#### デフォルト値
```typescript
area: false
```

#### 視覚的な違い

```
Area OFF:               Area ON:
    ●───●                   ●───●
   /     \                 /█████\
  ●       ●               ●███████●
                         ▓▓▓▓▓▓▓▓▓▓
```

#### 適用範囲
- ✅ **Line** (折れ線)
- ✅ **Smooth** (スムーズ折れ線)
- ✅ **Step** (階段グラフ)
- ❌ **Scatter** (散布図) - 無視される
- ❌ **Bar** (棒グラフ) - 無視される

#### 使用例
- **エリアチャート**: Series type = Line、Area = ON、Opacity = 0.5
- **積み上げエリアチャート**: Series type = Line、Area = ON、Stack = ON

---

### 4. Show Values（数値表示）

**パラメータ名**: `show_value` (Query A) / `showValueB` (Query B)

#### 説明
データポイント上に数値を表示します。

#### 設定値
- **false** (デフォルト): 数値を表示しない
- **true**: 数値を表示

#### デフォルト値
```typescript
showValue: false
```

#### 視覚的な違い

```
Show Values OFF:        Show Values ON:
    ●───●                  100●───●200
   /     \                   /     \
  ●       ●                50●     ●150
```

#### 使用例
- **棒グラフに数値表示**: Series type = Bar、Show Values = ON
- **折れ線グラフに数値表示**: Series type = Line、Show Values = ON

#### 注意点
- データポイントが多い場合、数値が重なって見づらくなることがあります
- 積み上げグラフでは、各セグメントの値が表示されます

---

### 5. Opacity（透明度）

**パラメータ名**: `opacity` (Query A) / `opacityB` (Query B)

#### 説明
グラフの透明度を設定します。主にエリアチャートで使用されます。

#### 設定値
- **範囲**: 0.0〜1.0
- **ステップ**: 0.1
- **デフォルト**: 0.2

#### デフォルト値
```typescript
opacity: 0.2
```

#### 視覚的な違い

```
Opacity = 0.2 (薄い):   Opacity = 0.8 (濃い):
    ●───●                   ●───●
   /░░░░░\                 /█████\
  ●░░░░░░░●               ●███████●

Opacity = 1.0 (完全不透明):
    ●───●
   /█████\
  ●███████●
```

#### 主な用途
- **エリアチャート**: Area = ONの場合、塗りつぶし部分の透明度を制御
- **重ねて表示**: 複数のグラフを重ねる際、下のグラフが見えるように透明度を調整

#### 設定例
- **半透明のエリアチャート**: Area = ON、Opacity = 0.3
- **ほぼ不透明のエリアチャート**: Area = ON、Opacity = 0.9

#### 注意点
- Opacity = 0.0にすると完全に透明になり、グラフが見えなくなります
- 折れ線のみの場合（Area = OFF）、透明度の効果は限定的です

---

### 6. Marker（マーカー表示）

**パラメータ名**: `markerEnabled` (Query A) / `markerEnabledB` (Query B)

#### 説明
データポイントにマーカー（●）を表示します。

#### 設定値
- **false** (デフォルト): マーカーを表示しない
- **true**: マーカーを表示

#### デフォルト値
```typescript
markerEnabled: false
```

#### 視覚的な違い

```
Marker OFF:             Marker ON:
    ────                   ●───●
   /     \                /     \
  ─       ─              ●       ●
```

#### 適用範囲
- ✅ **Line** (折れ線)
- ✅ **Smooth** (スムーズ折れ線)
- ✅ **Step** (階段グラフ)
- ✅ **Scatter** (散布図) - 常にマーカー表示（この設定は無視される）
- ❌ **Bar** (棒グラフ) - 無視される

#### 使用例
- **マーカー付き折れ線**: Series type = Line、Marker = ON、Marker size = 8
- **大きなマーカー**: Series type = Line、Marker = ON、Marker size = 20

#### 注意点
- データポイントが多い場合、マーカーが重なり見づらくなることがあります
- マーカーサイズは`Marker size`プロパティで調整できます

---

### 7. Marker size（マーカーサイズ）

**パラメータ名**: `markerSize` (Query A) / `markerSizeB` (Query B)

#### 説明
マーカーのサイズを設定します。

#### 設定値
- **範囲**: 0〜100
- **デフォルト**: 6

#### デフォルト値
```typescript
markerSize: 6
```

#### 視覚的な違い

```
Size = 4 (小):          Size = 10 (中):
    •───•                   ●───●
   /     \                 /     \
  •       •               ●       ●

Size = 20 (大):
    ◉───◉
   /     \
  ◉       ◉
```

#### 使用例
- **小さなマーカー**: Marker = ON、Marker size = 4
- **標準サイズ**: Marker = ON、Marker size = 6（デフォルト）
- **強調表示**: Marker = ON、Marker size = 15
- **非常に目立つマーカー**: Marker = ON、Marker size = 30

#### 注意点
- `Marker`が有効（ON）でない場合、このサイズ設定は無視されます
- 散布図（Scatter）では、常にマーカーが表示されるため、このサイズ設定が重要です
- サイズが大きすぎると、データポイント同士が重なり見づらくなることがあります

---

### 8. Y Axis（Y軸の選択）

**パラメータ名**: `yAxisIndex` (Query A) / `yAxisIndexB` (Query B)

#### 説明
クエリをプライマリY軸（左側）またはセカンダリY軸（右側）のどちらに表示するかを選択します。

#### 設定値
- **0** (Primary): プライマリY軸（左側）- デフォルト
- **1** (Secondary): セカンダリY軸（右側）

#### デフォルト値
```typescript
yAxisIndex: 0  // Primary
```

#### 視覚的な違い

```
両方ともPrimary (左軸):
   100 │    ●
    90 │   /
    80 │  ●
       └─────────

Query A = Primary (左軸)、Query B = Secondary (右軸):
   100 │    ●           │ 500
    90 │   /            │ 450
    80 │  ●      ▲      │ 400
       │        /       │ 350
       │       ▲        │ 300
       └─────────────────
```

#### 使用例

**パターン1: 同じスケールの値を比較**
```
Query A: Primary（売上 - 左軸）
Query B: Primary（費用 - 左軸）
```
→ 両方とも金額なので、同じ軸で比較

**パターン2: 異なるスケールの値を比較**
```
Query A: Primary（売上額: 1000万円 - 左軸）
Query B: Secondary（顧客数: 100人 - 右軸）
```
→ スケールが異なるため、別々の軸を使用

**パターン3: 2つのKPIを同時に監視**
```
Query A: Primary（サーバー負荷: 0-100% - 左軸）
Query B: Secondary（レスポンス時間: 0-500ms - 右軸）
```
→ 単位が異なるため、別々の軸を使用

#### Y軸の設定

セカンダリY軸を使用する場合、「Chart Options」セクションで以下の設定が可能です：

| 設定項目 | 説明 |
|---------|------|
| **Secondary y-axis Bounds** | セカンダリY軸の範囲（最小値・最大値） |
| **Secondary y-axis format** | セカンダリY軸の数値フォーマット |
| **Secondary currency format** | セカンダリY軸の通貨フォーマット |
| **Secondary y-axis title** | セカンダリY軸のタイトル |
| **Logarithmic y-axis** | セカンダリY軸を対数スケールにする |

#### 注意点
- セカンダリY軸を使用する場合、適切なラベルやタイトルを設定して、ユーザーが混乱しないようにします
- 両方のクエリを同じY軸に設定する場合、スケールの違いに注意が必要です
- セカンダリY軸は右側に表示されます

---

## 実践的な設定例

### 例1: 売上トレンドと顧客数の比較

**目的**: 売上の推移（折れ線）と顧客数（棒グラフ）を同時に表示

**設定**:
```
Query A（売上）:
  ├─ Series type: Line
  ├─ Stack series: OFF
  ├─ Area chart: ON
  ├─ Show Values: OFF
  ├─ Opacity: 0.3
  ├─ Marker: ON
  ├─ Marker size: 8
  └─ Y Axis: Primary

Query B（顧客数）:
  ├─ Series type: Bar
  ├─ Stack series: OFF
  ├─ Area chart: OFF (無視される)
  ├─ Show Values: ON
  ├─ Opacity: 0.7
  ├─ Marker: OFF (無視される)
  ├─ Marker size: 6 (無視される)
  └─ Y Axis: Secondary
```

**結果**:
- 売上は半透明のエリアチャートとして左軸に表示
- 顧客数は数値付きの棒グラフとして右軸に表示
- 異なる単位のデータを1つのチャートで視覚化

---

### 例2: 今年と昨年の売上比較

**目的**: 今年と昨年の売上を折れ線グラフで比較

**設定**:
```
Query A（今年の売上）:
  ├─ Series type: Line
  ├─ Stack series: OFF
  ├─ Area chart: OFF
  ├─ Show Values: OFF
  ├─ Opacity: 1.0
  ├─ Marker: ON
  ├─ Marker size: 10
  └─ Y Axis: Primary

Query B（昨年の売上）:
  ├─ Series type: Line
  ├─ Stack series: OFF
  ├─ Area chart: OFF
  ├─ Show Values: OFF
  ├─ Opacity: 0.5
  ├─ Marker: ON
  ├─ Marker size: 6
  └─ Y Axis: Primary
```

**結果**:
- 今年の売上は濃い線＋大きなマーカーで強調
- 昨年の売上は薄い線＋小さなマーカーで背景として表示
- 同じY軸を使用して直接比較が可能

---

### 例3: 積み上げエリアチャート（内訳表示）

**目的**: 2つのカテゴリの合計値を積み上げエリアチャートで表示

**設定**:
```
Query A（カテゴリA）:
  ├─ Series type: Line
  ├─ Stack series: ON
  ├─ Area chart: ON
  ├─ Show Values: OFF
  ├─ Opacity: 0.6
  ├─ Marker: OFF
  ├─ Marker size: 6
  └─ Y Axis: Primary

Query B（カテゴリB）:
  ├─ Series type: Line
  ├─ Stack series: ON
  ├─ Area chart: ON
  ├─ Show Values: OFF
  ├─ Opacity: 0.6
  ├─ Marker: OFF
  ├─ Marker size: 6
  └─ Y Axis: Primary
```

**結果**:
- カテゴリAとBが積み上げられて表示
- 各カテゴリの内訳と合計が一目でわかる
- 半透明なので、重なり部分も視認可能

---

### 例4: 実績値（散布図）と予測値（スムーズ折れ線）

**目的**: 実績データと予測データを区別して表示

**設定**:
```
Query A（実績値）:
  ├─ Series type: Scatter
  ├─ Stack series: OFF
  ├─ Area chart: OFF (無視される)
  ├─ Show Values: OFF
  ├─ Opacity: 1.0
  ├─ Marker: ON (常にON)
  ├─ Marker size: 8
  └─ Y Axis: Primary

Query B（予測値）:
  ├─ Series type: Smooth Line
  ├─ Stack series: OFF
  ├─ Area chart: ON
  ├─ Show Values: OFF
  ├─ Opacity: 0.3
  ├─ Marker: OFF
  ├─ Marker size: 6
  └─ Y Axis: Primary
```

**結果**:
- 実績値は散布図として明確に表示
- 予測値は滑らかな曲線＋半透明エリアで表示
- 視覚的に実績と予測を区別可能

---

### 例5: 在庫数（階段グラフ）と出荷数（棒グラフ）

**目的**: 段階的に変化する在庫数と日次の出荷数を表示

**設定**:
```
Query A（在庫数）:
  ├─ Series type: Step - end
  ├─ Stack series: OFF
  ├─ Area chart: OFF
  ├─ Show Values: ON
  ├─ Opacity: 1.0
  ├─ Marker: ON
  ├─ Marker size: 6
  └─ Y Axis: Primary

Query B（出荷数）:
  ├─ Series type: Bar
  ├─ Stack series: OFF
  ├─ Area chart: OFF (無視される)
  ├─ Show Values: ON
  ├─ Opacity: 0.8
  ├─ Marker: OFF (無視される)
  ├─ Marker size: 6 (無視される)
  └─ Y Axis: Secondary
```

**結果**:
- 在庫数は階段グラフで段階的な変化を表示
- 出荷数は棒グラフで日次の値を表示
- 両方に数値を表示して正確な値を確認可能

---

## 技術的詳細

### 型定義

**ファイル**: `superset-frontend/plugins/plugin-chart-echarts/src/MixedTimeseries/types.ts:44-91`

```typescript
export type EchartsMixedTimeseriesFormData = QueryFormData & {
  // ... 共通プロパティ

  // Query A固有のプロパティ
  area: boolean;
  markerEnabled: boolean;
  markerSize: number;
  opacity: number;
  seriesType: EchartsTimeseriesSeriesType;
  showValue: boolean;
  stack: StackType;
  yAxisIndex?: number;

  // Query B固有のプロパティ
  areaB: boolean;
  markerEnabledB: boolean;
  markerSizeB: number;
  opacityB: number;
  seriesTypeB: EchartsTimeseriesSeriesType;
  showValueB: boolean;
  stackB: StackType;
  yAxisIndexB?: number;

  // ... その他
};
```

### デフォルト値

**ファイル**: `superset-frontend/plugins/plugin-chart-echarts/src/Timeseries/constants.ts:64-82`

```typescript
export const DEFAULT_FORM_DATA: EchartsTimeseriesFormData = {
  // Customizeセクションのデフォルト値
  area: false,               // エリアチャート: OFF
  logAxis: false,            // 対数軸: OFF
  markerEnabled: false,      // マーカー: OFF
  markerSize: 6,             // マーカーサイズ: 6
  opacity: 0.2,              // 透明度: 0.2
  seriesType: EchartsTimeseriesSeriesType.Line,  // Line
  stack: false,              // 積み上げ: OFF
  showValue: false,          // 値表示: OFF
  // ...
};
```

**ファイル**: `superset-frontend/plugins/plugin-chart-echarts/src/MixedTimeseries/types.ts:111-130`

```typescript
export const DEFAULT_FORM_DATA: EchartsMixedTimeseriesFormData = {
  // Query Aのデフォルト値
  area: TIMESERIES_DEFAULTS.area,                   // false
  markerEnabled: TIMESERIES_DEFAULTS.markerEnabled, // false
  markerSize: TIMESERIES_DEFAULTS.markerSize,       // 6
  opacity: TIMESERIES_DEFAULTS.opacity,             // 0.2
  seriesType: TIMESERIES_DEFAULTS.seriesType,       // Line
  showValue: TIMESERIES_DEFAULTS.showValue,         // false
  stack: TIMESERIES_DEFAULTS.stack,                 // false
  yAxisIndex: 0,                                    // Primary

  // Query Bのデフォルト値（Query Aと同じ）
  areaB: TIMESERIES_DEFAULTS.area,                  // false
  markerEnabledB: TIMESERIES_DEFAULTS.markerEnabled,// false
  markerSizeB: TIMESERIES_DEFAULTS.markerSize,      // 6
  opacityB: TIMESERIES_DEFAULTS.opacity,            // 0.2
  seriesTypeB: TIMESERIES_DEFAULTS.seriesType,      // Line
  showValueB: TIMESERIES_DEFAULTS.showValue,        // false
  stackB: TIMESERIES_DEFAULTS.stack,                // false
  yAxisIndexB: 0,                                   // Primary
  // ...
};
```

### UI実装

**ファイル**: `superset-frontend/plugins/plugin-chart-echarts/src/MixedTimeseries/controlPanel.tsx:132-262`

```typescript
function createCustomizeSection(
  label: string,
  controlSuffix: string,
): ControlSetRow[] {
  return [
    [<ControlSubSectionHeader>{label}</ControlSubSectionHeader>],

    // 1. Series type
    [{
      name: `seriesType${controlSuffix}`,
      config: {
        type: 'SelectControl',
        label: t('Series type'),
        renderTrigger: true,
        default: seriesType,
        choices: [...],
        description: t('Series chart type (line, bar etc)'),
      },
    }],

    // 2. Stack series
    [{
      name: `stack${controlSuffix}`,
      config: {
        type: 'CheckboxControl',
        label: t('Stack series'),
        renderTrigger: true,
        default: stack,
        description: t('Stack series on top of each other'),
      },
    }],

    // 3. Area chart
    [{
      name: `area${controlSuffix}`,
      config: {
        type: 'CheckboxControl',
        label: t('Area chart'),
        renderTrigger: true,
        default: area,
        description: t(
          'Draw area under curves. Only applicable for line types.',
        ),
      },
    }],

    // 4. Show Values
    [{
      name: `show_value${controlSuffix}`,
      config: {
        type: 'CheckboxControl',
        label: t('Show Values'),
        renderTrigger: true,
        default: showValues,
        description: t(
          'Whether to display the numerical values within the cells',
        ),
      },
    }],

    // 5. Opacity
    [{
      name: `opacity${controlSuffix}`,
      config: {
        type: 'SliderControl',
        label: t('Opacity'),
        renderTrigger: true,
        min: 0,
        max: 1,
        step: 0.1,
        default: opacity,
        description: t('Opacity of area chart.'),
      },
    }],

    // 6. Marker
    [{
      name: `markerEnabled${controlSuffix}`,
      config: {
        type: 'CheckboxControl',
        label: t('Marker'),
        renderTrigger: true,
        default: markerEnabled,
        description: t(
          'Draw a marker on data points. Only applicable for line types.',
        ),
      },
    }],

    // 7. Marker size
    [{
      name: `markerSize${controlSuffix}`,
      config: {
        type: 'SliderControl',
        label: t('Marker size'),
        renderTrigger: true,
        min: 0,
        max: 100,
        default: markerSize,
        description: t(
          'Size of marker. Also applies to forecast observations.',
        ),
      },
    }],

    // 8. Y Axis
    [{
      name: `yAxisIndex${controlSuffix}`,
      config: {
        type: 'SelectControl',
        label: t('Y Axis'),
        choices: [
          [0, t('Primary')],
          [1, t('Secondary')],
        ],
        default: yAxisIndex,
        clearable: false,
        renderTrigger: true,
        description: t('Primary or secondary y-axis'),
      },
    }],
  ];
}
```

**解説**:
- `controlSuffix`が空文字列（Query A）または`'B'`（Query B）
- 例: `seriesType` (Query A) と `seriesTypeB` (Query B)

### Chart Optionsセクションでの使用

**ファイル**: `controlPanel.tsx:297-304`

```typescript
{
  label: t('Chart Options'),
  expanded: true,
  controlSetRows: [
    ['color_scheme'],
    ['time_shift_color'],
    ...createCustomizeSection(t('Query A'), ''),    // Query A
    ...createCustomizeSection(t('Query B'), 'B'),   // Query B
    // ...
  ],
}
```

### transformPropsでの適用

**ファイル**: `superset-frontend/plugins/plugin-chart-echarts/src/MixedTimeseries/transformProps.ts`

```typescript
// formDataから各プロパティを取得
const {
  area,
  areaB,
  markerEnabled,
  markerEnabledB,
  markerSize,
  markerSizeB,
  opacity,
  opacityB,
  seriesType,
  seriesTypeB,
  showValue,
  showValueB,
  stack,
  stackB,
  yAxisIndex,
  yAxisIndexB,
  // ...
} = formData;

// Query Aのシリーズ設定
const seriesA = {
  seriesType,
  stack,
  area,
  opacity,
  markerEnabled,
  markerSize,
  showValue,
  yAxisIndex,
  // ...
};

// Query Bのシリーズ設定
const seriesB = {
  seriesType: seriesTypeB,
  stack: stackB,
  area: areaB,
  opacity: opacityB,
  markerEnabled: markerEnabledB,
  markerSize: markerSizeB,
  showValue: showValueB,
  yAxisIndex: yAxisIndexB,
  // ...
};
```

---

## よくある質問

### Q1: Query AとQuery Bで異なるグラフタイプを組み合わせることはできますか？

**A**: はい、完全に可能です。

例:
- Query A: Line（折れ線）
- Query B: Bar（棒グラフ）

これは非常に一般的な組み合わせです。

---

### Q2: 両方のクエリを同じY軸（Primary）に表示した場合、スケールはどうなりますか？

**A**: 両方のクエリのデータ範囲を考慮した共通のスケールが自動的に設定されます。

例:
- Query A: 10〜100
- Query B: 50〜200

→ Y軸の範囲は自動的に10〜200になります。

手動で範囲を指定したい場合は、「Primary y-axis Bounds」で設定できます。

---

### Q3: 積み上げ（Stack）を使う場合、Query AとQuery Bで異なるグラフタイプにできますか？

**A**: 技術的には可能ですが、推奨されません。

例（非推奨）:
- Query A: Line + Stack ON
- Query B: Bar + Stack ON

→ 視覚的に混乱しやすいため、同じグラフタイプを使用することを推奨します。

推奨される組み合わせ:
- 両方とも Line + Area ON + Stack ON（積み上げエリアチャート）
- 両方とも Bar + Stack ON（積み上げ棒グラフ）

---

### Q4: Markerを有効にしたのに表示されません。なぜですか？

**A**: 以下の原因が考えられます：

1. **グラフタイプが棒グラフ（Bar）**
   - 棒グラフではマーカーは無視されます

2. **Marker sizeが0**
   - サイズが0の場合、マーカーは表示されません

3. **データポイントが多すぎる**
   - データポイントが非常に多い場合、マーカーが小さくて見えないことがあります
   - Marker sizeを大きくしてみてください

---

### Q5: Opacityを変更しても変化が見られません。

**A**: Opacityは主に**エリアチャート（Area = ON）**で効果があります。

- Area = OFFの場合、折れ線自体の透明度は変わりません
- エリアチャートにして、塗りつぶし部分の透明度を確認してください

---

### Q6: Show Valuesを有効にすると、数値が重なって見づらいです。

**A**: 以下の対策があります：

1. **データの粒度を粗くする**
   - 日次データを週次や月次に集計

2. **時間範囲を狭める**
   - 表示する期間を短くしてデータポイントを減らす

3. **Show Valuesを片方のクエリのみに適用**
   - Query A: Show Values = ON
   - Query B: Show Values = OFF

---

### Q7: プライマリY軸とセカンダリY軸の設定はどこで変更できますか？

**A**: 「Chart Options」セクションのY Axisサブセクションで設定できます。

**プライマリY軸の設定**:
- Primary y-axis Bounds: 範囲（最小値・最大値）
- Primary y-axis format: 数値フォーマット
- Currency format: 通貨フォーマット
- Logarithmic y-axis: 対数スケール

**セカンダリY軸の設定**:
- Secondary y-axis Bounds: 範囲（最小値・最大値）
- Secondary y-axis format: 数値フォーマット
- Secondary currency format: 通貨フォーマット
- Secondary y-axis title: タイトル
- Logarithmic y-axis (Secondary): 対数スケール

---

### Q8: JSON/APIでこれらのプロパティを設定する方法は？

**A**: チャート設定のJSONで以下のように指定できます。

```json
{
  "viz_type": "echarts_timeseries_mixed",

  // Query Aの設定
  "seriesType": "line",
  "stack": false,
  "area": true,
  "show_value": false,
  "opacity": 0.3,
  "markerEnabled": true,
  "markerSize": 8,
  "yAxisIndex": 0,

  // Query Bの設定
  "seriesTypeB": "bar",
  "stackB": false,
  "areaB": false,
  "showValueB": true,
  "opacityB": 0.7,
  "markerEnabledB": false,
  "markerSizeB": 6,
  "yAxisIndexB": 1,

  // その他の設定
  "metrics": [...],
  "metrics_b": [...],
  ...
}
```

---

### Q9: 設定を変更したのにチャートに反映されません。

**A**: 以下を確認してください：

1. **「Update chart」ボタンをクリック**
   - 設定を変更した後、必ず「Update chart」をクリックしてください

2. **チャートを保存**
   - 変更を永続化するには、チャートを保存する必要があります

3. **ブラウザをリフレッシュ**
   - キャッシュの問題がある場合、ブラウザをリフレッシュしてみてください

4. **renderTrigger設定の確認**
   - すべてのCustomizeプロパティは`renderTrigger: true`に設定されているため、変更はすぐに反映されるはずです

---

### Q10: デフォルト値をカスタマイズする方法はありますか？

**A**: デフォルト値はコードレベルで定義されています。

変更する場合は、以下のファイルを編集する必要があります：

**ファイル**: `superset-frontend/plugins/plugin-chart-echarts/src/Timeseries/constants.ts:64-82`

```typescript
export const DEFAULT_FORM_DATA: EchartsTimeseriesFormData = {
  area: false,          // ← ここを変更
  markerEnabled: false, // ← ここを変更
  markerSize: 6,        // ← ここを変更
  opacity: 0.2,         // ← ここを変更
  // ...
};
```

ただし、**コアファイルの変更は推奨されません**。代わりに、チャートテンプレートを作成して、よく使う設定を保存することをお勧めします。

---

## まとめ

### 重要なポイント

1. ✅ **8つのプロパティ**: 各クエリに対して8つのカスタマイズ項目が利用可能
2. ✅ **完全な独立性**: Query AとQuery Bは完全に独立して設定可能
3. ✅ **柔軟な組み合わせ**: グラフタイプ、スタイル、Y軸を自由に組み合わせ可能
4. ✅ **リアルタイムプレビュー**: すべての設定は即座にチャートに反映
5. ✅ **デフォルト値**: すべてのプロパティに合理的なデフォルト値が設定済み

### プロパティ早見表

| プロパティ | Query A | Query B | 型 | デフォルト | 範囲/選択肢 |
|-----------|---------|---------|-----|----------|-----------|
| Series type | `seriesType` | `seriesTypeB` | Select | Line | Line, Scatter, Smooth, Bar, Step |
| Stack series | `stack` | `stackB` | Checkbox | false | true/false |
| Area chart | `area` | `areaB` | Checkbox | false | true/false |
| Show Values | `show_value` | `showValueB` | Checkbox | false | true/false |
| Opacity | `opacity` | `opacityB` | Slider | 0.2 | 0.0〜1.0 |
| Marker | `markerEnabled` | `markerEnabledB` | Checkbox | false | true/false |
| Marker size | `markerSize` | `markerSizeB` | Slider | 6 | 0〜100 |
| Y Axis | `yAxisIndex` | `yAxisIndexB` | Select | 0 (Primary) | 0 (Primary), 1 (Secondary) |

### 推奨される設定パターン

| 用途 | Query A | Query B |
|------|---------|---------|
| **トレンド比較** | Line | Line |
| **トレンド+値** | Line + Area | Bar |
| **積み上げエリア** | Line + Area + Stack | Line + Area + Stack |
| **積み上げ棒グラフ** | Bar + Stack | Bar + Stack |
| **実績+予測** | Scatter | Smooth + Area |
| **段階的変化+トレンド** | Step | Line |

### 参考ファイル

| ファイル | 内容 |
|---------|------|
| `MixedTimeseries/types.ts:66-87` | プロパティの型定義 |
| `MixedTimeseries/types.ts:111-130` | デフォルト値 |
| `Timeseries/constants.ts:64-82` | デフォルト値の元 |
| `MixedTimeseries/controlPanel.tsx:132-262` | UI実装 |
| `MixedTimeseries/transformProps.ts` | プロパティの適用ロジック |

---

**作成者**: Claude Code
**作成日**: 2026-04-02
**最終更新**: 2026-04-02
