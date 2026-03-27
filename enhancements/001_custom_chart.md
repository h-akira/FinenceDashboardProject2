# ENH-001: カスタムチャート機能

## ステータス

未着手

## 起票日

2026-03-26

## 種別

仕様追加

## 概要

複数の金融データソースをユーザーが自由に組み合わせてチャート表示できるカスタムチャート機能を追加する。ダッシュボードとは別の専用ページとして提供する。

## 背景・動機

現在のダッシュボードでは政策金利・長期金利が固定のウィジェットとして表示されており、他の指標（S&P 500、Baa社債スプレッド、ドル指数、複合スコアなど）は experimental で個別に検証されている状態にある。これらの指標をユーザーが任意に組み合わせて一つのチャート上で比較・分析できるようにすることで、ダッシュボードの分析能力を大幅に向上させる。

## 要件

### 機能要件

- ダッシュボードとは別の専用ページ (`/custom-chart`) を設け、ナビゲーションバーから遷移できること
- バックエンドから利用可能なデータソース一覧を取得し、チェックリストとして表示すること
- ユーザーがチェックリストでソースを選択し、**反映ボタン**を押すとデータを取得してチャートに描画すること（チェック変更だけでは反映しない）
- Y軸は最大2つまでとし、異なる軸グループのソースは別の軸に割り当てること
- 反映ボタン押下時に選択されたソースの軸グループが3種類以上ある場合は、**警告メッセージを表示し、チャートは更新しない**
- 軸の最大数（初期値: 2）はバックエンドでソースコード直書きの設定値として管理し、ソース一覧 API のレスポンスに含めてフロントエンドに伝達する。将来的に軸数の上限を増やす場合はバックエンドの設定値変更のみで対応できる設計とする
- チャートの選択状態を保存する機能は不要（毎回選択する）

### 軸グループと制約ルール

各ソースは `axis_group` を持ち、同じ `axis_group` のソースは同じ Y 軸を共有する。`axis_group` は単位とスケールの両方を考慮したグルーピングであり、同じ単位でもスケールが大きく異なる場合は別の `axis_group` に分ける。

`axis_group` の命名規則は `{意味}_{単位}{連番}` とする（例: `rate_pct1`）。連番は同一系統でスケールが異なるものが追加された場合の拡張用。

- 同一 `axis_group` のソースは同じ Y 軸を共有する
- 最大軸数はバックエンドから取得した `max_axes` 値に従う（初期値: 2）
- 反映ボタン押下時にフロントエンドで `axis_group` の種類数をチェックし、`max_axes` を超える場合は警告を表示してデータ取得を行わない

### データ仕様

- 全ソース共通で半月次（1日付近と15日付近、または近傍の営業日）のデータを返す
- 2025年以前のデータは DynamoDB に格納し、2026年以降のデータは外部 API（FRED, yfinance 等）からリアルタイム取得して結合する（既存の金利データと同じ方式）

## データソース定義

バックエンドのソースコードで直書き管理する。初期リリース時のソース一覧:

| ID | 名称 | 軸グループ | 軸ラベル | FRED Series / データ元 | 開始年 |
|---|---|---|---|---|---|
| `target_rate` | 政策金利 (FF金利誘導目標上限) | `rate_pct1` | % | DFEDTAR / DFEDTARU | 1982 |
| `dgs10` | 米国10年国債利回り | `rate_pct1` | % | DGS10 | 1982 |
| `baa10y` | Baa社債スプレッド | `rate_pct1` | % | BAA10Y | 1986 |
| `sp500` | S&P 500 | `price_usd1` | USD | yfinance `^GSPC` | 1982 |
| `sp500_yoy` | S&P 500 前年比 | `ratio1` | 倍率 | yfinance `^GSPC` (YoY) | 1983 |
| `dtwexbgs` | ドル指数 (実効為替レート) | `index1` | 指数 | DTWEXBGS | 2006 |
| `score` | 複合スコア | `score1` | スコア | 算出値（EFFR, DGS10, BAA10Y, DTWEXBGS から） | 2007 |

### 軸グループ一覧

