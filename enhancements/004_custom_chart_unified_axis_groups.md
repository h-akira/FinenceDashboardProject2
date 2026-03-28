# ENH-004: カスタムチャート axis_groups 統合・local_groups 導入

## ステータス

完了（検証済み）

## 起票日

2026-03-28

## 種別

改善

## 概要

カスタムチャートの `custom_chart_sources.json` において、独立軸ソースの `axis_group` が辞書形式で特殊扱いされている現状を改善する。`independent_groups` を `axis_groups` に統合し、`independent` フラグで区別する。また、全 axis_group に `local_groups`（サブカテゴリ）を導入し、非独立軸でもサブカテゴリを使えるようにする。

## 背景・動機

ENH-003 で独立軸のサブグループ化を導入したが、以下の課題がある：

1. **`axis_group` の型が不統一**: 通常ソースは文字列、独立軸ソースは辞書 → バックエンドで `isinstance` 分岐が必要で複雑
2. **`independent_groups` が `axis_groups` と別管理**: 構造が分散しており、独立軸と通常軸で異なる仕組みが必要
3. **非独立軸にサブカテゴリがない**: 例えば金利・スプレッドグループ内で国別に分類したい場合に対応できない
4. **`other_display_name` という特殊キーが必要**: 独立軸のグループ名が別管理されている

## 要件

### JSON 構造の変更

#### `axis_groups` の統合

- `independent_groups` を廃止し、`axis_groups` に統合する
- 各 axis_group に `independent` フラグ（boolean, **必須**）を追加
- 各 axis_group に `local_groups`（optional）を追加し、サブカテゴリを定義できるようにする
- `other_display_name` を廃止する（独立軸グループも `axis_groups` 内の `display_name` で表示）

#### sources の変更

- `axis_group` は常に文字列（`axis_groups` のキーを参照）とする（辞書形式を廃止）
- `local_group`（optional）を追加し、`axis_groups[axis_group].local_groups` のキーを参照する
- `independent: true` のソースは `label` を sources 側に持つ（ソースごとに単位が異なるため）
- `independent: false`（通常）のソースは `label` を持たない（axis_groups 側の `label` を使用）

### バリデーションルール

バリデーションスクリプト `bin/validate_config.py` を作成し、以下のルールを検証する：

| # | ルール | エラー条件 |
|---|--------|-----------|
| V1 | axis_group 参照の存在 | source の `axis_group` が `axis_groups` に存在しないキーを参照 |
| V2 | local_group 参照の存在 | source の `local_group` が指定されているが、該当 axis_group に `local_groups` がない、またはキーが存在しない |
| V3 | 独立軸に label 禁止 | `independent: true` の axis_group に `label` が定義されている |
| V4 | 通常軸に label 必須 | `independent: false` の axis_group に `label` が定義されていない |
| V5 | independent 必須 | axis_group に `independent` フィールドが存在しない |
| V6 | 独立ソースに label 必須 | `independent: true` の axis_group を参照する source に `label` がない |
| V7 | 通常ソースに label 禁止 | `independent: false` の axis_group を参照する source に `label` がある |

### `custom_chart_sources.json` 新構造

```json
{
  "max_axes": 2,
  "axis_groups": {
    "rate_pct1": {
      "label": "%",
      "display_name": "金利・スプレッド",
      "independent": false,
      "local_groups": {
        "us": { "display_name": "米国" }
      }
    },
    "ratio1": {
      "label": "倍率",
      "display_name": "前年比",
      "independent": false,
      "local_groups": {
        "stock_index": { "display_name": "株価指数" }
      }
    },
    "index1": {
      "label": "指数",
      "display_name": "通貨指数",
      "independent": false
    },
    "independent1": {
      "display_name": "独立軸",
      "independent": true,
      "local_groups": {
        "stock_index": { "display_name": "株価指数" },
        "investment_env": { "display_name": "投資環境" }
      }
    }
  },
  "sources": {
    "target_rate": {
      "name": "政策金利 (FF金利誘導目標上限)",
      "axis_group": "rate_pct1",
      "local_group": "us",
      "default": true,
      "source_type": "fred",
      "fred_series": ["DFEDTAR", "DFEDTARU"],
      "fred_boundary": "2008-12-16",
      "start": "1982-09-27"
    },
    "dgs10": {
      "name": "米国10年国債利回り",
      "axis_group": "rate_pct1",
      "local_group": "us",
      "default": true,
      "source_type": "fred",
      "fred_series": ["DGS10"],
      "start": "1982-09-27"
    },
    "baa10y": {
      "name": "米国Baa社債スプレッド",
      "axis_group": "rate_pct1",
      "local_group": "us",
      "default": false,
      "source_type": "fred",
      "fred_series": ["BAA10Y"],
      "start": "1986-01-01"
    },
    "sp500": {
      "name": "S&P 500",
      "axis_group": "independent1",
      "local_group": "stock_index",
      "label": "USD",
      "default": false,
      "source_type": "yfinance",
      "ticker": "^GSPC",
      "start": "1982-09-27"
    },
    "sp500_yoy": {
      "name": "S&P 500 前年比",
      "axis_group": "ratio1",
      "local_group": "stock_index",
      "default": false,
      "source_type": "yfinance",
      "ticker": "^GSPC",
      "transform": "yoy_ratio",
      "start": "1983-09-27"
    },
    "dtwexbgs": {
      "name": "ドル指数 (実効為替レート)",
      "axis_group": "index1",
      "default": false,
      "source_type": "fred",
      "fred_series": ["DTWEXBGS"],
      "start": "2006-01-01"
    },
    "score": {
      "name": "投資環境スコア（堀井）",
      "axis_group": "independent1",
      "local_group": "investment_env",
      "label": "スコア",
      "default": false,
      "source_type": "calculated",
      "start": "2007-01-01"
    }
  }
}
```

