# WORK-009 Warning Notification

Status: Completed

## 目的
不審な利用を定期判定し、対象がある場合だけ管理者へメール通知する。

## 実装
- Cloudflare Cron Trigger（1時間ごと）
- ログ集計（T_LOGIN_LOG / T_ACCESS_LOG）
- M_SYSTEM_SETTING読込（閾値・抑止間隔）
- Warning判定（初版4種）
- T_WARNING登録
- 重複通知防止（T_WARNING.NOTIFIED_DATE + WARNING_MAIL_INTERVAL_MIN）
- 管理者メール（Resend、Warning発生時のみ）

## 初版で実装するWarning（4種）

判定窓は初版では下記の固定値とする（判定窓を変更するSETTING_KEYは初版では設けない）。

- LOGIN_FAILURE: T_LOGIN_LOG。直近60分に同一ユーザーのログイン失敗が LOGIN_FAIL_LIMIT（既定5）回以上。
- MANY_DEVICES: T_ACCESS_LOG.DEVICE_ID。直近24時間に同一ユーザーで異なる DEVICE_ID が MANY_DEVICES_LIMIT（既定4）台以上。NULL/空の DEVICE_ID は端末数として数えない。
- MANY_REGIONS: T_ACCESS_LOG。直近24時間に同一ユーザーで異なる地域（COUNTRY_CODE + REGION）が MANY_REGIONS_LIMIT（既定3）地域以上。COUNTRY_CODE / REGION が判定不能なログを独立地域として水増ししない。
- COUNTRY_CHANGE: T_ACCESS_LOG.COUNTRY_CODE。直近24時間に同一ユーザーで異なる有効な COUNTRY_CODE が 2か国以上。NULL/空の COUNTRY_CODE は国数として数えない。これ単独でユーザー停止等は行わない。固定条件のため国数用 SETTING_KEY は追加しない。

## 初版で保留するWarning（2種）

Warning種別としては正本に残すが、初版では判定処理を実装しない。

- IMPOSSIBLE_TRAVEL: 現行ログに緯度経度がなく、距離・速度の客観判定仕様も未確定。CITY/REGION だけから推測実装しない。緯度経度カラム追加も WORK-009 では行わない。将来必要になった時点で別途仕様化する。
- CONCURRENT_ACCESS: 同時利用のセッション・時間差・DEVICE_ID 等の判定条件が未確定。初版では推測実装しない。将来必要になった時点で別途仕様化する。

## M_SYSTEM_SETTING

既存キーを再利用し、不足分のみ追加する（同義キーを重複作成しない）。

再利用（既存）:
- LOGIN_FAIL_LIMIT = 5（LOGIN_FAILURE 閾値）
- WARNING_MAIL_INTERVAL_MIN = 60（同一対象・同一種別の通知抑止間隔・分）
- DEVICE_CHANGE_SCORE / REGION_CHANGE_SCORE / COUNTRY_CHANGE_SCORE / WARNING_SCORE（既存スコア方式を維持）

新規追加（実装工程で migration 化。今回は正本記載のみ）:
- MANY_DEVICES_LIMIT = 4（TEXT保持、利用側で整数変換）
- MANY_REGIONS_LIMIT = 3（TEXT保持、利用側で整数変換）

COUNTRY_CHANGE は「24時間以内に有効な2か国以上」の固定条件のため、国数用 SETTING_KEY は追加しない。

## スコア記録（確定）

初版は各種別の閾値超過そのものを検知条件とし、通知可否に WARNING_SCORE の合算判定は用いない。Warning は種別ごとに独立して T_WARNING へ登録し、複数種別が同時検知されてもスコアは合算しない。

T_WARNING.WARNING_SCORE に記録する値:

- LOGIN_FAILURE: 0（初版では対応スコアキーを設けない）
- MANY_DEVICES: DEVICE_CHANGE_SCORE の値
- MANY_REGIONS: REGION_CHANGE_SCORE の値
- COUNTRY_CHANGE: COUNTRY_CHANGE_SCORE の値

既存 WARNING_SCORE（既定100）は削除・変更せず維持するが、初版では通知可否の判定に使用しない。将来の優先度判定・スコア方式拡張のために残す。検知・個別スコア・WARNING_SCORE のいずれも自動停止には使用しない。

## メール送信（Resend）

- サービス: Resend。用途は Warning 発生時の管理者通知のみ。現在の想定送信量では Free プランの範囲で運用する。
- 呼び出し: Cloudflare Workers から Resend API。API Key は既存 Secret MAIL_API_KEY を使用（新しい Key 用 SETTING_KEY / Secret は追加しない）。API Key の実値は正本・コード・Git に保存しない。
- From: warning@shingo-camera.com（Resend 側で shingo-camera.com のドメイン認証が必要。Production E2E 前に設定）。
- To: ADMIN_AUTH_USER_ID に対応する M_USER.LOGIN_MAIL。管理者メール用の新 SETTING_KEY / Secret は追加しない。ADMIN_AUTH_USER_ID は従来どおり管理者認証・識別の基準であり、LOGIN_MAIL はその管理者への通知先取得にのみ使用する。メールアドレスだけで管理者判定は行わない。
- 送信条件: Warning が存在する場合のみ送信。0件なら送信しない。ユーザー本人へ自動送信しない。
- 本文の最低項目: Warning種別 / 対象ユーザーを管理者が特定できる情報 / 検知日時 / 判定根拠の概要。Password / JWT / Secret / API Key 等は含めない。

