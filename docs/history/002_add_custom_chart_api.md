# 002: カスタムチャート API の追加（ENH-001）

## 対象ファイル

- `docs/openapi_main.yaml` — カスタムチャート関連スキーマ・エンドポイント追加

## 変更内容

`docs/openapi_main.yaml` にカスタムチャート機能の API 定義を追加した。

### スキーマ追加

- `CustomChartSource` — データソース定義
- `CustomChartSourcesResponse` — ソース一覧レスポンス
- `CustomChartDataPoint` — 時系列データポイント
- `CustomChartSeries` — 1ソースの時系列データ
- `CustomChartDataResponse` — データ取得レスポンス

### エンドポイント追加

- `GET /api/v1/main/finance/custom-chart/sources` — ソース一覧取得
- `GET /api/v1/main/finance/custom-chart/data` — チャートデータ取得

## 変更理由

- ENH-001（カスタムチャート機能）の実装に伴い、フロントエンド・バックエンド間の API 契約を定義する必要があった
- ダッシュボードの固定金利チャート（`/finance/interest-rate`）とは別に、ユーザーが自由にソースを選択できるカスタムチャート用の API を新設した

## 影響を受けるユニット

### インフラ（Infra）

- 対応不要（API Gateway のルーティングはバックエンドの Lambda 統合で処理）

### フロントエンド（Frontend）

- Orval による型再生成が必要
- `CustomChartPage.vue` の新規作成で API を利用

### バックエンド・メイン（Backend/main）

- `custom_chart_service.py` の新規作成
- `app.py` にルーティング追加

### CI/CD

- 影響なし

## 関連

- [enhancements/001_custom_chart.md](../enhancements/001_custom_chart.md)
