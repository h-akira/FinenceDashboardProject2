# 003: カスタムチャート UX 改善（ENH-002）

## 対象ファイル

- `docs/openapi_main.yaml` — `CustomChartSource` に `default` 追加、`CustomChartAxisGroup` スキーマ新規追加、`CustomChartSourcesResponse` に `axis_groups`・`other_display_name` 追加
- `docs/user_stories.md` — セクション6を「バックエンドデータ チャート」に改名しダッシュボードの金利チャート（US-6.1）を分離、セクション7（ナビゲーション）は番号維持、セクション8「カスタムチャート」新設（US-8.1〜8.5）
- `docs/units_definition.md` — US-6.1 の説明更新、US-8.1〜8.5 行追加

## 変更内容

### `docs/openapi_main.yaml`

- `CustomChartSource` スキーマに `default`（boolean, required）を追加
- `CustomChartSource.axis_group` の description を更新（`"other"` の説明追加）
- `CustomChartAxisGroup` スキーマを新規追加（`label` + `display_name`）
- `CustomChartSourcesResponse` に以下を追加:
  - `axis_groups`: 軸グループ定義（`additionalProperties: CustomChartAxisGroup`）
  - `other_display_name`: 独立軸ソースの表示グループ名
- `CustomChartSeries.axis_group` の description 更新

### `docs/user_stories.md`

- 旧セクション6「カスタムチャート（バックエンドデータ）」を再構成:
  - セクション6「バックエンドデータ チャート」（US-6.1: ダッシュボードの金利チャートウィジェット）
  - セクション7「ナビゲーション」（US-7.1: 番号維持）
  - セクション8「カスタムチャート」（US-8.1〜8.5: カスタムチャートページの機能）
- US-8.1: ソース選択と表示（旧 US-6.2 相当）
- US-8.2: 軸グループ制約（旧 US-6.3 相当）
- US-8.3: デフォルト表示（ENH-002 新規）
- US-8.4: グループ化表示（ENH-002 新規）
- US-8.5: 独立軸ソース（ENH-002 新規）

### `docs/units_definition.md`

- US-6.1 の説明を「バックエンドデータ チャート表示」に更新
- US-8.1〜8.5（カスタムチャート）の行を追加

## 変更理由

- ENH-002 の実装に伴い、API レスポンスにデフォルト選択・軸グループ情報を追加する必要があった
- `user_stories.md` のセクション6にダッシュボードの固定金利チャート（US-6.1）とカスタムチャートページ（US-6.2〜）が混在していた。これらは別機能（別 API・別ページ）であるため、セクションを分離して整理した
- US 番号の変更（旧 6.2〜6.6 → 8.1〜8.5）は規約上は原則禁止だが、旧番号が外部から参照されていないことを確認の上、特例として実施した

## 影響を受けるユニット

### インフラ（Infra）

- 対応不要

### フロントエンド（Frontend）

- Orval による型再生成が必要
- `CustomChartPage.vue` のレイアウト変更・デフォルト選択・グループ化表示の実装

### バックエンド・メイン（Backend/main）

- `custom_chart_sources.json` の変更（`default`・`display_name`・`other_display_name` 追加、`score` の `axis_group` オブジェクト化）
- `custom_chart_service.py` の `get_sources` レスポンス変更

### CI/CD

- 影響なし

## 関連

- [enhancements/002_custom_chart_ux.md](../enhancements/002_custom_chart_ux.md)
- [docs/history/002_add_custom_chart_api.md](002_add_custom_chart_api.md) — ENH-001 の変更通知