## 重複通知防止

新しい通知履歴テーブルは作らない。既存 T_WARNING と NOTIFIED_DATE を使用する。

Warning レコードの重複防止とメール再送抑止を分離する。

- T_WARNING の継続判定: 同一 AUTH_USER_ID + WARNING_TYPE について、PERIOD_END が GAP（= 各種別の判定窓。LOGIN_FAILURE=60分 / MANY_DEVICES・MANY_REGIONS・COUNTRY_CHANGE=24時間）以内の既存行があれば、STATUS(0/1/2/9) を問わず同一の継続事象としてその1行を再利用（UPDATE）する。管理者が STATUS を変更していても、条件が継続している同じ Warning の新規行は作らない。PERIOD_END が GAP を超えていれば「解消済み」とみなし、再発は新規行とする。継続中は DETECT_DATE / PERIOD_START を初回値で固定し、PERIOD_END のみ更新する。STATUS / MEMO / LAST_ACTION_DATE は Cron から変更しない。

- メール送信（初版: 1事象 = T_WARNING 1行 = メール1通）: 新規 T_WARNING 作成時に NOTIFIED_DATE が NULL の場合のみ管理者へ送信し、送信成功時のみ NOTIFIED_DATE を記録する。同じ WARNING_ID を継続再利用している間は、NOTIFIED_DATE が入っていれば（STATUS=0 のままでも、STATUS=1/2/9 でも）再送しない。メール送信失敗時は NOTIFIED_DATE を NULL のまま残し、新規行も作らず、次回 Cron で同じ WARNING_ID の再送を試行する。GAP 超過後に再発して新しい WARNING_ID が作られた場合は、その新事象について改めて1通送る。

既存 WARNING_MAIL_INTERVAL_MIN=60 は削除・変更せず維持するが、WORK-009 初版では同一 WARNING_ID の再送条件には使用しない（将来のリマインド通知等のために残す）。

## Cron

Cloudflare Cron Trigger で1時間ごとに判定を実行する。wrangler.toml への設定変更は実装工程で行う（本ドキュメント時点では未変更）。実装時の必須確認事項として、keep_vars=true および既存 D1 binding・database_id・[vars] を壊さないこと。

## 管理画面

WORK-006 で実装済みの Warning 一覧（GET /api/admin/warnings）・ユーザー詳細を利用する。新しい管理画面を別系統で作らない。T_WARNING へ登録された Warning を既存管理画面から確認し、管理者が STATUS（0/1/2/9）・MEMO で対応する現行設計を維持する。

## 自動停止禁止（重要）

Warning検知・スコア超過・メール通知のいずれを理由としても、アカウント（M_USER.STATUS）や商品権限（T_USER_PRODUCT.STATUS）を自動停止しない。Warning は管理者判断の材料に限る。停止・再開・最終判断は管理者のみが行う。

## 完了条件
初版4種の検知、同一事象の重複メールなし、自動停止なし、閾値変更で再デプロイ不要、Warning 0件時にメール非送信、本文に秘密情報を含めないこと。

## Production E2E結果（Completed）

2026-08-09 に Production 環境で E2E を実施し、以下を確認した。

- Cron（`0 * * * *`）による LOGIN_FAILURE Warning の検知成功。
- T_WARNING への登録成功（WARNING_ID=1 / WARNING_TYPE=LOGIN_FAILURE / STATUS=0）。
- Resend による管理者宛メールの実送信成功、管理者による実受信を確認。
- 送信成功後の NOTIFIED_DATE 記録を確認（2026-08-09T08:00:53.178+09:00）。
- 次回 Cron（9時台）通過後も同一継続事象は T_WARNING 1行のまま（重複 Warning なし）。
- NOTIFIED_DATE は初回値のまま変化せず、Resend 送信・受信メールとも再送なし（重複通知抑止をProductionで確認）。
- 「1事象 = T_WARNING 1行 = メール1通」および継続事象の1行再利用が Production 実データで成立することを確認。

E2E 用に投入したテストデータ（識別子 `WORK009_PROD_E2E` のログイン失敗ログ、およびそれにより生成された WARNING_ID=1 / LOGIN_FAILURE）は、E2E 完了後に特定・削除済み（実運用データは削除していない）。

デプロイ情報: Version ID `1defe567-c824-4f20-81ad-7a0889b443cd`、migration 0004 適用済み、Resend `shingo-camera.com` Verified、Production `MAIL_API_KEY` 登録済み。実装は実装リポジトリの `820dae4 Implement WORK-009 warning notification` にて commit / push 済み。