| 軸グループ | 軸ラベル | 該当ソース | 備考 |
|---|---|---|---|
| `rate_pct1` | % | target_rate, dgs10, baa10y | 金利系（0〜20%程度のスケール） |
| `price_usd1` | USD | sp500 | 株価指数（数千USD） |
| `ratio1` | 倍率 | sp500_yoy | 前年比（0.5〜2.0程度） |
| `index1` | 指数 | dtwexbgs | 通貨指数（80〜130程度） |
| `score1` | スコア | score | 複合スコア（-10〜+10） |

> 将来的にソースが追加される可能性がある。ソース定義の追加はバックエンドのソースコードへの追記のみで完結する設計とする。同一系統でスケールが異なるソースが追加された場合は、連番を増やして新しい軸グループを作成する（例: `rate_pct2`）。

## 影響範囲

- Backend: 新規エンドポイント追加（ソース一覧取得、ソースデータ取得）
- Backend: DynamoDB へ新規データ種別の格納（既存の PK/SK スキーマを流用）
- Frontend: 新規ページ・コンポーネント追加
- Frontend: ナビゲーションバー・ルーター変更
- OpenAPI: 新規エンドポイント定義追加
- SAM テンプレート: Lambda IAM ポリシー変更不要（既存の DynamoDB Query/GetItem 権限で対応可能）

## 実現案

### バックエンド

#### ソース定義ファイル（JSON）

ソース定義・軸グループ・設定値は `Backend/main/src/custom_chart_sources.json` に JSON ファイルとして配置する。`CodeUri: src/` により Lambda デプロイパッケージに同梱され、Lambda 本体から参照できる。また、`bin/` 配下のデータ投入スクリプトからも相対パスで同じファイルを参照することで、ソース定義の一元管理を実現する。

```json
{
  "max_axes": 2,
  "axis_groups": {
    "rate_pct1": { "label": "%" },
    "price_usd1": { "label": "USD" },
    "ratio1": { "label": "倍率" },
    "index1": { "label": "指数" },
    "score1": { "label": "スコア" }
  },
  "sources": {
    "target_rate": {
      "name": "政策金利 (FF金利誘導目標上限)",
      "axis_group": "rate_pct1",
      "fred_series": ["DFEDTAR", "DFEDTARU"],
      "fred_boundary": "2008-12-16"
    },
    "dgs10": {
      "name": "米国10年国債利回り",
      "axis_group": "rate_pct1",
      "fred_series": ["DGS10"]
    },
    "baa10y": {
      "name": "Baa社債スプレッド",
      "axis_group": "rate_pct1",
      "fred_series": ["BAA10Y"]
    },
    "sp500": {
      "name": "S&P 500",
      "axis_group": "price_usd1",
      "source_type": "yfinance",
      "ticker": "^GSPC"
    },
    "sp500_yoy": {
      "name": "S&P 500 前年比",
      "axis_group": "ratio1",
      "source_type": "yfinance",
      "ticker": "^GSPC",
      "transform": "yoy_ratio"
    },
    "dtwexbgs": {
      "name": "ドル指数 (実効為替レート)",
      "axis_group": "index1",
      "fred_series": ["DTWEXBGS"]
    },
    "score": {
      "name": "複合スコア",
      "axis_group": "score1",
      "source_type": "calculated"
    }
  }
}
```

**参照方法:**

- **Lambda 本体** (`src/`): `pathlib.Path(__file__).parent / "custom_chart_sources.json"` で読み込み
- **データ投入スクリプト** (`bin/`): `pathlib.Path(__file__).parent / ".." / "src" / "custom_chart_sources.json"` で読み込み

#### API エンドポイント

**1. ソース一覧取得: `GET /finance/custom-chart/sources`**

フロントエンドがチェックリストを構築するために使用する。

```json
{
  "sources": [
    {
      "id": "target_rate",
      "name": "政策金利 (FF金利誘導目標上限)",
      "axis_group": "rate_pct1",
      "axis_label": "%"
    },
    {
      "id": "sp500",
      "name": "S&P 500",
      "axis_group": "price_usd1",
      "axis_label": "USD"
    }
  ],
  "max_axes": 2
}
```

**2. ソースデータ取得: `GET /finance/custom-chart/data?sources=target_rate,dgs10,sp500`**

クエリパラメータで指定されたソースのデータを返す。

