# ENH-003: カスタムチャート UX 改善2（単位表示簡素化・独立軸サブグループ化）

## ステータス

完了（検証済み）

## 起票日

2026-03-28

## 種別

改善

## 概要

カスタムチャートページの UX をさらに改善する。軸グループ内のソースは全て同一単位であるため各チェックボックスから単位表示を除去しグループヘッダーに集約する。また、独立軸ソースの表示グループ名を「その他」→「独立軸」に改名し、独立軸ソースが増えた場合に備えてサブグループで分類できる仕組みを導入する。

## 背景・動機

ENH-002 で軸グループごとのチェックボックス整理を実施したが、以下の課題が残っている:

1. **単位の冗長表示**: 同一軸グループ内のソースは全て同じ単位（例: 金利・スプレッドグループは全て `%`）であるにもかかわらず、各チェックボックスに単位が表示されている。グループヘッダーに単位を表示すれば十分である
2. **独立軸ソースの分類不足**: 現在、独立軸ソースは全て「その他」グループに一括表示される。将来的に独立軸ソースが増えた場合（例: 投資環境スコアに加えてセンチメント指標等）、分類なしでは見通しが悪くなる。独立軸ソースをサブグループで分類できる仕組みが必要である

## 要件

### 単位表示の簡素化

- 各ソースのチェックボックスから単位（`axis_label`）の表示を除去する
- 軸グループのヘッダーに単位を表示する（例: `金利・スプレッド（%）`）
- 独立軸ソースの場合はソース名の横に単位を表示する（グループ内で単位が異なるため）

### 独立軸のサブグループ化

- 「その他」を「独立軸」に改名する（`other_display_name` の値を変更）
- `custom_chart_sources.json` にトップレベルの `independent_groups` 定義を追加する
- 独立軸ソースの `axis_group` オブジェクトに `independent_group` キーを追加し、`independent_groups` のキーを参照させる
- フロントエンドは `independent_group` ごとにサブグループ化して表示する

#### `independent_groups` の定義

```json
{
  "independent_groups": {
    "stock_index": { "display_name": "株価指数" },
    "investment_env": { "display_name": "投資環境" }
  }
}
```

#### 独立軸ソースの `axis_group` 変更

```json
{
  "score": {
    "name": "投資環境スコア（堀井）",
    "axis_group": { "label": "スコア", "independent_group": "investment_env" },
    "default": false,
    "source_type": "calculated",
    "start": "2007-01-01"
  }
}
```

#### `custom_chart_sources.json` 全体イメージ

```json
{
  "max_axes": 2,
  "other_display_name": "独立軸",
  "axis_groups": {
    "rate_pct1": { "label": "%", "display_name": "金利・スプレッド" },
    "ratio1": { "label": "倍率", "display_name": "前年比" },
    "index1": { "label": "指数", "display_name": "通貨指数" }
  },
  "independent_groups": {
    "stock_index": { "display_name": "株価指数" },
    "investment_env": { "display_name": "投資環境" }
  },
  "sources": {
    "target_rate": {
      "name": "政策金利 (FF金利誘導目標上限)",
      "axis_group": "rate_pct1",
      "default": true,
      "...": "..."
    },
    "sp500": {
      "name": "S&P 500",
      "axis_group": { "label": "USD", "independent_group": "stock_index" },
      "default": false,
      "...": "..."
    },
    "score": {
      "name": "投資環境スコア（堀井）",
      "axis_group": { "label": "スコア", "independent_group": "investment_env" },
      "default": false,
      "...": "..."
    }
  }
}
```

### フロントエンド表示イメージ

