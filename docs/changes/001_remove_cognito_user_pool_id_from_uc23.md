# 001: UC-2.3 から CognitoUserPoolId エクスポートを削除

## 変更概要

`docs/units_contracts.md` の UC-2.3（インフラ → フロントエンド）から `${project}-${env}-infra-CognitoUserPoolId` の行を削除した。

### 理由

- このエクスポートの用途は `VITE_COGNITO_AUTHORITY` の生成とされていたが、フロントエンドのソースコードで `VITE_COGNITO_AUTHORITY` を参照している箇所が存在しない
- フロントエンドの認証フローは `VITE_COGNITO_DOMAIN` を使った OAuth2 エンドポイント直接呼び出しで完結しており、Cognito Authority URL（`https://cognito-idp.{region}.amazonaws.com/{userPoolId}`）は不要
- JWT の issuer 検証は API Gateway の Cognito Authorizer がサーバー側で行っており、フロントエンドでの検証は不要

## 対象ファイル

- `docs/units_contracts.md` — UC-2.3 テーブルから 1 行削除

## 影響を受けるユニット

### インフラ（Infra）

- `CognitoUserPoolId` の `CfnOutput` / エクスポート自体は**削除不要**。バックエンド（Lambda 環境変数）が引き続き参照している
- 対応不要

### フロントエンド（Frontend）

- `buildspec.yml` で `CognitoUserPoolId` を取得していないことが契約上も正当となった
- 対応不要

### バックエンド（Backend）

- 影響なし（バックエンドは別の契約で `CognitoUserPoolId` を参照している）

### CI/CD

- 影響なし
