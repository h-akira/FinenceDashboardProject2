# ENH-002: カスタムチャート UX 改善（デフォルト選択・チェックボックス整理・レイアウト変更）

## ステータス

完了（検証済み）

## 起票日

2026-03-28

## 種別

改善

## 概要

カスタムチャートページのユーザー体験を改善する。初回表示時にデフォルトで政策金利と米国10年債利回りを表示し、チェックボックスを軸グループごとに整理し、チャートの下に配置する。また、`score` の名称を「投資環境スコア（堀井）」に変更し、独立軸ソースの仕組みを導入する。

## 背景・動機

現在のカスタムチャートページには以下の課題がある:

1. **初回表示が空**: ページを開いた直後はチャートが空で、ユーザーが毎回手動でソースを選択して反映ボタンを押す必要がある。代表的な指標（政策金利・米国10年債利回り）をデフォルトで表示することで、即座に有用な情報を提示できる
2. **チェックボックスがフラットに並んでいる**: 7つのソースが軸グループの区別なく一列に並んでおり、どのソースが同じ Y 軸を共有するのか視覚的に判別しにくい。軸グループごとにまとめることで、軸数制約（最大2軸）を意識した選択がしやすくなる
3. **チェックボックスがチャートより上にある**: チャートが主要コンテンツであるにもかかわらず、ソース選択パネルがチャートの上に配置されている。チャートを上に、操作パネルを下に配置することで、チャートの視認性を高める
4. **「その他」カテゴリの不在**: 投資環境スコアのように他のソースとスケール・単位が異なり、同じ軸を共有するソースが増えない指標を適切に分類する仕組みがない

## 要件

### デフォルト選択

- バックエンドのソース定義（`custom_chart_sources.json`）に各ソースの `default` フラグ（boolean）を追加する
- ソース一覧 API（`GET /finance/custom-chart/sources`）のレスポンスに `default` フィールドを含める
- フロントエンドは初回ロード時に `default: true` のソースを自動選択し、自動的にデータを取得してチャートに描画する（反映ボタンを押す必要なし）
- 初期リリースでは `target_rate` と `dgs10` を `default: true` とする
- デフォルト選択の変更はバックエンドの JSON 修正のみで完結すること

### チェックボックスのグループ化表示

- チェックボックスを `axis_group` ごとにまとめて表示する
- `axis_groups` に `display_name`（日本語の表示名）を追加し、グループヘッダーとして使用する
- `display_group` の概念は導入せず、`axis_group` でグルーピングを一本化する
- グループ内のソースは JSON 定義順で表示する
- `axis_groups` の定義順がグループの表示順となる

### 独立軸ソース（「その他」グループ）

投資環境スコアのように、他のソースと Y 軸を共有せず、単位も独自である指標を扱う仕組みを導入する。

- ソース定義の `axis_group` に文字列（グループ ID）ではなく **オブジェクト `{"label": "..."}`** を指定した場合、そのソースは「独立軸ソース」として扱われる
  - 文字列の場合: `axis_groups` で定義されたグループへの参照（従来通り）
  - オブジェクトの場合: 独立軸ソース（そのソース固有の軸ラベルを `label` に持つ）
- `axis_groups` に独立軸ソース用の定義は不要（型で自動判別）
- 独立軸ソースの表示グループ名は `custom_chart_sources.json` のトップレベルに `other_display_name` として定義する
- 独立軸ソースは UI 上「その他」グループにまとめて表示する（`axis_groups` の後に表示）
- 軸数カウントのルール:
  - 通常の `axis_group`: 選択されたソースが何個あっても **グループで1軸**
  - 独立軸ソース: **選択されたソースごとに1軸**
- 独立軸ソースの複数選択は可能（軸数制約の範囲内で）
- 初期リリースでは `score`（投資環境スコア（堀井））を独立軸ソースとする

### ソース名称変更

- `score` の `name` を「複合スコア」→「投資環境スコア（堀井）」に変更する

### レイアウト変更

- ページ構成をチャート → 操作パネル（チェックボックス + 反映ボタン）の順に変更する
- 警告メッセージはチャートと操作パネルの間に表示する

## 影響範囲

- Backend: `custom_chart_sources.json` の変更（`axis_groups` に `display_name` 追加、`other_display_name` 追加、ソースに `default` 追加、`score` の `axis_group` をオブジェクト化・名称変更、`score1` 軸グループ削除）
- Backend: `custom_chart_service.py` の `get_sources` レスポンス変更（`default` 追加、`axis_label` の取得元ロジック変更、`axis_groups` 情報の追加）
- Frontend: `CustomChartPage.vue` のレイアウト変更・グループ化表示・デフォルト選択ロジック・軸数カウントロジック変更
- OpenAPI: `CustomChartSource` スキーマにフィールド追加、レスポンスに `axis_groups` 追加
- テスト: バックエンドユニットテスト・フロントエンドテストの更新

## 実現案

### バックエンド

#### `custom_chart_sources.json` 変更

