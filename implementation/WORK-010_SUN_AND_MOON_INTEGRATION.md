# WORK-010 SUN AND MOON Integration

Status: Completed

## 目的
既存SUN AND MOONへ共通認証・商品権限確認を追加する。

## 前提
最新`portrait.html`を唯一の実装基準とし、Moonは統合済みとして扱う。

## 実装
- 未ログイン時ログインへ
- `SUN_AND_MOON`権限確認
- 権限なし時商品案内
- 計算APIでも権限確認
- アクセスログ

## 禁止
天体計算、APIレスポンス、既存UIの無関係な変更。

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
