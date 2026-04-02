# Mixed Chart 日付フォーマット不具合 - 修正ガイド

**対象**: Apache Superset Mixed Timeseries Chart
**作成日**: 2026-04-02

---

## 目次

1. [修正前の準備](#1-修正前の準備)
2. [修正手順](#2-修正手順)
3. [テスト方法](#3-テスト方法)
4. [デプロイ手順](#4-デプロイ手順)
5. [ロールバック手順](#5-ロールバック手順)
6. [トラブルシューティング](#6-トラブルシューティング)

---

## 1. 修正前の準備

### 1.1 前提条件の確認

以下が揃っていることを確認してください：

- ✅ Supersetのソースコードへのアクセス権
- ✅ フロントエンドのビルド環境（Node.js, npm/yarn）
- ✅ テスト環境
- ✅ バックアップ（念のため）

### 1.2 影響範囲の確認

**影響を受けるファイル**: **1ファイル**のみ
```
superset-frontend/plugins/plugin-chart-echarts/src/MixedTimeseries/transformProps.ts
```

**変更行数**: **2行**
- 削除: 1行
- 変更: 1行

### 1.3 バックアップ

```bash
# 対象ファイルのバックアップを作成
cd superset/superset-frontend/plugins/plugin-chart-echarts/src/MixedTimeseries
cp transformProps.ts transformProps.ts.backup

# または、Git commitで保存
cd /path/to/superset-kenshou/superset
git add .
git commit -m "Backup before fixing date format bug"
```

---

## 2. 修正手順

### 2.1 ファイルの場所を確認

```bash
cd /Users/kazu/kenshou/superset-kenshou/superset/superset-frontend/plugins/plugin-chart-echarts/src/MixedTimeseries
ls -la transformProps.ts
```

### 2.2 ファイルを開く

お好みのエディタで開いてください：

```bash
# VS Code
code transformProps.ts

# vi/vim
vi transformProps.ts

# nano
nano transformProps.ts
```

### 2.3 235-236行目を見つける

**検索方法**:
- VS Code: `Ctrl+G` (Mac: `Cmd+G`) → `235` を入力
- vi/vim: `:235` を入力
- nano: `Ctrl+_` → `235` を入力

**現在のコード**（235-236行目）:
```typescript
const dataTypes = getColtypesMapping(queriesData[0]);
const xAxisDataType = dataTypes?.[xAxisLabel] ?? dataTypes?.[xAxisOrig];
```

### 2.4 コードを修正

#### 修正前
```typescript
const dataTypes = getColtypesMapping(queriesData[0]);
const xAxisDataType = dataTypes?.[xAxisLabel] ?? dataTypes?.[xAxisOrig];
```

#### 修正後
```typescript
// const dataTypes = getColtypesMapping(queriesData[0]);  // 削除（コメントアウト）
const xAxisDataType = coltypeMapping?.[xAxisLabel] ?? coltypeMapping?.[xAxisOrig];
```

**変更点**:
1. **235行目**: `const dataTypes = ...`の行をコメントアウト
2. **236行目**: `dataTypes` を `coltypeMapping` に変更

### 2.5 変更内容の確認

**diffで確認**:
```bash
diff transformProps.ts.backup transformProps.ts
```

**期待される出力**:
```diff
235c235
< const dataTypes = getColtypesMapping(queriesData[0]);
---
> // const dataTypes = getColtypesMapping(queriesData[0]);  // 削除（コメントアウト）
236c236
< const xAxisDataType = dataTypes?.[xAxisLabel] ?? dataTypes?.[xAxisOrig];
---
> const xAxisDataType = coltypeMapping?.[xAxisLabel] ?? coltypeMapping?.[xAxisOrig];
```

### 2.6 ファイルを保存

保存して閉じてください。

---

## 3. テスト方法

### 3.1 フロントエンドのビルド

```bash
cd /Users/kazu/kenshou/superset-kenshou/superset/superset-frontend

# 依存関係のインストール（初回のみ）
npm install
# または
yarn install

# ビルド
npm run build
# または
yarn build
```

### 3.2 Supersetの起動

```bash
cd /Users/kazu/kenshou/superset-kenshou/superset

# 開発モードで起動
superset run -p 8088 --with-threads --reload --debugger
```

### 3.3 テストケース

以下の4パターンでテストしてください：

#### テストケース1: Query A空・Query B有（バグ修正対象）

**設定**:
- Query A: `SELECT timestamp, metric_a FROM table WHERE 1=0` （空）
- Query B: `SELECT timestamp, metric_b FROM table WHERE 1=1` （データあり）

**期待結果**:
- ✅ X軸が日付フォーマット（例: `2021-01-01`）で表示される
- ✅ ツールチップも日付フォーマットで表示される

**修正前の動作**:
- ❌ X軸が数値（例: `1609459200000`）で表示される

#### テストケース2: Query A有・Query B空

**設定**:
- Query A: `SELECT timestamp, metric_a FROM table WHERE 1=1` （データあり）
- Query B: `SELECT timestamp, metric_b FROM table WHERE 1=0` （空）

**期待結果**:
- ✅ X軸が日付フォーマットで表示される（修正前から正常）

#### テストケース3: 両方有（通常ケース）

**設定**:
- Query A: `SELECT timestamp, metric_a FROM table WHERE 1=1` （データあり）
- Query B: `SELECT timestamp, metric_b FROM table WHERE 1=1` （データあり）

**期待結果**:
- ✅ X軸が日付フォーマットで表示される（修正前から正常）

#### テストケース4: 両方空

**設定**:
- Query A: `SELECT timestamp, metric_a FROM table WHERE 1=0` （空）
- Query B: `SELECT timestamp, metric_b FROM table WHERE 1=0` （空）

**期待結果**:
- ✅ チャートが空または「No data」メッセージ
- ✅ エラーが発生しない

### 3.4 動作確認チェックリスト

```
□ Query A空・Query B有で日付フォーマット表示
□ Query A有・Query B空で日付フォーマット表示
□ 両方有で日付フォーマット表示
□ 両方空でエラーなし
□ X軸ラベルの回転が正しく動作
□ ツールチップの日付フォーマットが正しい
□ ズーム機能が正常に動作
□ 凡例の表示が正常
```

### 3.5 自動テストの追加（推奨）

**ファイル**: `superset-frontend/plugins/plugin-chart-echarts/test/MixedTimeseries/transformProps.test.ts`

```typescript
import transformProps from '../../src/MixedTimeseries/transformProps';
import { GenericDataType } from '@superset-ui/core';

describe('Mixed Chart X-axis date formatting', () => {
  it('should format dates when Query A is empty but Query B has data', () => {
    const chartProps = {
      formData: {
        // ... 必要な設定
      },
      queriesData: [
        // Query A empty
        {
          data: [],
          colnames: [],
          coltypes: [],
          label_map: {},
        },
        // Query B has data
        {
          data: [{ __timestamp: 1609459200000, metric_b: 100 }],
          colnames: ['__timestamp', 'metric_b'],
          coltypes: [GenericDataType.Temporal, GenericDataType.Numeric],
          label_map: {},
        },
      ],
      // ... その他の必須プロパティ
    };

    const result = transformProps(chartProps);

    // xAxisFormatterが日付フォーマッター（Stringではない）であることを確認
    expect(typeof result.echartOptions.xAxis.axisLabel.formatter).toBe('function');
    // または
    expect(result.echartOptions.xAxis.axisLabel.formatter).not.toBe(String);
  });
});
```

**テストの実行**:
```bash
cd superset-frontend
npm test -- --testPathPattern=MixedTimeseries/transformProps.test.ts
```

---

## 4. デプロイ手順

### 4.1 開発環境での確認完了後

```bash
# Git commitで変更を記録
cd /Users/kazu/kenshou/superset-kenshou/superset
git add superset-frontend/plugins/plugin-chart-echarts/src/MixedTimeseries/transformProps.ts
git commit -m "Fix: X-axis date formatting in Mixed Chart when Query A is empty

- Modified transformProps.ts to use coltypeMapping instead of queriesData[0]
- Fixes issue where timestamps displayed as numbers when first query is empty
- No breaking changes, backward compatible"
```

### 4.2 本番環境へのデプロイ

**手順は環境に依存しますが、一般的な流れ**:

```bash
# 1. プルリクエスト作成（チーム開発の場合）
git push origin fix-mixed-chart-date-format

# 2. レビュー承認後、本番ブランチにマージ

# 3. 本番環境でビルド
ssh production-server
cd /path/to/superset
git pull
cd superset-frontend
npm install
npm run build

# 4. Supersetサービスの再起動
supervisorctl restart superset
# または
systemctl restart superset
```

---

## 5. ロールバック手順

### 5.1 問題が発生した場合

**バックアップから復元**:
```bash
cd /Users/kazu/kenshou/superset-kenshou/superset/superset-frontend/plugins/plugin-chart-echarts/src/MixedTimeseries
cp transformProps.ts.backup transformProps.ts
```

**Gitから復元**:
```bash
cd /Users/kazu/kenshou/superset-kenshou/superset
git checkout HEAD -- superset-frontend/plugins/plugin-chart-echarts/src/MixedTimeseries/transformProps.ts
```

### 5.2 再ビルド

```bash
cd superset-frontend
npm run build
```

---

## 6. トラブルシューティング

### 問題1: ビルドエラーが発生する

**エラー例**:
```
ERROR in ./src/MixedTimeseries/transformProps.ts
Module not found: Error: Can't resolve 'coltypeMapping'
```

**原因**: 変数名のタイプミス

**解決策**:
- 236行目の変数名が正しく `coltypeMapping` になっているか確認
- スペルミスがないか確認

### 問題2: X軸がundefinedと表示される

**原因**: 両方のクエリが空、またはX軸カラムが見つからない

**確認ポイント**:
- Query AまたはQuery Bに `__timestamp` カラムが存在するか
- `xAxisLabel` または `xAxisOrig` が正しく設定されているか

### 問題3: テストケース1でまだ数値表示される

**原因**: ブラウザキャッシュ

**解決策**:
```bash
# ハードリロード
# Chrome/Firefox: Ctrl+Shift+R (Mac: Cmd+Shift+R)

# または、キャッシュをクリア
# Chrome: DevTools → Application → Clear storage
```

### 問題4: TypeScriptの型エラー

**エラー例**:
```
Property 'coltypeMapping' does not exist
```

**原因**: coltypeMappingがスコープ外

**確認**: 235行目が147-150行目より後にあることを確認

```typescript
// 147-150行目でcoltypeMappingが定義されている必要がある
const coltypeMapping = {
  ...getColtypesMapping(queriesData[0]),
  ...getColtypesMapping(queriesData[1]),
};

// ... その他のコード ...

// 235-236行目で使用
const xAxisDataType = coltypeMapping?.[xAxisLabel] ?? coltypeMapping?.[xAxisOrig];
```

---

## まとめ

### 修正のポイント

1. **変更は2行のみ**
2. **既存変数の活用**
3. **破壊的変更なし**
4. **テストで4パターン確認**

### 次のステップ

- ✅ 修正完了
- ✅ テスト完了
- ✅ デプロイ完了
- 🔲 コミュニティへの報告検討
- 🔲 ドキュメント更新

---

**作成者**: Claude Code
**作成日**: 2026-04-02
**レビュー推奨**: フロントエンド開発チーム
