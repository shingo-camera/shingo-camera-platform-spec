# Claude Current Baseline / Handoff

Status: Active (2026-08-18)

## 最重要

実装修正を行う場合、**2026-08-18に実機確認済みのcurrent Platform sourceをbaseline**とする。古い成果物ZIPをファイル単位で上書きして、後から入った修正を巻き戻してはならない。

## 巻き戻し禁止項目

少なくとも以下を保持する。

- 正式DEV: `/dev/*`、DEV D1、Stripe Test、Cloudflare Access。
- `apiBase/apiUrl/appUrl/apiFetch` とDEV HTML resolver inline注入。
- DEV 3xx Locationの`/dev`保持。
- Stripe `resolveBaseUrl()`が`APP_BASE_URL`のpathname（`/dev`）を保持する修正。
- DEV reset APIで`APP_ENV=development`を許可し、Productionはdenyする環境ガード。
- `site.js`のアプリ起動導線は`launchHref(meta)`経由。生`meta.appUrl`をhrefへ入れない。
- `test/dev_env.test.mjs`のWindowsパス解決は`fileURLToPath(new URL("../public", import.meta.url))`を使用する。`.pathname`へ戻さない。
- SUN AND MOONの対象offsetは対象データ`elevOffset`へ一本化。旧`targetOffset`を復活させない。
- SUN AND MOONの撮影計画保存は撮影地点＋対象の両方必須。未選択時はno-op。
- 撮影計画名`planName`は保存時に入力し、既存一覧ラベルの末尾へ表示。旧データ互換を維持。
- スマホ太陽/月トグルは衛星ボタン直下の右端縦列・safe-area対応。

## 成果物の作り方

1. current baselineへ差分適用する。
2. 変更対象だけをdiffとして提出する。古いフルファイルを無条件上書きしない。
3. 変更前に対象ファイルのcurrent内容を読み、保持必須項目が存在することを確認する。
4. 変更後に保持必須項目の静的回帰テストを実行する。
5. Windows/PowerShell実行を前提に、Node URL pathnameをWindowsファイルパスとして直接使わない。
6. Productionのbinding/URL/Stripe/D1をDEVへfallbackさせない。
7. commit/push/deployはユーザーの明示指示なしに行わない。

## 現行テスト注意

2026-08-18時点の運用者実行では403 tests / 400 pass / 3 fail。DEV変更による新規failは確認されていない。3 failの内訳は、Windowsで`python3`コマンドが見つからないEarth icon検査1件と、既知SUN AND MOON chance/pinpoint系2件。これらをDEV修正と混同しない。