### API レスポンス変更

#### `GET /finance/custom-chart/sources`

```json
{
  "sources": [
    {
      "id": "target_rate",
      "name": "政策金利 (FF金利誘導目標上限)",
      "axis_group": "rate_pct1",
      "axis_label": "%",
      "local_group": "us",
      "default": true
    },
    {
      "id": "sp500",
      "name": "S&P 500",
      "axis_group": "independent1",
      "axis_label": "USD",
      "local_group": "stock_index",
      "default": false
    },
    {
      "id": "dtwexbgs",
      "name": "ドル指数 (実効為替レート)",
      "axis_group": "index1",
      "axis_label": "指数",
      "default": false
    }
  ],
  "axis_groups": {
    "rate_pct1": {
      "label": "%",
      "display_name": "金利・スプレッド",
      "independent": false,
      "local_groups": {
        "us": { "display_name": "米国" }
      }
    },
    "independent1": {
      "display_name": "独立軸",
      "independent": true,
      "local_groups": {
        "stock_index": { "display_name": "株価指数" },
        "investment_env": { "display_name": "投資環境" }
      }
    }
  },
  "max_axes": 2
}
```

変更点:
- `independent_groups` 廃止 → `axis_groups` に統合（`independent` フラグ付き）
- `other_display_name` 廃止
- `CustomChartSource` に `local_group`（optional）追加
- `CustomChartSource` から `independent_group` 削除
- `CustomChartAxisGroup` に `independent`（boolean）と `local_groups`（optional）追加
- `CustomChartIndependentGroup` スキーマ削除
- `CustomChartSourcesResponse` から `independent_groups` と `other_display_name` 削除
- `CustomChartAxisGroup` に `local_groups` 追加（`CustomChartLocalGroup` を参照）
- `CustomChartLocalGroup` スキーマ新規追加（`display_name` のみ）

### フロントエンド表示

```
├── 金利・スプレッド（%）
│   └── 米国
│       ├── ☑ 政策金利 (FF金利誘導目標上限)
│       ├── ☑ 米国10年国債利回り
│       └── ☐ 米国Baa社債スプレッド
├── 前年比（倍率）
│   └── 株価指数
│       └── ☐ S&P 500 前年比
├── 通貨指数（指数）
│   └── ☐ ドル指数 (実効為替レート)
└── 独立軸
    ├── 株価指数
    │   └── ☐ S&P 500 [USD]
    └── 投資環境
        └── ☐ 投資環境スコア（堀井）[スコア]
```

- 通常グループ: ヘッダーに `display_name（label）`、local_group があればサブヘッダー表示
- 独立軸グループ: ヘッダーに `display_name`（`label` なし）、local_group のサブヘッダー表示、各ソース横に `[label]`
- local_group なしのソースはサブカテゴリなしとしてグループ先頭に表示

### 軸数カウントの変更

- 現状: `axis_group === "other"` で独立軸を判定
- 変更後: `axis_groups[axis_group].independent === true` で判定
- 通常グループ: 同一 axis_group は1軸として集約カウント（変更なし）
- 独立軸グループ: 選択ソースごとに1軸消費（変更なし）

## 影響範囲

- Backend: `custom_chart_sources.json`（構造変更）
- Backend: `custom_chart_service.py`（`_resolve_axis` の `isinstance` 分岐廃止、`local_group` 対応）
- Backend: `bin/validate_config.py`（新規作成）
- Frontend: `CustomChartPage.vue`（グループ化ロジック変更、`local_groups` 対応の一般化）
- Frontend: MSW モックデータ更新
- OpenAPI: スキーマ変更（`CustomChartAxisGroup` 拡張、`CustomChartIndependentGroup` 削除、`CustomChartLocalGroup` 新規）
- テスト: バックエンド・フロントエンドテストの更新

