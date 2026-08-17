# ADR-013 Formal DEV Environment

Status: Accepted (2026-08-18)

## Context

従来の Local / Production 2環境方針では、公開Productionへ反映する前に、実ドメイン・PC/iPhone・Stripe Checkout・Cloudflare Accessを含む実機E2Eを安全に確認できなかった。SUN AND MOON発売前検証で、本番を試験場にしないための正式DEV環境が必要になった。

## Decision

環境は **Local / DEV / Production** の3環境を正式方針とする。

- Production: `main` → Production Worker → `https://shingo-camera.com/*`
- DEV: DEV Worker → `https://shingo-camera.com/dev/*`
- DEVは専用D1、Stripe Test Mode、Cloudflare Access allowlistを使用する。
- Supabase Authは現行設計どおり共用するが、DEVのAPI/DB/Stripe書込みはProductionへfallbackさせない。
- DEVは同一`public/`を配信し、`DEV_BASE_PATH=/dev`のprefix-strip shimで既存routeへ接続する。DEV専用HTMLコピーは作らない。
- `apiUrl()` / `appUrl()` / `apiFetch()` を環境URL解決の共通正本とする。
- DEV HTMLではresolverを`<head>`先頭へinline注入し、外部`api-base.js`ロード失敗でProductionへ脱出するfail-openを許さない。
- DEV内3xxのルート相対`Location`には`/dev`を維持する。
- Stripe Checkoutのsuccess/cancel URLは`APP_BASE_URL`のpathnameを保持する。`request.url.origin`を正本にしない。
- 商品設定の`appUrl`を起動hrefへ直接入れず、共通resolverを通す。
- `POST /dev/api/stripe/webhook` exact pathのみCloudflare Accessのユーザー認証対象外とし、Stripe署名検証を認証境界とする。その他のDEV APIはAccess保護を維持する。

## Promotion

DEV実機確認と自動テストに合格した変更のみProductionへ昇格する。DB migrationを伴う場合は後方互換性と適用順を個別に確定する。

## Supersedes

ADR-011の「Local / Productionのみ」の環境数に関する決定を置換する。ADR-011自体は履歴として保持する。
