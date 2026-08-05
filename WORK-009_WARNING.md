# WORK-007 Stripe Purchase

Status: Approved

## 目的
SUN AND MOON PLANNERの買い切り販売を実装する。

## 実装
- Checkout Session作成
- metadata設定
- Webhook署名検証
- raw body保持
- D1 batch
- 冪等性
- 購入反映待ち画面

## 環境
Productionアプリ内でStripe Test Modeを使用し、公開前にLive Modeへ切り替える。

## 完了条件
Test Mode購入成功、購入・権限登録、再送で二重登録なし、不正署名拒否。
