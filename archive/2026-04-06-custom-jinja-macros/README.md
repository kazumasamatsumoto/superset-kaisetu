# カスタムJinjaマクロ調査レポート - 2026-04-06

Apache Superset のカスタムJinjaマクロ実装に関する包括的な調査レポート

---

## 📋 調査概要

**調査日**: 2026-04-06
**対象**: Apache Superset カスタムJinjaマクロ機能
**調査者**: Claude Code

### 調査の目的

Supersetでカスタムマクロを作成する際、ソースコードを編集せず、`superset_config.py`にカスタムマクロの定義を実装することで、Superset側で取り込むことができるか調査する。

---

## 📚 レポート

### カスタムJinjaマクロ実装ガイド

**ファイル**: `CUSTOM_JINJA_MACROS_GUIDE.md`

**内容**:
- 実現可能性の結論（✅ 100%実現可能）
- 仕組みの全体像（図式化）
- superset_config.pyの読み込みメカニズム
- JINJA_CONTEXT_ADDONSの仕組み
- カスタムマクロの登録・実行フロー（詳細図）
- 実装例7パターン
- セキュリティ上の注意点
- ベストプラクティス
- トラブルシューティング
- 主要ファイル一覧

**重要度**: ⭐⭐⭐⭐⭐

---

## 🔍 調査結果サマリー

### ✅ 実現可能性

**Yes - 完全に実現可能**

Supersetは`superset_config.py`を使ったカスタムマクロの追加を**公式にサポート**しています。

### 🔑 キー設定項目

| 設定項目 | 用途 |
|---------|------|
| `JINJA_CONTEXT_ADDONS` | カスタムマクロ関数の辞書 |
| `CUSTOM_TEMPLATE_PROCESSORS` | データベースエンジン固有のテンプレートプロセッサー |

### 📊 実装の難易度

- **設定ファイル編集**: ⭐☆☆☆☆（非常に簡単）
- **カスタム関数作成**: ⭐⭐☆☆☆（簡単）
- **デプロイ**: ⭐☆☆☆☆（設定ファイルをコピーするだけ）

---

## 🎯 重要な発見

### 1. superset_config.pyの読み込みメカニズム

Supersetは**2つの方法**で`superset_config.py`を読み込みます：

#### 方法1: 環境変数SUPERSET_CONFIG_PATH

```bash
export SUPERSET_CONFIG_PATH=/path/to/custom_config.py
superset run
```

- 任意のパスから設定ファイルを読み込める
- PEX環境で有用
- **大文字で始まる変数のみ**がマージされる

#### 方法2: PYTHONPATHからのインポート

```bash
export PYTHONPATH=/etc/superset:$PYTHONPATH
superset run
```

- `from superset_config import *` ですべてをインポート
- **最も一般的な方法**

---

### 2. JINJA_CONTEXT_ADDONSの自動マージ

**ファイル**: `jinja_context.py:599`

```python
def set_context(self, **kwargs: Any) -> None:
    self._context.update(kwargs)
    self._context.update(context_addons())  # ← 自動マージ
```

**仕組み**:
1. `context_addons()`が`current_app.config`から`JINJA_CONTEXT_ADDONS`を取得
2. `@lru_cache`でキャッシュされる（高速化）
3. `_context`辞書に自動的にマージされる

---

### 3. セキュリティ対策: safe_proxy()

**ファイル**: `jinja_context.py:464-486`

```python
def safe_proxy(func: Callable[..., Any], *args: Any, **kwargs: Any) -> Any:
    return_value = func(*args, **kwargs)
    value_type = type(return_value).__name__

    # 許可された型のみ通過
    if value_type not in ALLOWED_TYPES:
        raise SupersetTemplateException(...)

    return return_value
```

**許可された型**:
- `None`, `bool`, `str`, `int`, `float`, `list`, `dict`, `tuple`, `set`, `TimeFilter`

---

### 4. クエリレンダリングフロー

```
SQL Labでクエリ実行
    ↓
SqlQueryRenderImpl.render()
    ↓
get_template_processor()
    ↓
JinjaTemplateProcessor生成
    ↓
set_context() → context_addons()
    ↓
process_template()
    ↓
Jinjaレンダリング（カスタムマクロ実行）
    ↓
レンダリング済みSQL
```

---

## 💡 実装例

### 基本的なカスタムマクロ

```python
# superset_config.py

def get_company_id():
    """会社IDを返す"""
    return 123

def get_date_range(days=7):
    """指定日数分の日付範囲を返す"""
    from datetime import datetime, timedelta
    end_date = datetime.now()
    start_date = end_date - timedelta(days=days)
    return {
        "start": start_date.strftime("%Y-%m-%d"),
        "end": end_date.strftime("%Y-%m-%d")
    }

JINJA_CONTEXT_ADDONS = {
    "get_company_id": get_company_id,
    "get_date_range": get_date_range,
}
```

### SQL Labでの使用

```sql
-- 会社IDでフィルター
SELECT *
FROM sales
WHERE company_id = {{ get_company_id() }}

-- 日付範囲でフィルター
{% set date_range = get_date_range(30) %}
SELECT *
FROM sales
WHERE date >= '{{ date_range.start }}'
  AND date <= '{{ date_range.end }}'
```

---

## ⚠️ セキュリティ上の注意点

### 危険な実装例

```python
# ❌ 危険！任意コード実行
def execute_code(code):
    return eval(code)

# ❌ 危険！機密情報の漏洩
def get_credentials():
    return {"user": "admin", "password": "secret"}

# ❌ 危険！SQLインジェクション
def get_table_name(table):
    return table  # 検証なし
```

### 安全な実装例

