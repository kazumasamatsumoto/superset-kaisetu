# Supersetクロスフィルターとfilter_values()完全ガイド

## 📋 目次

1. [概要](#概要)
2. [クロスフィルターとは](#クロスフィルターとは)
3. [全体アーキテクチャ](#全体アーキテクチャ)
4. [データフロー（図式付き）](#データフロー図式付き)
5. [filter_values()の仕組み](#filter_valuesの仕組み)
6. [extra_form_dataとadhoc_filtersのマージ処理](#extra_form_dataとadhoc_filtersのマージ処理)
7. [実装例](#実装例)
8. [主要ファイルと関数](#主要ファイルと関数)
9. [トラブルシューティング](#トラブルシューティング)
10. [結論](#結論)

---

## 概要

### 調査の目的

Supersetにおいて、**クロスフィルター**（チャート間フィルター連携）で値を受け取る仕組みを解説します。特に、`filter_values()`マクロがどのようにクロスフィルターの値を受け取るかを、図式を含めて詳細に説明します。

### クロスフィルターとfilter_values()の関係

```
┌────────────────────────────────────────────────────────┐
│ ユーザーがChart Aでフィルター選択                          │
│ （例: "Tokyo"を選択）                                    │
└────────────────────────────────────────────────────────┘
                      ↓
       （クロスフィルター機能により）
                      ↓
┌────────────────────────────────────────────────────────┐
│ Chart Bに extra_form_data としてフィルター情報が渡される   │
│ extra_form_data = {                                    │
│   "adhoc_filters": [                                   │
│     {                                                  │
│       "subject": "city",                               │
│       "operator": "IN",                                │
│       "comparator": ["Tokyo"]                          │
│     }                                                  │
│   ]                                                    │
│ }                                                      │
└────────────────────────────────────────────────────────┘
                      ↓
       （merge_extra_form_data()により）
                      ↓
┌────────────────────────────────────────────────────────┐
│ Chart Bの form_data.adhoc_filters にマージされる         │
│ form_data = {                                          │
│   "adhoc_filters": [                                   │
│     { "subject": "city", "comparator": ["Tokyo"],      │
│       "isExtra": true }                                │
│   ]                                                    │
│ }                                                      │
└────────────────────────────────────────────────────────┘
                      ↓
    （Chart BのSQLクエリ内で filter_values()を使用）
                      ↓
┌────────────────────────────────────────────────────────┐
│ {{ filter_values('city') }} が呼ばれる                  │
│ → ["Tokyo"] が返される                                 │
│                                                        │
│ SQL:                                                   │
│ SELECT * FROM sales                                    │
│ WHERE city IN ('Tokyo')                                │
└────────────────────────────────────────────────────────┘
```

---

## クロスフィルターとは

### 定義

**クロスフィルター**とは、ダッシュボード上の**あるチャートで選択したフィルター値を、他のチャートに自動的に適用する機能**です。

### 用途

- インタラクティブなダッシュボード作成
- ドリルダウン分析
- 複数チャート間の連携フィルタリング

### Supersetでのサポート

Supersetは以下の2つの方法でクロスフィルターをサポートしています：

1. **ネイティブフィルター** - Dashboard Native Filters（UI上で設定）
2. **チャート間フィルター** - Chart Emits Filter Events（チャートクリックでフィルター発火）

---

## 全体アーキテクチャ

### コンポーネント構成

```
┌─────────────────────────────────────────────────────────────┐
│ Frontend (React/Redux)                                      │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  Dashboard                                                  │
│  ├─ Chart A (Filter Emitter)                                │
│  │  └─ ユーザーが値を選択 → filterStateを更新               │
│  │                                                          │
│  └─ Chart B (Filter Receiver)                               │
│     └─ filterStateを監視 → extra_form_dataを生成            │
│                                                             │
└─────────────────────────────────────────────────────────────┘
                         ↓ (HTTP Request)
┌─────────────────────────────────────────────────────────────┐
│ Backend (Flask/Python)                                      │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  Chart API Endpoint                                         │
│  ├─ views/utils.py: get_form_data()                         │
│  │  └─ request.jsonから form_data と extra_form_data を取得 │
│  │                                                          │
│  ├─ utils/core.py: merge_extra_form_data()                  │
│  │  └─ extra_form_dataを form_dataにマージ                  │
│  │                                                          │
│  └─ jinja_context.py: filter_values()                       │
│     └─ form_data.adhoc_filtersから値を抽出                  │
│                                                             │
└─────────────────────────────────────────────────────────────┘
                         ↓ (SQL Execution)
┌─────────────────────────────────────────────────────────────┐
│ Database                                                    │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  SELECT * FROM sales                                        │
│  WHERE city IN ('Tokyo')                                    │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

---

## データフロー（図式付き）

### ステップ1: ユーザーがChart Aで値を選択

```
┌───────────────────────────────────────────────────────┐
│ Dashboard (Frontend)                                  │
├───────────────────────────────────────────────────────┤
│                                                       │
│  Chart A: "Sales by City" (Filter Emitter)           │
│  ┌─────────────────────────────────────────────────┐ │
│  │ 棒グラフ:                                        │ │
│  │   Tokyo    ████████████ (100)                    │ │
│  │   Osaka    ██████ (60)        ← クリック！       │ │
│  │   Kyoto    ████ (40)                             │ │
│  └─────────────────────────────────────────────────┘ │
│                                                       │
└───────────────────────────────────────────────────────┘
                       ↓
        （Redux storeのfilterStateが更新）
                       ↓
┌───────────────────────────────────────────────────────┐
│ Redux Store                                           │
├───────────────────────────────────────────────────────┤
│ filterState: {                                        │
│   "native-filter-1": {                                │
│     "filterKey": "city",                              │
│     "values": ["Osaka"]                               │
│   }                                                   │
│ }                                                     │
└───────────────────────────────────────────────────────┘
```

---

### ステップ2: Chart BのReact ComponentがfilterStateを監視

```
┌───────────────────────────────────────────────────────┐
│ Chart B: "Product Sales" (React Component)            │
├───────────────────────────────────────────────────────┤
│                                                       │
│ useSelector(state => state.filterState)               │
│   ↓                                                   │
│ filterStateが変更された！                              │
│   ↓                                                   │
│ extra_form_dataを生成:                                │
│ {                                                     │
│   "adhoc_filters": [                                  │
│     {                                                 │
│       "clause": "WHERE",                              │
│       "subject": "city",                              │
│       "operator": "IN",                               │
│       "comparator": ["Osaka"],                        │
│       "expressionType": "SIMPLE"                      │
│     }                                                 │
│   ]                                                   │
│ }                                                     │
│   ↓                                                   │
│ Chart APIにリクエスト送信                              │
│                                                       │
└───────────────────────────────────────────────────────┘
```

---

### ステップ3: Backend - get_form_data()でformDataを取得

**ファイル**: `views/utils.py:146-196`

```python
def get_form_data(...) -> tuple[dict[str, Any], Optional[Slice]]:
    form_data: dict[str, Any] = initial_form_data or {}

    if has_request_context():
        json_data = request.get_json(cache=True) if request.is_json else {}

        # Chart Data API requests are JSON
        first_query = (
            json_data["queries"][0]
            if "queries" in json_data and json_data["queries"]
            else None
        )

        if first_query:
            form_data.update(first_query)  # ← extra_form_data含む
        # ...

    return form_data, slc
```

**リクエストJSON例**:

```json
{
  "queries": [
    {
      "datasource": "1__table",
      "viz_type": "table",
      "adhoc_filters": [],  // Chart B自身のフィルター（空）
      "extra_form_data": {  // ← クロスフィルターからの値
        "adhoc_filters": [
          {
            "clause": "WHERE",
            "subject": "city",
            "operator": "IN",
            "comparator": ["Osaka"],
            "expressionType": "SIMPLE"
          }
        ]
      }
    }
  ]
}
```

**取得後のform_data**:

```python
form_data = {
    "datasource": "1__table",
    "viz_type": "table",
    "adhoc_filters": [],  # まだ空
    "extra_form_data": {
        "adhoc_filters": [
            {
                "clause": "WHERE",
                "subject": "city",
                "operator": "IN",
                "comparator": ["Osaka"],
                "expressionType": "SIMPLE"
            }
        ]
    }
}
```

---

### ステップ4: Backend - merge_extra_form_data()でマージ

**ファイル**: `utils/core.py:890-942`

```python
def merge_extra_form_data(form_data: dict[str, Any]) -> None:
    """
    Merge extra form data (appends and overrides) into the main payload
    """
    filter_keys = ["filters", "adhoc_filters"]
    extra_form_data = form_data.pop("extra_form_data", {})  # ① extra_form_dataを取り出す

    # ...

    adhoc_filters: list[AdhocFilterClause] = form_data.get("adhoc_filters", [])
    form_data["adhoc_filters"] = adhoc_filters

    append_adhoc_filters: list[AdhocFilterClause] = extra_form_data.get(
        "adhoc_filters", []
    )  # ② extra_form_dataからadhoc_filtersを取得

    adhoc_filters.extend(
        {"isExtra": True, **adhoc_filter}  # ③ isExtra: trueを追加
        for adhoc_filter in append_adhoc_filters
    )  # ④ form_data.adhoc_filtersに追加（extend）
```

**マージ処理の詳細図**:

```
Before merge_extra_form_data():
┌─────────────────────────────────────────────────────┐
│ form_data = {                                       │
│   "adhoc_filters": [],  // Chart B自身のフィルター   │
│   "extra_form_data": {  // クロスフィルター          │
│     "adhoc_filters": [                              │
│       {subject: "city", comparator: ["Osaka"]}      │
│     ]                                               │
│   }                                                 │
│ }                                                   │
└─────────────────────────────────────────────────────┘
                      ↓
          merge_extra_form_data()
                      ↓
After merge_extra_form_data():
┌─────────────────────────────────────────────────────┐
│ form_data = {                                       │
│   "adhoc_filters": [                                │
│     {                                               │
│       "subject": "city",                            │
│       "operator": "IN",                             │
│       "comparator": ["Osaka"],                      │
│       "expressionType": "SIMPLE",                   │
│       "isExtra": true  // ← 追加されたマーカー       │
│     }                                               │
│   ],                                                │
│   // extra_form_data は削除される（pop）            │
│ }                                                   │
└─────────────────────────────────────────────────────┘
```

**重要ポイント**:
- `extra_form_data.adhoc_filters`は`form_data.adhoc_filters`に**extend**される（上書きではない）
- `isExtra: true`マーカーが追加される
- `extra_form_data`自体は`form_data`から削除される（`pop()`）

---

### ステップ5: Backend - filter_values()でフィルター値を取得

**ファイル**: `jinja_context.py:240-279`

```python
def filter_values(
    self, column: str, default: Optional[str] = None, remove_filter: bool = False
) -> list[Any]:
    """Gets a values for a particular filter as a list

    Usage example::
        SELECT action, count(*) as times
        FROM logs
        WHERE
            action in ({{ "'" + "','".join(filter_values('action_type')) + "'" }})
        GROUP BY action

    :param column: column/filter name to lookup
    :param default: default value to return if there's no matching columns
    :param remove_filter: When set to true, mark the filter as processed
    :return: returns a list of filter values
    """
    return_val: list[Any] = []
    filters = self.get_filters(column, remove_filter)  # ① get_filters()を呼び出し

    for flt in filters:
        val = flt.get("val")
        if isinstance(val, list):
            return_val.extend(val)  # リストの場合はextend
        elif val:
            return_val.append(val)  # 単一値の場合はappend

    if (not return_val) and default:
        return_val = [default]  # デフォルト値を返す

    return return_val
```

---

### ステップ6: Backend - get_filters()でadhoc_filtersから検索

**ファイル**: `jinja_context.py:281-384`

```python
def get_filters(self, column: str, remove_filter: bool = False) -> list[Filter]:
    """Get the filters applied to the given column"""
    from superset.views.utils import get_form_data

    form_data, _ = get_form_data()  # ① form_dataを取得
    convert_legacy_filters_into_adhoc(form_data)
    merge_extra_filters(form_data)  # ② 再度マージ（念のため）

    filters: list[Filter] = []

    for flt in form_data.get("adhoc_filters", []):  # ③ adhoc_filtersをループ
        val: Union[Any, list[Any]] = flt.get("comparator")
        op: str = flt["operator"].upper() if flt.get("operator") else None

        if (
            flt.get("expressionType") == "SIMPLE"
            and flt.get("clause") == "WHERE"
            and flt.get("subject") == column  # ④ 指定されたcolumnと一致するかチェック
            and (val or op in (FilterOperator.IS_NULL.value, FilterOperator.IS_NOT_NULL.value))
        ):
            if remove_filter:
                if column not in self.removed_filters:
                    self.removed_filters.append(column)
            if column not in self.applied_filters:
                self.applied_filters.append(column)

            if op in (FilterOperator.IN.value, FilterOperator.NOT_IN.value) and not isinstance(val, list):
                val = [val]

            filters.append({"op": op, "col": column, "val": val})  # ⑤ フィルターを追加

    return filters
```

**検索処理の詳細図**:

```
form_data.adhoc_filters = [
  {
    "subject": "city",        ← filter_values('city')で検索
    "operator": "IN",
    "comparator": ["Osaka"],  ← これを取得したい
    "expressionType": "SIMPLE",
    "clause": "WHERE",
    "isExtra": true
  },
  {
    "subject": "category",    ← 別のフィルター（無視）
    "operator": "==",
    "comparator": "Electronics",
    "expressionType": "SIMPLE",
    "clause": "WHERE"
  }
]
         ↓
get_filters('city')が呼ばれる
         ↓
adhoc_filtersをループして、subject == 'city'を探す
         ↓
見つかった！
{
  "subject": "city",
  "operator": "IN",
  "comparator": ["Osaka"]
}
         ↓
return [{"op": "IN", "col": "city", "val": ["Osaka"]}]
         ↓
filter_values()に戻る
         ↓
val = ["Osaka"]をextract
         ↓
return ["Osaka"]
```

---

### ステップ7: SQL Labでfilter_values()を使用

**SQL クエリ例**:

```sql
SELECT product_name, SUM(amount) as total_sales
FROM sales
WHERE city IN ({{ "'" + "','".join(filter_values('city')) + "'" }})
GROUP BY product_name
ORDER BY total_sales DESC
```

**Jinjaテンプレート処理**:

```
1. filter_values('city') が呼ばれる
   ↓
2. ["Osaka"] が返される
   ↓
3. "','".join(["Osaka"]) → "Osaka"
   ↓
4. "'" + "Osaka" + "'" → "'Osaka'"
   ↓
5. レンダリング後のSQL:
   SELECT product_name, SUM(amount) as total_sales
   FROM sales
   WHERE city IN ('Osaka')
   GROUP BY product_name
   ORDER BY total_sales DESC
```

---

### ステップ8: データベースでSQL実行 → 結果を返す

```
┌─────────────────────────────────────────────────────┐
│ Database                                            │
├─────────────────────────────────────────────────────┤
│ SELECT product_name, SUM(amount) as total_sales    │
│ FROM sales                                          │
│ WHERE city IN ('Osaka')  ← フィルターが適用される   │
│ GROUP BY product_name                               │
│ ORDER BY total_sales DESC                           │
│                                                     │
│ 結果:                                               │
│ ┌──────────────┬─────────────┐                     │
│ │ product_name │ total_sales │                     │
│ ├──────────────┼─────────────┤                     │
│ │ Laptop       │ 50000       │                     │
│ │ Mouse        │ 3000        │                     │
│ │ Keyboard     │ 2500        │                     │
│ └──────────────┴─────────────┘                     │
└─────────────────────────────────────────────────────┘
                  ↓
            (JSON Response)
                  ↓
┌─────────────────────────────────────────────────────┐
│ Chart B (Frontend)                                  │
├─────────────────────────────────────────────────────┤
│ データを受信してチャートを更新                        │
│                                                     │
│ Table:                                              │
│ ┌──────────────┬─────────────┐                     │
│ │ Product      │ Sales       │                     │
│ ├──────────────┼─────────────┤                     │
│ │ Laptop       │ $50,000     │                     │
│ │ Mouse        │ $3,000      │                     │
│ │ Keyboard     │ $2,500      │                     │
│ └──────────────┴─────────────┘                     │
│                                                     │
│ ※ Osaka の商品のみ表示されている！                  │
└─────────────────────────────────────────────────────┘
```

---

## filter_values()の仕組み

### 関数シグネチャ

```python
def filter_values(
    self,
    column: str,
    default: Optional[str] = None,
    remove_filter: bool = False
) -> list[Any]:
```

### パラメータ

| パラメータ | 型 | 説明 | デフォルト |
|-----------|------|------|-----------|
| `column` | `str` | フィルター名（カラム名） | - (必須) |
| `default` | `Optional[str]` | フィルターがない場合のデフォルト値 | `None` |
| `remove_filter` | `bool` | `True`の場合、フィルターを「処理済み」としてマーク（外側のクエリから削除） | `False` |

### 戻り値

- `list[Any]`: フィルター値のリスト
- フィルターがない場合は空リスト `[]`（`default`が指定されていない場合）
- `default`が指定されている場合は `[default]`

---

### 内部処理フロー

```
filter_values('city', default='Tokyo', remove_filter=False)
         ↓
┌─────────────────────────────────────────────────────┐
│ 1. get_filters('city', remove_filter=False)を呼び出し │
└─────────────────────────────────────────────────────┘
         ↓
┌─────────────────────────────────────────────────────┐
│ 2. get_form_data()で現在のform_dataを取得           │
│    form_data = {                                    │
│      "adhoc_filters": [                             │
│        {"subject": "city", "comparator": ["Osaka"]} │
│      ]                                              │
│    }                                                │
└─────────────────────────────────────────────────────┘
         ↓
┌─────────────────────────────────────────────────────┐
│ 3. adhoc_filtersをループして、                       │
│    subject == 'city'のフィルターを検索               │
│         ↓                                           │
│    見つかった！                                      │
│    {"subject": "city", "comparator": ["Osaka"]}     │
└─────────────────────────────────────────────────────┘
         ↓
┌─────────────────────────────────────────────────────┐
│ 4. Filter形式に変換                                 │
│    {"op": "IN", "col": "city", "val": ["Osaka"]}    │
│         ↓                                           │
│    return [{"op": "IN", "col": "city",              │
│             "val": ["Osaka"]}]                      │
└─────────────────────────────────────────────────────┘
         ↓
┌─────────────────────────────────────────────────────┐
│ 5. filter_values()に戻る                            │
│    filters = [{"op": "IN", "col": "city",           │
│                "val": ["Osaka"]}]                   │
│         ↓                                           │
│    for flt in filters:                              │
│        val = flt.get("val")  # ["Osaka"]            │
│        if isinstance(val, list):                    │
│            return_val.extend(val)                   │
│         ↓                                           │
│    return_val = ["Osaka"]                           │
└─────────────────────────────────────────────────────┘
         ↓
┌─────────────────────────────────────────────────────┐
│ 6. return ["Osaka"]                                 │
└─────────────────────────────────────────────────────┘
```

---

### remove_filterパラメータの動作

**`remove_filter=False`（デフォルト）**:

```python
{{ filter_values('city') }}  # ['Osaka']
```

- フィルターは`form_data.adhoc_filters`に残る
- 外側のクエリでも適用される

**`remove_filter=True`**:

```python
{{ filter_values('city', remove_filter=True) }}  # ['Osaka']
```

- フィルター値は取得できる
- `self.removed_filters`に`'city'`が追加される
- **外側のクエリ**（Supersetが自動生成するWHERE句）からこのフィルターが削除される

**使用例**:

```sql
-- remove_filter=Trueを使用して、内側のクエリでのみフィルターを適用
SELECT product_name, SUM(amount) as total_sales
FROM (
    SELECT *
    FROM sales
    WHERE city IN ({{ "'" + "','".join(filter_values('city', remove_filter=True)) + "'" }})
    -- ↑ 内側のクエリで city フィルターを適用
) AS subquery
GROUP BY product_name
-- ↑ 外側のクエリでは city フィルターが適用されない（remove_filter=Trueのため）
```

---

## extra_form_dataとadhoc_filtersのマージ処理

### EXTRA_FORM_DATA_APPEND_KEYS

**ファイル**: `constants.py:178-185`

```python
EXTRA_FORM_DATA_APPEND_KEYS = {
    "adhoc_filters",      # ← クロスフィルターで主に使用
    "filters",
    "interactive_groupby",
    "interactive_highlight",
    "interactive_drilldown",
    "custom_form_data",
}
```

**意味**: これらのキーは`extra_form_data`から`form_data`に**追加**（append）される

---

### EXTRA_FORM_DATA_OVERRIDE_REGULAR_MAPPINGS

**ファイル**: `constants.py:187-193`

```python
EXTRA_FORM_DATA_OVERRIDE_REGULAR_MAPPINGS = {
    "granularity_sqla": "granularity",
    "time_column": "time_column",
    "time_grain": "time_grain",
    "time_range": "time_range",
    "time_grain_sqla": "time_grain_sqla",
}
```

**意味**: これらのキーは`extra_form_data`から`form_data`に**上書き**（override）される

---

### マージ処理の詳細

**ファイル**: `utils/core.py:890-942`

```python
def merge_extra_form_data(form_data: dict[str, Any]) -> None:
    extra_form_data = form_data.pop("extra_form_data", {})

    # 1. APPEND処理（adhoc_filtersなど）
    append_adhoc_filters = extra_form_data.get("adhoc_filters", [])
    adhoc_filters.extend(
        {"isExtra": True, **adhoc_filter}
        for adhoc_filter in append_adhoc_filters
    )

    # 2. OVERRIDE処理（time_rangeなど）
    for src_key, target_key in EXTRA_FORM_DATA_OVERRIDE_REGULAR_MAPPINGS.items():
        value = extra_form_data.get(src_key)
        if value is not None:
            form_data[target_key] = value  # 上書き
```

---

### マージ処理の具体例

#### 例1: adhoc_filtersのAPPEND

**Before**:

```python
form_data = {
    "adhoc_filters": [
        {"subject": "category", "comparator": "Electronics"}  # Chart自身のフィルター
    ],
    "extra_form_data": {
        "adhoc_filters": [
            {"subject": "city", "comparator": ["Osaka"]}  # クロスフィルター
        ]
    }
}
```

**After**:

```python
form_data = {
    "adhoc_filters": [
        {"subject": "category", "comparator": "Electronics"},  # 元のフィルター
        {"subject": "city", "comparator": ["Osaka"], "isExtra": True}  # 追加されたフィルター
    ]
}
```

---

#### 例2: time_rangeのOVERRIDE

**Before**:

```python
form_data = {
    "time_range": "Last 30 days",  # Chart自身の時間範囲
    "extra_form_data": {
        "time_range": "Last 7 days"  # クロスフィルターからの時間範囲
    }
}
```

**After**:

```python
form_data = {
    "time_range": "Last 7 days"  # 上書きされた
}
```

---

### isExtraマーカーの役割

```python
{"isExtra": True, **adhoc_filter}
```

**目的**:
- クロスフィルターから追加されたフィルターであることを識別
- デバッグ時に便利
- UIで表示する際に「外部フィルター」として識別可能

---

## 実装例

### 例1: 基本的なfilter_values()の使用

**シナリオ**: ダッシュボードで都市を選択すると、その都市の商品売上のみを表示

**Chart A**: "Sales by City" (Filter Emitter)

- 設定: "Emit cross filter" を有効化
- カラム: `city`

**Chart B**: "Product Sales" (Filter Receiver)

**SQLクエリ**:

```sql
SELECT
    product_name,
    SUM(amount) as total_sales,
    COUNT(*) as num_orders
FROM sales
WHERE city IN ({{ "'" + "','".join(filter_values('city', default='Tokyo')) + "'" }})
GROUP BY product_name
ORDER BY total_sales DESC
LIMIT 10
```

**動作**:
1. ユーザーがChart Aで"Osaka"をクリック
2. Chart Bの`filter_values('city')`が`['Osaka']`を返す
3. SQL: `WHERE city IN ('Osaka')`
4. Osakaの商品売上のみが表示される

---

### 例2: 複数値のフィルター

**シナリオ**: 複数の都市を選択してフィルター

**SQLクエリ**:

```sql
{% set cities = filter_values('city', default='Tokyo') %}
SELECT
    city,
    product_name,
    SUM(amount) as total_sales
FROM sales
WHERE city IN ({{ "'" + "','".join(cities) + "'" }})
GROUP BY city, product_name
ORDER BY city, total_sales DESC
```

**クロスフィルターで選択**: ["Tokyo", "Osaka", "Kyoto"]

**レンダリング後のSQL**:

```sql
SELECT
    city,
    product_name,
    SUM(amount) as total_sales
FROM sales
WHERE city IN ('Tokyo','Osaka','Kyoto')
GROUP BY city, product_name
ORDER BY city, total_sales DESC
```

---

### 例3: デフォルト値の使用

**シナリオ**: フィルターが選択されていない場合、デフォルト値を使用

**SQLクエリ**:

```sql
SELECT *
FROM sales
WHERE city IN ({{ "'" + "','".join(filter_values('city', default='Tokyo')) + "'" }})
```

**動作**:
- クロスフィルターで都市が選択されていない場合
- `filter_values('city', default='Tokyo')`が`['Tokyo']`を返す
- SQL: `WHERE city IN ('Tokyo')`

---

### 例4: remove_filterを使った内側クエリのフィルター

**シナリオ**: サブクエリ内でフィルターを適用し、外側のクエリでは適用しない

**SQLクエリ**:

```sql
WITH filtered_sales AS (
    SELECT *
    FROM sales
    WHERE city IN ({{ "'" + "','".join(filter_values('city', remove_filter=True)) + "'" }})
)
SELECT
    product_name,
    SUM(amount) as total_sales
FROM filtered_sales
GROUP BY product_name
-- 外側のクエリでは city フィルターが適用されない
```

**`remove_filter=True`の効果**:
- CTEまたはサブクエリ内でフィルターを適用
- Supersetが自動生成する外側のWHERE句にはこのフィルターが含まれない

---

### 例5: 条件付きフィルター適用

**シナリオ**: フィルターが選択されている場合のみWHERE句を適用

**SQLクエリ**:

```sql
SELECT product_name, SUM(amount) as total_sales
FROM sales
WHERE 1=1
{% if filter_values('city') %}
    AND city IN ({{ "'" + "','".join(filter_values('city')) + "'" }})
{% endif %}
{% if filter_values('category') %}
    AND category IN ({{ "'" + "','".join(filter_values('category')) + "'" }})
{% endif %}
GROUP BY product_name
```

**動作**:
- `city`フィルターが選択されている場合のみ、`AND city IN (...)`が追加される
- `category`フィルターが選択されている場合のみ、`AND category IN (...)`が追加される

---

### 例6: get_filters()を使った詳細なフィルター情報取得

**シナリオ**: フィルターのオペレーターも取得したい

**SQLクエリ**:

```sql
{% set city_filters = get_filters('city') %}
SELECT *
FROM sales
WHERE 1=1
{% for filter in city_filters %}
    {% if filter.op == 'IN' %}
        AND city IN ({{ "'" + "','".join(filter.val) + "'" }})
    {% elif filter.op == '==' %}
        AND city = '{{ filter.val }}'
    {% elif filter.op == 'IS NULL' %}
        AND city IS NULL
    {% endif %}
{% endfor %}
```

**`get_filters('city')`の戻り値例**:

```python
[
    {"op": "IN", "col": "city", "val": ["Tokyo", "Osaka"]}
]
```

---

### 例7: 複数のfilter_values()の組み合わせ

**SQLクエリ**:

```sql
SELECT
    city,
    category,
    product_name,
    SUM(amount) as total_sales
FROM sales
WHERE
    city IN ({{ "'" + "','".join(filter_values('city', default='Tokyo')) + "'" }})
    AND category IN ({{ "'" + "','".join(filter_values('category', default='Electronics')) + "'" }})
    AND product_type IN ({{ "'" + "','".join(filter_values('product_type', default='Laptop')) + "'" }})
GROUP BY city, category, product_name
```

**複数のクロスフィルター**:
- City Filter → `['Osaka']`
- Category Filter → `['Electronics', 'Books']`
- Product Type Filter → フィルターなし（デフォルト値使用）

**レンダリング後のSQL**:

```sql
SELECT
    city,
    category,
    product_name,
    SUM(amount) as total_sales
FROM sales
WHERE
    city IN ('Osaka')
    AND category IN ('Electronics','Books')
    AND product_type IN ('Laptop')
GROUP BY city, category, product_name
```

---

## 主要ファイルと関数

### ファイル一覧

| ファイル | 役割 | 重要度 |
|---------|------|--------|
| `superset/jinja_context.py` | `filter_values()`, `get_filters()`の実装 | ⭐⭐⭐⭐⭐ |
| `superset/utils/core.py` | `merge_extra_form_data()`, `merge_extra_filters()`の実装 | ⭐⭐⭐⭐⭐ |
| `superset/views/utils.py` | `get_form_data()`の実装 | ⭐⭐⭐⭐☆ |
| `superset/constants.py` | `EXTRA_FORM_DATA_*`定数の定義 | ⭐⭐⭐⭐☆ |

---

### 主要関数

#### 1. filter_values()

**ファイル**: `jinja_context.py:240-279`

```python
def filter_values(
    self, column: str, default: Optional[str] = None, remove_filter: bool = False
) -> list[Any]:
```

**役割**: 指定されたカラムのフィルター値をリストで返す

**使用例**:

```python
filter_values('city')  # ['Tokyo', 'Osaka']
filter_values('city', default='Tokyo')  # ['Tokyo'] or default
filter_values('city', remove_filter=True)  # 外側クエリから削除
```

---

#### 2. get_filters()

**ファイル**: `jinja_context.py:281-384`

```python
def get_filters(self, column: str, remove_filter: bool = False) -> list[Filter]:
```

**役割**: 指定されたカラムのフィルター情報（オペレーター含む）を返す

**戻り値**:

```python
[{"op": "IN", "col": "city", "val": ["Tokyo", "Osaka"]}]
```

---

#### 3. merge_extra_form_data()

**ファイル**: `utils/core.py:890-942`

```python
def merge_extra_form_data(form_data: dict[str, Any]) -> None:
```

**役割**: `extra_form_data`を`form_data`にマージ

**処理**:
- `extra_form_data.adhoc_filters`を`form_data.adhoc_filters`に**extend**
- `isExtra: True`マーカーを追加
- `extra_form_data`を`form_data`から削除（pop）

---

#### 4. get_form_data()

**ファイル**: `views/utils.py:146-196`

```python
def get_form_data(
    slice_id: Optional[int] = None,
    use_slice_data: bool = False,
    initial_form_data: Optional[dict[str, Any]] = None,
) -> tuple[dict[str, Any], Optional[Slice]]:
```

**役割**: リクエストから`form_data`を取得

**データソース**:
1. `request.get_json()` - Chart Data API requests
2. `request.form.get("form_data")` - Form data
3. `request.args.get("form_data")` - Query params
4. `g.form_data` - Flask globals (cache warmup)

---

## トラブルシューティング

### 問題1: filter_values()が空のリストを返す

**症状**:

```python
{{ filter_values('city') }}  # []
```

**原因**:

1. クロスフィルターが設定されていない
2. カラム名が間違っている
3. `extra_form_data`が正しくマージされていない

**解決策**:

```python
# 1. デフォルト値を設定
{{ filter_values('city', default='Tokyo') }}  # ['Tokyo']

# 2. カラム名を確認
{{ filter_values('city_name') }}  # 正しいカラム名を使用

# 3. デバッグ: 全フィルターを確認
{% for filter in get_filters('city') %}
    {{ filter }}
{% endfor %}
```

---

### 問題2: フィルターが二重に適用される

**症状**: WHERE句に同じフィルターが2回適用される

```sql
WHERE city IN ('Tokyo')  -- SQLクエリ内
  AND city IN ('Tokyo')  -- Supersetが自動生成
```

**原因**: `remove_filter=False`（デフォルト）のため、Supersetが自動生成するWHERE句にもフィルターが含まれる

**解決策**:

```python
# remove_filter=Trueを使用
{{ filter_values('city', remove_filter=True) }}
```

---

### 問題3: "Unsafe return type" エラー

**症状**:

```
SupersetTemplateException: Unsafe return type for function filter_values: ...
```

**原因**: `filter_values()`自体は安全だが、返された値を不適切に処理している

**解決策**:

```sql
-- ❌ Bad: 値を直接埋め込む（SQLインジェクションのリスク）
WHERE city = {{ filter_values('city')[0] }}

-- ✅ Good: INリストで使用
WHERE city IN ({{ "'" + "','".join(filter_values('city')) + "'" }})
```

---

### 問題4: 複数値のフィルターが正しくレンダリングされない

**症状**:

```sql
WHERE city IN ('['Tokyo', 'Osaka']')  -- 誤ったSQL
```

**原因**: `filter_values()`はリストを返すため、直接埋め込むと文字列化される

**解決策**:

```sql
-- ❌ Bad
WHERE city IN ('{{ filter_values('city') }}')

-- ✅ Good: join()を使用
WHERE city IN ({{ "'" + "','".join(filter_values('city')) + "'" }})
```

---

### 問題5: デフォルト値が適用されない

**症状**: フィルターがない場合、WHEREクエリがエラーになる

```sql
WHERE city IN ()  -- エラー
```

**原因**: `default`が指定されていない

**解決策**:

```python
# デフォルト値を指定
{{ filter_values('city', default='Tokyo') }}
```

または

```sql
-- 条件付きWHERE句
WHERE 1=1
{% if filter_values('city') %}
    AND city IN ({{ "'" + "','".join(filter_values('city')) + "'" }})
{% endif %}
```

---

## 結論

### クロスフィルターとfilter_values()のまとめ

#### ✅ 仕組みの全体像

```
ユーザーがChart Aでフィルター選択
    ↓
Redux storeのfilterState更新
    ↓
Chart Bが extra_form_data を生成
    ↓
Backend: get_form_data()で form_data 取得
    ↓
Backend: merge_extra_form_data()でマージ
    ↓
Backend: filter_values()でフィルター値抽出
    ↓
Jinjaテンプレートレンダリング
    ↓
SQLクエリ実行
    ↓
Chart Bに結果表示
```

---

#### 🔑 キーポイント

1. **extra_form_data**: クロスフィルターの情報を運ぶコンテナ
2. **merge_extra_form_data()**: `extra_form_data`を`form_data`にマージ（extend）
3. **filter_values()**: `form_data.adhoc_filters`からフィルター値を抽出
4. **isExtraマーカー**: クロスフィルター由来のフィルターを識別
5. **remove_filter**: 外側クエリからフィルターを削除するフラグ

---

#### 📊 データ構造の変化

```
extra_form_data (Frontend)
    ↓ (HTTP Request)
form_data.extra_form_data (Backend)
    ↓ (merge_extra_form_data)
form_data.adhoc_filters (Backend)
    ↓ (filter_values)
list[Any] (Python)
    ↓ (Jinja Render)
SQL IN clause
```

---

#### 💡 ベストプラクティス

1. **デフォルト値を設定**: `filter_values('city', default='Tokyo')`
2. **SQLインジェクション対策**: `join()`で適切にエスケープ
3. **条件付きフィルター**: `{% if filter_values('city') %}`で存在確認
4. **remove_filterの適切な使用**: サブクエリ内でのみフィルター適用する場合に使用
5. **get_filters()の活用**: オペレーター情報が必要な場合に使用

---

#### ⚠️ 注意点

- `filter_values()`は**リスト**を返す（単一値ではない）
- `extra_form_data.adhoc_filters`は**extend**される（上書きではない）
- `remove_filter=True`を使用すると、Superset自動生成のWHERE句からフィルターが削除される
- カラム名は`form_data.adhoc_filters[].subject`と完全一致する必要がある

---

**調査実施**: Claude Code
**作成日**: 2026-04-06
**Supersetバージョン**: 4.x以降対応
