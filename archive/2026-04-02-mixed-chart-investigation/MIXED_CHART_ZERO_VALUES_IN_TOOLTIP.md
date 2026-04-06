# Mixed Chart - ツールチップに大量の0が表示される問題 調査レポート

**対象**: Apache Superset Mixed Timeseries Chart
**問題**: ツールチップに大量の「0」が表示され、視認性が低下する
**作成日**: 2026-04-02
**最終更新**: 2026-04-02（重要な修正）

---

## 🔴 重要な発見

**検証結果**: Stack series を OFF にしても、0は表示される

**結論**: **Rich Tooltip（axisモード）が有効な限り、0値は必ず表示される**

---

## 目次

1. [問題の概要](#問題の概要)
2. [0が表示される2つの原因](#0が表示される2つの原因)
3. [Rich Tooltipと0表示の関係](#rich-tooltipと0表示の関係)
4. [Stack seriesの役割（誤解の訂正）](#stack-seriesの役割誤解の訂正)
5. [データの確認方法](#データの確認方法)
6. [解決策](#解決策)
7. [技術的詳細](#技術的詳細)
8. [よくある質問](#よくある質問)

---

## 問題の概要

### 症状

Mixed Chartのツールチップに**大量の0が表示**され、重要な情報が埋もれてしまう。

**ツールチップの表示例**:
```
┌─────────────────────────┐
│ 2024-07-01              │
├─────────────────────────┤
│ ● Series A: 100         │ ← 重要な値
│ ● Series B: 0           │
│ ● Series C: 0           │
│ ● Series D: 0           │
│ ● Series E: 0           │
│ ● Series F: 0           │
│ ... (20個以上)          │ ← ノイズ
└─────────────────────────┘
```

### 影響

- ❌ **ツールチップが巨大になる**: 画面の大部分を覆い隠す
- ❌ **視認性の低下**: 重要な非ゼロ値が大量の0に埋もれる
- ❌ **ユーザビリティの低下**: スクロールしないと全体が見えない
- ❌ **パフォーマンス低下**: HTMLレンダリングに時間がかかる

### 発生条件

以下の条件を満たす場合に発生：

| # | 条件 | 影響度 |
|---|------|--------|
| 1 | **Rich tooltip が ON** | **最重要★★★★★** |
| 2 | データに0値が多数存在 | 重要★★★★☆ |
| 3 | シリーズ数が多い（10個以上） | 重要★★★☆☆ |
| 4 | Stack series が ON（N/Aの場合） | 補助的★★☆☆☆ |

**重要**: Rich Tooltip が ON の場合、**0は必ず表示される**

---

## 0が表示される2つの原因

### 原因1: 実際に0のデータが存在する（主要因）

```
DATAタブ:
timestamp | Series A | Series B | Series C
----------+----------+----------+---------
7/1       | 100      | 0        | 0        ← 実際に0
7/2       | 200      | 0        | 0        ← 実際に0
7/3       | 150      | 0        | 0        ← 実際に0
```

**このケース**:
- データベースに実際に0が保存されている
- または、集計結果が0
- Stack series の設定に関係なく0として表示される

**判別方法**: DATAタブで「0」と表示される

---

### 原因2: N/Aが0に変換される（補助的）

```
DATAタブ:
timestamp | Series A | Series B | Series C
----------+----------+----------+---------
7/1       | 100      | N/A      | N/A      ← N/A
7/2       | 200      | N/A      | N/A      ← N/A
7/3       | 150      | N/A      | N/A      ← N/A

↓ Stack series: ON の場合
↓
内部的に0に変換される
↓
ツールチップでは「0」として表示
```

**このケース**:
- データベースにNULLまたはレコードなし
- Stack series が ON の場合のみ0に変換
- Stack series が OFF なら表示されない

**判別方法**: DATAタブで「N/A」と表示される

---

## Rich Tooltipと0表示の関係

### Rich Tooltip（axisモード）の動作

**設定**: Chart Options → Tooltip → **Rich tooltip** チェックボックス

```
Rich tooltip: ON → X軸上のすべてのシリーズを表示
Rich tooltip: OFF → マウスを置いたシリーズのみ表示
```

### なぜRich Tooltip ONで0が表示されるのか

**コードレベルの処理**:

```typescript
// extractForecastValuesFromTooltipParams
if (typeof numericValue === 'number') {  // ← 重要
  values[context.name] = {
    marker: marker || '',
    observation: numericValue,  // 0も含まれる
  };
}
```

**JavaScriptの型チェック**:

```javascript
typeof 0 === 'number'         // true ← 0は数値型
typeof null === 'object'      // true
typeof undefined === 'undefined'  // true
```

**結論**:
- **0は有効な数値データとして扱われる**
- Rich Tooltip ON では、X軸上のすべての数値データを表示
- したがって、**0も必ず表示される**

### Rich Tooltip ON vs OFF の違い

**Rich Tooltip ON（axisモード）**:
```
マウスをX軸上の7/1に置く
↓
7/1時点のすべてのシリーズを表示

┌─────────────────────────┐
│ 7/1                     │
├─────────────────────────┤
│ ● Series A: 100         │
│ ● Series B: 0           │ ← 0も表示
│ ● Series C: 0           │ ← 0も表示
│ ● Series D: 0           │ ← 0も表示
│ ... (すべて表示)        │
└─────────────────────────┘
```

**Rich Tooltip OFF（itemモード）**:
```
マウスをSeries Aのバーに置く
↓
Series Aのみを表示

┌─────────────────────────┐
│ 7/1                     │
├─────────────────────────┤
│ ● Series A: 100         │ ← これだけ
└─────────────────────────┘

0のシリーズは見ない
```

---

## Stack seriesの役割（誤解の訂正）

### よくある誤解

❌ **誤解**: Stack series を OFF にすれば0が消える

✅ **正解**: Stack series は N/A → 0 変換のみを制御

### Stack seriesの実際の役割

**Stack series の役割**: スタック表示のために、N/A（null/undefined）を0に変換する

```
元のデータ（バックエンドから）:
Series A: 100
Series B: null  ← N/A

↓ Stack series: ON の場合
↓
変換後:
Series A: 100
Series B: 0     ← null → 0 に変換

↓ Stack series: OFF の場合
↓
変換後:
Series A: 100
Series B: null  ← そのまま（ツールチップに表示されない）
```

### Stack series が影響するケース・しないケース

| データの状態 | Stack ON | Stack OFF |
|-------------|----------|-----------|
| **実際に0のデータ** | 0として表示 | 0として表示 ← **変わらない** |
| **N/A（null）のデータ** | 0に変換→表示 | 表示されない ← **変わる** |

**重要な発見**:
- 実際に0のデータが存在する場合、Stack series の設定は無関係
- Stack series OFF にしても、0は消えない

---

## データの確認方法

### 手順1: DATAタブで確認（最重要）

1. チャート編集画面を開く
2. 画面上部の **DATA** タブをクリック
3. データテーブルを確認

**確認ポイント**:

```
パターンA: 実際に0
timestamp | Series A | Series B | Series C
----------+----------+----------+---------
7/1       | 100      | 0        | 0        ← 「0」と表示
7/2       | 200      | 0        | 0

→ Stack series の設定に関係なく0として表示される


パターンB: N/A
timestamp | Series A | Series B | Series C
----------+----------+----------+---------
7/1       | 100      | N/A      | N/A      ← 「N/A」と表示
7/2       | 200      | N/A      | N/A

→ Stack series ON の場合のみ0として表示される
```

### 手順2: Rich tooltipの設定を確認

1. チャート編集画面を開く
2. **Chart Options** セクションを展開
3. **Tooltip** サブセクションを探す
4. **Rich tooltip** のチェックボックスを確認

```
┌─────────────────────────────────────────┐
│ Tooltip                                 │
├─────────────────────────────────────────┤
│ ☑ Rich tooltip  ← これがONか確認        │
│   Shows a list of all series at that    │
│   point in time                         │
└─────────────────────────────────────────┘
```

### 手順3: Stack seriesの設定を確認（N/Aの場合のみ）

DATAタブで「N/A」が多い場合のみ確認：

1. **Customize Query A** セクションを探す
2. **Stack series** のチェックボックスを確認

```
┌─────────────────────────────────────────┐
│ Customize Query A                       │
├─────────────────────────────────────────┤
│ Series type: [Line ▼]                   │
│ ☑ Stack series  ← これがONか確認        │
└─────────────────────────────────────────┘
```

### 診断フローチャート

```
┌─────────────────────────────────┐
│ DATAタブを確認                  │
└─────────────────────────────────┘
           ↓
   ┌──────┴──────┐
   ↓             ↓
「0」と表示   「N/A」と表示
   ↓             ↓
【原因1】     【原因2】
実際に0      N/A→0変換
   ↓             ↓
Stack設定     Stack ON?
無関係           ↓
              YES/NO
```

---

## 解決策

### 解決策の優先順位（重要な修正）

| 優先度 | 解決策 | 工数 | 効果 | 適用条件 |
|--------|--------|------|------|----------|
| **🥇 1位** | **Rich Tooltip OFF** | **1分** | **★★★★★** | itemモードで問題ない |
| 🥈 2位 | SQLで0を除外 | 1時間 | ★★★★☆ | データから除外OK |
| 🥉 3位 | tooltipHideZeroValues実装 | 2-3日 | ★★★★★ | axisモード維持 |
| 4位 | シリーズ数を削減 | 1時間 | ★★★☆☆ | シリーズを減らせる |
| 5位 | Stack series OFF | 1分 | ★☆☆☆☆ | N/Aの場合のみ有効 |

---

### 解決策1: Rich Tooltip を OFF にする（最推奨★★★★★）

**最も効果的で即効性のある解決策**

#### 設定方法

1. チャート編集画面を開く
2. **Chart Options** セクションを展開
3. **Tooltip** サブセクションを探す
4. **Rich tooltip** のチェックを外す
5. **Update chart** をクリック

#### 効果

**変更前（Rich Tooltip ON）**:
```
X軸上にマウス → すべてのシリーズが表示

┌─────────────────────────┐
│ 7/1                     │
├─────────────────────────┤
│ ● Series A: 100         │
│ ● Series B: 0           │
│ ● Series C: 0           │
│ ● Series D: 0           │
│ ... (20個)              │
└─────────────────────────┘
```

**変更後（Rich Tooltip OFF）**:
```
特定のバーにマウス → そのシリーズのみ表示

┌─────────────────────────┐
│ 7/1                     │
├─────────────────────────┤
│ ● Series A: 100         │ ← これだけ
└─────────────────────────┘

0のシリーズは見なくて済む
```

#### メリット

- ✅ **即座に対応可能**（1分で完了）
- ✅ **0が自然と目に入らなくなる**
- ✅ **ツールチップがシンプルになる**
- ✅ **コード変更不要**
- ✅ **シリーズ数が多くても問題ない**

#### デメリット

- ❌ 複数シリーズを同時に比較できない
- ❌ 各シリーズに個別にマウスオーバーが必要
- ❌ 小さいバー/ポイントにマウスを合わせにくい

#### 推奨するケース

- ✅ シリーズ数が多い（10個以上）
- ✅ 特定のシリーズのみに注目したい
- ✅ 即座に対応が必要

---

### 解決策2: SQLクエリで0を除外（推奨★★★★☆）

データレベルで0を除外します。

#### SQLクエリ例

**パターンA: 0のみ除外**
```sql
SELECT
  timestamp,
  series_name,
  metric_value
FROM metrics
WHERE metric_value != 0  -- 0を除外
```

**パターンB: 0とNULLの両方を除外**
```sql
SELECT
  timestamp,
  series_name,
  metric_value
FROM metrics
WHERE metric_value IS NOT NULL  -- NULLを除外
  AND metric_value != 0          -- 0を除外
```

**パターンC: 0以上の値のみ取得**
```sql
SELECT
  timestamp,
  series_name,
  metric_value
FROM metrics
WHERE metric_value > 0  -- 正の値のみ
```

#### メリット

- ✅ **パフォーマンス向上**（データ転送量削減）
- ✅ **Rich Tooltip ON のまま使える**
- ✅ **Stack series の設定に関係ない**
- ✅ **バックエンドで処理完了**

#### デメリット

- ❌ **すべてのクエリに手動で追加**が必要
- ❌ **チャート描画にも影響**（0のデータポイントが消える）
- ❌ **「値が0」の情報が失われる**

#### 推奨するケース

- ✅ 0は不要なノイズである
- ✅ チャートから0を完全に削除してよい
- ✅ Rich Tooltip ON を維持したい

---

### 解決策3: tooltipHideZeroValues機能の実装（推奨★★★★★）

**中期的な根本解決策**

#### 概要

`TOOLTIP_COMPREHENSIVE_GUIDE.md` で提案されている機能を実装します。

**新しい設定**:
- `tooltipHideZeroValues`: ツールチップで0値を非表示
- `tooltipHideNullValues`: ツールチップでnull値を非表示

#### 設定UI（実装後）

```
┌─────────────────────────────────────────┐
│ Tooltip                                 │
├─────────────────────────────────────────┤
│ ☑ Rich tooltip                          │
│                                         │
│ ☑ Hide zero values in tooltip           │ ← 新規追加
│   When enabled, series with zero values │
│   will not be shown in the tooltip      │
└─────────────────────────────────────────┘
```

#### 実装の核心部分

**ファイル**: `transformProps.ts`

```typescript
// formatter関数内でフィルタリング
sortedKeys
  .filter(key => keys.includes(key))
  .forEach(key => {
    const value = forecastValues[key];

    // 新規追加: 0値のフィルタリング
    if (tooltipHideZeroValues && value.observation === 0) {
      return;  // 0をスキップ
    }

    // 既存の処理
    const row = formatForecastTooltipSeries({...});
    rows.push(row);
  });
```

#### 効果

**設定例**:
```
Rich tooltip: ON  （axisモード維持）
Hide zero values: ON
```

**結果**:
```
データ:
7/1: Series A = 100, Series B = 0, Series C = 0

ツールチップ:
┌─────────────────────┐
│ 7/1                 │
├─────────────────────┤
│ ● Series A: 100     │ ← 0値は非表示
└─────────────────────┘
```

#### メリット

- ✅ **Rich Tooltip ON のまま使える**（axisモード維持）
- ✅ **チャート描画に影響しない**（ツールチップのみ制御）
- ✅ **柔軟な制御が可能**（設定で切り替え）
- ✅ **実際の0とN/Aから変換された0を区別可能**
- ✅ **すべてのチャートに適用可能**

#### デメリット

- ❌ 実装工数が必要（2-3日）
- ❌ Supersetのコード変更が必要
- ❌ アップグレード時にマージ対応が必要

#### 工数見積もり

- **実装**: 1-2日
- **テスト**: 1日
- **合計**: 2-3日

詳細は `TOOLTIP_COMPREHENSIVE_GUIDE.md` の「セクション7: 実装ガイド」を参照。

---

### 解決策4: シリーズ数を削減（推奨★★★☆☆）

不要なシリーズをチャートから除外します。

#### 方法

**方法A: メトリクスを絞る**
```
Query A:
  Metrics: [重要な指標のみ]
  （不要な指標を削除）
```

**方法B: フィルターを活用**
```sql
SELECT *
FROM metrics
WHERE series_name IN ('Series A', 'Series D')  -- 必要なものだけ
```

#### メリット

- ✅ 即座に対応可能
- ✅ チャート全体が見やすくなる

#### デメリット

- ❌ 必要な情報が失われる可能性

---

### 解決策5: Stack series を OFF にする（推奨★☆☆☆☆）

**N/Aが多い場合のみ有効**

#### 重要な注意

⚠️ **この解決策は、DATAタブで「N/A」が多い場合のみ有効です**

⚠️ **実際に0のデータが存在する場合、効果はありません**

#### 設定方法

1. **Customize Query A** セクションを探す
2. **Stack series** のチェックを外す
3. **Customize Query B** も同様
4. **Update chart** をクリック

#### 効果（N/Aの場合のみ）

```
データ（N/A）:
7/1: Series A = 100, Series B = N/A, Series C = N/A

Stack ON:
→ N/Aが0に変換される
→ ツールチップに0が表示

Stack OFF:
→ N/Aはそのまま
→ ツールチップに表示されない
```

#### 効果がないケース

```
データ（実際に0）:
7/1: Series A = 100, Series B = 0, Series C = 0

Stack ON:
→ 0として表示

Stack OFF:
→ 0として表示（変わらない！）
```

---

## 推奨される解決策フロー

```
┌─────────────────────────────────────┐
│ ツールチップに大量の0が表示される   │
└─────────────────────────────────────┘
              ↓
┌─────────────────────────────────────┐
│ DATAタブで確認                      │
└─────────────────────────────────────┘
       ↓               ↓
     「0」           「N/A」
       ↓               ↓
   【実際に0】      【N/A→0変換】
       ↓               ↓
       ├───────────────┤
       ↓
┌─────────────────────────────────────┐
│ Rich Tooltip ONのまま使いたい？     │
└─────────────────────────────────────┘
    ↓ YES              ↓ NO
    ↓                  ↓
┌──────────┐      【Rich Tooltip OFF】
│ すぐに   │         最も簡単
│ 対応？   │
└──────────┘
↓ YES   ↓ NO
↓       ↓
【SQL    【tooltipHide
 で除外】  ZeroValues実装】
```

---

## 技術的詳細

### Rich Tooltip ON で0が表示される理由

#### JavaScriptの型システム

```javascript
// 0は number 型
typeof 0 === 'number'  // true

// null/undefined は number 型ではない
typeof null === 'object'  // true
typeof undefined === 'undefined'  // true
```

#### extractForecastValuesFromTooltipParams

**ファイル**: `utils/forecast.ts:66-80`

```typescript
export const extractForecastValuesFromTooltipParams = (
  params: any[],
  isHorizontal = false,
): Record<string, ForecastValue> => {
  const values: Record<string, ForecastValue> = {};
  params.forEach(param => {
    const { marker, seriesId, value } = param;
    const numericValue = isHorizontal ? value[0] : value[1];

    // 重要: 0も number 型なのでこの条件を通過する
    if (typeof numericValue === 'number') {  // ← ここ
      if (!(context.name in values))
        values[context.name] = {
          marker: marker || '',
        };
      const forecastValues = values[context.name];
      if (context.type === ForecastSeriesEnum.Observation)
        forecastValues.observation = numericValue;  // ← 0も含まれる
    }
  });
  return values;
};
```

**重要ポイント**:
- `typeof 0 === 'number'` → `true`
- **0は有効な数値として扱われる**
- したがって、**0も必ず抽出される**

#### formatForecastTooltipSeries

**ファイル**: `utils/forecast.ts:99`

```typescript
export const formatForecastTooltipSeries = ({
  seriesName,
  observation,
  marker,
  formatter,
}: ForecastValue & {
  seriesName: string;
  marker: TooltipMarker;
  formatter: ValueFormatter;
}): string[] => {
  const name = `${marker}${sanitizeHtml(seriesName)}`;

  // 重要: 0も number 型なので formatter(0) が実行される
  let value = typeof observation === 'number' ? formatter(observation) : '';

  return [name, value];
};
```

**重要ポイント**:
- `typeof 0 === 'number'` → `true`
- **0も`formatter(0)`が実行される**
- フォーマッター（例: `getNumberFormatter(',.0f')`）は0を`"0"`として整形
- **結果: 0値も正常にツールチップに表示される**

### Stack series の処理

#### fillNeighborValueの決定

**ファイル**: `transformProps.ts:226, 231`

```typescript
const [rawSeriesA] = extractSeries(rebasedDataA, {
  fillNeighborValue: stack ? 0 : undefined,  // Stack ONなら0
  xAxis: xAxisLabel,
});

const [rawSeriesB] = extractSeries(rebasedDataB, {
  fillNeighborValue: stackB ? 0 : undefined,  // Stack ONなら0
  xAxis: xAxisLabel,
});
```

#### extractSeriesでのN/A→0変換

**ファイル**: `utils/series.ts:330-338`

```typescript
const isFillNeighborValue =
  !isDefined(currentValue) &&           // currentValueがnull/undefined
  isNextToDefinedValue &&               // 隣接する値がある
  fillNeighborValue !== undefined;      // fillNeighborValueが指定されている

let value: DataRecordValue | undefined = currentValue;

if (isFillNeighborValue) {
  value = fillNeighborValue;  // ← null → 0 に変換
}
```

**重要**: この処理は**N/A（null/undefined）のみ**に適用される

実際に0のデータには適用されない：
```typescript
isDefined(0) === true  // 0は「定義されている」
→ isFillNeighborValue === false
→ 変換されない
```

### データフロー全体像

```
┌─────────────────────────────────────────┐
│ データベース                            │
│ - Series A: 100                         │
│ - Series B: 0 (実際に0)                 │
│ - Series C: NULL (N/A)                  │
└─────────────────────────────────────────┘
              ↓
┌─────────────────────────────────────────┐
│ DATAタブ                                │
│ - Series A: 100                         │
│ - Series B: 0                           │
│ - Series C: N/A                         │
└─────────────────────────────────────────┘
              ↓
┌─────────────────────────────────────────┐
│ extractSeries (Stack ONの場合)          │
│ - Series A: 100 (そのまま)              │
│ - Series B: 0 (そのまま)                │
│ - Series C: 0 (null→0に変換)           │
└─────────────────────────────────────────┘
              ↓
┌─────────────────────────────────────────┐
│ Rich Tooltip ON の場合                  │
│ extractForecastValuesFromTooltipParams  │
│ - typeof 0 === 'number' → true          │
│ - すべての0を抽出                       │
└─────────────────────────────────────────┘
              ↓
┌─────────────────────────────────────────┐
│ ツールチップ表示                        │
│ - Series A: 100                         │
│ - Series B: 0 (実際の0)                 │
│ - Series C: 0 (N/Aから変換)            │
└─────────────────────────────────────────┘
```

### 関連ファイルとコード

| ファイル | 行番号 | 内容 |
|---------|--------|------|
| `transformProps.ts` | 584 | Rich tooltipの設定 `trigger: richTooltip ? 'axis' : 'item'` |
| `transformProps.ts` | 226, 231 | Stack seriesによる fillNeighborValue決定 |
| `forecast.ts` | 66-80 | `typeof numericValue === 'number'` チェック（0を含む） |
| `forecast.ts` | 99 | `typeof observation === 'number'` でフォーマット（0を含む） |
| `series.ts` | 54-56 | `isDefined(0) === true` （0は定義されている） |
| `series.ts` | 330-338 | N/A → 0 変換処理 |
| `controlPanel.tsx` | 322 | Rich tooltip設定のUI |

---

## よくある質問

### Q1: Stack series を OFF にしたのに、まだ0が表示されます。なぜですか？

**A**: **実際に0のデータが存在する**ためです。

Stack series の役割は「N/A → 0 変換」のみです。実際に0のデータには影響しません。

**確認方法**:
1. DATAタブを開く
2. 「0」と表示される → 実際に0のデータ
3. 「N/A」と表示される → N/Aが0に変換されている

**解決策**:
- Rich Tooltip OFF
- SQLで0を除外
- tooltipHideZeroValues実装

---

### Q2: Rich Tooltip ON のまま、0を非表示にできませんか？

**A**: 現在の標準機能では不可能です。以下の選択肢があります：

**選択肢A: tooltipHideZeroValues機能を実装**（推奨）
- 工数: 2-3日
- Rich Tooltip ON のまま使える
- 設定で柔軟に制御可能

**選択肢B: SQLで0を除外**
- 即座に対応可能
- ただしチャート描画にも影響

---

### Q3: 「実際の0」と「N/Aから変換された0」を区別できますか？

**A**: Rich Tooltip ON の場合、ツールチップでは区別できません。

**Stack series ON の場合**:
```
ツールチップ:
● Series B: 0  ← 実際の0？N/A？ 区別不可
```

**区別する方法**:

1. **DATAタブで確認**
   ```
   Series B: 0   → 実際の0
   Series B: N/A → N/Aから変換された0
   ```

2. **Stack series OFF にする**
   ```
   N/Aは表示されなくなる
   → 表示される0は「実際の0」と確定
   ```

3. **tooltipHideZeroValues + tooltipHideNullValues実装**
   ```
   Hide zero values: OFF → 実際の0は表示
   Hide null values: ON → N/Aは非表示
   ```

---

### Q4: すべてのシリーズが0の時点があります。ツールチップはどうなりますか？

**A**: Rich tooltipの設定により異なります。

**Rich Tooltip ON の場合**:
```
すべて0 → すべて表示

ツールチップ:
┌─────────────────────┐
│ 7/1                 │
├─────────────────────┤
│ ● Series A: 0       │
│ ● Series B: 0       │
│ ● Series C: 0       │
└─────────────────────┘
```

**Rich Tooltip OFF の場合**:
```
0のバーにマウス → そのシリーズのみ表示

ツールチップ:
┌─────────────────────┐
│ 7/1                 │
├─────────────────────┤
│ ● Series A: 0       │
└─────────────────────┘
```

---

### Q5: itemモード（Rich Tooltip OFF）でも使いにくくないですか？

**A**: ユースケース次第です。

**itemモードが適している場合**:
- ✅ シリーズ数が多い（10個以上）
- ✅ 特定のシリーズのみに注目したい
- ✅ 0値が多く、axisモードではノイズが多い

**itemモードが不適切な場合**:
- ❌ 複数シリーズを同時に比較したい
- ❌ 時点ごとの全体像を把握したい
- ❌ バー/ポイントが小さくてマウスを合わせにくい

**推奨**: まず試してみて、使いにくければtooltipHideZeroValues実装を検討

---

### Q6: tooltipHideZeroValues機能を実装する価値はありますか？

**A**: 以下の場合、実装する価値があります：

**実装推奨**:
- ✅ **多数のチャートで0表示問題が発生**
- ✅ **Rich Tooltip ON（axisモード）を維持したい**
- ✅ **チャート描画に影響させたくない**（ツールチップのみ制御）
- ✅ **柔軟な制御が必要**

**実装不要**:
- ❌ 少数のチャートのみで発生
- ❌ Rich Tooltip OFF で問題ない
- ❌ SQLで対応可能

**工数対効果**:
```
実装工数: 2-3日
対応可能チャート数: すべて
持続性: 高
柔軟性: 高（設定で切り替え可能）

→ 多数のチャートがある場合は価値あり
```

---

### Q7: N/Aと実際の0を同時に扱っています。どうすればいいですか？

**A**: 以下のステップで対応してください：

**Step 1: DATAタブで確認**
```
どのシリーズがN/Aで、どのシリーズが実際の0かを確認
```

**Step 2: 優先順位を決める**
```
オプションA: Rich Tooltip OFF
→ 最も簡単、すべての0が目立たなくなる

オプションB: SQLで0とN/Aを除外
→ データから削除してOKな場合

オプションC: tooltipHideZeroValues実装
→ Rich Tooltip ONを維持、最も柔軟
```

**推奨フロー**:
1. まず Rich Tooltip OFF を試す（1分）
2. 問題なければそのまま
3. 問題があれば tooltipHideZeroValues実装を検討

---

### Q8: SQLで0を除外すると、グラフにどう影響しますか？

**A**: **グラフ描画にも影響**があります。

```sql
WHERE metric_value != 0
```

**影響**:
- ✅ ツールチップから0が消える
- ❌ **グラフ上で0のデータポイントが消える**
- ❌ **折れ線グラフの場合、線が途切れる**
- ❌ **棒グラフの場合、0のバーが表示されない**

**ツールチップのみに影響させたい場合**:
- tooltipHideZeroValues実装を検討

---

### Q9: Rich Tooltip ON と OFF、どちらを使うべきですか？

**A**: シリーズ数と0値の多さで判断してください。

| シリーズ数 | 0値の多さ | 推奨モード |
|-----------|----------|-----------|
| 1-5個 | 少ない | **Rich Tooltip ON** |
| 1-5個 | 多い | どちらでも |
| 6-10個 | 少ない | **Rich Tooltip ON** |
| 6-10個 | 多い | **Rich Tooltip OFF** |
| 11個以上 | 少ない | どちらでも |
| 11個以上 | 多い | **Rich Tooltip OFF** |

**一般的なガイドライン**:
- 0値が全シリーズの50%以上 → Rich Tooltip OFF 推奨
- シリーズ数が10個以上 → Rich Tooltip OFF 推奨

---

### Q10: 将来的に公式機能として実装される可能性はありますか？

**A**: 可能性は低いですが、ゼロではありません。

**現状**:
- Issue #4462（2018年開設）は Closed
- 2025年のロードマップに含まれていない
- 公式実装予定はなし

**将来の見通し**:
- 公式実装の可能性: 10-20%
- コミュニティPRのマージ可能性: 60-70%

**推奨アクション**:
1. **短期**: Rich Tooltip OFF または SQLで対応
2. **中期**: tooltipHideZeroValues実装
3. **長期**: コミュニティ貢献（PR作成）を検討

---

## まとめ

### 問題の本質（重要な修正）

**ツールチップに大量の0が表示される理由**:

1. ✅ **Rich Tooltip が ON**（axisモード） → すべてのシリーズを表示
2. ✅ **`typeof 0 === 'number'` が true** → 0は有効な数値として扱われる
3. ✅ **データに0が多数存在** → 実際の0、またはN/Aから変換された0

**重要な発見**:
- **Stack series OFF にしても、実際の0は消えない**
- **Rich Tooltip ON の限り、0は必ず表示される**

### 推奨される解決策（優先順）

| 優先度 | 解決策 | 工数 | 即効性 | 持続性 |
|--------|--------|------|--------|--------|
| **🥇 1位** | **Rich Tooltip OFF** | **1分** | ★★★★★ | ★★★☆☆ |
| 🥈 2位 | SQLで0を除外 | 1時間 | ★★★★☆ | ★★★★☆ |
| 🥉 3位 | tooltipHideZeroValues実装 | 2-3日 | ★★☆☆☆ | ★★★★★ |

### 診断と対応フロー

```
1. DATAタブで確認
   - 「0」→ 実際の0
   - 「N/A」→ N/Aから変換された0

2. Rich Tooltip の必要性を判断
   - 必要ない → Rich Tooltip OFF（1分）
   - 必要 → 次のステップへ

3. 対応方針を選択
   - すぐ対応 → SQLで0除外
   - 中期的 → tooltipHideZeroValues実装
```

### 技術的ポイント

1. **Rich Tooltip と 0表示**:
   ```typescript
   typeof 0 === 'number'  // true
   → Rich Tooltip ON では0も表示される
   ```

2. **Stack series の役割**:
   ```typescript
   N/A → 0 変換のみ
   実際の0には影響しない
   ```

3. **解決の方向性**:
   - ツールチップレベルでの制御（tooltipHideZeroValues）
   - データレベルでの除外（SQL）
   - モード切り替え（Rich Tooltip OFF）

### 次のステップ

1. **現状確認**:
   - DATAタブで0とN/Aの分布を確認
   - Rich Tooltipの必要性を判断

2. **解決策選択**:
   - シリーズ数、0の多さ、要件から判断

3. **実施**:
   - 選択した解決策を適用
   - テスト・検証

4. **必要に応じて**:
   - tooltipHideZeroValues機能の実装検討
   - 詳細は `TOOLTIP_COMPREHENSIVE_GUIDE.md` を参照

---

**作成者**: Claude Code
**作成日**: 2026-04-02
**最終更新**: 2026-04-02（重要な修正: Stack series OFF では実際の0は消えない）
**関連ドキュメント**:
- `TOOLTIP_COMPREHENSIVE_GUIDE.md` - ツールチップ機能の包括的ガイド
- `MIXED_CHART_CUSTOMIZE_PROPERTIES.md` - Customizeプロパティ完全ガイド
