# 005: カスタムチャート axis_groups 統合・local_groups 導入（ENH-004）

## 対象ファイル

- `docs/openapi_main.yaml`
- `docs/user_stories.md`

## 変更内容

### `docs/openapi_main.yaml`

- `CustomChartSource` スキーマ:
  - `axis_group` の description から `"other"` の記述を削除。`axis_groups` のキーを参照する旨に変更
  - `independent_group` を削除し、`local_group`（string, optional）に置き換え
  - `axis_label` の description を `independent` フラグベースの説明に更新
- `CustomChartAxisGroup` スキーマ:
  - `label` を optional に変更（`independent: true` の場合は存在しない）
  - `independent`（boolean, required）を追加
  - `local_groups`（optional, `additionalProperties: CustomChartLocalGroup`）を追加
  - `required` を `[display_name, independent]` に変更（`label` を除外）
- `CustomChartIndependentGroup` スキーマを削除し、`CustomChartLocalGroup` スキーマを新規追加（`display_name` のみ、required）
- `CustomChartSourcesResponse` スキーマ:
  - `independent_groups` を削除
  - `other_display_name` を削除
  - `axis_groups` の description を「通常・独立軸の両方を含む」に更新
  - `required` を `[sources, axis_groups, max_axes]` に変更
- `CustomChartSeries` スキーマ:
  - `axis_group` の description から `"other"` の記述を削除

### `docs/user_stories.md`

- US-8.4（グループ化表示）の受け入れ条件を更新:
  - サブカテゴリ表示について追記
  - サブカテゴリに属さないソースのグループ先頭表示について追記
- US-8.5（独立軸ソース）の受け入れ条件を更新:
  - サブカテゴリによる分類が可能である旨に変更

## 変更理由

- `axis_group` の辞書形式と文字列形式の二重構造を廃止し、`axis_groups` に統合することで設計を一般化する
- `independent_groups` を `axis_groups` 内の `local_groups` に統合し、通常軸・独立軸の両方でサブカテゴリを利用可能にする
- `independent` フラグを必須にし、軸の独立性を明示的に宣言する

## 影響を受けるユニット

### インフラ（Infra）

- 対応不要

### フロントエンド（Frontend）

- Orval による型再生成が必要
- `CustomChartPage.vue` のグループ化ロジック変更（`"other"` 判定 → `independent` フラグ判定、`local_groups` 対応の一般化）
- MSW モックデータの更新

### バックエンド・メイン（Backend/main）

- `custom_chart_sources.json` の構造変更（`independent_groups` 廃止、`axis_groups` に統合、`local_groups` 追加）
- `custom_chart_service.py` の `_resolve_axis` リファクタリング（`isinstance` 分岐廃止）、`get_sources` レスポンス変更
- `bin/validate_config.py` の新規作成

### CI/CD

- 影響なし

## 関連

- [enhancements/004_custom_chart_unified_axis_groups.md](../enhancements/004_custom_chart_unified_axis_groups.md)
- [docs/history/004_custom_chart_independent_groups.md](004_custom_chart_independent_groups.md) — ENH-003 の変更通知