```python
# ✅ 安全：ホワイトリスト方式
def get_table_name(table):
    allowed_tables = {"users", "products", "orders"}
    if table not in allowed_tables:
        raise ValueError(f"Table {table} not allowed")
    return table

# ✅ 安全：プリミティブ型のみ返す
def get_config():
    return {
        "value1": "text",
        "value2": 123,
    }

# ✅ 安全：例外処理とデフォルト値
def get_external_data():
    try:
        response = requests.get("https://api.example.com/data", timeout=5)
        return response.json()
    except Exception:
        return {"status": "error"}
```

---

## 🛠️ 主要ファイル

| ファイル | 役割 | 重要度 |
|---------|------|--------|
| `superset/config.py` | デフォルト設定、superset_config.pyの読み込み | ⭐⭐⭐⭐⭐ |
| `superset/jinja_context.py` | Jinjaコンテキスト、テンプレートプロセッサー | ⭐⭐⭐⭐⭐ |
| `superset/sqllab/query_render.py` | SQLクエリレンダリング | ⭐⭐⭐⭐☆ |

### 主要関数

| 関数 | 説明 | ファイル:行数 |
|------|------|--------------|
| `context_addons()` | `JINJA_CONTEXT_ADDONS`を取得 | `jinja_context.py:76-78` |
| `set_context()` | コンテキストを設定 | `jinja_context.py:597-599` |
| `process_template()` | Jinjaテンプレートをレンダリング | `jinja_context.py:607-623` |
| `safe_proxy()` | 関数の戻り値を安全にラップ | `jinja_context.py:464-486` |
| `get_template_processor()` | テンプレートプロセッサーのファクトリー | `jinja_context.py:819-831` |

---

## 📊 メリット vs デメリット

### メリット

| メリット | 説明 |
|---------|------|
| ✅ **ソースコード編集不要** | Supersetのソースコードを一切変更しない |
| ✅ **アップグレード安全** | Supersetのアップグレード時にカスタマイズが失われない |
| ✅ **メンテナンス容易** | 設定ファイル1つで管理 |
| ✅ **デプロイ簡単** | `superset_config.py`をコピーするだけ |
| ✅ **公式サポート** | Supersetが公式にサポートする方法 |
| ✅ **柔軟性** | 任意のPythonコードを実行可能 |

### 注意点

| 注意点 | 対策 |
|--------|------|
| ⚠️ **セキュリティリスク** | プリミティブ型のみ返す、入力検証、safe_proxy()使用 |
| ⚠️ **パフォーマンス** | `@lru_cache`でキャッシュ、タイムアウト設定 |
| ⚠️ **再起動必要** | 設定変更後は必ずSuperset再起動 |
| ⚠️ **エラーハンドリング** | 適切な例外処理と安全なデフォルト値 |

---

## 🎓 ベストプラクティス

### 1. 命名規則

- 動詞で始める: `get_`, `fetch_`, `calculate_`, `format_`
- 小文字とアンダースコア（snake_case）

### 2. ドキュメント化

- すべての関数にdocstringを追加
- 引数、戻り値、使用例を記述

### 3. テスト

- カスタムマクロのユニットテストを作成
- 型チェック、エラーケースをカバー

### 4. パフォーマンス最適化

- `@lru_cache`でキャッシュ
- 外部API呼び出しにタイムアウト設定

### 5. エラーハンドリング

- 適切な例外処理
- 安全なデフォルト値を返す

---

## 🚀 クイックスタート

### 1. superset_config.pyを作成

```bash
sudo mkdir -p /etc/superset
sudo vi /etc/superset/superset_config.py
```

### 2. カスタムマクロを定義

```python
def my_custom_macro():
    return "Hello from custom macro!"

JINJA_CONTEXT_ADDONS = {
    "my_custom_macro": my_custom_macro,
}
```

### 3. PYTHONPATHに追加してSuperset起動

```bash
export PYTHONPATH=/etc/superset:$PYTHONPATH
superset run
```

### 4. SQL Labで使用

```sql
SELECT '{{ my_custom_macro() }}' AS message
-- 結果: 'Hello from custom macro!'
```

---

## 📝 まとめ

### 調査の成果

1. ✅ カスタムマクロの実装が**100%実現可能**であることを確認
2. ✅ `superset_config.py`の読み込みメカニズムを解明
3. ✅ `JINJA_CONTEXT_ADDONS`の自動マージ処理を解析
4. ✅ クエリレンダリングフローを図式化
5. ✅ 7パターンの実装例を提供
6. ✅ セキュリティベストプラクティスを文書化
7. ✅ トラブルシューティングガイドを作成

### 推奨される使用例

- ✅ 会社ID、部署コードなどの組織固有の値
- ✅ 会計年度、四半期などの複雑な日付計算
- ✅ 外部APIからマスターデータを取得（キャッシュ付き）
- ✅ 環境変数から設定値を取得
- ✅ 複雑なフィルター条件の生成

### 避けるべき使用例

- ❌ 任意のPythonコード実行（`eval()`, `exec()`）
- ❌ データベース認証情報の返却
- ❌ ファイルシステムへのアクセス
- ❌ クラスインスタンスの返却
- ❌ キャッシュなしの外部API呼び出し

---

## 🔗 関連リソース

### 内部ドキュメント

- なし（初回調査）

### 外部リソース

- [Superset公式ドキュメント - Jinja Templating](https://superset.apache.org/docs/installation/sql-templating)
- [Superset公式ドキュメント - Configuration](https://superset.apache.org/docs/installation/configuring-superset)
- [GitHub - superset/config.py](https://github.com/apache/superset/blob/master/superset/config.py)
- [GitHub - superset/jinja_context.py](https://github.com/apache/superset/blob/master/superset/jinja_context.py)

---

**調査実施**: Claude Code
**作成日**: 2026-04-06
**Supersetバージョン**: 4.x以降対応
