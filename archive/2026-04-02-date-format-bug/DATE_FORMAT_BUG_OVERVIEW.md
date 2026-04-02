# Mixed Chart 日付フォーマット不具合 - 概要

**影響範囲**: Mixed Timeseries Chart
**作成日**: 2026-04-02

---

## 目次

1. [問題の概要](#1-問題の概要)
2. [発生条件](#2-発生条件)
3. [症状](#3-症状)
4. [根本原因](#4-根本原因)
5. [解決策](#5-解決策)
6. [関連ドキュメント](#6-関連ドキュメント)

---

## 1. 問題の概要

Mixed Timeseries Chartにおいて、**Query A（最初のクエリ）が空データを返す場合**、X軸の日時が正しくフォーマットされず、タイムスタンプの数値（例: `1609459200000`）がそのまま表示される不具合が発生しています。

### 視覚的な説明

**正常時（Query Aにデータがある）**:

```
X軸: 2021-01-01  2021-01-02  2021-01-03
```

**不具合発生時（Query Aが空）**:

```
X軸: 1609459200000  1609545600000  1609632000000
     ↑ タイムスタンプが数値のまま表示される
```

---

## 2. 発生条件

以下のすべての条件を満たす場合に発生します：

1. ✅ **Mixed Timeseries Chart を使用**
2. ✅ **Query A（最初のクエリ）が空データを返す**
3. ✅ **Query B（2番目のクエリ）に有効なデータが存在**
4. ✅ **X軸が日時カラム（Temporal型）**

### 具体例

```sql
-- Query A（空になるクエリ）
SELECT
  timestamp_column,
  metric_a
FROM table_a
WHERE condition = 'no_match'  -- データが返らない

-- Query B（データを返すクエリ）
SELECT
  timestamp_column,
  metric_b
FROM table_b
WHERE condition = 'has_data'  -- データが返る
```

**結果**:

- Query A: 0行（空）
- Query B: 100行（データあり）
- **症状**: X軸の日時が数値表示される

---

## 3. 症状

### 3.1 ユーザーから見た症状

**チャート画面**:

```
┌─────────────────────────────────────────┐
│ Mixed Timeseries Chart                  │
├─────────────────────────────────────────┤
│                           ■             │
│                     ■                   │
│               ■                         │
│         ■                               │
└─────────────────────────────────────────┘
  1609459200000  1609545600000  1609632000000
  ↑ 本来は "2021-01-01" などと表示されるべき
```

**ツールチップ**:

```
┌──────────────────────┐
│ 1609459200000        │  ← 日付フォーマットされていない
├──────────────────────┤
│ ● Metric B    100    │
└──────────────────────┘
```

### 3.2 システムログ（該当なし）

- エラーログは出力されません
- JavaScriptコンソールにも警告は表示されません
- **見た目だけの問題として顕在化**するため、気づきにくい

---

## 4. 根本原因

### 4.1 単純な説明

Mixed Chartは内部で2つのクエリ（Query A / Query B）を実行しますが、**X軸のデータ型を判定する際にQuery Aの型情報のみを参照**しているため、Query Aが空の場合に型情報を取得できず、日時フォーマットが適用されません。

### 4.2 コードレベルの原因

**ファイル**: `superset-frontend/plugins/plugin-chart-echarts/src/MixedTimeseries/transformProps.ts`

#### 問題の箇所（235-236行目）

```typescript
const dataTypes = getColtypesMapping(queriesData[0]); // ← Query Aのみ参照
const xAxisDataType = dataTypes?.[xAxisLabel] ?? dataTypes?.[xAxisOrig];
```

**動作**:

1. `queriesData[0]`（Query A）から型情報を取得
2. Query Aが空の場合 → `dataTypes = {}`（空オブジェクト）
3. `xAxisDataType = undefined`
4. 日付フォーマッターが適用されない

#### 既に存在する正しいコード（使われていない）

**147-150行目**:

```typescript
const coltypeMapping = {
  ...getColtypesMapping(queriesData[0]), // Query A
  ...getColtypesMapping(queriesData[1]), // Query B
};
```

**重要**: このコードは**Query AとQuery Bの両方から型情報をマージ**していますが、`xAxisDataType`の判定には使用されていません。

### 4.3 なぜこの問題が発生するのか

**設計の想定**:

- Mixed Chartの開発時、通常は両方のクエリにデータがあることを想定
- Query Aのみから型情報を取得する実装になった

**実際の利用シーン**:

- フィルター条件によってQuery Aが空になるケースがある
- 時系列で特定期間のみデータがあるケース
- A/Bテストで片方のグループにデータがないケース

**結果**:

- 想定外のケースに対応できていなかった

---

## 5. 解決策

### 5.1 修正内容（最小限）

**対象ファイル**: `transformProps.ts:235-236`

#### 修正前

```typescript
const dataTypes = getColtypesMapping(queriesData[0]);
const xAxisDataType = dataTypes?.[xAxisLabel] ?? dataTypes?.[xAxisOrig];
```

#### 修正後

```typescript
// const dataTypes = getColtypesMapping(queriesData[0]); // 削除
const xAxisDataType =
  coltypeMapping?.[xAxisLabel] ?? coltypeMapping?.[xAxisOrig];
```

**変更点**:

- `dataTypes`変数を削除
- 既存の`coltypeMapping`（両クエリマージ済み）を使用

### 5.2 修正の効果

**修正前の動作**:

```
Query A: 空（型情報なし）
   ↓
dataTypes = {}
   ↓
xAxisDataType = undefined
   ↓
フォーマッター = String
   ↓
タイムスタンプが数値表示
```

**修正後の動作**:

```
Query A: 空（型情報なし）
Query B: データあり（型情報あり）
   ↓
coltypeMapping = { __timestamp: GenericDataType.Temporal, ... }
   ↓
xAxisDataType = GenericDataType.Temporal
   ↓
フォーマッター = getXAxisFormatter()
   ↓
日時フォーマット適用 ✅
```

### 5.3 リスク評価

| 項目                 | 評価        | 理由                     |
| -------------------- | ----------- | ------------------------ |
| **破壊的変更**       | ❌ なし     | 既存変数の再利用のため   |
| **パフォーマンス**   | ✅ 影響なし | 計算量は同じ             |
| **テストカバレッジ** | ⚠️ 追加推奨 | 新しいテストケースが必要 |
| **後方互換性**       | ✅ 保持     | 既存の動作を維持         |

**総合評価**: 🟢 **低リスク**

---

## 6. 関連ドキュメント

### 詳細ドキュメント

1. **[DATE_FORMAT_BUG_TECHNICAL.md](./DATE_FORMAT_BUG_TECHNICAL.md)**
   - 日付フォーマット決定のフロー全体
   - コードレベルの詳細解説
   - データフロー図

2. **[DATE_FORMAT_BUG_FIX_GUIDE.md](./DATE_FORMAT_BUG_FIX_GUIDE.md)**
   - 具体的な修正手順
   - テストケース
   - デプロイ方法

### 元のレポート

3. **[REPORT.md](./REPORT.md)**
   - 初回調査レポート
   - お客様向け説明
   - コミュニティ報告案

---

## 次のステップ

### 即座に対応すべきこと

1. ✅ **DATE_FORMAT_BUG_TECHNICAL.md を読む**
   - 技術的な詳細を理解

2. ✅ **DATE_FORMAT_BUG_FIX_GUIDE.md を読む**
   - 修正手順を確認

3. ✅ **修正を適用**
   - 2行の変更を実施

4. ✅ **テストを実施**
   - 4つのパターンで動作確認

### 中長期的に検討すべきこと

5. 🔲 **Apache Supersetコミュニティへの報告**
   - GitHub Issueまたは Pull Request
   - 他のユーザーへの貢献

6. 🔲 **ドキュメント化**
   - 社内ナレッジベースに追加
   - 再発防止策の検討

---

**作成者**: Claude Code
**作成日**: 2026-04-02
**レビュー推奨**: フロントエンド開発チーム、プロダクトマネージャー