```json
{
  "series": [
    {
      "id": "target_rate",
      "name": "政策金利 (FF金利誘導目標上限)",
      "axis_group": "rate_pct1",
      "axis_label": "%",
      "data": [
        { "time": "2024-01-15", "value": 5.33 },
        { "time": "2024-01-31", "value": 5.33 }
      ]
    },
    {
      "id": "sp500",
      "name": "S&P 500",
      "axis_group": "price_usd1",
      "axis_label": "USD",
      "data": [
        { "time": "2024-01-15", "value": 4783.45 },
        { "time": "2024-01-31", "value": 4845.65 }
      ]
    }
  ]
}
```

#### データ取得ロジック

各ソースについて既存の `_fetch_fred_series` と同じ半月次リサンプリングロジックを適用する。

- DynamoDB から 2025 年以前のデータを `KIND#{source_id}` で取得
- 2026 年以降は FRED / yfinance からリアルタイム取得
- 結合・重複排除は既存の金利データと同じ方式

yfinance ソース（S&P 500）の場合も同様に半月次でリサンプリングし、前年比は取得後に算出する。複合スコアは構成要素（EFFR, DGS10, BAA10Y, DTWEXBGS）を取得してから算出する。

### フロントエンド

#### ページ構成

```
/custom-chart (CustomChartPage.vue)
├── ソース選択パネル
│   ├── チェックリスト（ソース一覧）
│   └── 反映ボタン
├── 警告メッセージ表示エリア（軸数超過時）
└── チャート表示エリア（Lightweight Charts）
```

#### チェックリスト UI と反映フロー

- `GET /finance/custom-chart/sources` でソース一覧と `max_axes` を取得
- 各ソースをチェックボックスで表示（軸ラベルをバッジ等で表示）
- チェックボックスの選択は自由に行える（選択時点での制約チェックは行わない）
- **反映ボタン**を設置し、押下時に以下のフローを実行:
  1. 選択されたソースの `axis_group` の種類数をカウント
  2. 種類数が `max_axes` を超える場合: 警告メッセージ（例:「Y軸は最大2種類までです。選択を見直してください」）を表示し、データ取得もチャート更新も行わない
  3. 種類数が `max_axes` 以内の場合: `GET /finance/custom-chart/data` を呼び出してチャートを描画・更新

#### チャート描画

- Lightweight Charts を使用（既存の InterestRateWidget と同じライブラリ）
- 左 Y 軸と右 Y 軸で最大2軸グループに対応（軸ラベルを表示）
- 同一 `axis_group` のシリーズは同じ Y 軸を共有
- シリーズごとに色を変え、凡例を表示
- `timeScale().fitContent()` で全データが表示されるよう自動調整

#### ナビゲーション

- ナビゲーションバーに「カスタムチャート」リンクを追加
- ルーターに `/custom-chart` を追加（`requiresAuth: true`）

### OpenAPI 追加定義

```yaml
# ----- Custom Chart -----
CustomChartSource:
  type: object
  properties:
    id:
      type: string
    name:
      type: string
    axis_group:
      type: string
      description: 軸グループID（同一グループは同じY軸を共有）
    axis_label:
      type: string
      description: 軸の表示ラベル（例 "%" "USD"）
  required: [id, name, axis_group, axis_label]

CustomChartSourcesResponse:
  type: object
  properties:
    sources:
      type: array
      items:
        $ref: "#/components/schemas/CustomChartSource"
    max_axes:
      type: integer
      description: Y軸の最大数
      example: 2
  required: [sources, max_axes]

CustomChartDataPoint:
  type: object
  properties:
    time:
      type: string
    value:
      type: number
      format: double
  required: [time, value]

CustomChartSeries:
  type: object
  properties:
    id:
      type: string
    name:
      type: string
    axis_group:
      type: string
    axis_label:
      type: string
    data:
      type: array
      items:
        $ref: "#/components/schemas/CustomChartDataPoint"
  required: [id, name, axis_group, axis_label, data]

CustomChartDataResponse:
  type: object
  properties:
    series:
      type: array
      items:
        $ref: "#/components/schemas/CustomChartSeries"
  required: [series]
```

### DynamoDB データ格納

既存の PK/SK スキーマをそのまま流用する。

