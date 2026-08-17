# DEV Environment Architecture

Status: Active (2026-08-18)

## 1. 目的

Productionへ反映する前に、実ドメイン上で認証・購入・アプリ起動までを実機確認する正式DEV環境を提供する。

## 2. URL / Worker

- Production: `https://shingo-camera.com/...`
- DEV: `https://shingo-camera.com/dev/...`
- DEV Workerは`/dev`をstripして既存route/ASSETSへ渡す。同一コード・同一`public/`を使用する。

## 3. 分離

| 項目 | DEV | Production |
|---|---|---|
| Worker | DEV専用 | Production |
| D1 | DEV専用 | Production専用 |
| Stripe | Test | Live/Production設定 |
| Access | Cloudflare Access allowlist | 通常公開境界 |
| APP_ENV | development | production |
| APP_BASE_URL | `https://shingo-camera.com/dev` | `https://shingo-camera.com` |
| DEV_BASE_PATH | `/dev` | 未設定 |

Productionへfallbackしない。DEV binding/secret不足はfail-closedとする。

## 4. URL resolver

フロントの自前APIは`apiFetch()`、Platform内navigationは`appUrl()`を使用する。商品設定のルート相対`appUrl`は`launchHref()`等の共通resolver経由でhref化する。

DEV HTMLではresolverをhead先頭へinline注入し、外部JSロードに単一依存しない。3xx Location、静的href/src/action/posterもDEV prefixを維持する。

## 5. Stripe return URL

Checkout Sessionのsuccess/cancel URLは`env.APP_BASE_URL`を唯一の正とし、originだけでなくpathnameも保持する。

- Production → `/purchase/...`
- DEV → `/dev/purchase/...`

## 6. DEV購入リセット

テスト購入のやり直し用`POST /api/admin/test/reset-purchases`はdeny-by-defaultとし、`local` / `test` / `development`のみ許可する。Productionでは利用不可。

## 7. 2026-08-18 実機E2E

運用者がDEVで以下を実機確認済み。

1. DEVでログイン。
2. Stripe Test Checkoutを完了。
3. `/dev/purchase/success/`へreturn。
4. DEV D1へ注文・購入・商品権限が反映。
5. Store/My Pageの「利用する」から`/dev/apps/sun-and-moon/`を起動。
6. SUN AND MOONの`app-start`が`/dev/api/apps/sun-and-moon/app-start`へPOSTし200 OK。

この経路でProduction root/APIへの脱出がないことを確認した。