`axis_groups` に `display_name` を追加する。`score1` 軸グループを削除し、`score` の `axis_group` をオブジェクト `{"label": "スコア"}` に変更する。トップレベルに `other_display_name` を追加する。

```json
{
  "max_axes": 2,
  "other_display_name": "その他",
  "axis_groups": {
    "rate_pct1": { "label": "%", "display_name": "金利・スプレッド" },
    "price_usd1": { "label": "USD", "display_name": "米国株価指数" },
    "ratio1": { "label": "倍率", "display_name": "前年比" },
    "index1": { "label": "指数", "display_name": "通貨指数" }
  },
  "sources": {
    "target_rate": {
      "name": "政策金利 (FF金利誘導目標上限)",
      "axis_group": "rate_pct1",
      "default": true,
      "source_type": "fred",
      "fred_series": ["DFEDTAR", "DFEDTARU"],
      "fred_boundary": "2008-12-16",
      "start": "1982-09-27"
    },
    "dgs10": {
      "name": "米国10年国債利回り",
      "axis_group": "rate_pct1",
      "default": true,
      "source_type": "fred",
      "fred_series": ["DGS10"],
      "start": "1982-09-27"
    },
    "baa10y": {
      "name": "米国Baa社債スプレッド",
      "axis_group": "rate_pct1",
      "default": false,
      "source_type": "fred",
      "fred_series": ["BAA10Y"],
      "start": "1986-01-01"
    },
    "sp500": {
      "name": "S&P 500",
      "axis_group": "price_usd1",
      "default": false,
      "source_type": "yfinance",
      "ticker": "^GSPC",
      "start": "1982-09-27"
    },
    "sp500_yoy": {
      "name": "S&P 500 前年比",
      "axis_group": "ratio1",
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
      "axis_group": { "label": "スコア" },
      "default": false,
      "source_type": "calculated",
      "start": "2007-01-01"
    }
  }
}
```

> **設計方針**:
> - `axis_group` の型で通常ソースと独立軸ソースを判別する
>   - 文字列 → `axis_groups` のグループを参照。`axis_label` はグループの `label` から取得
>   - オブジェクト → 独立軸ソース。`axis_label` はオブジェクトの `label` から取得
> - `axis_groups` に「その他」の定義は不要（独立軸ソースは `axis_group` の型で自動判別される）
> - 「その他」の表示名は `other_display_name` で管理する
> - 将来、独立軸ソースを追加する場合は、ソース定義に `"axis_group": {"label": "..."}` を書くだけで完結する

#### ソース一覧 API レスポンス変更

`GET /finance/custom-chart/sources` のレスポンスに `default` を追加し、`axis_groups` 情報と `other_display_name` を含める。独立軸ソースの `axis_group` は API レスポンスでは `"other"` 文字列に正規化する（フロントエンドでの判定を簡素化するため）。

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
      "id": "dgs10",
      "name": "米国10年国債利回り",
      "axis_group": "rate_pct1",
      "axis_label": "%",
      "default": true
    },
    {
      "id": "score",
      "name": "投資環境スコア（堀井）",
      "axis_group": "other",
      "axis_label": "スコア",
      "default": false
    },
    "..."
  ],
  "axis_groups": {
    "rate_pct1": { "label": "%", "display_name": "金利・スプレッド" },
    "price_usd1": { "label": "USD", "display_name": "米国株価指数" },
    "ratio1": { "label": "倍率", "display_name": "前年比" },
    "index1": { "label": "指数", "display_name": "通貨指数" }
  },
  "other_display_name": "その他",
  "max_axes": 2
}
```

> **`axis_group` の正規化**: JSON 定義では `axis_group` がオブジェクトの場合、API レスポンスでは `"other"` 文字列に変換する。フロントエンドは `axis_group === "other"` で独立軸ソースを判定できる。`axis_groups` オブジェクトに `"other"` キーは含まれないため、`axis_groups` に存在しない `axis_group` = 独立軸ソース、という判定も可能。

#### `axis_label` の取得元ロジック

サービス層でソースの `axis_label` を構築する際:
- `axis_group` が文字列の場合: `axis_groups[axis_group]["label"]` を使用
- `axis_group` がオブジェクトの場合: `axis_group["label"]` を使用

### フロントエンド

#### レイアウト変更

```
/custom-chart (CustomChartPage.vue)
├── チャート表示エリア（Lightweight Charts）
├── 警告メッセージ表示エリア（軸数超過時）
└── ソース選択パネル
    ├── グループ化チェックリスト
    │   ├── 金利・スプレッド
    │   │   ├── ☑ 政策金利 (FF金利誘導目標上限)
    │   │   ├── ☑ 米国10年国債利回り
    │   │   └── ☐ 米国Baa社債スプレッド
    │   ├── 米国株価指数
    │   │   └── ☐ S&P 500
    │   ├── 前年比
    │   │   └── ☐ S&P 500 前年比
    │   ├── 通貨指数
    │   │   └── ☐ ドル指数 (実効為替レート)
    │   └── その他
    │       └── ☐ 投資環境スコア（堀井）
    └── 反映ボタン