```
PK: KIND#sp500         SK: TIME#2024-01-15   value: 4783.45
PK: KIND#sp500         SK: TIME#2024-01-31   value: 4845.65
PK: KIND#baa10y        SK: TIME#2024-01-15   value: 2.15
PK: KIND#dtwexbgs      SK: TIME#2024-01-15   value: 123.45
...
```

既存の `target_rate`, `dgs10` データはそのまま活用する。

## テスト計画

### Phase 1: データ取得検証（bin/load_data.py）

`--dry-run` で各ソースのデータ取得・リサンプリングが正しく動作するか確認する。外部 API（FRED, yfinance）への接続を含む実データテスト。

| # | テストケース | コマンド | 確認内容 |
|---|------------|---------|---------|
| D1 | 全ソース一括 dry-run | `python bin/load_data.py --env dev --dry-run` | 全7ソースのデータ取得成功、レコード数が妥当 |
| D2 | FRED 系個別 (target_rate) | `--sources target_rate --dry-run` | DFEDTAR/DFEDTARU の結合、半月次リサンプリング |
| D3 | FRED 系個別 (dgs10) | `--sources dgs10 --dry-run` | DGS10 取得、半月次リサンプリング |
| D4 | FRED 系個別 (baa10y) | `--sources baa10y --dry-run` | BAA10Y 取得（1986年〜） |
| D5 | FRED 系個別 (dtwexbgs) | `--sources dtwexbgs --dry-run` | DTWEXBGS 取得（2006年〜） |
| D6 | yfinance 系 (sp500) | `--sources sp500 --dry-run` | ^GSPC 取得、半月次リサンプリング |
| D7 | yfinance YoY (sp500_yoy) | `--sources sp500_yoy --dry-run` | 前年比算出、1年ルックバック |
| D8 | 算出値 (score) | `--sources score --dry-run` | 複合スコア算出（EFFR, DGS10, BAA10Y, DTWEXBGS から） |
| D9 | 無効なソースID | `--sources invalid_id` | エラーメッセージが表示される |

### Phase 2: バックエンド ユニットテスト（pytest）

既存テストパターン（moto + conftest）に合わせて追加。外部 API はモックする。

#### サービス層 (test_services.py)

| # | テストケース | 検証内容 |
|---|------------|---------|
| S1 | `get_sources` 正常系 | 7ソース返却、`max_axes=2`、各ソースに `id/name/axis_group/axis_label` |
| S2 | `get_data` DynamoDB のみ | DynamoDB のデータが正しく返却される |
| S3 | `get_data` DynamoDB + 外部API結合 | 重複排除（DynamoDB優先）が正しく動作する |
| S4 | `get_data` 無効なソースID | `ValidationError` が発生する |

#### ルート層 (test_routes.py)

| # | テストケース | 検証内容 |
|---|------------|---------|
| R1 | `GET /custom-chart/sources` 正常系 | 200, `sources` 配列と `max_axes` |
| R2 | `GET /custom-chart/data?sources=target_rate,dgs10` 正常系 | 200, `series` 配列 |
| R3 | `GET /custom-chart/data` パラメータなし | 400 |
| R4 | `GET /custom-chart/data?sources=invalid_id` | 400 |
| R5 | `GET /custom-chart/data?sources=` 空文字 | 400 |

### Phase 3: フロントエンド手動確認

| # | 確認項目 |
|---|---------|
| F1 | `/custom-chart` にアクセスでき、ソース一覧がチェックリストで表示される |
| F2 | ナビバーに「カスタムチャート」リンクがあり遷移できる |
| F3 | ソース選択 → 反映ボタンでチャートが描画される |
| F4 | 軸グループ3種以上選択 → 反映ボタンで警告メッセージが表示される |
| F5 | 未認証時に `/custom-chart` にアクセスするとホームにリダイレクトされる |
| F6 | 選択なしで反映ボタン → チャートがクリアされる |

### テスト結果

#### Phase 1: データ取得検証（2026-03-27 実施）

