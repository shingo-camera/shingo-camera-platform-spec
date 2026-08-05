# ADR-007 Stripe CheckoutとWebhookを購入権限付与の正本にする

Status: Approved

## 決定

Stripe Checkoutを使用し、署名検証済みWebhookによってT_PURCHASEおよびT_USER_PRODUCTを更新する。

購入完了画面から直接権限を付与しない。

## 理由

- 完了画面へ戻らないケースでも処理できる
- Stripeによる再送へ対応できる
- 決済結果と権限付与をサーバー側で確実に連携できる
- カード情報を自サイトで保持しない

## セキュリティ

Webhookは未加工bodyとStripe-Signatureで検証する。

秘密キーはCloudflare Secretsで管理する。
