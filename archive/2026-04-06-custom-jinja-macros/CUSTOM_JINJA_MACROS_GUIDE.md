# Superset カスタムJinjaマクロ実装ガイド

## 📋 目次

1. [概要](#概要)
2. [調査結果サマリー](#調査結果サマリー)
3. [実現可能性の結論](#実現可能性の結論)
4. [仕組みの全体像](#仕組みの全体像)
5. [superset_config.pyの読み込みメカニズム](#superset_configpyの読み込みメカニズム)
6. [JINJA_CONTEXT_ADDONSの仕組み](#jinja_context_addonsの仕組み)
7. [カスタムマクロの登録・実行フロー](#カスタムマクロの登録実行フロー)
8. [実装例](#実装例)
9. [セキュリティ上の注意点](#セキュリティ上の注意点)
10. [ベストプラクティス](#ベストプラクティス)
11. [トラブルシューティング](#トラブルシューティング)
12. [主要ファイル一覧](#主要ファイル一覧)
13. [結論](#結論)

---

## 概要

### 調査の目的

**質問**: Supersetでカスタムマクロを作成する際、ソースコードを編集せず、`superset_config.py`にカスタムマクロの定義を実装することで、Superset側で取り込むことができるのか？

### 背景

Supersetのデフォルトでは、以下のような組み込みマクロが提供されています：

- `{{ current_user_id() }}` - 現在のユーザーID
- `{{ current_username() }}` - 現在のユーザー名
- `{{ url_param('param_name') }}` - URLパラメータ
- `{{ filter_values('column_name') }}` - フィルター値

しかし、カスタムロジックを実装したい場合、ソースコードを編集するのは以下の理由で望ましくありません：

- Supersetのアップグレード時にカスタマイズが失われる
- メンテナンスが困難
- デプロイメントが複雑化

---

## 調査結果サマリー

### ✅ 実現可能性

**Yes - 完全に実現可能です！**

Supersetは`superset_config.py`を使ったカスタムマクロの追加を**公式にサポート**しています。

### 🔑 キー設定項目

| 設定項目 | 用途 | 型 |
|---------|------|-----|
| **JINJA_CONTEXT_ADDONS** | カスタムマクロ関数の辞書 | `dict[str, Callable[..., Any]]` |
| **CUSTOM_TEMPLATE_PROCESSORS** | データベースエンジン固有のテンプレートプロセッサー | `dict[str, type[BaseTemplateProcessor]]` |

### 📊 実装の難易度

- **設定ファイル編集**: ⭐☆☆☆☆（非常に簡単）
- **カスタム関数作成**: ⭐⭐☆☆☆（簡単）
- **デプロイ**: ⭐☆☆☆☆（設定ファイルをコピーするだけ）

---

## 実現可能性の結論

### ✅ 100%実現可能

**理由**:

1. **公式サポート**: `superset/config.py:1193-1203`で`JINJA_CONTEXT_ADDONS`が定義されている
2. **自動マージ**: `jinja_context.py:599`で自動的にコンテキストにマージされる
3. **ソースコード編集不要**: `superset_config.py`のみで完結

**証拠**:

```python
# superset/config.py:1193-1203
# A dictionary of items that gets merged into the Jinja context for
# SQL Lab. The existing context gets updated with this dictionary,
# meaning values for existing keys get overwritten by the content of this
# dictionary.
JINJA_CONTEXT_ADDONS: dict[str, Callable[..., Any]] = {}
```

```python
# jinja_context.py:76-78
@lru_cache(maxsize=LRU_CACHE_MAX_SIZE)
def context_addons() -> dict[str, Any]:
    return current_app.config.get("JINJA_CONTEXT_ADDONS", {})
```

```python
# jinja_context.py:599
def set_context(self, **kwargs: Any) -> None:
    self._context.update(kwargs)
    self._context.update(context_addons())  # ← ここでマージ
```

---

## 仕組みの全体像

### アーキテクチャ図

```
┌─────────────────────────────────────────────────────────────────┐
│ 1. アプリケーション起動                                            │
├─────────────────────────────────────────────────────────────────┤
│ Superset起動                                                    │
│    ↓                                                            │
│ superset/config.py を読み込み                                   │
│    ├─ デフォルト設定をロード                                     │
│    └─ JINJA_CONTEXT_ADDONS = {} （デフォルトは空）               │
│    ↓                                                            │
│ superset_config.py を探す                                       │
│    ├─ 環境変数 SUPERSET_CONFIG_PATH が設定されているか？         │
│    │   └─ Yes → 指定パスからロード                              │
│    └─ No → PYTHONPATH から superset_config.py を探す           │
│         └─ 見つかった場合、`from superset_config import *`     │
│    ↓                                                            │
│ JINJA_CONTEXT_ADDONS がマージされる                             │
│    └─ カスタムマクロ定義が上書き                                │
└─────────────────────────────────────────────────────────────────┘
                          ↓
┌─────────────────────────────────────────────────────────────────┐
│ 2. ユーザーがSQLクエリを実行                                       │
├─────────────────────────────────────────────────────────────────┤
│ ユーザーがSQL Labで以下のクエリを入力:                            │
│ SELECT * FROM users WHERE id = {{ my_custom_macro() }}          │
│    ↓                                                            │
│ SqlQueryRenderImpl.render() が呼ばれる                          │
│    └─ query_render.py:53-65                                    │
│    ↓                                                            │
│ get_template_processor() が呼ばれる                             │
│    └─ jinja_context.py:819-831                                 │
│    ↓                                                            │
│ JinjaTemplateProcessor が生成される                             │
│    └─ set_context() でコンテキストを設定                        │
│         └─ context_addons() を呼び出し                         │
│              └─ JINJA_CONTEXT_ADDONS を取得                    │
│                   └─ カスタムマクロがコンテキストに追加         │
│    ↓                                                            │
│ process_template() が呼ばれる                                   │
│    └─ jinja_context.py:607-623                                 │
│    └─ Jinjaテンプレートがレンダリングされる                      │
│         └─ {{ my_custom_macro() }} が実行される                │
│    ↓                                                            │
│ レンダリング後のSQL:                                             │
│ SELECT * FROM users WHERE id = 123                              │
└─────────────────────────────────────────────────────────────────┘
                          ↓
┌─────────────────────────────────────────────────────────────────┐
│ 3. SQL実行                                                       │
├─────────────────────────────────────────────────────────────────┤
│ レンダリングされたSQLがデータベースに送信される                   │
│    ↓                                                            │
│ 結果がユーザーに返される                                         │
└─────────────────────────────────────────────────────────────────┘
```

---

## superset_config.pyの読み込みメカニズム

### 読み込みの仕組み

Supersetは**2つの方法**で`superset_config.py`を読み込みます：

#### 方法1: 環境変数SUPERSET_CONFIG_PATH

**ファイル**: `superset/config.py:1933-1951`

```python
if CONFIG_PATH_ENV_VAR in os.environ:
    # SUPERSET_CONFIG_PATH 環境変数が設定されている場合
    cfg_path = os.environ[CONFIG_PATH_ENV_VAR]
    try:
        module = sys.modules[__name__]
        spec = importlib.util.spec_from_file_location("superset_config", cfg_path)
        override_conf = importlib.util.module_from_spec(spec)
        spec.loader.exec_module(override_conf)

        # 大文字で始まる変数のみをコピー
        for key in dir(override_conf):
            if key.isupper():
                setattr(module, key, getattr(override_conf, key))

        click.secho(f"Loaded your LOCAL configuration at [{cfg_path}]", fg="cyan")
    except Exception:
        logger.exception("Failed to import config for %s=%s", CONFIG_PATH_ENV_VAR, cfg_path)
        raise
```

**特徴**:
- 任意のパスから設定ファイルを読み込める
- PEX（Python Executable）環境で有用
- **大文字で始まる変数のみ**がマージされる

**使用例**:
```bash
export SUPERSET_CONFIG_PATH=/path/to/my/custom_config.py
superset run
```

---

#### 方法2: PYTHONPATHからのインポート

**ファイル**: `superset/config.py:1952-1964`

```python
elif importlib.util.find_spec("superset_config"):
    try:
        # superset_config モジュールが見つかった場合
        import superset_config
        from superset_config import *  # すべてをインポート

        click.secho(
            f"Loaded your LOCAL configuration at [{superset_config.__file__}]",
            fg="cyan",
        )
    except Exception:
        logger.exception("Found but failed to import local superset_config")
        raise
```

**特徴**:
- `PYTHONPATH`に含まれるディレクトリから`superset_config.py`を探す
- `from superset_config import *` ですべての変数をインポート
- **最も一般的な方法**

**使用例**:
```bash
# 1. /etc/superset/ に superset_config.py を配置
sudo mkdir -p /etc/superset
sudo vi /etc/superset/superset_config.py

# 2. PYTHONPATHに追加
export PYTHONPATH=/etc/superset:$PYTHONPATH

# 3. Superset起動
superset run
```

---

### 読み込み優先順位

```
1. SUPERSET_CONFIG_PATH環境変数
   ↓
2. PYTHONPATHからsuperset_configモジュール
   ↓
3. どちらも見つからない場合、デフォルト設定のみ
```

---

### 読み込み確認方法

Supersetが起動時に以下のメッセージを表示します：

```bash
Loaded your LOCAL configuration at [/etc/superset/superset_config.py]
```

**ログ出力箇所**:
- 方法1: `config.py:1946`
- 方法2: `config.py:1958-1960`

---

## JINJA_CONTEXT_ADDONSの仕組み

### 定義と型

**ファイル**: `superset/config.py:1193-1203`

```python
# A dictionary of items that gets merged into the Jinja context for
# SQL Lab. The existing context gets updated with this dictionary,
# meaning values for existing keys get overwritten by the content of this
# dictionary. Exposing functionality through JINJA_CONTEXT_ADDONS has security
# implications as it opens a window for a user to execute untrusted code.
# It's important to make sure that the objects exposed (as well as objects attached
# to those objects) are harmless. We recommend only exposing simple/pure functions that
# return native types.
JINJA_CONTEXT_ADDONS: dict[str, Callable[..., Any]] = {}
```

**型の解説**:
- `dict[str, Callable[..., Any]]` - キーが文字列、値が呼び出し可能なオブジェクト（関数）
- `Callable[..., Any]` - 任意の引数を受け取り、任意の値を返す関数

---

### コンテキストへのマージ処理

#### ステップ1: context_addons() 関数

**ファイル**: `jinja_context.py:76-78`

```python
@lru_cache(maxsize=LRU_CACHE_MAX_SIZE)
def context_addons() -> dict[str, Any]:
    return current_app.config.get("JINJA_CONTEXT_ADDONS", {})
```

**重要ポイント**:
- `@lru_cache` でキャッシュされる（高速化）
- `current_app.config` からFlaskアプリケーション設定を取得
- `JINJA_CONTEXT_ADDONS`が未設定の場合は空の辞書`{}`を返す

---

#### ステップ2: set_context() でマージ

**ファイル**: `jinja_context.py:597-599` (BaseTemplateProcessor)

```python
def set_context(self, **kwargs: Any) -> None:
    self._context.update(kwargs)
    self._context.update(context_addons())  # ← ここでマージ！
```

**処理の流れ**:

```python
# 1. デフォルトコンテキスト
self._context = {}

# 2. 引数で渡された値をマージ
self._context.update(kwargs)
# 例: {'from_dttm': '2024-01-01', 'to_dttm': '2024-12-31'}

# 3. context_addons() を呼び出し、JINJA_CONTEXT_ADDONSをマージ
self._context.update(context_addons())
# 例: {'my_custom_macro': <function my_custom_macro>}

# 最終的なコンテキスト:
# {
#     'from_dttm': '2024-01-01',
#     'to_dttm': '2024-12-31',
#     'my_custom_macro': <function my_custom_macro>,
#     'current_user_id': <function current_user_id>,
#     'url_param': <function url_param>,
#     ...
# }
```

---

#### ステップ3: JinjaTemplateProcessor でさらに追加

**ファイル**: `jinja_context.py:640-691`

```python
def set_context(self, **kwargs: Any) -> None:
    super().set_context(**kwargs)  # ← BaseTemplateProcessor.set_context() 呼び出し

    extra_cache = ExtraCache(...)

    # 組み込みマクロを追加
    self._context.update(
        {
            "url_param": partial(safe_proxy, extra_cache.url_param),
            "current_user_id": partial(safe_proxy, extra_cache.current_user_id),
            "current_username": partial(safe_proxy, extra_cache.current_username),
            "current_user_email": partial(safe_proxy, extra_cache.current_user_email),
            "filter_values": partial(safe_proxy, extra_cache.filter_values),
            "get_filters": partial(safe_proxy, extra_cache.get_filters),
            "get_time_filter": partial(safe_proxy, extra_cache.get_time_filter),
            # ...
        }
    )
```

**最終的なマージ順序**:

```
1. デフォルトコンテキスト (空)
   ↓
2. kwargs (引数で渡された値)
   ↓
3. context_addons() (JINJA_CONTEXT_ADDONS)  ← カスタムマクロ
   ↓
4. 組み込みマクロ (current_user_id, url_param, etc.)
```

**重要**: `JINJA_CONTEXT_ADDONS`で定義したマクロは、組み込みマクロと**同じ名前の場合、組み込みマクロに上書きされます**。

---

### safe_proxy() の仕組み

**ファイル**: `jinja_context.py:464-486`

```python
def safe_proxy(func: Callable[..., Any], *args: Any, **kwargs: Any) -> Any:
    return_value = func(*args, **kwargs)
    value_type = type(return_value).__name__

    # 許可された型のみ通過
    if value_type not in ALLOWED_TYPES:
        raise SupersetTemplateException(
            _(
                "Unsafe return type for function %(func)s: %(value_type)s",
                func=func.__name__,
                value_type=value_type,
            )
        )

    # コレクション型の場合、JSON変換でサニタイズ
    if value_type in COLLECTION_TYPES:
        try:
            return_value = json.loads(json.dumps(return_value))
        except TypeError as ex:
            raise SupersetTemplateException(...) from ex

    return return_value
```

**許可された型**:

```python
ALLOWED_TYPES = (
    NONE_TYPE,  # None
    "bool",
    "str",
    "unicode",
    "int",
    "long",
    "float",
    "list",
    "dict",
    "tuple",
    "set",
    "TimeFilter",
)
```

**セキュリティ対策**:
- 関数の戻り値の型をチェック
- 許可されていない型（クラスインスタンス、lambdaなど）を拒否
- コレクション型はJSON変換でサニタイズ

---

## カスタムマクロの登録・実行フロー

### フロー図

```
┌─────────────────────────────────────────────────────────────────┐
│ superset_config.py                                              │
├─────────────────────────────────────────────────────────────────┤
│ def my_custom_macro():                                          │
│     return 42                                                   │
│                                                                 │
│ JINJA_CONTEXT_ADDONS = {                                        │
│     "my_custom_macro": my_custom_macro,                         │
│ }                                                               │
└─────────────────────────────────────────────────────────────────┘
                          ↓
                  (アプリケーション起動)
                          ↓
┌─────────────────────────────────────────────────────────────────┐
│ superset/config.py                                              │
├─────────────────────────────────────────────────────────────────┤
│ JINJA_CONTEXT_ADDONS: dict = {}  # デフォルト                   │
│                                                                 │
│ ...（1900行後）...                                              │
│                                                                 │
│ if importlib.util.find_spec("superset_config"):                │
│     from superset_config import *  # マージ！                   │
│                                                                 │
│ # 結果:                                                         │
│ # JINJA_CONTEXT_ADDONS = {"my_custom_macro": my_custom_macro}  │
└─────────────────────────────────────────────────────────────────┘
                          ↓
                  (Flask app.config に保存)
                          ↓
┌─────────────────────────────────────────────────────────────────┐
│ current_app.config                                              │
├─────────────────────────────────────────────────────────────────┤
│ {                                                               │
│     "JINJA_CONTEXT_ADDONS": {                                   │
│         "my_custom_macro": <function my_custom_macro>           │
│     },                                                          │
│     ...                                                         │
│ }                                                               │
└─────────────────────────────────────────────────────────────────┘
                          ↓
              (ユーザーがSQLクエリを実行)
                          ↓
┌─────────────────────────────────────────────────────────────────┐
│ SQL Lab                                                         │
├─────────────────────────────────────────────────────────────────┤
│ SELECT * FROM users WHERE id = {{ my_custom_macro() }}          │
└─────────────────────────────────────────────────────────────────┘
                          ↓
┌─────────────────────────────────────────────────────────────────┐
│ sqllab/query_render.py                                          │
├─────────────────────────────────────────────────────────────────┤
│ def render(self, execution_context):                            │
│     sql_template_processor = get_template_processor(            │
│         database=query_model.database,                          │
│         query=query_model                                       │
│     )                                                           │
│     rendered_query = sql_template_processor.process_template(   │
│         query_model.sql,                                        │
│         **execution_context.template_params                     │
│     )                                                           │
│     return rendered_query                                       │
└─────────────────────────────────────────────────────────────────┘
                          ↓
┌─────────────────────────────────────────────────────────────────┐
│ jinja_context.py: get_template_processor()                     │
├─────────────────────────────────────────────────────────────────┤
│ def get_template_processor(database, table, query, **kwargs):  │
│     template_processor = JinjaTemplateProcessor                 │
│     return template_processor(                                  │
│         database=database,                                      │
│         table=table,                                            │
│         query=query,                                            │
│         **kwargs                                                │
│     )                                                           │
└─────────────────────────────────────────────────────────────────┘
                          ↓
┌─────────────────────────────────────────────────────────────────┐
│ JinjaTemplateProcessor.__init__()                              │
├─────────────────────────────────────────────────────────────────┤
│ def __init__(self, ...):                                        │
│     super().__init__(...)                                       │
│     self.env = SandboxedEnvironment()                           │
│     self.set_context(**kwargs)  # ← コンテキスト設定            │
└─────────────────────────────────────────────────────────────────┘
                          ↓
┌─────────────────────────────────────────────────────────────────┐
│ JinjaTemplateProcessor.set_context()                           │
├─────────────────────────────────────────────────────────────────┤
│ def set_context(self, **kwargs):                               │
│     super().set_context(**kwargs)  # BaseTemplateProcessor     │
│     # 組み込みマクロを追加                                       │
│     self._context.update({                                      │
│         "current_user_id": ...,                                 │
│         "url_param": ...,                                       │
│         ...                                                     │
│     })                                                          │
└─────────────────────────────────────────────────────────────────┘
                          ↓
┌─────────────────────────────────────────────────────────────────┐
│ BaseTemplateProcessor.set_context()                            │
├─────────────────────────────────────────────────────────────────┤
│ def set_context(self, **kwargs):                               │
│     self._context.update(kwargs)                                │
│     self._context.update(context_addons())  # ← ここ！          │
└─────────────────────────────────────────────────────────────────┘
                          ↓
┌─────────────────────────────────────────────────────────────────┐
│ context_addons()                                                │
├─────────────────────────────────────────────────────────────────┤
│ @lru_cache(maxsize=LRU_CACHE_MAX_SIZE)                          │
│ def context_addons() -> dict[str, Any]:                        │
│     return current_app.config.get("JINJA_CONTEXT_ADDONS", {})  │
│                                                                 │
│ # 返り値:                                                       │
│ # {"my_custom_macro": <function my_custom_macro>}              │
└─────────────────────────────────────────────────────────────────┘
                          ↓
          (self._contextに my_custom_macro が追加される)
                          ↓
┌─────────────────────────────────────────────────────────────────┐
│ self._context                                                   │
├─────────────────────────────────────────────────────────────────┤
│ {                                                               │
│     "my_custom_macro": <function my_custom_macro>,              │
│     "current_user_id": <function current_user_id>,              │
│     "url_param": <function url_param>,                          │
│     ...                                                         │
│ }                                                               │
└─────────────────────────────────────────────────────────────────┘
                          ↓
┌─────────────────────────────────────────────────────────────────┐
│ BaseTemplateProcessor.process_template()                       │
├─────────────────────────────────────────────────────────────────┤
│ def process_template(self, sql, **kwargs):                     │
│     template = self.env.from_string(sql)                        │
│     kwargs.update(self._context)  # ← コンテキストをマージ      │
│     context = validate_template_context(self.engine, kwargs)    │
│     return template.render(context)  # ← Jinjaレンダリング     │
└─────────────────────────────────────────────────────────────────┘
                          ↓
┌─────────────────────────────────────────────────────────────────┐
│ Jinja2 テンプレートエンジン                                       │
├─────────────────────────────────────────────────────────────────┤
│ SQL: SELECT * FROM users WHERE id = {{ my_custom_macro() }}     │
│                                                                 │
│ {{ my_custom_macro() }} を評価:                                 │
│     context["my_custom_macro"]() を呼び出し                     │
│     → <function my_custom_macro>()                              │
│     → return 42                                                 │
│                                                                 │
│ レンダリング結果:                                                │
│ SELECT * FROM users WHERE id = 42                               │
└─────────────────────────────────────────────────────────────────┘
                          ↓
                  (レンダリングされたSQLを返す)
                          ↓
┌─────────────────────────────────────────────────────────────────┐
│ SQL実行                                                          │
├─────────────────────────────────────────────────────────────────┤
│ SELECT * FROM users WHERE id = 42                               │
└─────────────────────────────────────────────────────────────────┘
```

---

## 実装例

### 例1: 基本的なカスタムマクロ

**ファイル**: `/etc/superset/superset_config.py`

```python
# カスタムマクロ関数の定義
def get_company_id():
    """
    会社IDを返すカスタムマクロ
    """
    return 123


def get_date_range(days=7):
    """
    指定された日数分の日付範囲を返すカスタムマクロ
    """
    from datetime import datetime, timedelta
    end_date = datetime.now()
    start_date = end_date - timedelta(days=days)
    return {
        "start": start_date.strftime("%Y-%m-%d"),
        "end": end_date.strftime("%Y-%m-%d")
    }


# JINJA_CONTEXT_ADDONSに登録
JINJA_CONTEXT_ADDONS = {
    "get_company_id": get_company_id,
    "get_date_range": get_date_range,
}
```

**SQL Lab での使用例**:

```sql
-- 例1: 会社IDでフィルター
SELECT *
FROM sales
WHERE company_id = {{ get_company_id() }}

-- 例2: 日付範囲でフィルター
{% set date_range = get_date_range(30) %}
SELECT *
FROM sales
WHERE date >= '{{ date_range.start }}'
  AND date <= '{{ date_range.end }}'
```

---

### 例2: 引数を受け取るカスタムマクロ

**ファイル**: `/etc/superset/superset_config.py`

```python
def get_region_codes(region_name):
    """
    地域名から地域コードのリストを返す
    """
    region_mapping = {
        "north": ["01", "02", "03"],
        "south": ["04", "05", "06"],
        "east": ["07", "08", "09"],
        "west": ["10", "11", "12"],
    }
    return region_mapping.get(region_name, [])


def format_currency(amount):
    """
    金額をフォーマットする
    """
    return f"${amount:,.2f}"


JINJA_CONTEXT_ADDONS = {
    "get_region_codes": get_region_codes,
    "format_currency": format_currency,
}
```

**SQL Lab での使用例**:

```sql
-- 例1: 地域コードでフィルター
SELECT *
FROM sales
WHERE region_code IN ({{ "'" + "','".join(get_region_codes('north')) + "'" }})
-- 結果: WHERE region_code IN ('01','02','03')

-- 例2: 金額フォーマット（SELECT句で使用）
SELECT
    product_name,
    '{{ format_currency(price) }}' AS formatted_price
FROM products
```

---

### 例3: 外部APIを呼び出すカスタムマクロ

**ファイル**: `/etc/superset/superset_config.py`

```python
import requests
from functools import lru_cache


@lru_cache(maxsize=128)
def get_exchange_rate(from_currency="USD", to_currency="JPY"):
    """
    為替レートを取得する（キャッシュ付き）
    """
    try:
        # 実際のAPI呼び出し（例）
        # response = requests.get(f"https://api.example.com/rates?from={from_currency}&to={to_currency}")
        # return response.json()["rate"]

        # デモ用の固定値
        if from_currency == "USD" and to_currency == "JPY":
            return 150.0
        return 1.0
    except Exception as e:
        # エラー時はデフォルト値を返す
        return 1.0


@lru_cache(maxsize=128)
def get_active_campaigns():
    """
    現在アクティブなキャンペーンIDのリストを取得
    """
    try:
        # 実際のAPI呼び出し（例）
        # response = requests.get("https://api.example.com/campaigns/active")
        # return response.json()["campaign_ids"]

        # デモ用の固定値
        return ["CAMP001", "CAMP002", "CAMP003"]
    except Exception:
        return []


JINJA_CONTEXT_ADDONS = {
    "get_exchange_rate": get_exchange_rate,
    "get_active_campaigns": get_active_campaigns,
}
```

**SQL Lab での使用例**:

```sql
-- 例1: 為替レート適用
SELECT
    product_name,
    price_usd,
    price_usd * {{ get_exchange_rate('USD', 'JPY') }} AS price_jpy
FROM products

-- 例2: アクティブキャンペーンのフィルター
SELECT *
FROM campaign_results
WHERE campaign_id IN ({{ "'" + "','".join(get_active_campaigns()) + "'" }})
```

**注意**: 外部API呼び出しは**必ず`@lru_cache`でキャッシュ**してください。そうしないと、クエリが実行されるたびにAPI呼び出しが発生します。

---

### 例4: 複雑なロジックのカスタムマクロ

**ファイル**: `/etc/superset/superset_config.py`

```python
from datetime import datetime, timedelta


def get_fiscal_year_range(fiscal_year=None):
    """
    会計年度の開始日と終了日を返す
    会計年度: 4月1日〜翌年3月31日
    """
    if fiscal_year is None:
        # 現在の会計年度を自動判定
        now = datetime.now()
        if now.month >= 4:
            fiscal_year = now.year
        else:
            fiscal_year = now.year - 1

    start_date = datetime(fiscal_year, 4, 1)
    end_date = datetime(fiscal_year + 1, 3, 31)

    return {
        "fy": fiscal_year,
        "start": start_date.strftime("%Y-%m-%d"),
        "end": end_date.strftime("%Y-%m-%d"),
        "start_ts": int(start_date.timestamp()),
        "end_ts": int(end_date.timestamp()),
    }


def get_quarter_range(year, quarter):
    """
    指定された四半期の開始日と終了日を返す
    """
    quarter_months = {
        1: (1, 3),
        2: (4, 6),
        3: (7, 9),
        4: (10, 12),
    }

    if quarter not in quarter_months:
        raise ValueError("Quarter must be 1, 2, 3, or 4")

    start_month, end_month = quarter_months[quarter]
    start_date = datetime(year, start_month, 1)

    # 終了日は月の最終日
    if end_month == 12:
        end_date = datetime(year + 1, 1, 1) - timedelta(days=1)
    else:
        end_date = datetime(year, end_month + 1, 1) - timedelta(days=1)

    return {
        "start": start_date.strftime("%Y-%m-%d"),
        "end": end_date.strftime("%Y-%m-%d"),
    }


JINJA_CONTEXT_ADDONS = {
    "get_fiscal_year_range": get_fiscal_year_range,
    "get_quarter_range": get_quarter_range,
}
```

**SQL Lab での使用例**:

```sql
-- 例1: 会計年度でフィルター
{% set fy = get_fiscal_year_range(2024) %}
SELECT *
FROM sales
WHERE sale_date >= '{{ fy.start }}'
  AND sale_date <= '{{ fy.end }}'

-- 例2: 四半期でフィルター
{% set q1 = get_quarter_range(2024, 1) %}
SELECT *
FROM sales
WHERE sale_date BETWEEN '{{ q1.start }}' AND '{{ q1.end }}'
```

---

### 例5: safe_proxy()を使った安全なマクロ

**ファイル**: `/etc/superset/superset_config.py`

```python
from functools import partial
from superset.jinja_context import safe_proxy


def _get_department_filter(department_name):
    """
    部署名から部署コードを取得する内部関数
    """
    departments = {
        "sales": "DEPT001",
        "engineering": "DEPT002",
        "marketing": "DEPT003",
        "hr": "DEPT004",
    }
    return departments.get(department_name, "UNKNOWN")


# safe_proxy()でラップして安全性を確保
JINJA_CONTEXT_ADDONS = {
    "get_department_filter": partial(safe_proxy, _get_department_filter),
}
```

**SQL Lab での使用例**:

```sql
SELECT *
FROM employees
WHERE department_code = '{{ get_department_filter("sales") }}'
```

**重要**: `safe_proxy()`は戻り値の型をチェックし、安全でない型（クラスインスタンス、lambdaなど）を拒否します。

---

### 例6: データベース接続を使ったマクロ

**ファイル**: `/etc/superset/superset_config.py`

```python
from sqlalchemy import create_engine, text
from functools import lru_cache


@lru_cache(maxsize=128)
def get_user_segment(user_id):
    """
    ユーザーIDからユーザーセグメントを取得
    """
    try:
        # 別のデータベースに接続してセグメント情報を取得
        engine = create_engine("postgresql://user:pass@localhost/user_db")
        with engine.connect() as conn:
            result = conn.execute(
                text("SELECT segment FROM users WHERE user_id = :user_id"),
                {"user_id": user_id}
            )
            row = result.fetchone()
            if row:
                return row[0]
            return "unknown"
    except Exception:
        return "unknown"


JINJA_CONTEXT_ADDONS = {
    "get_user_segment": get_user_segment,
}
```

**SQL Lab での使用例**:

```sql
SELECT *
FROM purchases
WHERE user_segment = '{{ get_user_segment(123) }}'
```

**注意**: データベース接続は**コストが高い**ため、必ず`@lru_cache`でキャッシュしてください。

---

### 例7: 環境変数を使ったマクロ

**ファイル**: `/etc/superset/superset_config.py`

```python
import os


def get_env_variable(var_name, default=""):
    """
    環境変数を取得するマクロ
    """
    return os.getenv(var_name, default)


def get_data_warehouse_schema():
    """
    環境に応じたデータウェアハウススキーマを返す
    """
    env = os.getenv("SUPERSET_ENV", "development")
    schema_mapping = {
        "development": "dev_schema",
        "staging": "stg_schema",
        "production": "prod_schema",
    }
    return schema_mapping.get(env, "dev_schema")


JINJA_CONTEXT_ADDONS = {
    "get_env_variable": get_env_variable,
    "get_data_warehouse_schema": get_data_warehouse_schema,
}
```

**SQL Lab での使用例**:

```sql
SELECT *
FROM {{ get_data_warehouse_schema() }}.sales
WHERE created_at >= '{{ get_env_variable("REPORT_START_DATE", "2024-01-01") }}'
```

---

## セキュリティ上の注意点

### ⚠️ 重要な警告

**ファイル**: `superset/config.py:1198-1202`

> Exposing functionality through JINJA_CONTEXT_ADDONS has security implications as it opens a window for a user to execute untrusted code. It's important to make sure that the objects exposed (as well as objects attached to those objects) are harmless. We recommend only exposing simple/pure functions that return native types.

---

### セキュリティリスク

#### 1. SQLインジェクションのリスク

**危険な例**:

```python
def get_table_name(table):
    # ❌ 危険！ユーザー入力をそのままテーブル名に使用
    return table


JINJA_CONTEXT_ADDONS = {
    "get_table_name": get_table_name,
}
```

**悪用例**:

```sql
SELECT * FROM {{ get_table_name("users; DROP TABLE users; --") }}
-- 結果: SELECT * FROM users; DROP TABLE users; --
```

**安全な実装**:

```python
def get_table_name(table):
    # ✅ 安全：ホワイトリスト方式
    allowed_tables = {"users", "products", "orders"}
    if table not in allowed_tables:
        raise ValueError(f"Table {table} not allowed")
    return table


JINJA_CONTEXT_ADDONS = {
    "get_table_name": get_table_name,
}
```

---

#### 2. 任意コード実行のリスク

**危険な例**:

```python
def execute_python_code(code):
    # ❌ 危険！任意のPythonコードを実行
    return eval(code)


JINJA_CONTEXT_ADDONS = {
    "execute_python_code": execute_python_code,
}
```

**悪用例**:

```sql
SELECT {{ execute_python_code("__import__('os').system('rm -rf /')") }}
```

**対策**: `eval()`、`exec()`は**絶対に使用しない**

---

#### 3. 情報漏洩のリスク

**危険な例**:

```python
def get_database_credentials():
    # ❌ 危険！データベース認証情報を返す
    return {
        "user": "admin",
        "password": "secret123",
    }


JINJA_CONTEXT_ADDONS = {
    "get_credentials": get_database_credentials,
}
```

**悪用例**:

```sql
SELECT '{{ get_credentials().password }}'  -- 結果: 'secret123'
```

**対策**: 機密情報を返す関数は**絶対に公開しない**

---

### セキュリティベストプラクティス

#### ✅ 1. プリミティブ型のみを返す

```python
# ✅ Good
def get_company_id():
    return 123  # int

def get_region_name():
    return "Tokyo"  # str

def get_date_range():
    return {"start": "2024-01-01", "end": "2024-12-31"}  # dict (JSONシリアライズ可能)
```

```python
# ❌ Bad
def get_database_connection():
    return engine.connect()  # Connection object - 安全でない

def get_lambda():
    return lambda x: x * 2  # lambda - 安全でない
```

---

#### ✅ 2. 入力値の検証（ホワイトリスト方式）

```python
def get_department_code(department_name):
    # ホワイトリスト
    allowed_departments = {
        "sales": "DEPT001",
        "engineering": "DEPT002",
        "marketing": "DEPT003",
    }

    # ホワイトリストにない場合はエラー
    if department_name not in allowed_departments:
        raise ValueError(f"Department {department_name} not allowed")

    return allowed_departments[department_name]
```

---

#### ✅ 3. safe_proxy() の使用

```python
from functools import partial
from superset.jinja_context import safe_proxy


def _internal_function():
    # 内部実装
    return "value"


# safe_proxy()でラップ
JINJA_CONTEXT_ADDONS = {
    "my_macro": partial(safe_proxy, _internal_function),
}
```

**safe_proxy()の効果**:
- 戻り値の型をチェック
- 許可されていない型を拒否
- コレクション型をJSON変換でサニタイズ

---

#### ✅ 4. 例外処理の実装

```python
def get_external_data():
    try:
        # 外部API呼び出し
        response = requests.get("https://api.example.com/data", timeout=5)
        return response.json()
    except requests.RequestException as e:
        # エラー時は安全なデフォルト値を返す
        return {"status": "error", "message": "API unavailable"}
    except Exception as e:
        # 予期しないエラー
        return {"status": "error", "message": "Unknown error"}
```

---

#### ✅ 5. レート制限とキャッシュ

```python
from functools import lru_cache
import time


@lru_cache(maxsize=128)
def get_expensive_data(param):
    # 重い処理（外部API、データベースクエリなど）
    time.sleep(1)  # デモ用
    return f"Result for {param}"


# キャッシュされるため、同じparamでの再呼び出しは高速
```

---

### セキュリティチェックリスト

- [ ] プリミティブ型（str, int, float, bool, list, dict）のみを返している
- [ ] `eval()`, `exec()`を使用していない
- [ ] ユーザー入力をホワイトリストで検証している
- [ ] 機密情報（パスワード、API キーなど）を返していない
- [ ] 例外処理を実装している
- [ ] 外部API呼び出しは`@lru_cache`でキャッシュしている
- [ ] タイムアウトを設定している（外部API、データベースクエリ）
- [ ] ログに機密情報を出力していない

---

## ベストプラクティス

### 1. 関数の命名規則

**推奨**:
- 動詞で始める: `get_`, `fetch_`, `calculate_`, `format_`
- 明確で説明的な名前を使用
- 小文字とアンダースコアを使用（snake_case）

**例**:

```python
# ✅ Good
get_company_id()
fetch_user_segment(user_id)
calculate_tax_rate(region)
format_currency(amount)

# ❌ Bad
company()  # 不明確
fetchSeg()  # camelCase（Pythonでは非推奨）
x()  # 短すぎる
getUserSegmentFromDatabaseByUserId()  # 長すぎる
```

---

### 2. ドキュメント化

**推奨**: すべてのカスタムマクロにdocstringを追加

```python
def get_fiscal_year_range(fiscal_year=None):
    """
    会計年度の開始日と終了日を返す

    会計年度は4月1日〜翌年3月31日とする。
    fiscal_yearが指定されない場合、現在の会計年度を自動判定する。

    Args:
        fiscal_year (int, optional): 会計年度（例: 2024）。デフォルトはNone（自動判定）。

    Returns:
        dict: 以下のキーを持つ辞書
            - fy (int): 会計年度
            - start (str): 開始日（YYYY-MM-DD形式）
            - end (str): 終了日（YYYY-MM-DD形式）
            - start_ts (int): 開始日のUNIXタイムスタンプ
            - end_ts (int): 終了日のUNIXタイムスタンプ

    Examples:
        >>> get_fiscal_year_range(2024)
        {'fy': 2024, 'start': '2024-04-01', 'end': '2025-03-31', ...}

        >>> get_fiscal_year_range()  # 現在が2024年5月の場合
        {'fy': 2024, 'start': '2024-04-01', 'end': '2025-03-31', ...}
    """
    # 実装
    ...
```

---

### 3. テスト

**推奨**: カスタムマクロのユニットテストを作成

**ファイル**: `tests/test_custom_macros.py`

```python
import unittest
from superset_config import get_company_id, get_date_range


class TestCustomMacros(unittest.TestCase):

    def test_get_company_id(self):
        """get_company_id()が正しい値を返すことを確認"""
        result = get_company_id()
        self.assertEqual(result, 123)
        self.assertIsInstance(result, int)

    def test_get_date_range(self):
        """get_date_range()が正しい形式を返すことを確認"""
        result = get_date_range(7)
        self.assertIn("start", result)
        self.assertIn("end", result)
        self.assertIsInstance(result["start"], str)
        self.assertIsInstance(result["end"], str)

    def test_get_date_range_default(self):
        """get_date_range()がデフォルト引数で動作することを確認"""
        result = get_date_range()
        self.assertIn("start", result)
        self.assertIn("end", result)


if __name__ == "__main__":
    unittest.main()
```

---

### 4. ログ記録

**推奨**: 重要な処理やエラーをログに記録

```python
import logging

logger = logging.getLogger(__name__)


def get_external_data(param):
    """
    外部APIからデータを取得
    """
    try:
        logger.info(f"Fetching external data for param: {param}")
        response = requests.get(f"https://api.example.com/data?param={param}", timeout=5)
        response.raise_for_status()
        data = response.json()
        logger.info(f"Successfully fetched data: {len(data)} items")
        return data
    except requests.RequestException as e:
        logger.error(f"Failed to fetch external data: {e}")
        return []
    except Exception as e:
        logger.exception(f"Unexpected error in get_external_data: {e}")
        return []
```

---

### 5. 設定の外部化

**推奨**: ハードコードを避け、環境変数や設定ファイルを使用

```python
import os


# ✅ Good: 環境変数から取得
API_BASE_URL = os.getenv("EXTERNAL_API_URL", "https://api.example.com")
API_TIMEOUT = int(os.getenv("API_TIMEOUT", "5"))


def get_external_data():
    response = requests.get(f"{API_BASE_URL}/data", timeout=API_TIMEOUT)
    return response.json()


# ❌ Bad: ハードコード
def get_external_data_bad():
    response = requests.get("https://api.example.com/data", timeout=5)
    return response.json()
```

---

### 6. パフォーマンス最適化

#### キャッシュの活用

```python
from functools import lru_cache


@lru_cache(maxsize=128)
def get_expensive_computation(param):
    """
    重い計算処理（キャッシュ付き）
    """
    # 重い処理
    result = complex_calculation(param)
    return result
```

#### タイムアウトの設定

```python
import requests


def get_api_data():
    # タイムアウトを必ず設定
    response = requests.get("https://api.example.com/data", timeout=5)
    return response.json()
```

---

### 7. エラーハンドリング

**推奨**: 適切なエラーメッセージと安全なデフォルト値

```python
def get_user_segment(user_id):
    try:
        # データベースクエリ
        segment = fetch_from_database(user_id)
        return segment
    except DatabaseConnectionError as e:
        logger.error(f"Database connection failed: {e}")
        return "unknown"  # 安全なデフォルト値
    except ValueError as e:
        logger.warning(f"Invalid user_id: {user_id}")
        return "unknown"
    except Exception as e:
        logger.exception(f"Unexpected error: {e}")
        return "unknown"
```

---

## トラブルシューティング

### 問題1: カスタムマクロが認識されない

**症状**:
```
jinja2.exceptions.UndefinedError: 'my_custom_macro' is undefined
```

**原因と解決策**:

#### 1-1. superset_config.pyが読み込まれていない

**確認方法**:

```bash
# Superset起動時のログを確認
superset run

# 以下のメッセージが表示されるか確認:
# "Loaded your LOCAL configuration at [/path/to/superset_config.py]"
```

**解決策**:

```bash
# 方法1: PYTHONPATHに追加
export PYTHONPATH=/etc/superset:$PYTHONPATH
superset run

# 方法2: 環境変数で指定
export SUPERSET_CONFIG_PATH=/etc/superset/superset_config.py
superset run
```

---

#### 1-2. JINJA_CONTEXT_ADDONSに登録されていない

**確認**:

```python
# superset_config.py
def my_custom_macro():
    return "value"

# ❌ 登録忘れ
# JINJA_CONTEXT_ADDONS = {}  # 空のまま

# ✅ 正しい登録
JINJA_CONTEXT_ADDONS = {
    "my_custom_macro": my_custom_macro,
}
```

---

#### 1-3. Supersetの再起動が必要

**解決策**:

```bash
# Supersetを再起動
superset stop
superset run
```

**注意**: `superset_config.py`の変更は**再起動が必要**です。

---

### 問題2: "Unsafe return type" エラー

**症状**:
```
SupersetTemplateException: Unsafe return type for function my_macro: MyClass
```

**原因**: 許可されていない型（クラスインスタンス、lambda など）を返している

**解決策**:

```python
# ❌ Bad: クラスインスタンスを返す
def get_config():
    return MyConfigClass()  # Unsafe

# ✅ Good: プリミティブ型を返す
def get_config():
    config = MyConfigClass()
    return {
        "value1": config.value1,
        "value2": config.value2,
    }  # dict - Safe
```

---

### 問題3: パフォーマンスが遅い

**症状**: クエリ実行が非常に遅い

**原因**: カスタムマクロで重い処理を実行している（外部API、データベースクエリなど）

**解決策**:

```python
from functools import lru_cache


# ❌ Bad: キャッシュなし
def get_external_data():
    response = requests.get("https://api.example.com/data")
    return response.json()  # 毎回API呼び出し


# ✅ Good: キャッシュ付き
@lru_cache(maxsize=128)
def get_external_data():
    response = requests.get("https://api.example.com/data", timeout=5)
    return response.json()  # 結果がキャッシュされる
```

---

### 問題4: ImportError

**症状**:
```
ImportError: No module named 'my_module'
```

**原因**: superset_config.pyで使用しているモジュールがインストールされていない

**解決策**:

```bash
# 必要なモジュールをインストール
pip install my_module

# または requirements.txt に追加
echo "my_module==1.0.0" >> requirements.txt
pip install -r requirements.txt
```

---

### 問題5: "Infinite recursion detected in template"

**症状**:
```
SupersetTemplateException: Infinite recursion detected in template
```

**原因**: マクロが自分自身を呼び出している（循環参照）

**解決策**:

```python
# ❌ Bad: 循環参照
def macro_a():
    return macro_b()

def macro_b():
    return macro_a()  # 無限ループ


# ✅ Good: 循環参照を避ける
def macro_a():
    return "value_a"

def macro_b():
    return "value_b"
```

---

## 主要ファイル一覧

### Supersetソースコード

| ファイル | 役割 | 重要度 |
|---------|------|--------|
| **`superset/config.py`** | デフォルト設定、superset_config.pyの読み込み | ⭐⭐⭐⭐⭐ |
| **`superset/jinja_context.py`** | Jinjaコンテキスト、テンプレートプロセッサー | ⭐⭐⭐⭐⭐ |
| **`superset/sqllab/query_render.py`** | SQLクエリレンダリング | ⭐⭐⭐⭐☆ |
| `superset/models/sql_lab.py` | SQL Labモデル | ⭐⭐⭐☆☆ |
| `superset/views/core.py` | コアビュー | ⭐⭐⭐☆☆ |

---

### 主要関数・クラス

#### `superset/config.py`

| 行数 | 定義 | 説明 |
|-----|------|------|
| `1193-1203` | `JINJA_CONTEXT_ADDONS: dict[str, Callable[..., Any]]` | カスタムマクロを登録する辞書 |
| `1211` | `CUSTOM_TEMPLATE_PROCESSORS: dict[str, type[BaseTemplateProcessor]]` | カスタムテンプレートプロセッサー |
| `1933-1951` | 環境変数からの設定読み込み | `SUPERSET_CONFIG_PATH` 経由 |
| `1952-1964` | PYTHONPATHからの設定読み込み | `from superset_config import *` |

---

#### `superset/jinja_context.py`

| 行数 | 定義 | 説明 |
|-----|------|------|
| `76-78` | `context_addons() -> dict[str, Any]` | `JINJA_CONTEXT_ADDONS`を取得 |
| `464-486` | `safe_proxy(func, *args, **kwargs) -> Any` | 関数の戻り値を安全にラップ |
| `489-510` | `validate_context_types(context) -> dict` | コンテキストの型を検証 |
| `561-593` | `BaseTemplateProcessor.__init__()` | テンプレートプロセッサーの初期化 |
| `597-599` | `BaseTemplateProcessor.set_context()` | コンテキストを設定（`context_addons()`呼び出し） |
| `607-623` | `BaseTemplateProcessor.process_template()` | Jinjaテンプレートをレンダリング |
| `626-691` | `JinjaTemplateProcessor.set_context()` | 組み込みマクロを追加 |
| `808-816` | `get_template_processors() -> dict` | テンプレートプロセッサーの取得 |
| `819-831` | `get_template_processor() -> BaseTemplateProcessor` | テンプレートプロセッサーのファクトリー |

---

#### `superset/sqllab/query_render.py`

| 行数 | 定義 | 説明 |
|-----|------|------|
| `45-68` | `SqlQueryRenderImpl.render()` | SQLクエリをレンダリング |
| `53-63` | テンプレートプロセッサーの生成と実行 | `get_template_processor()` → `process_template()` |

---

## 結論

### ✅ 実現可能性の最終確認

**質問**: Supersetでカスタムマクロを作成する際、ソースコードを編集せず、`superset_config.py`にカスタムマクロの定義を実装することで、Superset側で取り込むことができるのか？

**答え**: **Yes - 100%実現可能です！**

---

### 仕組みのまとめ

1. **設定ファイルの読み込み**:
   - Supersetは起動時に`superset_config.py`を自動的に読み込む
   - 環境変数`SUPERSET_CONFIG_PATH`またはPYTHONPATHから検索

2. **カスタムマクロの登録**:
   - `superset_config.py`で`JINJA_CONTEXT_ADDONS`辞書にカスタム関数を追加
   - 関数名がキー、関数オブジェクトが値

3. **コンテキストへのマージ**:
   - `context_addons()`関数が`JINJA_CONTEXT_ADDONS`を取得
   - `BaseTemplateProcessor.set_context()`で自動的にコンテキストにマージ

4. **クエリ実行時の使用**:
   - SQL Labでクエリを実行
   - `get_template_processor()`でテンプレートプロセッサーを生成
   - `process_template()`でJinjaテンプレートをレンダリング
   - カスタムマクロが呼び出される

---

### メリット

| メリット | 説明 |
|---------|------|
| ✅ **ソースコード編集不要** | Supersetのソースコードを一切変更しない |
| ✅ **アップグレード安全** | Supersetのアップグレード時にカスタマイズが失われない |
| ✅ **メンテナンス容易** | 設定ファイル1つで管理 |
| ✅ **デプロイ簡単** | `superset_config.py`をコピーするだけ |
| ✅ **公式サポート** | Supersetが公式にサポートする方法 |
| ✅ **柔軟性** | 任意のPythonコードを実行可能 |

---

### 注意点

| 注意点 | 対策 |
|--------|------|
| ⚠️ **セキュリティリスク** | プリミティブ型のみ返す、入力検証、safe_proxy()使用 |
| ⚠️ **パフォーマンス** | `@lru_cache`でキャッシュ、タイムアウト設定 |
| ⚠️ **再起動必要** | 設定変更後は必ずSuperset再起動 |
| ⚠️ **エラーハンドリング** | 適切な例外処理と安全なデフォルト値 |

---

### 推奨される使用例

- ✅ 会社ID、部署コードなどの組織固有の値を返す
- ✅ 会計年度、四半期などの複雑な日付計算
- ✅ 外部APIからマスターデータを取得（キャッシュ付き）
- ✅ 環境変数から設定値を取得
- ✅ 複雑なフィルター条件の生成

---

### 避けるべき使用例

- ❌ 任意のPythonコード実行（`eval()`, `exec()`）
- ❌ データベース認証情報の返却
- ❌ ファイルシステムへのアクセス
- ❌ クラスインスタンスの返却
- ❌ キャッシュなしの外部API呼び出し

---

## 参考リンク

### Superset公式ドキュメント

- [Jinja Templating](https://superset.apache.org/docs/installation/sql-templating)
- [Configuration](https://superset.apache.org/docs/installation/configuring-superset)

### ソースコード

- [superset/config.py](https://github.com/apache/superset/blob/master/superset/config.py)
- [superset/jinja_context.py](https://github.com/apache/superset/blob/master/superset/jinja_context.py)
- [superset/sqllab/query_render.py](https://github.com/apache/superset/blob/master/superset/sqllab/query_render.py)

---

**作成者**: Claude Code
**作成日**: 2026-04-06
**調査対象**: Apache Superset カスタムJinjaマクロ
**Supersetバージョン**: 4.x以降対応