| # | ソース | 結果 | レコード数 | 期間 | 備考 |
|---|--------|------|-----------|------|------|
| D2 | target_rate | ✅ OK | 1039 | 1982-09-27 〜 2025-12-15 | DFEDTAR/DFEDTARU 結合正常 |
| D3 | dgs10 | ✅ OK | 1039 | 1982-09-27 〜 2025-12-15 | |
| D4 | baa10y | ✅ OK | 960 | 1986-01-02 〜 2025-12-15 | |
| D5 | dtwexbgs | ✅ OK | 480 | 2006-01-02 〜 2025-12-15 | |
| D6 | sp500 | ✅ OK | 1039 | 1982-09-27 〜 2025-12-15 | |
| D7 | sp500_yoy | ✅ OK | 1015 | 1983-09-27 〜 2025-12-16 | 前年比算出正常（値域 0.5〜2.0 程度） |
| D8 | score | ✅ OK | 456 | 2007-01-02 〜 2025-12-16 | 複合スコア算出正常（値域 -10〜+10） |
| D9 | 無効ID | ✅ OK | - | - | エラーメッセージ正常表示 |

#### Phase 2: バックエンド ユニットテスト（2026-03-27 実施）

テスト用 venv に yfinance 未インストールだったため追加。既存テスト含め全 26 件パス。

##### 前提修正
- テスト用 venv (`env/`) に `yfinance` をインストール（`custom_chart_service.py` の import で既存ルートテストも全滅していた）

##### サービス層 (test_services.py)

| # | テストケース | 結果 |
|---|------------|------|
| S1 | `get_sources` 正常系（7ソース、max_axes=2） | ✅ OK |
| S2 | `get_data` DynamoDB のみ | ✅ OK |
| S3 | `get_data` 重複排除（DynamoDB優先） | ✅ OK |
| S4 | `get_data` 無効なソースID → ValidationError | ✅ OK |

##### ルート層 (test_routes.py)

| # | テストケース | 結果 |
|---|------------|------|
| R1 | `GET /custom-chart/sources` → 200 | ✅ OK |
| R2 | `GET /custom-chart/data?sources=target_rate,dgs10` → 200 | ✅ OK |
| R3 | `GET /custom-chart/data` パラメータなし → 400 | ✅ OK |
| R4 | `GET /custom-chart/data?sources=invalid_id` → 400 | ✅ OK |
| R5 | `GET /custom-chart/data?sources=` 空文字 → 400 | ✅ OK |

#### Phase 3: フロントエンド確認（2026-03-27 実施）

Playwright による自動テスト + MSW モックで検証。

##### バグ修正
- `CustomChartPage.vue`: `onMounted` で `initChart()` → `fetchSources()` の順に呼んでいたが、`fetchSources` が `sourcesLoading = true` にすると `v-if/v-else` により `chartContainer` の DOM が消え、`initChart()` が空振りしていた。`fetchSources` 完了後に `nextTick` → `initChart()` を呼ぶよう修正
- `CustomChartPage.vue`: `setData` 前にデータを time でソートするよう修正（Lightweight Charts の要件）

##### テスト結果

| # | 確認項目 | 結果 |
|---|---------|------|
| F1 | ソース一覧がチェックリスト（7件）で表示される | ✅ OK |
| F2 | ナビバーから「カスタムチャート」に遷移できる | ✅ OK |
| F3 | ソース選択 → 反映ボタンでチャートが描画される | ✅ OK |
| F4 | 軸グループ3種以上 → 警告メッセージ表示 | ✅ OK |
| F5 | 未認証時にホームにリダイレクトされる | ✅ OK |
| F6 | 選択なしで反映 → チャートクリア | ✅ OK |

## 関連

- [experimental/interest_rate_sp500_raw/](../experimental/interest_rate_sp500_raw/) - 金利 + S&P 500 の検証
- [experimental/baa_spread/](../experimental/baa_spread/) - Baa社債スプレッドの検証
- [experimental/score_sp500/](../experimental/score_sp500/) - 複合スコアの検証
- [Backend/main/src/services/finance_service.py](../Backend/main/src/services/finance_service.py) - 既存の金利データ取得ロジック（リサンプリング・結合の参考）
- [Frontend/finance-dashboard/src/components/widgets/InterestRateWidget.vue](../Frontend/finance-dashboard/src/components/widgets/InterestRateWidget.vue) - 既存のチャート実装（Lightweight Charts の参考）
