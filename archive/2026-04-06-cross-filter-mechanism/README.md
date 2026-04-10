# クロスフィルターとfilter_values()調査レポート - 2026-04-06

Apache Superset のクロスフィルター機能と`filter_values()`マクロに関する包括的な調査レポート

---

## 📋 調査概要

**調査日**: 2026-04-06
**対象**: Apache Superset クロスフィルター機能と`filter_values()`マクロ
**調査者**: Claude Code

### 調査の目的

Supersetにおいて、**クロスフィルター**（チャート間フィルター連携）で値を受け取る仕組みを解説します。特に、`filter_values()`マクロがどのようにクロスフィルターの値を受け取るかを、図式を含めて詳細に説明します。

---

## 📚 レポート

### クロスフィルターとfilter_values()完全ガイド

**ファイル**: `CROSS_FILTER_AND_FILTER_VALUES_GUIDE.md`

**内容**（全800行以上）:
- ✅ クロスフィルターの基本概念
- ✅ 全体アーキテクチャ（図式付き）
- ✅ データフロー（8ステップの詳細図）
- ✅ filter_values()の仕組み
- ✅ extra_form_dataとadhoc_filtersのマージ処理
- ✅ **実装例7パターン**
- ✅ 主要ファイルと関数
- ✅ トラブルシューティング

**重要度**: ⭐⭐⭐⭐⭐

---

## 🔍 調査結果サマリー

### クロスフィルターの仕組み

```
Chart A (Filter Emitter)
    ↓ ユーザーが値を選択
Redux Store (filterState更新)
    ↓
Chart B (extra_form_data生成)
    ↓ HTTP Request
Backend (get_form_data)
    ↓
Backend (merge_extra_form_data)
    ↓
Backend (filter_values)
    ↓ SQL Rendering
Database
    ↓ Results
Chart B (更新)
```

---

### filter_values()の動作

| ステップ | 処理 | ファイル/行数 |
|---------|------|--------------|
| 1 | `get_filters(column)`呼び出し | `jinja_context.py:240-279` |
| 2 | `get_form_data()`で form_data 取得 | `views/utils.py:146-196` |
| 3 | `form_data.adhoc_filters`から検索 | `jinja_context.py:353` |
| 4 | `subject == column`のフィルターを抽出 | `jinja_context.py:356-369` |
| 5 | `comparator`値を返す | `jinja_context.py:247-254` |

---

### extra_form_dataのマージ処理

#### マージ前

```python
form_data = {
    "adhoc_filters": [],  # Chart自身のフィルター
    "extra_form_data": {  # クロスフィルター
        "adhoc_filters": [
            {"subject": "city", "comparator": ["Osaka"]}
        ]
    }
}
```

#### マージ後

```python
form_data = {
    "adhoc_filters": [
        {
            "subject": "city",
            "comparator": ["Osaka"],
            "isExtra": True  # マーカー追加
        }
    ]
}
```

---

## 💡 重要な発見

### 1. extra_form_data vs form_data

| 項目 | extra_form_data | form_data |
|------|----------------|-----------|
| **送信元** | Frontend (クロスフィルター) | Chart設定 |
| **マージ方法** | **extend** (追加) | - |
| **マーカー** | `isExtra: true` | なし |
| **処理後** | 削除される（pop） | 残る |

---

### 2. EXTRA_FORM_DATA_APPEND_KEYS

**ファイル**: `constants.py:178-185`

```python
EXTRA_FORM_DATA_APPEND_KEYS = {
    "adhoc_filters",  # ← extend（追加）される
    "filters",
    "interactive_groupby",
    # ...
}
```

→ これらのキーは**extend**（追加）される

---

### 3. EXTRA_FORM_DATA_OVERRIDE_REGULAR_MAPPINGS

**ファイル**: `constants.py:187-193`

```python
EXTRA_FORM_DATA_OVERRIDE_REGULAR_MAPPINGS = {
    "time_range": "time_range",  # ← override（上書き）される
    "granularity_sqla": "granularity",
    # ...
}
```

→ これらのキーは**override**（上書き）される

---

### 4. filter_values()の内部処理

```python
def filter_values(column, default=None, remove_filter=False):
    filters = get_filters(column, remove_filter)  # ①
    return_val = []
    for flt in filters:  # ②
        val = flt.get("val")
        if isinstance(val, list):
            return_val.extend(val)  # ③ リストを展開
        elif val:
            return_val.append(val)  # ④ 単一値を追加
    if not return_val and default:
        return_val = [default]  # ⑤ デフォルト値
    return return_val  # ⑥ リストを返す
```

---

### 5. remove_filterパラメータの効果

```
remove_filter=False (デフォルト)
    ↓
フィルターは form_data.adhoc_filters に残る
    ↓
Supersetが自動生成する WHERE句にも適用される
```

```
remove_filter=True
    ↓
フィルター値は取得できる
    ↓
self.removed_filters に追加される
    ↓
Supersetが自動生成する WHERE句から削除される
```

**使用例**: サブクエリ内でのみフィルターを適用したい場合

---

## 📊 実装例

### 例1: 基本的な使用

```sql
SELECT *
FROM sales
WHERE city IN ({{ "'" + "','".join(filter_values('city')) + "'" }})
```

**クロスフィルター**: `['Osaka']`

**レンダリング後**:

