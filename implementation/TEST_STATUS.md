# Test Status（2026-08-18 current baseline）

Status: Recorded

## 正本

2026-08-18に採取した、DEV実機E2E合格後のPlatform current sourceを現行baselineとする。GitHub `main` は当時 `7f3b466` で、DEV・発売前修正は未commitの作業ツリーに存在していたため、古いHEADだけを実装正本として扱わない。

## 現況

運用者のWindows環境での直近実行:

```text
403 tests
400 pass
3 fail
```

DEV追加・URL resolver修正・Stripe return修正・アプリ起動修正による新規failは確認されていない。

## 3 failの分類

1. Earth icon検査 1件: Windows環境で`python3`コマンドが見つからないテスト環境要因。画像不良を示すものではない。
2. SUN AND MOON chance/pinpoint系 2件: DEV追加前から追跡している既知検索テスト。DEV修正とは切り分ける。内訳は性質が異なるため分けて扱う：
   - `日別最小moveM 代表` 系: canonical 探索側（moveM最小の代表選定）の既知課題。表示時刻の丸めとは無関係。
   - `pinpoint 上端中央近傍 altPct≈100` 系: 現 Production で `time` / `moveM` / `ts` が canonical `fd.dt` 由来、`alt` / `azDiff` / `angDiam` が `dispDt` 由来という**時刻由来値の混在**（WORK-010「現Productionに存在する時刻不整合」）との**関連が未解決**。以前の表示時刻変更検証で自然に pass 化した実績があるため、「表示時刻とは無関係」とは断定しない。**次回 display-time 整合修正（表示依存値を同一 `displayDt` 由来へ統一）時に再評価する既知課題**とする。

   chance/pinpoint の評価正本（moveM / PINPOINT≤30m / CHANCE≤200m / canonical時刻）は `implementation/WORK-010_SUN_AND_MOON_INTEGRATION.md`「CHANCE / PINPOINT 検索仕様」を正本とする。

## Windows回帰防止

`test/dev_env.test.mjs`で`new URL("../public", import.meta.url).pathname`をWindowsファイルパスとして使わない。`fileURLToPath(new URL("../public", import.meta.url))`をcurrent仕様として固定する。古い成果物で`.pathname`へ巻き戻さない。

## DEV URL / E2E回帰

自動テスト・実機で以下を固定する。

- Production: `apiUrl/appUrl`は従来ルート相対URLと同値。
- DEV: `/api/*`→`/dev/api/*`、Platform navigation→`/dev/*`。
- DEV HTML resolverはhead先頭inline注入。外部resolverロード失敗でProductionへ脱出しない。
- DEV 3xx Locationは`/dev`を維持。
- Stripe success/cancel URLは`APP_BASE_URL=/dev`のpathnameを保持。
- Store/My Pageのアプリ起動は`launchHref()`経由。生`meta.appUrl`をhrefへ入れない。
- DEV reset APIは`APP_ENV=development`で許可、Productionで拒否。

## 2026-08-18 実機確認

DEVで Stripe Test Checkout → `/dev/purchase/success/` → DEV D1権限反映 → 「利用する」→ `/dev/apps/sun-and-moon/` → `/dev/api/apps/sun-and-moon/app-start` 200 OKまで確認済み。

## HANABI統合後のテスト結果（最新）

WORK-013（HANABI Integration）の実装・公開前調整・HANABI回帰テスト追加を反映したfixture込みの最新テスト一式で実行済み。上記の旧結果（403 tests / 400 pass / 3 fail、Windows環境での直近実行）は履歴として残し、最新結果を以下に追記する。

```text
680 tests
668 pass
12 fail
0 cancelled
0 skipped
0 todo
```

今回追加したHANABI回帰テストについて新規failは確認されていない。残存12 failは今回のHANABI追加差分とは独立していることを確認した。

12 failを「全テスト合格」とは表現しない。残存12 failは今回のWORK-013で修正対象とせず、勝手に修正しない。確認できている残存failの主な内訳は次のとおり（性質ごとに分けて扱う）。

- Platform frontend_fixes系: `site.js` cancelCheckout、startMultiCheckout、initStore、active-checkout completed、pending整合、古いopen checkoutの扱い。
- SUN AND MOON側: chanceの日別最小moveM、非Admin固有建物UI、非Admin時の復元経路遮断、KMZ debug値、円錐台topWidth、その他同系統の既存テスト。

これらは今回のHANABI追加差分とは独立した既存failであり、HANABI統合が新規に発生させたものではないことを確認済み。SUN AND MOON側 chance/pinpoint 系の評価正本・既知課題の扱いは `implementation/WORK-010_SUN_AND_MOON_INTEGRATION.md` を引き続き正本とする。