## テスト計画

### バックエンド ユニットテスト

| # | テストケース | 検証内容 |
|---|------------|---------|
| S1 | `get_sources` の通常ソース | `axis_label` が axis_groups の `label` から取得される |
| S2 | `get_sources` の独立軸ソース | `axis_label` が source の `label` から取得される |
| S3 | `get_sources` に `local_group` | `local_group` を持つソースのレスポンスに `local_group` が含まれる |
| S4 | `get_sources` に `local_group` なし | `local_group` がないソースのレスポンスに `local_group` が含まれない |
| S5 | `axis_groups` に `independent` と `local_groups` | レスポンスの axis_groups に `independent` と `local_groups` が含まれる |
| S6 | `independent_groups` と `other_display_name` が廃止 | レスポンスにこれらのキーが含まれない |

### バリデーションスクリプト テスト

| # | テストケース | 検証内容 |
|---|------------|---------|
| V1 | 正常な設定ファイル | エラーなしで通過 |
| V2 | 存在しない axis_group 参照 | エラー |
| V3 | 存在しない local_group 参照 | エラー |
| V4 | 独立軸 axis_group に label あり | エラー |
| V5 | 通常軸 axis_group に label なし | エラー |
| V6 | axis_group に independent なし | エラー |
| V7 | 独立軸ソースに label なし | エラー |
| V8 | 通常ソースに label あり | エラー |

### フロントエンド確認

| # | 確認項目 |
|---|---------|
| F1 | 通常グループにヘッダー `display_name（label）` が表示される |
| F2 | 通常グループ内に local_group のサブヘッダーが表示される |
| F3 | 独立軸グループにヘッダー `display_name` が表示される（label なし） |
| F4 | 独立軸内に local_group のサブヘッダーが表示される |
| F5 | 独立軸ソースの横に `[label]` が表示される |
| F6 | local_group なしのソースがグループ先頭に表示される |
| F7 | 軸数カウントが正しく動作する（independent フラグベース） |
| F8 | 既存の機能（デフォルト選択、軸数制約、反映ボタン等）が引き続き動作する |

## 対応

### 採用した方針

enhancement ファイルの実現案に従い、以下を実施した。

### 変更ファイル

#### ドキュメント
- `docs/openapi_main.yaml` — スキーマ変更（`CustomChartAxisGroup` に `independent`/`local_groups` 追加、`CustomChartIndependentGroup` → `CustomChartLocalGroup`、`CustomChartSourcesResponse` から `independent_groups`/`other_display_name` 削除）
- `docs/user_stories.md` — US-8.4, US-8.5 の受け入れ条件更新
- `docs/history/005_unified_axis_groups_local_groups.md` — 変更通知（新規作成）
- `Frontend/docs/openapi_main.yaml` — 同期
- `Backend/main/docs/openapi_main.yaml` — 同期

#### バックエンド
- `Backend/main/src/custom_chart_sources.json` — 構造変更（`independent_groups`/`other_display_name` 廃止、`axis_groups` に統合、`local_groups` 追加、sources の `axis_group` を文字列統一）
- `Backend/main/src/services/custom_chart_service.py` — `_resolve_axis` リファクタリング（`isinstance` 分岐廃止）、`get_sources` レスポンス変更
- `Backend/main/bin/validate_config.py` — 新規作成（7つのバリデーションルール）
- `Backend/main/tests/local/test_services.py` — `TestCustomChartService` テスト更新
- `Backend/main/tests/local/test_routes.py` — `TestCustomChartRoutes` テスト更新
- `Backend/main/tests/local/test_validate_config.py` — 新規作成（9テストケース）

#### フロントエンド
- `Frontend/finance-dashboard/src/pages/CustomChartPage.vue` — グループ化ロジック一般化、`independent` フラグベースの軸数カウント、サブカテゴリ表示の統一
- `Frontend/finance-dashboard/src/mocks/handlers.ts` — MSW モックデータ更新
- `Frontend/finance-dashboard/src/api/generated/` — Orval 型再生成
- `Frontend/tests/test_custom_chart.py` — E2E テスト更新

### テスト結果

- バックエンド: 41 passed, 0 failed
- バリデーションスクリプト: 9 passed（`test_validate_config.py`）
- フロントエンド E2E: 28 passed, 0 failed

## 関連

- [enhancements/003_custom_chart_ux3.md](003_custom_chart_ux3.md) - ENH-003（本改善の前提）
- [Backend/main/src/custom_chart_sources.json](../Backend/main/src/custom_chart_sources.json) - ソース定義ファイル
- [Frontend/finance-dashboard/src/pages/CustomChartPage.vue](../Frontend/finance-dashboard/src/pages/CustomChartPage.vue) - カスタムチャートページ