```
チャート表示エリア
警告メッセージ表示エリア

ソース選択パネル
├── 金利・スプレッド（%）
│   ├── ☑ 政策金利 (FF金利誘導目標上限)
│   ├── ☑ 米国10年国債利回り
│   └── ☐ 米国Baa社債スプレッド
├── 前年比（倍率）
│   └── ☐ S&P 500 前年比
├── 通貨指数（指数）
│   └── ☐ ドル指数 (実効為替レート)
├── 独立軸
│   ├── 株価指数
│   │   └── ☐ S&P 500 [USD]
│   └── 投資環境
│       └── ☐ 投資環境スコア（堀井）[スコア]
└── 反映ボタン
```

- 通常グループ: ヘッダーに `display_name（label）` 形式で単位を表示
- 独立軸グループ: `other_display_name` をヘッダーに、`independent_group` の `display_name` をサブヘッダーに表示
- 独立軸ソース: ソース名の横に `[label]` で単位を表示（ソースごとに単位が異なるため）

## 影響範囲

- Backend: `custom_chart_sources.json`（`other_display_name` 変更、`independent_groups` 追加、`score` の `axis_group` に `independent_group` 追加）
- Backend: `custom_chart_service.py`（`get_sources` レスポンスに `independent_groups` 追加、`_resolve_axis` で `independent_group` を処理）
- Frontend: `CustomChartPage.vue`（単位表示変更、独立軸サブグループ描画）
- OpenAPI: `CustomChartSourcesResponse` に `independent_groups` 追加、新規 `CustomChartIndependentGroup` スキーマ、`CustomChartSource` に `independent_group` 追加
- テスト: バックエンドユニットテスト・フロントエンドテストの更新
- MSW: モックデータ更新

## API レスポンス変更

### `GET /finance/custom-chart/sources`

```json
{
  "sources": [
    {
      "id": "target_rate",
      "name": "政策金利 (FF金利誘導目標上限)",
      "axis_group": "rate_pct1",
      "axis_label": "%",
      "default": true
    },
    {
      "id": "score",
      "name": "投資環境スコア（堀井）",
      "axis_group": "other",
      "axis_label": "スコア",
      "independent_group": "investment_env",
      "default": false
    }
  ],
  "axis_groups": {
    "rate_pct1": { "label": "%", "display_name": "金利・スプレッド" }
  },
  "independent_groups": {
    "investment_env": { "display_name": "投資環境" }
  },
  "other_display_name": "独立軸",
  "max_axes": 2
}
```

- `independent_groups`: 独立軸のサブグループ定義
- `CustomChartSource` に `independent_group`（optional）を追加: `axis_group === "other"` の場合のみ存在

## テスト計画

### バックエンド ユニットテスト

| # | テストケース | 検証内容 |
|---|------------|---------|
| S1 | `get_sources` に `independent_groups` | レスポンスに `independent_groups` が含まれる |
| S2 | 独立軸ソースに `independent_group` | `score` のレスポンスに `independent_group: "investment_env"` が含まれる |
| S3 | 通常ソースに `independent_group` なし | `target_rate` のレスポンスに `independent_group` が含まれない |
| S4 | `other_display_name` が「独立軸」 | レスポンスの `other_display_name` が `"独立軸"` |

### フロントエンド確認

| # | 確認項目 |
|---|---------|
| F1 | 通常グループのヘッダーに単位が表示され、各ソースのチェックボックスには単位が表示されない |
| F2 | 独立軸グループのヘッダーが「独立軸」 |
| F3 | 独立軸内にサブグループ「投資環境」が表示される |
| F4 | 独立軸ソースの横に単位（`[スコア]`）が表示される |
| F5 | 既存の機能（デフォルト選択、軸数制約、反映ボタン等）が引き続き動作する |

## 関連

- [enhancements/002_custom_chart_ux.md](002_custom_chart_ux.md) - ENH-002（本改善の前提）
- [Backend/main/src/custom_chart_sources.json](../Backend/main/src/custom_chart_sources.json) - ソース定義ファイル
- [Frontend/finance-dashboard/src/pages/CustomChartPage.vue](../Frontend/finance-dashboard/src/pages/CustomChartPage.vue) - カスタムチャートページ
