# WORK-010 SUN AND MOON Integration

Status: Completed

## 目的
既存SUN AND MOONへ共通認証・商品権限確認を追加する。

## 前提
SUN AND MOON本体の唯一の実装正本は `public/apps/sun-and-moon/index.html` とする。旧 `portrait.html` / `moon.html` は実装正本として扱わない（Moonは本体へ統合済み）。

## 実装
- 未ログイン時ログインへ
- `SUN_AND_MOON`権限確認
- 権限なし時商品案内
- 計算APIでも権限確認
- アクセスログ

## 禁止
天体計算、APIレスポンス、既存UIの無関係な変更。

## 発売前 実機修正（スマホ 太陽/月トグル）

SUN AND MOON本体 `public/apps/sun-and-moon/index.html` のスマホ地図上「太陽/月切替」トグルは、発売前のProductionスマホ実機確認を経て以下の現行仕様とする（実機で押下確認済み）。

- 配置（現行Production実装）: スマホ右上の右端縦列。**最上段が太陽/月トグル（`#map-body-toggle`, `top:8px`）**、**その直下が衛星切替（`#map-sat-btn`, `top:52px` = 8px + 高さ36px + 8pxギャップ）**。両者とも `right:8px`・高さ36px・8pxギャップで縦に重ならない。地図メニュー（☰ `#map-menu-btn`）は右下（`bottom`）。既存の検索／ズーム／対象追加のいずれとも重ならない。
- safe-area 考慮（`top`/`right` とも `env(safe-area-inset-top/right)` を加算）。
- 表示: PCは非表示（`#map-body-toggle{ display:none; }` / `#map-sat-btn{ display:none; }`）、スマホのみ `@media (max-width:600px)` で `display:flex !important`。モバイルでは Leaflet 右上コントロール群（`.leaflet-top.leaflet-right`）は `display:none` で非干渉。
- ロジック: 既存 `setSunsetMode(!sunsetMode)` を再利用（新規ロジックを作らない）。状態は `setSunsetMode` 内で一元管理し、ボタン表示（☀ 太陽／☾ 月）もそこで同期する。
- 既存の衛星ボタン（`#map-sat-btn`）・メニューボタン（`#map-menu-btn`）・検索・ズームの機能は変更しない。

補足: 初期実装では独立した固定座標（左上）に置いていたが、実機で既存コントロールと重なったため、右端縦列へ組み込む現行仕様に修正した。**旧Spec記述は「衛星を上（`top:8px`）・太陽/月を下（`top:52px`）」とする逆順だったが、現行は太陽/月が `top:8px`・衛星が `top:52px`。** 旧CSS配置（左上固定・`display:block` 切替、および衛星を上・太陽/月を下とする順序）は現行実装では使用しない。

## 完了条件
未ログイン・未購入・停止を拒否し、購入者が既存機能を問題なく利用できる。

## 実装概要（統合方針）

独立Pages版のまま認証連携する方式は採用せず、SUN AND MOON本体（`index.html`）と計算API群を Platform Worker（`shingo-camera-platform`）へ統合した。認証・商品権限・アクセスログ・公開URLを同一プラットフォームへ集約し、Pages→Platform 間の認証問い合わせ・別オリジン CORS・Supabaseセッション共有を恒久的に残さない。

- アプリ画面: `/apps/sun-and-moon/`（静的配信）
- SUN AND MOON 固有API: `/api/apps/sun-and-moon/*`（Worker、`run_worker_first=["/api/*"]`）
- 各計算APIは既存 `requireProduct(request, env, "SUN_AND_MOON")` を必ず通す。新しい認証方式は追加しない。
- 計算APIではアクセスログを記録せず、アプリ起動時に `ACCESS_TYPE=0 (APP_START)` を1回だけ記録する（既存 `recordEntitlementAccess` の ACCESS_TYPE=1 挙動は不変）。

実装は実装リポジトリの `e79f6a7 Integrate SUN AND MOON into platform` にて commit / push 済み、Production deploy 済み。

## Local / 実装検証

