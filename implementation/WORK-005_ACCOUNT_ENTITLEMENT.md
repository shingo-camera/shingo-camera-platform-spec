# WORK-005 Account and Entitlement

Status: Completed

## 目的
M_USER同期、ログイン後ホーム、商品一覧、商品権限確認を完成させる。

## API
- `POST /api/account/sync`
- `GET /api/account/me`
- `GET /api/account/products`
- `POST /api/account/password-changed`
- `GET /api/products`
- `GET /api/products/{code}`
- `GET /api/entitlements/{code}`

## 完了条件
商品0件ユーザーでログイン可能、M_USER自動作成、未購入・停止・期限外を正しく拒否する。
