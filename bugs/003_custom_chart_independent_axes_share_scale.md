# BUG-003: カスタムチャートの独立軸ソースが同じ軸グループに属していると同一軸を共有する

## ステータス

修正済み（検証済み）

## 発見日

2026-05-10

## 概要

カスタムチャートで独立軸（`independent: true`）の異なるサブカテゴリ（`local_group`）に属するソース（例: 「株価指数」配下の S&P 500 と「投資環境」配下の投資環境スコア）を同時に選択して反映すると、本来は独立軸として別々の軸が割り当てられるべきところ、同一の軸（同一スケール）にまとめられてしまう。

## 症状

- 独立軸グループ `independent1` 配下の「株価指数」（例: S&P 500）と「投資環境」（例: 投資環境スコア）を両方チェックして反映ボタンを押す
- 期待: 2 系列がそれぞれ独立した Y 軸（左右）に描画される（単位「USD」と「スコア」が別軸）
- 実際: 2 系列が同じ Y 軸スケールに描画され、値域が大きく異なる場合は片方が極端に潰れて見える、または軸ラベルが片側分しか表示されない
- `selectedAxisCount`（フロント側の軸カウント表示）では正しく独立軸ソース 1 つ＝1 軸として数えられているのに、実際の描画では合算されており、内部状態と描画結果が不整合

## 原因

[`Frontend/finance-dashboard/src/pages/CustomChartPage.vue`](Frontend/finance-dashboard/src/pages/CustomChartPage.vue) の `renderChart` 関数における軸割り当てロジックが、`axis_group` キーのみでユニーク化しているため、独立軸（`independent: true`）の挙動を考慮していない。

具体的には [`renderChart` L247](Frontend/finance-dashboard/src/pages/CustomChartPage.vue#L247):

```ts
const uniqueGroups = [...new Set(series.map((s) => s.axis_group))]
```

仕様（`docs/openapi_main.yaml` の `CustomChartAxisGroup.independent` の説明）では:

> 独立軸フラグ。trueの場合、選択ソースごとに1軸を消費する。falseの場合、同一グループのソースは同じ軸を共有する。

つまり独立軸の場合は **ソースごとに 1 軸**を割り当てる必要がある。ところが現実装では `axis_group === "independent1"` のソースが複数あっても 1 軸にまとめられてしまう。

一方、軸数の事前チェックを行う `selectedAxisCount`（[L60-69](Frontend/finance-dashboard/src/pages/CustomChartPage.vue#L60-L69)）では `independent` フラグを参照してソース数で数えているため、内部カウントと実際の描画ロジックの間で不整合がある。

なお、`max_axes`（=2）による軸数上限のチェックは `selectedAxisCount` が独立軸を「ソースごと 1 軸」として正しく数えているため、独立軸 ≥3 や「独立軸 + 通常軸」で合計 3 軸以上選択した場合は `handleApply`（[L212-215](Frontend/finance-dashboard/src/pages/CustomChartPage.vue#L212-L215)）で正しくブロックされる。本バグは制限内（独立軸 2 つ = 2 軸）で発生する描画割当の問題であり、軸数制限ロジックそのものに問題はない。

## 影響範囲

- `Frontend/finance-dashboard/src/pages/CustomChartPage.vue`
  - `renderChart` 関数（軸割り当てロジック）
  - 凡例表示の `axis_label` 表示にも影響する可能性あり（軸ラベルがソース固有のものになるべき）

バックエンドはソース定義（`Backend/main/src/custom_chart_sources.json`）と OpenAPI 仕様の通りに `axis_group` / `axis_label` を返しており、責務はフロントエンドの描画側にある。

## 再現手順

1. `/custom-chart` にアクセス
2. 「独立軸」グループ配下の「株価指数」サブカテゴリで S&P 500 をチェック
3. 「独立軸」グループ配下の「投資環境」サブカテゴリで投資環境スコア（堀井）をチェック
4. 反映ボタンを押す
5. 2 系列が同じ Y 軸スケールで描画される（期待: 左右別々の軸で描画される）

## 修正案

### 案1: 軸キーを `independent` を考慮して生成する（推奨）

`renderChart` 内の軸割り当てロジックで、独立軸ソースは `axis_group` ではなく `id`（または `axis_group:id` の合成キー）で軸をユニーク化するように変更する。これにより `selectedAxisCount` の数え方と一致し、仕様通り「独立軸ソースごとに 1 軸」を割り当てる挙動になる。

実装の要点:
- `axisGroups`（`CustomChartSourcesResponseAxisGroups`）を `renderChart` から参照可能にする（既に `ref` として保持済み）
- `series.map` 時に `axisGroups[s.axis_group]?.independent ? s.id : s.axis_group` のような軸キーを算出し、それでユニーク化・スケール割当・ラベル割当を行う
- 軸ラベル（`axis_label`）はソース固有の値が既に `series[i].axis_label` に入っているため、軸キー → ラベルのマップもこの新しい軸キーで構築する

### 案2: バックエンドで独立軸ソースに一意な `axis_group` を返す

バックエンドが独立軸ソースのレスポンス時に `axis_group` をソースごとにユニークな値に書き換えて返す。フロントの修正は不要になるが、`axis_groups` の定義（共有ドキュメント `openapi_main.yaml`）と矛盾するうえ、フロントの `groupedSources` 表示ロジック（`axisGroups` のキーを参照）も壊れるため非推奨。

## 対応

採用案: **案1（軸キーを `independent` を考慮して生成する）**

`renderChart` 内で軸キーを以下のように算出し、独立軸ソースは `id`、通常軸ソースは `axis_group` を軸キーとしてユニーク化するように変更した。

```ts
const axisKeyOf = (s: { id: string; axis_group: string }) =>
  axisGroups.value[s.axis_group]?.independent ? s.id : s.axis_group
```

これに伴い `uniqueGroups` を `uniqueAxisKeys` にリネームし、スケール（左右）割当・`series.forEach` 内の `priceScaleId` 参照もすべて新軸キーで行うように統一した。これにより `selectedAxisCount`（軸数の事前チェック）と描画ロジックの間の不整合が解消される。

変更ファイル:
- `Frontend/finance-dashboard/src/pages/CustomChartPage.vue`
  - `renderChart` 関数の軸割り当てロジック修正

検証:
- `npm run type-check` 通過
- `npx eslint src/pages/CustomChartPage.vue` エラーなし
- ブラウザでの挙動検証済み（独立軸の「株価指数」と「投資環境」を同時選択した際に左右別々の Y 軸で描画されることを確認）

## 関連

- ENH-001: カスタムチャート機能
- `docs/history/005_unified_axis_groups_local_groups.md` — 独立軸とサブカテゴリの統合仕様
- `docs/openapi_main.yaml` — `CustomChartAxisGroup.independent` の仕様
