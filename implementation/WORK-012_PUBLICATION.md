# WORK-012 Production Publication

Status: Approved

## 目的
非公開Productionを公開可能な状態へ切り替える。

## 作業
- 独自ドメイン接続
- Stripe Live Mode切替
- Live Webhook確認
- テストデータ確認
- 管理者権限確認
- バックアップ
- RELEASE_CHECKLIST実施
- 公開

## 完了条件
独自ドメインで登録、購入、利用、note移行、管理、Warning通知がすべて動作する。

## HANABI統合の公開前状態（2026-08-19 追補）

HANABIアプリのPlatform統合と公開前調整はWORK-013（HANABI Integration）で実施済み。実装は develop（HEAD `227858d`）に反映済み、DEVでPC/スマホ実機確認済み、今回発見した主要不具合は修正済み。これによりHANABIはProduction公開工程へ進める見込みとなった。

現時点でHANABIはProduction公開済みではない。公開前の残工程は次のとおり。

1. Spec正本反映（本追補を含む）
2. 実装正本（develop）とSpec正本の最終整合
3. Production deploy
4. 公開後スモーク確認

Production公開の確定は本WORK-012の完了条件で扱う。WORK-013の「実装済み / DEV実機確認済み」と本WORKのProduction公開完了を混同しない。
