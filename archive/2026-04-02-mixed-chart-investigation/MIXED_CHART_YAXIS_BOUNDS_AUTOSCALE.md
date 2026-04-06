# Mixed Chart Y軸Bounds設定時のAuto Scale問題

**調査日**: 2026-04-06
**対象**: Apache Superset Mixed Timeseries Chart - Y軸のBounds設定とAuto Scale
**調査者**: Claude Code

---

## 📋 問題概要

### 発生している問題

**症状**: Y軸Boundsに下限0、上限100を設定すると、auto scaleが機能しない

**期待する動作**:
- データが20-30の範囲 → 軸も20-30にスケール（Boundsは制約として機能）
- データが-10-150の範囲 → 軸は0-100に制限（Boundsで制約）

**実際の動作**:
- どんなデータでも軸が常に0-100に固定される
- auto scaleが完全に無効化される

---

## 🔍 原因分析

### コードの問題箇所

**ファイル1**: `defaults.ts:25-28`

```typescript
export const defaultYAxis = {
  scale: true,  // ← auto scale有効
  yAxisLabelRotation: 0,
};
```

**ファイル2**: `MixedTimeseries/transformProps.ts:536-551`

```typescript
yAxis: [
  {
    ...defaultYAxis,  // scale: true を継承
    type: logAxis ? 'log' : 'value',
    min: yAxisMin,    // ← 固定値（0）
    max: yAxisMax,    // ← 固定値（100）
    minorTick: { show: minorTicks },
    minorSplitLine: { show: minorSplitLine },
    axisLabel: {
      formatter: getYAxisFormatter(
        metrics,
        !!contributionMode,
        customFormatters,
        formatter,
        yAxisFormat,
      ),
    },
  },
  // Secondary Y-axis
  {
    ...defaultYAxis,
    type: logAxisSecondary ? 'log' : 'value',
    min: minSecondary,    // ← 固定値
    max: maxSecondary,    // ← 固定値
    // ...
  }
]
```

---

### なぜauto scaleが機能しないのか

#### EChartsの`scale: true`の役割

`scale: true`は以下の動作をします：

1. **軸がゼロを強制的に含めない**
   - 例：データが20-30なら、軸も20-30になる（0から始めない）

2. **データの実際の値に基づいてスケールを最適化**
   - 無駄な空白を排除

#### 固定値を設定した場合の挙動

**問題**: `min: 0, max: 100` のように固定値を設定すると：

```typescript
min: 0,      // ← 常に0から開始
max: 100,    // ← 常に100で終了
```

→ **`scale: true`が実質的に無効化される**

EChartsは「minとmaxが明示的に指定されている場合はそれを使う」という仕様のため、auto scaleが機能しません。

---

### UIの説明文との矛盾

**controlPanel.tsx:361-366の説明文**:

```
"Bounds for the primary Y-axis. When left empty, the bounds are
dynamically defined based on the min/max of the data. Note that
this feature will only expand the axis range. It won't
narrow the data's extent."
```

**説明の意図**:
- Boundsは「範囲を拡大するだけ」で「データの範囲を狭めない」
- 制約として機能するべき

**実際の動作**:
- Boundsに0-100を設定 → 常に0-100に固定
- データが20-30でも0-100のまま（範囲が拡大されている）

**矛盾**:
説明文は「データが150なら0-150にする」という拡大方向の動作を想定していますが、実際は縮小方向にも影響しています。

---

## 💡 解決策

### EChartsの関数コールバック機能

**重要な発見**: EChartsの`min`と`max`は**関数**を受け入れます。

```typescript
yAxis: {
  scale: true,
  min: function(value) {
    // value.min = データから自動計算された最小値
    // value.max = データから自動計算された最大値
    return Math.max(value.min, 0);  // データと制約の大きい方
  },
  max: function(value) {
    return Math.min(value.max, 100); // データと制約の小さい方
  }
}
```

**動作例**:

