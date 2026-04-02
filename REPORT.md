# Mixed Timeseries チャート日時変換不具合 調査レポート

**対象**: Apache Superset (superset-kenshou)
**調査日**: 2026-04-01
**優先度**: 🔴 高（お客様への説明が必要）

---

## 1. 問い

**なぜ Mixed Chart において、一つのフラグ（Query A）のデータがない状態で表示すると、タイムスタンプが日時フォーマットされずに数値として表示されるのか？**

---

## 2. 現状の動作メカニズム（詳細解析）

### エントリポイント

- **ファイル**: `superset/superset-frontend/plugins/plugin-chart-echarts/src/MixedTimeseries/transformProps.ts`
- **関数**: `transformProps()` (行117-721)

### 処理フロー

#### Step 1: データ型マッピングの取得（235-236行目）

```typescript
const dataTypes = getColtypesMapping(queriesData[0]);
const xAxisDataType = dataTypes?.[xAxisLabel] ?? dataTypes?.[xAxisOrig];
```

**問題点**:
- `getColtypesMapping()`は**Query A（`queriesData[0]`）のみ**から型情報を取得
- Mixed Chartは2つのクエリ（Query A / Query B）を持つが、**Query Bの型情報は使用されていない**

#### Step 2: `getColtypesMapping`の実装

**ファイル**: `superset/superset-frontend/plugins/plugin-chart-echarts/src/utils/series.ts:391-401`

```typescript
export const getColtypesMapping = ({
  coltypes = [],
  colnames = [],
}: Pick<ChartDataResponseResult, 'coltypes' | 'colnames'>): Record<
  string,
  GenericDataType
> =>
  colnames.reduce(
    (accumulator, item, index) => ({ ...accumulator, [item]: coltypes[index] }),
    {},
  );
```

**動作**:
- `colnames`（列名配列）と`coltypes`（型配列）からマッピングを作成
- **Query Aにデータがない場合**: `colnames`と`coltypes`が空配列 → 戻り値は`{}`（空オブジェクト）

#### Step 3: 日時フォーマッター判定（476-483行目）

```typescript
const tooltipFormatter =
  xAxisDataType === GenericDataType.Temporal
    ? getTooltipTimeFormatter(tooltipTimeFormat)
    : String;
const xAxisFormatter =
  xAxisDataType === GenericDataType.Temporal
    ? getXAxisFormatter(xAxisTimeFormat)
    : String;
```

**問題発生メカニズム**:

1. Query Aが空 → `getColtypesMapping(queriesData[0])` が `{}` を返す
2. `xAxisDataType = undefined`
3. `undefined === GenericDataType.Temporal` → `false`
4. フォーマッターに`String`が設定される
5. タイムスタンプ値（例: `1609459200000`）が`String(1609459200000)`として表示される

#### 既に存在する正しいコード（使われていない箇所）

**transformProps.ts:147-150**

```typescript
const coltypeMapping = {
  ...getColtypesMapping(queriesData[0]),
  ...getColtypesMapping(queriesData[1]),
};
```

**重要**: このコードは**両方のクエリ**からデータ型をマージしているが、`xAxisDataType`の判定には使用されていない。

### データフロー図

```
queriesData[0] (Query A) ┐
                         ├─> getColtypesMapping() ─> dataTypes ─> xAxisDataType
queriesData[1] (Query B) ┘                              ↓
                                                   (Query Aが空なら undefined)
                                                          ↓
                                              xAxisFormatter = String
                                                          ↓
                                              タイムスタンプが数値表示
```

**本来あるべき姿**:

```
queriesData[0] (Query A) ┐
                         ├─> coltypeMapping (マージ済) ─> xAxisDataType
queriesData[1] (Query B) ┘        ↓
                              (どちらかに型情報があればOK)
                                   ↓
                          xAxisFormatter = getXAxisFormatter()
                                   ↓
                          日時フォーマット適用
```

---

## 3. 既存機能による実現可能性

- [ ] 設定変更のみで実現可能
- [ ] カスタム設定ファイルで実現可能
- [x] **既存機能では実現不可能（コードの不具合）**

**理由**:
- この問題はロジックレベルの不具合であり、設定フラグでは解決できない
- 運用回避策（Query Aにダミーデータを追加）は現実的でない

---

## 4. 改修が必要な場合（OSS 非改変の原則を守る方法）

### 改修が必要な理由

**コードレベルの根拠**:
- `superset/superset-frontend/plugins/plugin-chart-echarts/src/MixedTimeseries/transformProps.ts:235-236`
- `xAxisDataType`の判定に使用するデータ型マッピングが、`queriesData[0]`のみから取得されている
- 既に両クエリをマージした`coltypeMapping`が147-150行目で生成されているが、使用されていない

### 修正内容（最小限のOSS改修）

**対象ファイル**: `transformProps.ts:235-236`

#### 修正前

```typescript
const dataTypes = getColtypesMapping(queriesData[0]);
const xAxisDataType = dataTypes?.[xAxisLabel] ?? dataTypes?.[xAxisOrig];
```

#### 修正後

