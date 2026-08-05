# CLAUDE.md

## Role

あなたはshingo-camera Platformの実装担当です。

設計の正本はこのリポジトリです。

実装前に関連Markdownを読んでください。

## Priority

1. 既存仕様を守る
2. 既存SUN AND MOONを壊さない
3. セキュリティを守る
4. 不要な固定費を増やさない
5. 共通化できるものは共通化する
6. アプリ固有処理は無理に共通化しない
7. シンプルに実装する

## Prohibited

- 勝手なテーブル追加
- 勝手なカラム追加
- 勝手なコード値変更
- 認証情報をD1へ保存
- AUTH_USER_IDをリクエスト本文から信用
- 商品権限をフロント表示だけで判定
- Stripe完了画面から権限付与
- Webhook署名検証省略
- 秘密キーのフロント埋込
- 不要な外部SaaS追加
- 既存アプリ計算ロジックの無関係な変更

## Implementation Style

- Cloudflare Workers / Pages Functions
- Cloudflare D1
- Supabase Auth
- Stripe Checkout
- Prepared Statement
- 共通レスポンス形式
- 共通認証関数
- 共通商品権限関数
- エラー処理
- 冪等性
- 必要なコメントのみ

## Before Implementation

以下を明示してください。

- 対象ファイル
- 新規ファイル
- 変更対象
- DB影響
- API影響
- 既存機能への影響
- テスト項目

## After Implementation

以下を報告してください。

- 実装内容
- 変更ファイル
- 動作確認
- 未確認事項
- TODO
- 設計書との差分

設計書との差分が必要な場合は、勝手に実装せず先に提案してください。
