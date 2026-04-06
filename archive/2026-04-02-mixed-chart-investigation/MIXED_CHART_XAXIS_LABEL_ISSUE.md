# Mixed Chart - X軸ラベル表示問題の解決ガイド

**対象**: Apache Superset Mixed Timeseries Chart
**問題**: 最終日のX軸ラベルが表示されない
**作成日**: 2026-04-02

---

## 目次

1. [問題の概要](#問題の概要)
2. [発生原因](#発生原因)
3. [解決策](#解決策)
4. [技術的詳細](#技術的詳細)
5. [よくある質問](#よくある質問)

---

## 問題の概要

### 症状

7月1日〜5日の5日分のデータがあるにもかかわらず、X軸のラベルが以下のように表示される：

```
表示されている: 7/1   7/2   7/3   7/4
表示されない:                        7/5 ←
```

**期待される表示**:
```
7/1   7/2   7/3   7/4   7/5
```

### 用語の確認

- **X軸ラベル**: チャートの横軸（時間軸）に表示される日付や時刻のテキスト
- **凡例（Legend）**: チャート上部または側面に表示される、各データ系列の名前と色を示すもの

※本問題は「X軸ラベル」の表示問題です（「凡例」ではありません）

---

## 発生原因

X軸の最終日のラベルが表示されない原因は、主に以下の3つです：

### 1. **EChartsの自動ラベル間引き機能**

EChartsは、ラベルが重ならないように、チャートの幅に応じて自動的にラベルを間引きます。

**メカニズム**:
- チャート幅が狭い場合、すべてのラベルを表示するとラベル同士が重なる
- EChartsは自動的に一部のラベルをスキップして表示
- 特に最初と最後のラベルは、表示スペースの都合で省略されやすい

**確認方法**:
- ブラウザの開発者ツールでチャートの幅を確認
- チャート幅が広い場合でも発生する場合は、他の原因を疑う

---

### 2. **X軸の範囲設定（X Axis Bounds）**

`X Axis Bounds`設定で最大値が7/4に設定されている場合、7/5は範囲外となり表示されません。

**確認方法**:
1. チャート編集画面を開く
2. 「Chart Options」セクションを展開
3. 「X Axis」サブセクションを探す
4. **「X Axis Bounds」**の設定を確認

**設定例**:
```
X Axis Bounds: [min: null, max: 2024-07-04]
                                    ↑
                                  7/4までに制限されている
```

---

### 3. **Truncate X Axis設定の影響**

`Truncate X Axis`がONの場合、X軸の範囲が自動的にデータの最小値・最大値に切り詰められますが、棒グラフの場合は挙動が異なります。

**棒グラフの場合の特殊な挙動**:
- 棒グラフは各データポイントに幅を持つため、EChartsは自動的に「境界の余白（boundaryGap）」を追加
- この余白により、最後のデータポイント（7/5）のラベルが範囲外に押し出される可能性がある

**確認方法**:
1. チャート編集画面を開く
2. 「Chart Options」→「X Axis」セクション
3. **「Truncate X Axis」**の設定を確認
4. Query AまたはQuery Bで棒グラフ（Bar）を使用しているか確認

---

## 解決策

問題の原因に応じて、以下の解決策を試してください。

### 解決策1: チャート幅を広げる

**最も簡単な解決策**です。チャートの幅を広げることで、すべてのラベルを表示するスペースを確保します。

#### 方法1: ダッシュボード上でチャートを広げる

1. ダッシュボード編集モードに入る
2. チャートの右下の角をドラッグして幅を広げる
3. 保存

#### 方法2: チャート設定でカラム数を調整

1. ダッシュボード編集モード→チャートの設定アイコンをクリック
2. 「Width」または「Column span」を増やす
3. 保存

**効果**:
- ✅ 簡単に実施可能
- ✅ 他の設定を変更する必要がない
- ❌ ダッシュボードのレイアウトが変わる

---

### 解決策2: X軸ラベルを回転させる

ラベルを斜めに表示することで、より多くのラベルを表示できます。

#### 設定方法

1. チャート編集画面を開く
2. 「Chart Options」セクションを展開
3. 「X Axis」サブセクションを探す
4. **「X Axis Label Rotation」**を設定

**推奨値**:
```
X Axis Label Rotation: 45  （45度回転）
または
X Axis Label Rotation: 90  （90度回転・縦書き）
```

**視覚的な違い**:

```
回転なし（0度）:
7/1  7/2  7/3  7/4
（ラベルが重なりやすい）

45度回転:
    7/1
       7/2
          7/3
             7/4
                7/5

90度回転（縦書き）:
7  7  7  7  7
/  /  /  /  /
1  2  3  4  5
```

**効果**:
- ✅ チャートの幅を変えずに対応可能
- ✅ より多くのラベルを表示可能
- ❌ ラベルが斜めになり、やや読みづらい

**設定場所**: `controlPanel.tsx:320-321`
```typescript
['x_axis_time_format'],
[xAxisLabelRotation],  // ← ここで設定
```

---

### 解決策3: X軸のフォーマットを短くする

日付フォーマットを短くすることで、ラベルの幅を狭め、より多くのラベルを表示できます。

#### 設定方法

1. チャート編集画面を開く
2. 「Chart Options」セクションを展開
3. 「X Axis」サブセクションを探す
4. **「X Axis Time Format」**を設定

**フォーマット例**:

| フォーマット | 表示例 | 幅 |
|------------|--------|-----|
| `%Y-%m-%d %H:%M:%S` | `2024-07-01 12:00:00` | 広い |
| `%Y-%m-%d` | `2024-07-01` | 中 |
| `%m/%d` | `07/01` | 狭い |
| `%m/%d` | `7/1` | 最も狭い |

**推奨設定**:
```
X Axis Time Format: %m/%d
→ 表示: 7/1, 7/2, 7/3, 7/4, 7/5
```

**効果**:
- ✅ ラベルの幅が狭くなり、より多くのラベルを表示可能
- ✅ 見た目がすっきりする
- ❌ 年や時刻の情報が省略される

**設定場所**: `controlPanel.tsx:320`
```typescript
['x_axis_time_format'],  // ← ここで設定
```

---

### 解決策4: X Axis Boundsの設定を確認・修正

X Axis Boundsで最大値が制限されている場合、それを解除または修正します。

#### 設定方法

1. チャート編集画面を開く
2. 「Chart Options」セクションを展開
3. 「X Axis」サブセクションを探す
4. **「X Axis Bounds」**を確認

**修正方法**:

**パターンA: 最大値が7/4に制限されている場合**
```
修正前: [min: null, max: 2024-07-04]
修正後: [min: null, max: null]
         ↑ 自動的にデータの範囲全体を使用
```

**パターンB: 最大値を7/5に設定する場合**
```
修正前: [min: null, max: 2024-07-04]
修正後: [min: null, max: 2024-07-05]
```

**効果**:
- ✅ 7/5が確実に範囲内に含まれる
- ✅ 手動で範囲をコントロール可能
- ❌ データが増えた場合、再度調整が必要

**設定場所**: `controlPanel.tsx:338`
```typescript
[xAxisBounds],  // ← ここで設定
```

---

### 解決策5: Truncate X Axisを無効化（棒グラフの場合）

棒グラフを使用している場合、Truncate X Axisを無効化することで、最後のラベルが表示される可能性があります。

#### 設定方法

1. チャート編集画面を開く
2. 「Chart Options」セクションを展開
3. 「X Axis」サブセクションを探す
4. **「Truncate X Axis」**のチェックを外す

**設定値**:
```
Truncate X Axis: OFF (チェックを外す)
```

**効果**:
- ✅ X軸の範囲に余裕ができ、最後のラベルが表示される可能性が高まる
- ❌ X軸の範囲が広くなり、データが狭い範囲に表示される可能性がある

**設定場所**: `controlPanel.tsx:337`
```typescript
[truncateXAxis],  // ← ここで設定
```

**技術的な詳細**:

`transformProps.ts:525-534`で、`getMinAndMaxFromBounds`関数が呼ばれています：

```typescript
...getMinAndMaxFromBounds(
  xAxisType,
  truncateXAxis,  // ← この値
  xAxisMin,
  xAxisMax,
  seriesType === EchartsTimeseriesSeriesType.Bar ||
    seriesTypeB === EchartsTimeseriesSeriesType.Bar
    ? EchartsTimeseriesSeriesType.Bar
    : undefined,
),
```

この関数は、棒グラフの場合に`scale: true`を設定しますが、X軸がTime型の場合は何も返しません（空オブジェクト`{}`）。

**`utils/series.ts:607-632`**:
```typescript
export function getMinAndMaxFromBounds(
  axisType: AxisType,
  truncateAxis: boolean,
  min?: number,
  max?: number,
  seriesType?: EchartsTimeseriesSeriesType,
): BoundsType | {} {
  if (axisType === AxisType.Value && truncateAxis) {
    // Y軸（Value型）の場合のみ処理
    const ret: BoundsType = {};
    if (seriesType === EchartsTimeseriesSeriesType.Bar) {
      ret.scale = true;
    }
    // ...
    return ret;
  }
  return {};  // ← X軸（Time型）の場合は空オブジェクト
}
```

したがって、**Truncate X AxisはX軸（Time型）には直接影響しません**。しかし、棒グラフの場合、ECharts自体が境界の余白（boundaryGap）を自動的に追加するため、間接的に影響する可能性があります。

---

### 解決策6: データの確認

そもそも7/5のデータが正しく取得されているかを確認します。

#### 確認方法

1. チャート編集画面を開く
2. 「DATA」タブに切り替え
3. データテーブルを確認し、7/5のデータが存在するか確認

**確認ポイント**:
- ✅ 7/5のデータが存在する → 他の解決策を試す
- ❌ 7/5のデータが存在しない → クエリやフィルタを確認

**クエリの確認**:
```sql
-- 日付範囲を確認
SELECT
  MIN(timestamp_column) as min_date,
  MAX(timestamp_column) as max_date,
  COUNT(*) as row_count
FROM your_table
WHERE your_filters
```

**期待される結果**:
```
min_date: 2024-07-01
max_date: 2024-07-05  ← 7/5が含まれているか確認
row_count: 5
```

---

## 推奨される解決策の組み合わせ

複数の解決策を組み合わせることで、より確実に問題を解決できます。

### パターンA: 見た目を重視する場合

```
1. X Axis Time Format: %m/%d （短いフォーマット）
2. X Axis Label Rotation: 0 （回転なし）
3. チャート幅を広げる
```

**結果**: すっきりとした見た目で、すべてのラベルが水平に表示される

---

### パターンB: チャート幅を変えたくない場合

```
1. X Axis Time Format: %m/%d （短いフォーマット）
2. X Axis Label Rotation: 45 （45度回転）
3. Truncate X Axis: OFF
```

**結果**: チャート幅を変えずに、すべてのラベルを表示

---

### パターンC: 確実にすべてのラベルを表示する場合

```
1. X Axis Bounds: [null, null] （範囲制限なし）
2. X Axis Time Format: %m/%d （短いフォーマット）
3. X Axis Label Rotation: 45 （45度回転）
4. チャート幅を広げる
```

**結果**: 最も確実にすべてのラベルが表示される

---

## 技術的詳細

### X軸ラベル表示の仕組み

Mixed ChartのX軸ラベル表示は、以下のフローで決定されます：

```
1. データ取得
   ↓
2. X軸の範囲決定
   - xAxisBoundsの設定を適用
   - truncateXAxisの設定を考慮
   - データの最小値・最大値を考慮
   ↓
3. X軸の型決定
   - Time型（日時データ）
   - Category型（カテゴリデータ）
   - Value型（数値データ）
   ↓
4. ラベルフォーマット適用
   - xAxisTimeFormatの設定を適用
   - 日時 → フォーマット済み文字列
   ↓
5. EChartsによる自動レイアウト
   - チャート幅を考慮
   - ラベルの幅を計算
   - ラベルが重なる場合は自動的に間引き
   - xAxisLabelRotationを適用
   ↓
6. 描画
```

### 関連する設定とコード

#### 1. X軸の設定（transformProps.ts:509-535）

```typescript
xAxis: {
  type: xAxisType,  // 'time' (Time型の場合)
  name: xAxisTitle,
  nameGap: convertInteger(xAxisTitleMargin),
  nameLocation: 'middle',
  axisLabel: {
    formatter: xAxisFormatter,  // 日付フォーマット関数
    rotate: xAxisLabelRotation,  // ラベルの回転角度
  },
  minorTick: { show: minorTicks },
  minInterval:
    xAxisType === AxisType.Time && timeGrainSqla
      ? TIMEGRAIN_TO_TIMESTAMP[
          timeGrainSqla as keyof typeof TIMEGRAIN_TO_TIMESTAMP
        ]
      : 0,
  ...getMinAndMaxFromBounds(
    xAxisType,
    truncateXAxis,
    xAxisMin,
    xAxisMax,
    seriesType === EchartsTimeseriesSeriesType.Bar ||
      seriesTypeB === EchartsTimeseriesSeriesType.Bar
      ? EchartsTimeseriesSeriesType.Bar
      : undefined,
  ),
},
```

#### 2. X軸フォーマッター（transformProps.ts:480-483）

```typescript
const xAxisFormatter =
  xAxisDataType === GenericDataType.Temporal
    ? getXAxisFormatter(xAxisTimeFormat)  // 日時フォーマット
    : String;  // カテゴリ・数値の場合は文字列化
```

#### 3. コントロールパネルの設定（controlPanel.tsx:319-338）

```typescript
[<ControlSubSectionHeader>{t('X Axis')}</ControlSubSectionHeader>],
['x_axis_time_format'],  // X軸の日時フォーマット
[xAxisLabelRotation],    // ラベルの回転角度
...richTooltipSection,
// eslint-disable-next-line react/jsx-key
[<ControlSubSectionHeader>{t('Y Axis')}</ControlSubSectionHeader>],
// ... Y軸の設定
[truncateXAxis],  // X軸の切り詰め
[xAxisBounds],    // X軸の範囲
```

### EChartsのデフォルト動作

EChartsは、Time型のX軸に対して以下のデフォルト動作を行います：

1. **自動ラベル間引き**:
   - チャート幅とラベル幅を計算
   - ラベルが重なる場合、自動的に一部をスキップ
   - 最初と最後のラベルは優先的に表示されるが、スペースが足りない場合は省略される

2. **boundaryGap（境界の余白）**:
   - 棒グラフの場合: `boundaryGap: true`（デフォルト）
     - X軸の両端に余白が追加される
     - 最初と最後のデータポイントは軸の端から少し内側に配置される
   - 折れ線グラフの場合: `boundaryGap: false`（デフォルト）
     - X軸の両端に余白なし
     - 最初と最後のデータポイントは軸の端に配置される

3. **minInterval（最小間隔）**:
   - 時間粒度（timeGrainSqla）が設定されている場合、その粒度に応じた最小間隔が設定される
   - 例: 日次データの場合、1日（86400000ミリ秒）が最小間隔

---

## よくある質問

### Q1: チャート幅を広げても7/5が表示されません。

**A**: 以下を確認してください：

1. **X Axis Boundsの設定を確認**
   - 最大値が7/4に制限されていないか確認
   - 制限されている場合は、nullに設定または7/5以降に設定

2. **データの存在確認**
   - DATAタブでデータテーブルを確認
   - 7/5のデータが実際に存在するか確認

3. **ブラウザのキャッシュをクリア**
   - Ctrl+Shift+R（Windows）またはCmd+Shift+R（Mac）でハードリフレッシュ

---

### Q2: X Axis Label Rotationを設定したのに反映されません。

**A**: 以下を確認してください：

1. **「Update chart」ボタンをクリック**
   - 設定を変更した後、必ず「Update chart」をクリック

2. **チャートを保存**
   - 変更を永続化するには、チャートを保存する必要があります

3. **設定値の確認**
   - 正しい数値（0〜360）が入力されているか確認
   - 推奨値: 45または90

---

### Q3: すべての解決策を試しても7/5が表示されません。

**A**: 以下の可能性があります：

1. **7/5のデータが実際に存在しない**
   - クエリを確認し、7/5のデータが取得されているか確認
   - フィルタ条件を確認

2. **X軸が日時型（Temporal）として認識されていない**
   - DATAタブでX軸カラムの型を確認
   - カテゴリ型として認識されている場合、日時型に変更

3. **ブラウザの互換性問題**
   - 別のブラウザで試してみる
   - ブラウザの開発者ツールでエラーログを確認

---

### Q4: 7/1も表示されません。両端が表示されないのですが。

**A**: これはチャート幅が非常に狭い場合に発生します。

**解決策**:
1. **チャート幅を大幅に広げる**
2. **X Axis Label Rotationを90度に設定**（縦書き）
3. **X Axis Time Formatを最短に設定**（例: `%d`で「1」のみ表示）

---

### Q5: X軸のラベルがすべて表示されるようにEChartsの設定を変更できますか？

**A**: 現在のSupersetの実装では、EChartsの`axisLabel.interval`設定は公開されていません。

**代替案**:
- 上記の解決策を組み合わせて対応
- コードを直接修正する場合は、`transformProps.ts`の`xAxis.axisLabel`に`interval: 0`を追加

**コード修正例**（上級者向け）:

```typescript
// transformProps.ts:514-517
axisLabel: {
  formatter: xAxisFormatter,
  rotate: xAxisLabelRotation,
  interval: 0,  // ← すべてのラベルを表示（重なる可能性あり）
},
```

⚠️ **注意**: この修正は推奨されません。ラベルが重なる可能性があります。

---

### Q6: 日付フォーマットの一覧はどこで確認できますか？

**A**: X Axis Time Formatは、D3.jsの時刻フォーマット記法を使用します。

**主なフォーマット記法**:

| 記法 | 説明 | 例 |
|-----|------|-----|
| `%Y` | 4桁の年 | `2024` |
| `%y` | 2桁の年 | `24` |
| `%m` | 月（01-12） | `07` |
| `%-m` | 月（1-12） | `7` |
| `%B` | 月名（完全） | `July` |
| `%b` | 月名（略） | `Jul` |
| `%d` | 日（01-31） | `05` |
| `%-d` | 日（1-31） | `5` |
| `%H` | 時（00-23） | `14` |
| `%M` | 分（00-59） | `30` |
| `%S` | 秒（00-59） | `45` |

**組み合わせ例**:
- `%Y-%m-%d`: `2024-07-05`
- `%m/%d`: `07/05`
- `%-m/%-d`: `7/5`
- `%b %d`: `Jul 05`
- `%Y-%m-%d %H:%M`: `2024-07-05 14:30`

**詳細**: [D3.js Time Format Documentation](https://github.com/d3/d3-time-format)

---

### Q7: X Axis Boundsはどのような形式で入力すればいいですか？

**A**: X Axis Boundsは、`[最小値, 最大値]`の形式で入力します。

**入力例**:

1. **制限なし（デフォルト）**:
   ```
   [null, null]
   ```

2. **最小値のみ指定**:
   ```
   [2024-07-01, null]
   ```

3. **最大値のみ指定**:
   ```
   [null, 2024-07-05]
   ```

4. **両方指定**:
   ```
   [2024-07-01, 2024-07-05]
   ```

**注意**:
- 日付形式は`YYYY-MM-DD`
- 時刻を含む場合は`YYYY-MM-DD HH:MM:SS`
- タイムスタンプ（ミリ秒）も使用可能: `[1719792000000, 1720137600000]`

---

## まとめ

### 問題の原因

X軸の最終日（7/5）のラベルが表示されない原因は：

1. ✅ **EChartsの自動ラベル間引き**（最も一般的）
2. ✅ **X Axis Boundsの設定**（手動で範囲制限している場合）
3. ✅ **棒グラフのboundaryGap**（棒グラフを使用している場合）
4. ✅ **データの不足**（7/5のデータが実際に存在しない場合）

### 推奨される解決策

**最も簡単で効果的な解決策**:

```
1. X Axis Time Format: %-m/%-d （例: 7/5）
2. X Axis Label Rotation: 45
3. チャート幅を広げる
```

この組み合わせで、ほとんどの場合、すべてのラベルが表示されます。

### チェックリスト

問題解決のために、以下を順番に確認してください：

- [ ] チャート幅は十分広いか？
- [ ] X Axis Time Formatは短いフォーマットか？
- [ ] X Axis Label Rotationは設定されているか？
- [ ] X Axis Boundsは制限されていないか？
- [ ] 7/5のデータは実際に存在するか？
- [ ] 「Update chart」ボタンをクリックしたか？
- [ ] チャートを保存したか？

---

**作成者**: Claude Code
**作成日**: 2026-04-02
**最終更新**: 2026-04-02