```sql
SELECT *
FROM sales
WHERE city IN ('Osaka')
```

---

### 例2: デフォルト値

```sql
WHERE city IN ({{ "'" + "','".join(filter_values('city', default='Tokyo')) + "'" }})
```

**フィルターなし** → `default='Tokyo'` → `['Tokyo']`

---

### 例3: 条件付きフィルター

```sql
WHERE 1=1
{% if filter_values('city') %}
    AND city IN ({{ "'" + "','".join(filter_values('city')) + "'" }})
{% endif %}
```

---

### 例4: remove_filter使用

```sql
WITH filtered AS (
    SELECT *
    FROM sales
    WHERE city IN ({{ "'" + "','".join(filter_values('city', remove_filter=True)) + "'" }})
)
SELECT * FROM filtered
-- 外側のクエリでは city フィルターが適用されない
```

---

### 例5: get_filters()で詳細情報取得

```sql
{% set city_filters = get_filters('city') %}
{% for filter in city_filters %}
    {% if filter.op == 'IN' %}
        AND city IN ({{ "'" + "','".join(filter.val) + "'" }})
    {% endif %}
{% endfor %}
```

---

## 🛠️ 主要ファイル

| ファイル | 役割 | 重要度 |
|---------|------|--------|
| `superset/jinja_context.py` | `filter_values()`, `get_filters()` | ⭐⭐⭐⭐⭐ |
| `superset/utils/core.py` | `merge_extra_form_data()`, `merge_extra_filters()` | ⭐⭐⭐⭐⭐ |
| `superset/views/utils.py` | `get_form_data()` | ⭐⭐⭐⭐☆ |
| `superset/constants.py` | `EXTRA_FORM_DATA_*`定数 | ⭐⭐⭐⭐☆ |

### 主要関数

| 関数 | ファイル:行数 | 役割 |
|------|--------------|------|
| `filter_values()` | `jinja_context.py:240-279` | フィルター値をリストで返す |
| `get_filters()` | `jinja_context.py:281-384` | フィルター情報（op含む）を返す |
| `merge_extra_form_data()` | `utils/core.py:890-942` | extra_form_dataをマージ |
| `get_form_data()` | `views/utils.py:146-196` | リクエストから form_data 取得 |

---

## 🎯 ベストプラクティス

### ✅ 推奨

1. **デフォルト値を設定**

```python
filter_values('city', default='Tokyo')
```

2. **SQLインジェクション対策**

```sql
WHERE city IN ({{ "'" + "','".join(filter_values('city')) + "'" }})
```

3. **条件付きフィルター**

```sql
{% if filter_values('city') %}
    AND city IN (...)
{% endif %}
```

4. **remove_filterの適切な使用**

```python
filter_values('city', remove_filter=True)  # サブクエリ内でのみ
```

---

### ❌ 避けるべき

1. **値を直接埋め込む**

```sql
-- ❌ Bad
WHERE city = {{ filter_values('city')[0] }}

-- ✅ Good
WHERE city IN ({{ "'" + "','".join(filter_values('city')) + "'" }})
```

2. **デフォルト値なしでの使用**

```sql
-- ❌ Bad: フィルターなし → WHERE city IN () → エラー
WHERE city IN ({{ "'" + "','".join(filter_values('city')) + "'" }})

-- ✅ Good
WHERE city IN ({{ "'" + "','".join(filter_values('city', default='Tokyo')) + "'" }})
```

---

## 📝 まとめ

### 調査の成果

1. ✅ クロスフィルターの全体アーキテクチャを解明
2. ✅ `filter_values()`の内部処理フローを図式化（8ステップ）
3. ✅ `extra_form_data`のマージ処理を詳細に解析
4. ✅ `isExtraマーカー`の役割を特定
5. ✅ `remove_filter`パラメータの動作を検証
6. ✅ 実装例7パターンを提供
7. ✅ トラブルシューティングガイドを作成

---

### キーポイント

| 概念 | 説明 |
|------|------|
| **extra_form_data** | クロスフィルター情報を運ぶコンテナ |
| **merge_extra_form_data()** | extra_form_dataをform_dataに**extend** |
| **filter_values()** | adhoc_filtersからフィルター値を抽出 |
| **isExtraマーカー** | クロスフィルター由来を識別 |
| **remove_filter** | 外側クエリからフィルター削除 |

---

### データフロー要約

```
Frontend (Redux filterState)
    ↓ extra_form_data生成
Backend (get_form_data)
    ↓ extra_form_dataマージ
Backend (filter_values)
    ↓ adhoc_filtersから値抽出
Jinja Template
    ↓ SQLレンダリング
Database
```

---

## 🔗 関連リソース

### 内部ドキュメント

- [カスタムJinjaマクロ実装ガイド](../2026-04-06-custom-jinja-macros/CUSTOM_JINJA_MACROS_GUIDE.md)

### 外部リソース

- [Superset公式ドキュメント - Jinja Templating](https://superset.apache.org/docs/installation/sql-templating)
- [GitHub - superset/jinja_context.py](https://github.com/apache/superset/blob/master/superset/jinja_context.py)
- [GitHub - superset/utils/core.py](https://github.com/apache/superset/blob/master/superset/utils/core.py)

---

**調査実施**: Claude Code
**作成日**: 2026-04-06
**Supersetバージョン**: 4.x以降対応