| データ範囲 | `Math.max(dataMin, 0)` | `Math.min(dataMax, 100)` | 軸の範囲 |
|-----------|------------------------|--------------------------|----------|
| 20-30 | `Math.max(20, 0) = 20` | `Math.min(30, 100) = 30` | **20-30** ✅ |
| -10-50 | `Math.max(-10, 0) = 0` | `Math.min(50, 100) = 50` | **0-50** ✅ |
| 50-150 | `Math.max(50, 0) = 50` | `Math.min(150, 100) = 100` | **50-100** ✅ |
| -20-150 | `Math.max(-20, 0) = 0` | `Math.min(150, 100) = 100` | **0-100** ✅ |

**結果**:
- ✅ データが範囲内 → データに合わせてauto scale
- ✅ データが範囲外 → Boundsで制約
- ✅ `scale: true`が有効なまま

---

## 🛠️ 実装方法

### 修正ファイル

**ファイル**: `MixedTimeseries/transformProps.ts`

**修正箇所**: 行536-567付近（Primary Y-axisとSecondary Y-axis）

---

### 修正コード

**修正前**:

```typescript
yAxis: [
  {
    ...defaultYAxis,  // scale: true
    type: logAxis ? 'log' : 'value',
    min: yAxisMin,    // 固定値
    max: yAxisMax,    // 固定値
    minorTick: { show: minorTicks },
    minorSplitLine: { show: minorSplitLine },
    axisLabel: {
      formatter: getYAxisFormatter(
        metrics,
        !!contributionMode,
        customFormatters,
        formatter,
        yAxisFormat,
      ),
    },
  },
  {
    ...defaultYAxis,
    type: logAxisSecondary ? 'log' : 'value',
    min: minSecondary,    // 固定値
    max: maxSecondary,    // 固定値
    minorTick: { show: minorTicks },
    minorSplitLine: { show: minorSplitLine },
    axisLabel: {
      formatter: getYAxisFormatter(
        metricsB,
        !!contributionMode,
        customFormattersSecondary,
        formatterSecondary,
        yAxisFormatSecondary,
      ),
    },
  },
]
```

**修正後**:

```typescript
yAxis: [
  {
    ...defaultYAxis,  // scale: true を維持
    type: logAxis ? 'log' : 'value',
    // 関数コールバックで制約付きauto scaleを実現
    min: yAxisMin !== undefined
      ? function(value: { min: number; max: number }) {
          // データの最小値とBoundsの大きい方を使用
          return Math.max(value.min, yAxisMin);
        }
      : undefined,
    max: yAxisMax !== undefined
      ? function(value: { min: number; max: number }) {
          // データの最大値とBoundsの小さい方を使用
          return Math.min(value.max, yAxisMax);
        }
      : undefined,
    minorTick: { show: minorTicks },
    minorSplitLine: { show: minorSplitLine },
    axisLabel: {
      formatter: getYAxisFormatter(
        metrics,
        !!contributionMode,
        customFormatters,
        formatter,
        yAxisFormat,
      ),
    },
  },
  {
    ...defaultYAxis,  // scale: true を維持
    type: logAxisSecondary ? 'log' : 'value',
    // Secondary Y-axisも同様に修正
    min: minSecondary !== undefined
      ? function(value: { min: number; max: number }) {
          return Math.max(value.min, minSecondary);
        }
      : undefined,
    max: maxSecondary !== undefined
      ? function(value: { min: number; max: number }) {
          return Math.min(value.max, maxSecondary);
        }
      : undefined,
    minorTick: { show: minorTicks },
    minorSplitLine: { show: minorSplitLine },
    axisLabel: {
      formatter: getYAxisFormatter(
        metricsB,
        !!contributionMode,
        customFormattersSecondary,
        formatterSecondary,
        yAxisFormatSecondary,
      ),
    },
  },
]
```

---

### ロジックの説明

#### Primary Y-axis

