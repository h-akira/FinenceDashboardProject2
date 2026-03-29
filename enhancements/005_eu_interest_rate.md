# ENH-005: 欧州の政策金利・長期金利データの追加

## ステータス

完了（検証済み）

## 起票日

2026-03-29

## 種別

仕様追加

## 概要

欧州（ECB 政策金利・ドイツ 10 年国債利回り）のデータソースをカスタムチャートに追加する。

## 背景・動機

現在、米国の政策金利（FF 金利）と長期金利（米国 10 年国債）のみ対応している。欧州の金利データを追加することで、米欧の金利比較が可能になる。

## 要件

- ECB 政策金利（MRO: Main Refinancing Operations Rate）を追加する
  - FRED シリーズ ID: `ECBMRRFR`
  - 日次データ、半月単位サンプリング可能
  - 開始日: 1999-01-01（ECB 発足）
  - ソース ID: `ecb_mro_rate`
- ドイツ 10 年国債利回りを追加する
  - FRED シリーズ ID: `IRLTLT01DEM156N`
  - 月次データのみ（FRED・yfinance ともに日次データが存在しないため）
  - 開始日: 1999-01-01
  - ソース ID: `de_10y`
- `rate_pct1` axis_group に `eu` local_group を追加する

## 影響範囲

- `Backend/main/src/custom_chart_sources.json`
- `Backend/main/bin/load_data.py`
- `Backend/main/src/services/custom_chart_service.py`
- `Backend/main/docs/data_policies.md`
- `Backend/main/docs/database_design.md`

## 実現案

### 案 1: FRED pandas_datareader 経由（採用）

既存の FRED データ取得パターンに合わせる。`ecb_mro_rate` は日次→半月サンプリング、`de_10y` は月次データをそのまま使用（半月サンプリング不可）。

## 対応

案 1 を採用。既存の FRED データ取得パターンをそのまま利用し、`load_data.py` / `custom_chart_service.py` のコード変更なしでデータソース定義の追加のみで対応。

変更ファイル:
- `Backend/main/src/custom_chart_sources.json`: `ecb_mro_rate`, `de_10y` ソース定義追加、`rate_pct1.local_groups` に `eu` 追加
- `Backend/main/docs/data_policies.md`: データ系列に `ecb_mro_rate`, `de_10y` 追記、月次データの注記追加
- `Backend/main/docs/database_design.md`: PK パターン・アクセスパターンに `ecb_mro_rate`, `de_10y` 追記
- `Backend/main/tests/local/test_services.py`: ソース数 7→9
- `Backend/main/tests/local/test_routes.py`: ソース数 7→9
- `Frontend/finance-dashboard/src/mocks/handlers.ts`: モックにソース・axis_groups・データ生成追加
- `Frontend/tests/test_custom_chart.py`: ソース数・サブグループ数・nth インデックス修正

`load_data.py` および `custom_chart_service.py` はコード変更不要（汎用的な FRED 取得ロジックがそのまま動作）。dry-run テストで `ecb_mro_rate` 648 件、`de_10y` 324 件の取得を確認済み。

## 関連

- FRED ECBMRRFR: https://fred.stlouisfed.org/series/ECBMRRFR
- FRED IRLTLT01DEM156N: https://fred.stlouisfed.org/series/IRLTLT01DEM156N
