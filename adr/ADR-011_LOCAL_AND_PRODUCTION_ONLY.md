# ADR-011 Initial Environment Uses Local and Production Only

Status: Approved

## 決定
初期構築では専用Staging環境を作らない。

```text
Local
Production（非公開）
```

Productionを開発・結合テストに使用し、完成後そのまま公開する。

## 理由
- 既存公開利用者がいない
- 環境維持の手間と設定差異を減らせる
- 個人開発の速度を優先できる
- 専用Stagingの費用対効果が低い

## 注意
StripeはProductionアプリ内でもTest Modeを使用し、公開前にLive Modeへ切り替える。