- SUN AND MOON 計算API・共通モジュール15ファイルは、元 `sun-and-moon.zip` から diff 0 で無改変移植。
- 計算ゴールデンテスト38項目 PASS（移植前 Pages版 と 移植後 Worker版で同一入力→同一出力）。
- アクセスログ E2E 11項目 PASS（APP_START記録・ENTITLEMENT_CHECK独立・60分抑制・別デバイス別レコード・計算APIでログ非増加）。
- entitlement 回帰7項目 PASS（recordEntitlementAccess の挙動不変）。
- typecheck 合格 / wrangler dry-run 成功。
- migration 追加なし / Secret 追加なし（ACCESS_TYPE=0 は既存 DDL の CHECK 制約で許可済み）。

## Production E2E結果（Completed）

Production 環境で以下を確認した。

認証・権限:
- Production Worker へ統合版 SUN AND MOON を deploy 済み。
- 無認証で計算API直叩き → `UNAUTHORIZED / 認証が必要です` で拒否、計算結果を返さない。
- 未ログインで `/apps/sun-and-moon/` → ログイン画面へ誘導。
- 認証済み・SUN_AND_MOON 権限なし → Platformトップ `/` へ排除。
- 権限あり ユーザーで SUN AND MOON が正常起動。
- 権限なし試験は、管理画面から非管理者ユーザーの商品権限を一時停止して確認し、試験後に再開して原状復帰済み。

機能（PC実機）:
- 既存撮影計画データの移行・読込成功。
- 過去撮影計画を呼び出し、地図クリックに天体表示が正常追従。
- 主要計算機能を PC 実機確認し問題なし。
- prefecture 都道府県判定 正常。
- KMZ 生成・Google Earth 表示 正常。
- Google Earth 連携 正常。
- 標高取得 正常。

アクセスログ:
- APP_START アクセスログ `ACCESS_TYPE=0` が Production D1 へ正常記録。
- 同一ユーザー・同一デバイスで60分以内に再読込しても APP_START ログは1件のまま。
- 計算APIを複数利用しても T_ACCESS_LOG は増加しない。
- 既存 ENTITLEMENT_CHECK `ACCESS_TYPE=1` と独立して記録される。

既存資産の維持:
- Production deploy 後も WORK-009 Cron `0 * * * *`、D1 binding、`APP_ENV=production` を維持。

## 認証前画面フラッシュ（修正済み）

初回 Production E2E で、未ログイン時に SUN AND MOON 本体UIが一瞬表示される問題を確認。認証・権限確認が完了するまで本体を `visibility:hidden` とする auth gate を追加して再 deploy 済み。修正後は SUN AND MOON 本体UIの認証前露出は解消。黒→白の背景だけの一瞬の切替は残るが、アプリUI・内容の露出ではないため WORK-010 のセキュリティ不具合とは扱わない。

## 既存不具合候補（WORK-010回帰ではない）

Windy 連携で、PLの塔・富田林 撮影地点から開くと中心が奈良県御所市付近へずれる現象を確認。旧独立 Pages版 SUN AND MOON でも同じ現象を再現したため、WORK-010 の回帰ではない。WORK-010 の不合格要因とはせず、「既存 SUN AND MOON 不具合候補」として記録する。

## PC / スマホ

今回の Production 主要機能 E2E は PC で実施。スマートフォン固有UI・PWA の網羅確認は WORK-010 Completed 条件には含めない。必要なら別確認項目として残す。


## 2026-08-18 発売前 current baseline 追補

以下はPlatform統合後のSUN AND MOON本体に入った発売前修正としてcurrent baselineへ固定する。旧ZIP・旧Pages版から上書きして戻さない。

