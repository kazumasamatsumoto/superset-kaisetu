# superset-kaisetu

Apache Supersetの解説リポジトリです。

個人で使っています。

---

## ドキュメント構成

### 📄 包括的なガイド

#### Mixed Chart 日付フォーマット不具合
**[DATE_FORMAT_BUG_COMPREHENSIVE.md](./DATE_FORMAT_BUG_COMPREHENSIVE.md)**
- Query Aが空の場合にX軸の日時が数値表示される問題の包括的ガイド
- 概要、技術詳細、修正手順、テスト方法をすべて含む
- 読了時間: 約30分（全体）、約5分（概要のみ）

#### ツールチップ機能
**[TOOLTIP_COMPREHENSIVE_GUIDE.md](./TOOLTIP_COMPREHENSIVE_GUIDE.md)**
- Mixed Timeseries Chartのツールチップ機能の包括的なガイド
- axisモード vs itemモード
- 0値表示の問題
- スタックモードとの関係
- 読了時間: 約40分

#### スタックモード
**[STACK_MODE_EXPLAINED.md](./STACK_MODE_EXPLAINED.md)**
- スタックモードの完全解説
- なぜ欠損値が0になるのか
- 視覚的な図解
- 読了時間: 約15分

#### シリーズタイプ設定
**[MIXED_CHART_SERIES_TYPE.md](./MIXED_CHART_SERIES_TYPE.md)**
- Query AとQuery Bのグラフタイプ設定方法
- 7種類のグラフタイプ（折れ線、棒グラフ、散布図など）
- 設定の組み合わせ例
- 読了時間: 約10分

---

## アーカイブ

過去の個別ドキュメントや調査レポートは、日付別のアーカイブディレクトリに整理されています。

### 📂 archive/2026-04-01-tooltip-investigation/
- ツールチップ関連の5つの個別レポート
- 初回の日付フォーマット不具合レポート
- [詳細はこちら](./archive/2026-04-01-tooltip-investigation/README.md)

### 📂 archive/2026-04-02-date-format-bug/
- 日付フォーマット不具合の4つの個別ドキュメント
  - インデックス（INDEX）
  - 概要（OVERVIEW）
  - 技術詳細（TECHNICAL）
  - 修正ガイド（FIX_GUIDE）
- [詳細はこちら](./archive/2026-04-02-date-format-bug/README.md)

---

## クイックスタート

### Mixed Chart 日付フォーマット不具合を修正したい方
1. [DATE_FORMAT_BUG_COMPREHENSIVE.md](./DATE_FORMAT_BUG_COMPREHENSIVE.md) のパート1（概要）を読む（5分）
2. パート3（修正ガイド）に従って修正を適用（30分）
3. 4パターンのテストを実施

### ツールチップの仕組みを理解したい方
1. [TOOLTIP_COMPREHENSIVE_GUIDE.md](./TOOLTIP_COMPREHENSIVE_GUIDE.md) を読む
2. 必要に応じて [STACK_MODE_EXPLAINED.md](./STACK_MODE_EXPLAINED.md) も参照

---

## 修正済み不具合

### ✅ Mixed Chart 日付フォーマット不具合（2026-04-02）
- **問題**: Query Aが空の場合、X軸が `1609459200000` のような数値表示
- **修正**: `transformProps.ts:235-236` の2行を修正
- **詳細**: [DATE_FORMAT_BUG_COMPREHENSIVE.md](./DATE_FORMAT_BUG_COMPREHENSIVE.md)

---

## 今後の予定

- 🔲 フロントエンドのビルドとテスト
- 🔲 本番環境へのデプロイ
- 🔲 Apache Supersetコミュニティへの報告

---

**作成者**: Claude Code
**最終更新**: 2026-04-02
