# Mixed Chart シリーズタイプ設定 - 調査レポート

**対象**: Apache Superset Mixed Timeseries Chart
**作成日**: 2026-04-02

---

## 質問

> Aのグラフは折れ線、Bのグラフは棒グラフになるのはデフォルトの機能でしょうか？それとも入れ替えや変更は可能でしょうか？

---

## 回答サマリー

**結論**: Mixed Chartでは、**Query AとQuery Bのグラフタイプは独立して自由に変更可能**です。

### デフォルトの設定

- **Query A**: 折れ線グラフ（Line）
- **Query B**: 折れ線グラフ（Line）

**重要**: デフォルトでは**両方とも折れ線グラフ**です。Query Aが折れ線、Query Bが棒グラフという固定の組み合わせではありません。

### 変更可能性

✅ **完全に変更可能**

- Query Aのグラフタイプを変更可能
- Query Bのグラフタイプを変更可能
- 入れ替えも自由（例: A=棒グラフ、B=折れ線）
- 同じタイプにすることも可能（例: A=棒グラフ、B=棒グラフ）

---

## 目次

1. [利用可能なグラフタイプ](#1-利用可能なグラフタイプ)
2. [設定方法](#2-設定方法)
3. [設定の組み合わせ例](#3-設定の組み合わせ例)
4. [技術的詳細](#4-技術的詳細)
5. [よくある質問](#5-よくある質問)

---

## 1. 利用可能なグラフタイプ

Mixed Chartでは、Query AとQuery Bのそれぞれに対して、以下の**7種類のグラフタイプ**から選択できます：

### 1.1 グラフタイプ一覧

| タイプ | 英語名 | 説明 | 用途 |
|--------|--------|------|------|
| **折れ線** | Line | 標準的な折れ線グラフ | トレンドの可視化 |
| **散布図** | Scatter | 点のみを表示（線なし） | データポイントの分布確認 |
| **スムーズ折れ線** | Smooth Line | 曲線で滑らかに接続 | 視覚的に滑らかなトレンド表示 |
| **棒グラフ** | Bar | 縦棒グラフ | 値の比較、カテゴリ別の表示 |
| **階段グラフ（開始）** | Step - start | 階段状（変化点が左） | 段階的な変化の表示 |
| **階段グラフ（中央）** | Step - middle | 階段状（変化点が中央） | 段階的な変化の表示 |
| **階段グラフ（終了）** | Step - end | 階段状（変化点が右） | 段階的な変化の表示 |

### 1.2 視覚的な違い

```
Line（折れ線）:
    ●───●
   /     \
  ●       ●

Scatter（散布図）:
    ●   ●

  ●       ●

Smooth Line（スムーズ折れ線）:
    ●───●
   ╱     ╲
  ●       ●

Bar（棒グラフ）:
  │ ││ │
  │ ││ │
  ┴─┴┴─┴

Step - start（階段グラフ - 開始）:
    ●───┐
  ┌─┘   │
  ●     └─●

Step - middle（階段グラフ - 中央）:
      ●─┐
    ┌─┘ │
  ●─┘   └─●

Step - end（階段グラフ - 終了）:
        ●
    ┌───┘
  ┌─┘
  ●
```

---

## 2. 設定方法

### 2.1 UI上での設定

Mixed Chartの設定画面には、**2つのカスタマイズセクション**があります：

```
┌─────────────────────────────────────────┐
│ Customize Query A                       │
├─────────────────────────────────────────┤
│ Series type: [Line ▼]                   │
│   ├─ Line                               │
│   ├─ Scatter                            │
│   ├─ Smooth Line                        │
│   ├─ Bar                                │
│   ├─ Step - start                       │
│   ├─ Step - middle                      │
│   └─ Step - end                         │
└─────────────────────────────────────────┘

┌─────────────────────────────────────────┐
│ Customize Query B                       │
├─────────────────────────────────────────┤
│ Series type: [Line ▼]                   │
│   ├─ Line                               │
│   ├─ Scatter                            │
│   ├─ Smooth Line                        │
│   ├─ Bar                                │
│   ├─ Step - start                       │
│   ├─ Step - middle                      │
│   └─ Step - end                         │
└─────────────────────────────────────────┘
```

### 2.2 設定手順

1. **Supersetでチャート編集画面を開く**
2. **左側のコントロールパネルをスクロール**
3. **「Customize Query A」セクションを見つける**
   - `Series type`ドロップダウンから好きなタイプを選択
4. **「Customize Query B」セクションを見つける**
   - `Series type`ドロップダウンから好きなタイプを選択
5. **「Update chart」ボタンをクリック**して変更を適用

### 2.3 注意点

- **独立した設定**: Query AとQuery Bは完全に独立しています
- **リアルタイムプレビュー**: 設定を変更すると、チャートがすぐに更新されます
- **保存が必要**: 設定を保存するには、チャートを保存する必要があります

---

## 3. 設定の組み合わせ例

### 3.1 よく使われる組み合わせ

#### パターン1: 折れ線 + 棒グラフ（トレンドと値の比較）
```
Query A: Line（折れ線） - 例: 売上推移
Query B: Bar（棒グラフ） - 例: 顧客数
```
**用途**: トレンドと絶対値を同時に表示したい場合

#### パターン2: 折れ線 + 折れ線（複数のトレンド比較）
```
Query A: Line（折れ線） - 例: 今年の売上
Query B: Line（折れ線） - 例: 昨年の売上
```
**用途**: 2つのトレンドを比較したい場合

#### パターン3: 棒グラフ + 棒グラフ（積み上げ棒グラフ）
```
Query A: Bar（棒グラフ） - 例: 商品A売上
Query B: Bar（棒グラフ） - 例: 商品B売上
```
**用途**: カテゴリ別の内訳を表示したい場合（スタックモードと併用）

#### パターン4: 散布図 + 折れ線（データポイントとトレンド）
```
Query A: Scatter（散布図） - 例: 実績値
Query B: Smooth Line（スムーズ折れ線） - 例: 予測値
```
**用途**: 実績と予測を区別して表示したい場合

#### パターン5: 階段グラフ + 折れ線（段階的変化とトレンド）
```
Query A: Step - start（階段グラフ） - 例: 在庫数
Query B: Line（折れ線） - 例: 出荷数
```
**用途**: 段階的に変化する指標とトレンドを比較したい場合

### 3.2 入れ替え例

**元の設定**:
```
Query A: Line（折れ線）
Query B: Bar（棒グラフ）
```

**入れ替え後**:
```
Query A: Bar（棒グラフ）
Query B: Line（折れ線）
```

**結果**: グラフの種類が入れ替わるだけで、機能的には同じです。視覚的な強調を変えたい場合に有効です。

---

## 4. 技術的詳細

### 4.1 設定パラメータ

**ファイル**: `superset-frontend/plugins/plugin-chart-echarts/src/MixedTimeseries/types.ts`

```typescript
export type EchartsMixedTimeseriesFormData = QueryFormData & {
  // Query Aのシリーズタイプ
  seriesType: EchartsTimeseriesSeriesType;

  // Query Bのシリーズタイプ
  seriesTypeB: EchartsTimeseriesSeriesType;

  // ... その他のプロパティ
};
```

### 4.2 デフォルト値

**ファイル**: `superset-frontend/plugins/plugin-chart-echarts/src/Timeseries/constants.ts:71`

```typescript
export const DEFAULT_FORM_DATA: EchartsTimeseriesFormData = {
  // ... 他の設定
  seriesType: EchartsTimeseriesSeriesType.Line,  // デフォルト: Line
  // ... 他の設定
};
```

**ファイル**: `superset-frontend/plugins/plugin-chart-echarts/src/MixedTimeseries/types.ts:123-124`

```typescript
export const DEFAULT_FORM_DATA: EchartsMixedTimeseriesFormData = {
  // ... 他の設定
  seriesType: TIMESERIES_DEFAULTS.seriesType,   // = Line
  seriesTypeB: TIMESERIES_DEFAULTS.seriesType,  // = Line
  // ... 他の設定
};
```

**重要**: デフォルトでは**両方ともLine（折れ線）**に設定されています。

### 4.3 グラフタイプの定義

**ファイル**: `superset-frontend/plugins/plugin-chart-echarts/src/Timeseries/types.ts:44-52`

```typescript
export enum EchartsTimeseriesSeriesType {
  Line = 'line',
  Scatter = 'scatter',
  Smooth = 'smooth',
  Bar = 'bar',
  Start = 'start',
  Middle = 'middle',
  End = 'end',
}
```

### 4.4 UI設定の実装

**ファイル**: `superset-frontend/plugins/plugin-chart-echarts/src/MixedTimeseries/controlPanel.tsx:132-157`

```typescript
function createCustomizeSection(
  label: string,
  controlSuffix: string,
): ControlSetRow[] {
  return [
    [<ControlSubSectionHeader>{label}</ControlSubSectionHeader>],
    [
      {
        name: `seriesType${controlSuffix}`,
        config: {
          type: 'SelectControl',
          label: t('Series type'),
          renderTrigger: true,
          default: seriesType,
          choices: [
            [EchartsTimeseriesSeriesType.Line, t('Line')],
            [EchartsTimeseriesSeriesType.Scatter, t('Scatter')],
            [EchartsTimeseriesSeriesType.Smooth, t('Smooth Line')],
            [EchartsTimeseriesSeriesType.Bar, t('Bar')],
            [EchartsTimeseriesSeriesType.Start, t('Step - start')],
            [EchartsTimeseriesSeriesType.Middle, t('Step - middle')],
            [EchartsTimeseriesSeriesType.End, t('Step - end')],
          ],
          description: t('Series chart type (line, bar etc)'),
        },
      },
    ],
    // ... 他のコントロール
  ];
}
```

**解説**:
- `createCustomizeSection`関数が各クエリのカスタマイズセクションを生成
- `controlSuffix`によって、空文字列（Query A）または`'B'`（Query B）が付与される
- つまり、`seriesType`（Query A）と`seriesTypeB`（Query B）が生成される

### 4.5 transformPropsでの使用

**ファイル**: `superset-frontend/plugins/plugin-chart-echarts/src/MixedTimeseries/transformProps.ts:170-171`

```typescript
const {
  // ... 他のプロパティ
  seriesType,
  seriesTypeB,
  // ... 他のプロパティ
} = formData;
```

**ファイル**: `transformProps.ts:393, 441`

```typescript
// Query Aのシリーズ設定
const seriesA = {
  // ... 他の設定
  seriesType,  // Query Aのグラフタイプ
  // ... 他の設定
};

// Query Bのシリーズ設定
const seriesB = {
  // ... 他の設定
  seriesType: seriesTypeB,  // Query Bのグラフタイプ
  // ... 他の設定
};
```

**解説**: Query AとQuery Bでそれぞれ独立してグラフタイプが適用されます。

---

## 5. よくある質問

### Q1: デフォルトではQuery Aが折れ線、Query Bが棒グラフですか？

**A**: いいえ、**デフォルトでは両方とも折れ線グラフ（Line）**です。

Query Bを棒グラフにしたい場合は、手動で「Customize Query B」セクションの`Series type`を`Bar`に変更する必要があります。

### Q2: Query AとQuery Bを入れ替えることは可能ですか？

**A**: はい、可能です。

以下の2つの方法があります：

**方法1: グラフタイプを入れ替える**
- Query Aの`Series type`をBarに変更
- Query Bの`Series type`をLineに変更

**方法2: クエリ自体を入れ替える**
- Query AとQuery BのSQL/メトリクスを入れ替える

どちらでも同じ結果が得られます。

### Q3: 両方とも棒グラフにすることは可能ですか？

**A**: はい、可能です。

両方の`Series type`を`Bar`に設定してください。スタックモードをONにすれば、積み上げ棒グラフになります。

### Q4: 同じグラフタイプで異なるスタイルにすることは可能ですか？

**A**: はい、可能です。

`Series type`以外にも、各クエリに対して以下の設定を個別に変更できます：

- **Area**: エリアチャートにするか（折れ線の下を塗りつぶす）
- **Opacity**: 透明度
- **Marker enabled**: データポイントのマーカー表示
- **Marker size**: マーカーのサイズ
- **Stack**: スタックモード
- **Show values**: 値の表示

例:
- Query A: Line、Area ON、Opacity 0.3
- Query B: Line、Area OFF、Opacity 1.0

これで、Query Aは半透明のエリアチャート、Query Bは通常の折れ線になります。

### Q5: 階段グラフ（Step）の3種類の違いは？

**A**: 階段グラフの3種類は、**変化するタイミング**が異なります。

```
Step - start（変化点が左）:
  値が変わる瞬間に、すぐに新しい値にジャンプ
  例: 2021-01-01 00:00から新価格が適用される場合

Step - middle（変化点が中央）:
  期間の中央で値が変わる
  例: 月の真ん中で値が変わる場合

Step - end（変化点が右）:
  期間の最後で値が変わる
  例: 月末時点での在庫数を表示する場合
```

**用途**:
- **Start**: 時点データ（その時点から新しい値が適用される）
- **Middle**: 期間の中央値
- **End**: 期末時点のデータ

### Q6: グラフタイプを変更すると、他の設定も変わりますか？

**A**: いいえ、**グラフタイプの変更は独立**しています。

他の設定（スタックモード、マーカー、透明度など）には影響しません。ただし、一部のグラフタイプでは、特定の設定が機能しない場合があります。

例:
- **Scatter（散布図）**: Area（エリア）設定は無視される
- **Bar（棒グラフ）**: Marker（マーカー）設定は無視される

### Q7: JSON/SQLでグラフタイプを設定できますか？

**A**: はい、可能です。

**チャート設定のJSONで指定**:
```json
{
  "seriesType": "bar",
  "seriesTypeB": "line"
}
```

**APIでチャートを作成する場合**:
```python
chart_config = {
    "viz_type": "echarts_timeseries_mixed",
    "seriesType": "bar",
    "seriesTypeB": "line",
    # ... その他の設定
}
```

---

## まとめ

### ポイント

1. ✅ **Query AとQuery Bのグラフタイプは独立して変更可能**
2. ✅ **デフォルトは両方とも折れ線グラフ（Line）**
3. ✅ **7種類のグラフタイプから選択可能**
4. ✅ **入れ替えや同じタイプの組み合わせも可能**
5. ✅ **UI上で簡単に変更できる**

### 推奨される使い方

- **トレンド比較**: 折れ線 + 折れ線
- **トレンドと値の比較**: 折れ線 + 棒グラフ
- **実績と予測**: 散布図 + スムーズ折れ線
- **積み上げ表示**: 棒グラフ + 棒グラフ（スタックモードON）
- **段階的変化**: 階段グラフ + 折れ線

### 参考ファイル

| ファイル | 内容 |
|---------|------|
| `MixedTimeseries/types.ts:80-81` | seriesType, seriesTypeBの定義 |
| `MixedTimeseries/types.ts:123-124` | デフォルト値 |
| `Timeseries/types.ts:44-52` | EchartsTimeseriesSeriesTypeの定義 |
| `Timeseries/constants.ts:71` | デフォルト値の元 |
| `MixedTimeseries/controlPanel.tsx:132-157` | UI設定の実装 |
| `MixedTimeseries/transformProps.ts:170-171, 393, 441` | グラフタイプの使用箇所 |

---

**作成者**: Claude Code
**作成日**: 2026-04-02
**最終更新**: 2026-04-02
