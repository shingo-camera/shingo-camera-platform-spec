# Implementation Plan

Status: Approved

## 1. 目的

shingo-camera Platformの共通認証・商品権限・購入・管理基盤を、Claudeが順序どおり実装できるよう作業単位を定義する。

## 2. 環境方針

初期環境は以下のみとする。

```text
Local
DEV（正式検証環境 / https://shingo-camera.com/dev）
Production
```

正式DEV環境を設ける。Productionを開発・結合テスト環境として使用せず、DEVで実機確認済みの変更のみProductionへ昇格する。詳細は `adr/ADR-013_FORMAL_DEV_ENVIRONMENT.md` と `architecture/DEV_ENVIRONMENT.md` を正本とする。

## 3. 原則

- 各WORKを順番に実施する
- 前工程の完了条件を満たすまで次工程へ進まない
- 既存SUN AND MOON本体を最初から移動・再構成しない
- 共通基盤を先に単体完成させる
- 既存設計書と差異がある場合は実装前に報告する
- 勝手なテーブル・カラム・SaaS追加をしない
- Productionの秘密情報をコードやGitへ保存しない

## 4. 実装順

```text
WORK-001 リポジトリ・Cloudflare基盤
↓
WORK-002 D1初期構築
↓
WORK-003 Supabase認証
↓
WORK-004 共通API基盤
↓
WORK-005 アカウント・商品権限
↓
WORK-006 管理画面
↓
WORK-007 Stripe決済
↓
WORK-008 note移行
↓
WORK-009 Warning通知
↓
WORK-010 SUN AND MOON統合
↓
WORK-011 公開サイト・法務導線（動線再編・複数商品Checkout等の追加対応を含む。詳細は WORK-011_APPENDIX_MULTI_CHECKOUT.md）
↓
WORK-012 本番公開確認
```

## 5. 実装報告

各WORK完了時に以下を報告する。

- 実装内容
- 新規ファイル
- 変更ファイル
- 環境変数・Secrets
- DB変更
- 動作確認結果
- 未確認事項
- 設計書との差分
- 次WORKへ進める状態か
