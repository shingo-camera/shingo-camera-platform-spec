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

- 配置: スマホ右上の衛星ボタン（`#map-sat-btn`）直下の右端縦列。衛星の直下（`top:52px` = 衛星 `top:8px` + 高さ36px + 8pxギャップ）へ同幅・同スタイルで積む。既存の衛星／メニュー／対象追加／ズーム／検索のいずれとも重ならない。
- safe-area 考慮（`env(safe-area-inset-top/right)`）。
- 表示: PCは非表示（`#map-body-toggle{ display:none; }`）、スマホのみ `@media (max-width:600px)` で `display:flex`。
- ロジック: 既存 `setSunsetMode(!sunsetMode)` を再利用（新規ロジックを作らない）。状態は `setSunsetMode` 内で一元管理し、ボタン表示（☀ 太陽／☾ 月）もそこで同期する。
- 既存の衛星ボタン（`#map-sat-btn`）・メニューボタン（`#map-menu-btn`）・検索・ズームの位置と機能は変更しない。

補足: 初期実装では独立した固定座標（左上）に置いていたが、実機で既存コントロールと重なったため、既存スマホボタン群と同じレイアウト方式（右端縦列）へ組み込む現行仕様に修正した。旧CSS配置（左上固定・`display:block` 切替）は現行実装では使用しない。

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
- スマホ太陽/月トグルは本書上部の現行仕様（衛星ボタン直下）を維持。
- DEV起動では `/dev/apps/sun-and-moon/` から自前APIを `/dev/api/apps/sun-and-moon/*` へ解決する。2026-08-18に `app-start` 200 OKを実機確認済み。

### 既知の自動テスト課題

2026-08-18 current baselineでSUN AND MOON chance/pinpoint系の既知failが2件残る。DEV環境追加による回帰として扱わず、検索アルゴリズム側の別課題として切り分ける。ただし発売判断時には再評価する。