```typescript
// const dataTypes = getColtypesMapping(queriesData[0]); // 削除
const xAxisDataType = coltypeMapping?.[xAxisLabel] ?? coltypeMapping?.[xAxisOrig];
```

**修正理由**:
1. `coltypeMapping`は既に147-150行目で両クエリをマージ済み
2. 既存変数の再利用のため、ロジック変更リスクが低い
3. Query A/Bどちらかにデータがあれば型情報を取得できる

### 副作用の検証

**影響を受けるコード**:
- `tooltipFormatter` (476-479行目) - X軸値のツールチップ表示
- `xAxisFormatter` (480-483行目) - X軸ラベルの表示

**影響範囲の確認**:
- `coltypeMapping`は同じ`getColtypesMapping()`で生成されており、マッピング構造は同一
- Query A/Bのどちらかに型情報があれば正しく動作する
- 両方が空の場合は従来通り`undefined`となり、既存の動作を維持
- **破壊的変更なし**

### 代替案の検討

#### 案A: 設定による回避（不採用）
- Query Aに必ずダミーデータを入れる運用ルール
- ❌ 運用負荷が高く、データの意味が不明確になる

#### 案B: フォーマット関数の拡張（不採用）
- `xAxisFormatter`でタイムスタンプ自動検出を実装
- ❌ 型情報を無視することになり、他のケースで誤動作のリスク

#### 案C: 本修正（採用）
- 既存の`coltypeMapping`を使用
- ✅ 最小限の変更、既存ロジックの活用、リスク低

---

## 5. お客様向け説明サマリー

### 問題の要約

Mixed Chart（混合グラフ）において、**Query Aにデータがなく、Query Bのみにデータが存在する場合**、X軸の日時が正しくフォーマットされず、タイムスタンプの数値（例: `1609459200000`）がそのまま表示される不具合が発生しています。

### 発生条件

1. Mixed Chartを使用
2. Query A（最初のクエリ）が空データを返す
3. Query B（2番目のクエリ）に有効なデータが存在
4. X軸が日時カラム（Temporal型）

### 原因

Mixed Chartは内部で2つのクエリ（Query A / Query B）を実行しますが、X軸のデータ型を判定する際に**Query Aの型情報のみ**を参照しているため、Query Aが空の場合に型情報を取得できず、日時フォーマットが適用されません。

コード内には既に両クエリの型情報をマージした変数が存在していますが、X軸の型判定でその変数が使用されていないことが根本原因です。

### 対処方法

**Supersetのソースコード1箇所（2行）を修正することで解決可能です。**

- 修正ファイル: `superset-frontend/plugins/plugin-chart-echarts/src/MixedTimeseries/transformProps.ts`
- 修正内容: 既に存在する両クエリマージ済み変数を使用するように変更
- リスク: 低（既存変数の再利用のため、破壊的変更なし）

### OSS改修の妥当性

- ✅ この問題は設定変更では解決できないコードの不具合
- ✅ 修正は既存変数の再利用のため、リスクが低い最小限の変更
- ✅ Supersetコミュニティへのバグ報告・パッチ提案も検討に値する
  - 他のユーザーも同じ問題に遭遇している可能性が高い
  - 本家に取り込まれれば、将来のアップグレードで修正の維持が不要になる

---

## 6. 次のアクション提案

### 短期（即時対応）
1. ✅ **修正の実施**: 上記の2行修正を適用
2. ✅ **動作検証**: 以下のパターンでテスト
   - Query A空・Query B有
   - Query A有・Query B空
   - 両方有
   - 両方空

### 中期（コミュニティ連携）
3. 🔲 **Apache Supersetコミュニティへの報告**:
   - GitHub Issueとして報告
   - または Pull Request を提出
   - タイトル案: "Fix x-axis date formatting in Mixed Chart when first query returns empty data"
   - 他のユーザーへの貢献 + 本家取り込みによる保守性向上

### 長期（ドキュメント化）
4. 🔲 **CLAUDE.md の既知の調査済み領域テーブルに追加**:
   ```markdown
   | Mixed Chart X軸 日時変換（Query A空） | queriesData[0]のみ参照。coltypeMapping使用で解決 | transformProps.ts:235 |
   ```

---

## 7. 技術的補足

### 関連する既知の問題

**CLAUDE.md記載の既知問題**:
- Mixed Chart ゼロ値 tooltip: Issue #4462 未解決（ECharts仕様）
- Mixed Chart バー幅（年単位データ）: Canvas制約

**本件との関連**:
- 今回の問題はSuperset固有のバグであり、ECharts側の制約ではない
- 修正によってSuperset側で完全に解決可能

### テストケース

```typescript
// テストシナリオ
describe('Mixed Chart X-axis date formatting', () => {
  it('should format dates when Query A is empty but Query B has data', () => {
    const queriesData = [
      { data: [], colnames: [], coltypes: [] }, // Query A empty
      {
        data: [{ __timestamp: 1609459200000, value: 100 }],
        colnames: ['__timestamp', 'value'],
        coltypes: [GenericDataType.Temporal, GenericDataType.Numeric]
      }
    ];
    // Expected: xAxisFormatter uses date formatter, not String
  });
});
```

---

**調査者**: Claude Code
**レビュー推奨**: フロントエンド開発チーム
