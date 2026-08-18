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
- スマホ地図ボタンは右上の右端縦列で、**上＝太陽/月トグル（`#map-body-toggle`, `top:8px`）、下＝衛星切替（`#map-sat-btn`, `top:52px`）**、safe-area対応。地図メニュー（☰）は右下。旧記述の「衛星を上・太陽/月を下」とする順序は上下が逆であり使用しない。
- SUN AND MOON chance/pinpoint の canonical event time は moveM最小時刻 `fd.dt`。検索側（moveM / 採否 PINPOINT≤30m・CHANCE≤200m / ★の基礎となる検索結果 / candidate / sort / dedup / canonical event time）は変更しない。次回修正は `fd.dt` から最近傍1分の `displayDt` を生成（秒<30→当該分／≥30→次分／日跨ぎは丸め済みDateから生成）し、`date`/`time`/`azDiff`/`alt`/`angDiam` など表示時刻依存値を同一 `displayDt` から生成する（現状の時刻由来値の混在を解消）。`azMatchTime`/angSep最小等の別時刻探索は行わない。**Rev1/Rev2の角距離(angSep)最小表示時刻探索はいずれも不採用**（Rev1＝成立区間外へ流出し得たため／Rev2＝canonical `fd.dt` と別の angSep最小時刻を表示時刻にする仕様変更自体を採らないため）。詳細は `implementation/WORK-010_SUN_AND_MOON_INTEGRATION.md`「CHANCE / PINPOINT 検索仕様」。
- 検索導線：一括CHANCE検索は廃止、一括PINPOINTは維持、通常画面のCHANCE検索は維持。

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
