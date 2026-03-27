# BUG-002: カスタムチャートの凡例がチェック変更時に即座に更新される

## ステータス

修正済み（検証済み）

## 発見日

2026-03-27

## 概要

カスタムチャートで反映ボタンを押してチャートを描画した後、チェックボックスを追加・解除すると、反映ボタンを押していないのに凡例が即座に更新されてしまう。

## 症状

- 反映ボタン押下後にチャートと凡例が表示される
- その後チェックボックスを変更すると、反映ボタンを押していないのに凡例の項目数・内容が変わる
- チャート自体は更新されないため、チャートと凡例が不整合になる

## 原因

テンプレートの凡例部分が `selectedSources`（computed、チェック状態にリアクティブに連動）を直接参照していたため、チェック変更が即座に凡例に反映されていた。

## 影響範囲

- `Frontend/finance-dashboard/src/pages/CustomChartPage.vue`

## 再現手順

1. `/custom-chart` にアクセス
2. 2つのソースを選択して反映ボタンを押す
3. 凡例が2つ表示されることを確認
4. 3つ目のソースをチェックする（反映ボタンは押さない）
5. 凡例が3つに変わってしまう（期待: 2つのまま）

## 対応

`renderedSources` という ref を追加し、反映ボタン押下でチャート描画が成功した時点でのみ更新するようにした。凡例のテンプレートを `selectedSources` → `renderedSources` に変更。

変更ファイル:
- `Frontend/finance-dashboard/src/pages/CustomChartPage.vue`
  - `renderedSources` ref 追加
  - `handleApply` 内で描画成功時に `renderedSources` を更新、クリア時に空配列を設定
  - テンプレートの凡例 `v-for` を `renderedSources` に変更

Playwright テスト（`tests/test_legend_bug.mjs`）で検証済み。

## 関連

- ENH-001: カスタムチャート機能