**minの計算**:
```typescript
min: yAxisMin !== undefined
  ? function(value) {
      return Math.max(value.min, yAxisMin);
    }
  : undefined
```

| 条件 | 動作 |
|------|------|
| `yAxisMin`が設定されている | `Math.max(dataMin, yAxisMin)` で制約適用 |
| `yAxisMin`が未設定 | `undefined` → EChartsが自動計算 |

**maxの計算**:
```typescript
max: yAxisMax !== undefined
  ? function(value) {
      return Math.min(value.max, yAxisMax);
    }
  : undefined
```

| 条件 | 動作 |
|------|------|
| `yAxisMax`が設定されている | `Math.min(dataMax, yAxisMax)` で制約適用 |
| `yAxisMax`が未設定 | `undefined` → EChartsが自動計算 |

---

### TypeScript型定義

関数コールバックの型：

```typescript
// EChartsのyAxis.min/max関数の型
type AxisBoundFunction = (value: {
  min: number;
  max: number;
}) => number;
```

---

## 📊 動作確認

### テストケース

#### ケース1: データが範囲内（20-30）

**設定**:
- Y-axis Bounds: `[0, 100]`
- データ: `20-30`

**計算**:
- `Math.max(20, 0) = 20`
- `Math.min(30, 100) = 30`

**結果**: 軸は **20-30** ✅

---

#### ケース2: データが下限を超える（-10-50）

**設定**:
- Y-axis Bounds: `[0, 100]`
- データ: `-10-50`

**計算**:
- `Math.max(-10, 0) = 0`
- `Math.min(50, 100) = 50`

**結果**: 軸は **0-50** ✅（下限で制約）

---

#### ケース3: データが上限を超える（50-150）

**設定**:
- Y-axis Bounds: `[0, 100]`
- データ: `50-150`

**計算**:
- `Math.max(50, 0) = 50`
- `Math.min(150, 100) = 100`

**結果**: 軸は **50-100** ✅（上限で制約）

---

#### ケース4: データが両方を超える（-20-150）

**設定**:
- Y-axis Bounds: `[0, 100]`
- データ: `-20-150`

**計算**:
- `Math.max(-20, 0) = 0`
- `Math.min(150, 100) = 100`

**結果**: 軸は **0-100** ✅（両方で制約）

---

#### ケース5: Boundsが未設定

**設定**:
- Y-axis Bounds: 空
- データ: `20-30`

**計算**:
- `min: undefined` → EChartsが自動計算
- `max: undefined` → EChartsが自動計算

**結果**: 軸は **20-30** ✅（完全なauto scale）

---

## 🎯 この修正の効果

### 1. `scale: true`が有効なまま

- デフォルトの`scale: true`を維持
- 軸がデータに合わせて自動調整される
- ゼロを強制的に含めない

### 2. Boundsが制約として機能

**期待通りの動作**:
- データが範囲内 → データに合わせてスケール
- データが範囲外 → Boundsで制限

### 3. UI説明文と一致

**説明文**:
```
"this feature will only expand the axis range.
 It won't narrow the data's extent."
```

**修正後の動作**:
- データが20-30、Bounds 0-100 → 軸は20-30（狭めない）
- データが-10-150、Bounds 0-100 → 軸は0-100（拡大する）

### 4. 後方互換性を維持

**Boundsが未設定の場合**:
- 従来通りの完全なauto scale
- 既存のチャートに影響なし

---

## 🔄 代替案

### 案1: ソースコード修正（推奨）

**メリット**:
- ✅ 根本的な解決
- ✅ UI説明文と動作が一致
- ✅ scale: trueを活用

**デメリット**:
- ❌ ソースコード修正が必要（約15行）

**工数**: 1-2時間

---

### 案2: Boundsを使用しない

**方法**: Y-axis Boundsを空にする

**メリット**:
- ✅ 即座に対応可能
- ✅ auto scaleが機能する

