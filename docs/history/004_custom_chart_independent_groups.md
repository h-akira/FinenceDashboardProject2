# 004: カスタムチャート 独立軸サブグループ化・単位表示簡素化（ENH-003）

## 対象ファイル

- `docs/openapi_main.yaml` — `CustomChartSource` に `independent_group` 追加、`CustomChartIndependentGroup` スキーマ新規追加、`CustomChartSourcesResponse` に `independent_groups` 追加、`other_display_name` の example を「独立軸」に変更
- `docs/user_stories.md` — US-8.4（グループ化表示）に単位表示の簡素化を反映、US-8.5（独立軸ソース）にサブグループ化・「独立軸」改名を反映

## 変更内容

### `docs/openapi_main.yaml`

- `CustomChartSource` スキーマに `independent_group`（string, optional）を追加
  - 独立軸ソース（`axis_group === "other"`）の場合のみ存在し、`independent_groups` のキーを参照する
- `CustomChartIndependentGroup` スキーマを新規追加（`display_name` のみ）
- `CustomChartSourcesResponse` に `independent_groups`（`additionalProperties: CustomChartIndependentGroup`）を追加し、required に含めた
- `other_display_name` の example を「その他」→「独立軸」に変更

### `docs/user_stories.md`

- US-8.4（グループ化表示）の受け入れ条件を更新:
  - 通常グループのヘッダーに単位を含める形式（「金利・スプレッド（%）」）
  - 通常グループ内の各ソースには単位を表示しない
- US-8.5（独立軸ソース）の受け入れ条件を更新:
  - 「その他」→「独立軸」に改名
  - サブグループで分類される旨を追加
  - ソース横にソース固有の単位を表示する旨を追加

## 変更理由

- 同一軸グループ内のソースは全て同一単位であるため、各チェックボックスへの単位表示は冗長である。グループヘッダーに集約することで UI を簡素化する
- 独立軸ソースが将来的に増加した場合、「その他」に一括表示すると分類がなく見通しが悪い。`independent_groups` による分類の仕組みを導入し、「独立軸」に改名することで意味を明確にする

## 影響を受けるユニット

### インフラ（Infra）

- 対応不要

### フロントエンド（Frontend）

- Orval による型再生成が必要
- `CustomChartPage.vue` の単位表示変更・独立軸サブグループ描画
- MSW モックデータの更新

### バックエンド・メイン（Backend/main）

- `custom_chart_sources.json` の変更（`other_display_name` 変更、`independent_groups` 追加、`score` の `axis_group` に `independent_group` 追加）
- `custom_chart_service.py` の `get_sources` レスポンスに `independent_groups` 追加、`independent_group` フィールドの処理

### CI/CD

- 影響なし

## 関連

- [enhancements/003_custom_chart_ux3.md](../enhancements/003_custom_chart_ux3.md)
- [docs/history/003_custom_chart_ux_improvements.md](003_custom_chart_ux_improvements.md) — ENH-002 の変更通知
