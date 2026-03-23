# BUG-001: セルフサインアップが無効化されていない

## ステータス

修正済み（検証待ち）

## 発見日

2026-03-24

## 概要

設計ドキュメント（user_stories.md, units_definition.md）ではセルフサインアップを無効とし管理者がマネジメントコンソールまたは CLI でユーザーを作成する方針だが、実装およびリファレンスドキュメントではセルフサインアップが有効のままになっている。

## 症状

- Cognito Managed Login のサインアップ画面からユーザーが自己登録できてしまう
- 設計方針と実装が乖離しており、意図しないユーザー作成が可能な状態

## 原因

設計ドキュメントの方針がインフラの CDK コード・フロントエンドの認証コード・リファレンスドキュメントに反映されていない。

## 影響範囲

- `Infra/stacks/cognito_stack.py` — `self_sign_up_enabled=True` になっている（L35）
- `Frontend/finance-dashboard/src/auth/auth.ts` — `signup()` 関数が存在する可能性（※ buildspec_review.md では削除済みとの記載あり。要確認）
- `docs/reference/authentication.md` — セルフサインアップ有効と記載（L56, L74）、サインアップフローの説明あり（L220-L241）

## 再現手順

1. Cognito Managed Login の URL にアクセスする
2. サインアップ画面が表示され、メールアドレスとパスワードでアカウントを作成できる

## 修正案

### 案1: CDK コード修正 + リファレンスドキュメント汎用化（推奨）

1. **Infra**: `cognito_stack.py` の `self_sign_up_enabled` を `False` に変更
2. **docs/reference/authentication.md**: セルフサインアップの記載を汎用的な表現に変更（要件次第でどちらもあり得る旨を記載）。authentication.md は他プロジェクトでも使う汎用リファレンスのため、プロジェクト固有の設定をハードコードしない
3. **Frontend**: `signup()` 関数は既に削除済みのため対応不要

## 対応

案1を採用。以下のファイルを修正:

- `Infra/stacks/cognito_stack.py` — `self_sign_up_enabled` を `True` → `False` に変更（L35）
- `docs/reference/authentication.md` — セルフサインアップの記載を汎用的な表現に変更:
  - 前提セクション: セルフサインアップ有効時のみ Managed Login を使う旨に修正
  - コードスニペット: 要件次第のコメントを追加
  - 設定テーブル: 「有効」→「要件次第」に変更
  - サインアップセクション: セルフサインアップ有効時のみ使用する旨の注記を追加

## 関連

- `docs/user_stories.md` L17: 「アカウントの作成は管理者がマネジメントコンソールまたは CLI で行う。セルフサインアップは提供しない。」
- `docs/units_definition.md` L74: 「セルフサインアップは無効。管理者がマネジメントコンソールまたは CLI でユーザーを作成する」