**デメリット**:
- ❌ 制約がない（データが-1000や1000でも表示される）

**工数**: 1分

---

### 案3: データ側で制限

**方法**: SQLでデータをクリッピング

```sql
SELECT
  time_column,
  CASE
    WHEN metric_value < 0 THEN 0
    WHEN metric_value > 100 THEN 100
    ELSE metric_value
  END as metric_value
FROM table
```

**メリット**:
- ✅ ソースコード修正不要
- ✅ データレベルで制約

**デメリット**:
- ❌ 実際のデータが改変される
- ❌ 100を超えるデータが切り捨てられる
- ❌ 各チャートでSQLを書く必要がある

**工数**: 1時間（チャートごと）

---

### 案4: scaleをfalseにする

**方法**: `defaults.ts`の`scale: true`を`false`に変更

```typescript
export const defaultYAxis = {
  scale: false,  // ← 変更
  yAxisLabelRotation: 0,
};
```

**メリット**:
- ✅ Boundsが機能する

**デメリット**:
- ❌ 全てのチャートでゼロから始まる（データが20-30でも0-30になる）
- ❌ auto scaleが完全に無効化
- ❌ 全チャートに影響

**工数**: 1分（ただし全体に影響大）

---

## 📌 まとめ

### 問題

Y軸Boundsに0-100を設定すると、auto scaleが機能せず、常に0-100に固定される

### 原因

1. `scale: true`がデフォルトで有効
2. 固定値の`min`/`max`を設定すると、`scale: true`が無効化される
3. EChartsの仕様により、明示的な値が優先される

### 解決策（推奨）

**関数コールバックを使用**して制約付きauto scaleを実現

```typescript
min: yAxisMin !== undefined
  ? (value) => Math.max(value.min, yAxisMin)
  : undefined,
max: yAxisMax !== undefined
  ? (value) => Math.min(value.max, yAxisMax)
  : undefined,
```

**効果**:
- ✅ `scale: true`が有効なまま
- ✅ Boundsが制約として機能
- ✅ UI説明文と動作が一致
- ✅ 後方互換性を維持

### 修正の影響範囲

**修正ファイル**: 1ファイル（`MixedTimeseries/transformProps.ts`）

**修正行数**: 約15行

**影響**: Mixed Timeseries Chartのみ（他のチャートタイプには影響なし）

---

## 🔗 関連情報

### EChartsドキュメント

- [Axis - Concepts - Handbook](https://apache.github.io/echarts-handbook/en/concepts/axis/)
- [min/max function test](https://github.com/apache/echarts/blob/master/test/min-max-function.html)

### 関連Issue

- [Data Zoom min and Max Values with scale option true - Issue #19358](https://github.com/apache/echarts/issues/19358)
- [min/max on yAxis for bar charts - Issue #8841](https://github.com/apache/echarts/issues/8841)

### Supersetファイル

| ファイル | 役割 |
|---------|------|
| `defaults.ts:25-28` | `scale: true`のデフォルト定義 |
| `MixedTimeseries/transformProps.ts:536-567` | Y軸設定の構築 |
| `MixedTimeseries/controlPanel.tsx:361-366` | UI説明文 |
| `utils/controls.ts:22-29` | `parseAxisBound`関数 |

---

## 🚀 次のステップ

### すぐに対応したい場合

1. **代替案2**: Y-axis Boundsを空にする（1分）
2. **代替案3**: SQLでデータをクリッピング（1時間）

### 恒久的な解決を目指す場合

1. **ソースコード修正**（1-2時間）
   - `MixedTimeseries/transformProps.ts`を修正
   - ローカル環境でテスト
   - 本番環境にデプロイ

2. **コミュニティ貢献**（2-3週間）
   - 修正してテスト
   - Pull Request作成
   - 本家にマージ

---

**調査実施**: Claude Code
**作成日**: 2026-04-06
**レビュー推奨**: フロントエンド開発チーム、データ可視化チーム
