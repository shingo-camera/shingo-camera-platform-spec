# Stripe Operation

Status: Approved

## 1. 採用理由

- 固定費なし
- 売れた時だけ決済手数料
- Checkoutによりカード情報を自サイトで保持しない
- Webhookで購入後処理を確実に実行可能
- 将来サブスクリプションにも拡張可能

## 2. 初期販売

初期対象:

```text
SUN AND MOON PLANNER
```

買い切り、円決済。

HANABIの新規販売移行は後続対応可能とする。

## 3. 本番前作業

- Stripeアカウント設定
- 本人確認・入金先設定
- 商品作成
- Price作成
- Checkout成功URL設定
- Checkout取消URL設定
- Webhook endpoint登録
- Webhook SecretをCloudflare Secretへ登録
- テストモード購入
- Webhook再送テスト
- 二重Webhookテスト
- 金額不一致テスト
- 本番切替

## 4. 運用

通常はStripe管理画面を毎日確認しない。

以下の場合に確認する。

- 購入問い合わせ
- 権限未反映
- 二重購入
- 返金
- Stripeからの警告

## 5. 費用

- 月額固定費: なし
- 決済手数料: 売上発生時のみ
- Cloudflare追加固定費: 原則なし
- D1追加固定費: 現規模では既存契約内を基本

Stripeの実際の手数料率は導入時点の日本向け公式料金を確認し、販売価格設計へ反映する。

## 6. 障害時

Stripeで決済成功、Webhook未反映の場合:

1. Stripe Checkout Session IDを確認
2. T_PURCHASEの有無を確認
3. Webhook再送
4. 改善しない場合は管理画面から購入実績または商品権限を手動補正

手動補正時もStripe側の購入IDを追跡可能な形で保持する。