- 対象標高補正は対象データの `elevOffset` へ一本化。旧撮影セッション `targetOffset` は廃止。
- 撮影計画保存は **撮影地点＋対象の両方が選択済み** の場合のみ実行。未選択時は警告を出さずno-op。
- 保存時に撮影計画名 `planName` を入力。既定値は「対象名称_」。通常一覧・共有/エクスポート系で使う共通ラベルの末尾へ表示し、`planName`を持たない旧データは従来表示のまま。
- 検索導線の現行構成：**一括CHANCE検索は廃止**（登録地点の全件チャンス検索は行わない。実装コメント「一括チャンス廃止によりpinpointのみ」＝`_pinSearchResults` は `pinpoint` のみ保持）。**一括PINPOINT（登録地点の全件ピンポイント検索, `searchPinpointBatch`）は維持**。**通常画面の CHANCE検索（`btn-chance` / `calcChances()` 「📅 チャンス検索（365日）」）は維持**。
- スマホ太陽/月トグルは本書上部の現行仕様（太陽/月＝最上段 `top:8px`／衛星＝直下 `top:52px`）を維持。
- DEV起動では `/dev/apps/sun-and-moon/` から自前APIを `/dev/api/apps/sun-and-moon/*` へ解決する。2026-08-18に `app-start` 200 OKを実機確認済み。

## CHANCE / PINPOINT 検索仕様（現行正本）

Production `7f3b466` の実コード（`_search.js` / `chance.js` / `pinpoint.js` / `_astro.js`）を根拠とする現行正本。

### 1. 評価の正本＝moveM

- 検索評価の正本は **moveM**。
- moveM ＝「その時刻に、現在の撮影地点から移動して、**対象の中心軸上端**（登録座標 + 高さ）を月・太陽の円盤内へ入れられる撮影地点までの**実移動距離**（横移動＋前後移動＝距離D変化を含む）」。
- 現在地点で既に対象上端が円盤内なら **moveM = 0**。
- **PINPOINT 表示対象：moveM ≤ 30m**。
- **CHANCE 表示対象：moveM ≤ 200m**。
- **fail-closed**：未収束 / NaN / Infinity は採用しない（`acceptMove` が収束かつ閾値内のみ採用）。
- pinpoint 候補集合 ⊆ chance 候補集合（同一候補・同一理想点・同一moveM、閾値のみ 30m/200m の差）。「現在地点固定（moveM=0）だけがPINPOINT」ではなく、「30m以内で成立する高精度候補」を PINPOINT と呼ぶ表示カテゴリである。

### 2. ★評価

moveM に対する★（`starOf`）：

| moveM | 表示 |
|---|---|
| ≤ 5m | 🎯 |
| ≤ 10m | ★★★ |
| ≤ 50m | ★★☆ |
| ≤ 100m | ★☆☆ |
| ≤ 200m | ☆☆☆ |

- ★境界（5/10/50/100/200m）と PINPOINT の 30m 境界は**別概念**（30m を跨いでも★は変わらない。★★☆ は 10〜50m で 30m 境界を跨ぐ）。

### 3. 代表時刻（canonical event time）

- 現行検索は候補区間内で **moveM(t) を最小化**し、その **moveM最小の sub-minute 時刻 `fd.dt` を canonical event time** とする。
- 「方位完全一致時刻（bodyAz = targetAz）」や「角距離最小時刻」を**イベント正本にはしない**。
- 撮影計画としての意味：**そのイベントで、現在地点から最も少ない移動で成立できる瞬間**。
- 実測上、moveM最小時刻と「方位完全一致時刻」「角距離最小時刻」は数分〜数十分ずれ得る（moveM最小 ≤ 方位一致時刻のmoveM が常に成立）。canonical を moveM最小に置くのは、撮影者にとって「最小移動で撮れる瞬間」を返すためである。

### 4. 対象評価点（検索・プレビュー共通）

- 全形状共通で、評価点は **登録座標の中心軸上端 ＝ target lat/lng + height**（`elAng(distH, sElev, t.elev, t.h)` と中心方位 `tAz = brng(→中心)`）。
- 対象形状：person / cylinder / box / cone / frustum / landmark building。
- `physW` / `topWidth` / `frontAz` / `silhouette` は**検索評価点には使わない**（主にプレビューのシルエット描画・KMZ 3D用）。
- **検索とプレビューの評価点は一致**している（プレビューの一直線判定も中心軸上端に対する角距離）。
- KMZ 輸出の一部形状（四角柱の上面角など）が天体軌跡アンカーになる差異は**別の既知課題として記録し、今回は修正しない**。