```

#### デフォルト選択フロー

1. `fetchSources()` でソース一覧を取得
2. `default: true` のソースを自動的にチェック状態にする
3. デフォルトソースが1つ以上ある場合、自動的に `applySelection()`（反映処理）を実行してチャートを描画する
4. ユーザーが反映ボタンを押した場合は従来通りの動作

#### グループ化チェックリスト描画

- API レスポンスの `axis_groups` のキー順で通常グループを描画する
- 通常グループの後に「その他」グループを描画する（`other_display_name` をヘッダーに使用）
- 「その他」グループには `axis_group === "other"` のソースをまとめる
- 各グループのヘッダーには `display_name`（通常グループ）または `other_display_name` を表示する
- グループ内のソースは API レスポンスの `sources` 配列順で描画する

#### 軸数カウントロジック変更

反映ボタン押下時の軸数チェックを以下のように変更する:

```
軸数 = (選択されたソースの通常axis_groupの種類数) + (選択されたotherソースの数)
```

- 通常の `axis_group`（`rate_pct1`, `price_usd1` 等）: 何個選択してもグループで1軸
- `other`（独立軸ソース）: 選択されたソース1つにつき1軸
- 合計が `max_axes` を超える場合は警告を表示し、チャートは更新しない

### OpenAPI 変更

```yaml
CustomChartSource:
  type: object
  properties:
    id:
      type: string
    name:
      type: string
    axis_group:
      type: string
      description: >
        軸グループID。axis_groupsに定義されたグループIDまたは "other"。
        "other" の場合はソースごとに独立した軸を消費する。
    axis_label:
      type: string
      description: >
        軸の表示ラベル。通常ソースはaxis_groupsのlabelから、
        独立軸ソース(other)はソース固有のlabelから取得。
    default:
      type: boolean
      description: デフォルトで選択状態にするかどうか
  required: [id, name, axis_group, axis_label, default]

CustomChartAxisGroup:
  type: object
  properties:
    label:
      type: string
      description: 軸の単位ラベル
    display_name:
      type: string
      description: グループの日本語表示名
  required: [label, display_name]

CustomChartSourcesResponse:
  type: object
  properties:
    sources:
      type: array
      items:
        $ref: "#/components/schemas/CustomChartSource"
    axis_groups:
      type: object
      additionalProperties:
        $ref: "#/components/schemas/CustomChartAxisGroup"
      description: 軸グループ定義（キー順が表示順）。独立軸ソース(other)は含まれない
    other_display_name:
      type: string
      description: 独立軸ソースの表示グループ名
      example: "その他"
    max_axes:
      type: integer
      example: 2
  required: [sources, axis_groups, other_display_name, max_axes]
```

## テスト計画

### バックエンド ユニットテスト

| # | テストケース | 検証内容 |
|---|------------|---------|
| S1 | `get_sources` に `default` フィールド | 各ソースに `default` が含まれ、`target_rate` と `dgs10` が `true` |
| S2 | `get_sources` に `axis_groups` | レスポンスに `axis_groups` が含まれ、各グループに `display_name` がある |
| S3 | `get_sources` に `other_display_name` | レスポンスに `other_display_name` が含まれる |
| S4 | 独立軸ソースの `axis_group` 正規化 | `score` の `axis_group` が `"other"` に正規化されている |
| S5 | 独立軸ソースの `axis_label` | `score` の `axis_label` が `"スコア"` （ソース定義のオブジェクトから取得） |
| S6 | 通常ソースの `axis_label` | `target_rate` の `axis_label` が `"%"` （`axis_groups` の `label` から取得） |

### フロントエンド手動確認

| # | 確認項目 |
|---|---------|
| F1 | 初回ロード時に政策金利と米国10年債利回りがチェック済みでチャートに描画されている |
| F2 | チェックボックスが軸グループごとにまとまり、`display_name` がグループヘッダーに表示されている |
| F3 | 独立軸ソース（投資環境スコア）が「その他」グループに表示されている |
| F4 | チャートが上、チェックボックスが下に配置されている |
| F5 | 警告メッセージがチャートとチェックボックスの間に表示される |
| F6 | デフォルト選択を解除して反映 → チャートがクリアされる |
| F7 | 独立軸ソース1つ + 通常1グループ選択 → 2軸で描画される |
| F8 | 独立軸ソース1つ + 通常2グループ選択 → 3軸超過で警告 |
| F9 | 既存の機能（反映ボタン、軸数制約チェック等）が引き続き動作する |

## 関連

- [enhancements/001_custom_chart.md](001_custom_chart.md) - カスタムチャート機能の元仕様（ENH-001）
- [Backend/main/src/custom_chart_sources.json](../Backend/main/src/custom_chart_sources.json) - ソース定義ファイル
- [Frontend/finance-dashboard/src/pages/CustomChartPage.vue](../Frontend/finance-dashboard/src/pages/CustomChartPage.vue) - カスタムチャートページ
