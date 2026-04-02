# アーカイブ: ツールチップ調査レポート（2026-04-01）

## このアーカイブについて

このディレクトリには、Apache Superset Mixed Timeseries Chart のツールチップ機能に関する詳細調査の個別レポートが保管されています。

**アーカイブ日**: 2026-04-01
**理由**: 5つの個別レポートを1つの包括的なガイドに統合したため

## 統合後の最新ドキュメント

**📄 [TOOLTIP_COMPREHENSIVE_GUIDE.md](../../TOOLTIP_COMPREHENSIVE_GUIDE.md)**

すべての情報を統合した包括的なガイドです。今後はこちらを参照してください。

## アーカイブされたファイル

### 1. TOOLTIP_REPORT.md
**内容**: ツールチップ表示メカニズムの詳細解説
**主なトピック**:
- ツールチップの動作フロー
- extractForecastValuesFromTooltipParams の処理
- formatForecastTooltipSeries の実装
- 0値が表示される理由

### 2. MISSING_VALUES_REPORT.md
**内容**: 欠損値（値がない場合）の扱いに関する詳細解説
**主なトピック**:
- データフロー全体像（バックエンド→フロントエンド）
- fillNeighborValue の決定ロジック
- extractSeries での変換処理
- スタックモードON/OFFによる違い

### 3. TOOLTIP_MODE_REPORT.md
**内容**: ツールチップモード（axis vs item）の違いと設定方法
**主なトピック**:
- axisモードとitemモードの動作比較
- Rich tooltip チェックボックスの設定方法
- シリーズ数による推奨モード
- params パラメータの構造の違い

### 4. TOOLTIP_HYBRID_FEATURE.md
**内容**: axisモード + 0値非表示の複合機能実装案
**主なトピック**:
- 技術的実装方法（コードレベル）
- types.ts, transformProps.ts, controlPanel.tsx の変更箇所
- UI設定の追加方法
- テストケース

### 5. ISSUE_STATUS_REPORT.md
**内容**: Issue #4462 の状態調査レポート
**主なトピック**:
- Issue #4462 の詳細（2018年から存在）
- 関連PR #21296 の内容
- 2025年ロードマップでの位置づけ
- 実装されない理由の分析
- コミュニティ貢献の道筋

### 6. DATE_FORMAT_ORIGINAL_REPORT.md
**内容**: Mixed Chart 日時変換不具合の初回調査レポート（2026-04-01）
**主なトピック**:
- Query Aが空の場合にX軸がタイムスタンプ数値で表示される問題
- transformProps.ts:235-236行目の実装ミス
- お客様向け説明サマリー
- 次のアクション提案

**注**: このレポートの内容は、より詳細な3つのドキュメントに分割されました：
- [DATE_FORMAT_BUG_OVERVIEW.md](../../DATE_FORMAT_BUG_OVERVIEW.md) - 概要
- [DATE_FORMAT_BUG_TECHNICAL.md](../../DATE_FORMAT_BUG_TECHNICAL.md) - 技術詳細
- [DATE_FORMAT_BUG_FIX_GUIDE.md](../../DATE_FORMAT_BUG_FIX_GUIDE.md) - 修正ガイド

## 調査の経緯

1. **初回調査**: Mixチャートで日時変換がうまくいかない現象の調査
2. **ツールチップ調査**: 0件がツールチップに表示される仕組みの解説
3. **欠損値調査**: 時系列データで値がない場合の0変換ロジックの解明
4. **モード調査**: axisモードとitemモードの違いと設定方法の特定
5. **複合機能調査**: axisモード + 0値非表示の実装可能性の検証
6. **Issue調査**: 公式実装の見通しと将来性の確認
7. **統合**: すべての知見を1つの包括的なガイドにまとめる

## 利用方法

### 包括的なガイドを参照する場合
```bash
# プロジェクトルートから
cat TOOLTIP_COMPREHENSIVE_GUIDE.md
```

### 特定のトピックのみ確認する場合
```bash
# アーカイブディレクトリから
cd archive/2026-04-01-tooltip-investigation/
cat TOOLTIP_MODE_REPORT.md  # 例: モードの違いのみ確認
```

## 関連リンク

- [Issue #4462](https://github.com/apache/incubator-superset/issues/4462) - Tooltip option to display or ignore 0 or null values
- [PR #21296](https://github.com/apache/superset/pull/21296) - fix(plugin-chart-echarts): show zero value in tooltip

## 注意事項

このアーカイブ内のドキュメントは参考資料として保管されています。最新の情報や統合された内容については、プロジェクトルートの `TOOLTIP_COMPREHENSIVE_GUIDE.md` を参照してください。

---

**アーカイブ作成者**: Claude Code
**作成日**: 2026-04-01