### 5. 現Productionに存在する時刻不整合（既知課題・現状記録）

- Production `7f3b466` の `_search.js` には `azMatchTime` / `dispDt`（方位差のみ最小の時刻）が存在し、結果の `azDiff` / `alt` / `angDiam` は **dispDt 由来**。
- 一方、`chance.js` / `pinpoint.js` の API `time` は **`jst(fd.dt)`** で canonical時刻を**秒切り捨て**して返す。
- したがって現在は **date / time / moveM / ts → canonical `fd.dt`**、**azDiff / alt / angDiam → dispDt** という**異なる時刻由来値の混在**がある。
- これは Production 現状の記録であり、**正しい最終仕様として固定しない**（下記「次回修正として確定した仕様」で整合させる）。

### 6. 次回修正として確定した仕様（表示時刻の丸めのみ）

次回 SUN AND MOON 修正では以下を確定仕様とする。

- **検索ロジックは変更しない。** canonical `fd.dt`（moveM最小 sub-minute）から**最近傍1分の `displayDt`** を生成する：
  - 秒 < 30 → 当該分 / 秒 ≥ 30 → 次の分
  - 日跨ぎも、Date 時刻そのものを丸めてから同一の丸め済み `displayDt` で生成する（例：23:59:30 → 翌日 00:00）。
- **`displayDt` から生成する（ユーザー表示時刻に依存する値）**：`date` / `time` / `azDiff` / `alt` / `angDiam` など「その表示時刻の瞬間」を表す値は、すべて同一の `displayDt` から生成する。これにより現状の時刻由来値の混在（§5）を解消する。
- **`fd.dt` / moveM最小探索を変更しない（検索側）**：`moveM` / 採否 / ★評価の基礎となる検索結果 / candidate / sort / dedup / canonical event time は現行のまま変更しない。
- `azMatchTime` / angSep 最小等の**別時刻探索は行わない**（方位一致時刻・角距離最小時刻を表示用に別途求めない）。
- **Rev1 / Rev2 の angSep（角距離）最小表示時刻探索は、いずれも不採用**。理由は Rev ごとに異なる：
  - **Rev1 不採用理由**：angSep 最小の探索が CHANCE/PINPOINT 成立区間（moveM≤閾値）の**外へ流出し得た**（実測：14:18 の canonical に対し表示が 13:14 まで移動、その地点の moveM=379.8m で chance≤200m を満たさない）。表示時刻が「そのチャンスに属さない別状況」を指してしまうため不採用。
  - **Rev2 不採用理由**：Rev2 は探索を moveM≤閾値区間内へ制限して Rev1 の流出欠陥を解消している。したがって「閾値外へ流出するから」ではない。不採用の理由は、**canonical `fd.dt`（moveM最小）とは別の angSep 最小時刻をユーザー表示時刻にするという仕様変更自体を採用しない**ため（表示時刻の意味を canonical event time から乖離させる設計を採らない）。

### 既知の自動テスト課題

2026-08-18 current baselineでSUN AND MOON chance/pinpoint系の既知failが2件残る。DEV環境追加による回帰として扱わず、検索アルゴリズム側の別課題として切り分ける。ただし発売判断時には再評価する。2件は性質が異なる：
- `日別最小moveM 代表` 系＝canonical 探索側（moveM最小の代表選定）の既知課題。表示時刻の丸めとは無関係。
- `pinpoint 上端中央近傍 altPct≈100` 系＝上記「現Productionに存在する時刻不整合」（`time`/`moveM`/`ts`=canonical、`alt`/`azDiff`/`angDiam`=dispDt の混在）との関連が未解決。以前の表示時刻変更検証で自然に pass 化した実績があり、「表示時刻とは無関係」とは断定しない。次回 display-time 整合修正（表示依存値を同一 `displayDt` 由来へ統一）時に再評価する。
