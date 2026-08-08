# Warning Operation

Status: Approved

## 1. 目的

不正利用を自動排除するのではなく、利用状況を把握し、共有利用への牽制と管理者判断を支援する。

## 2. 通常運用

普段は管理画面を毎日確認しない。

Warningが通知された時だけ対象ユーザー詳細を確認する。

## 3. 判定元

- T_LOGIN_LOG
- T_ACCESS_LOG
- DEVICE_ID
- IP
- COUNTRY_CODE
- REGION
- CITY
- 時系列

## 4. 初期Warning候補

初版（WORK-009）では LOGIN_FAILURE / MANY_DEVICES / MANY_REGIONS / COUNTRY_CHANGE の4種を実装する。IMPOSSIBLE_TRAVEL / CONCURRENT_ACCESS は種別として残すが初版では判定を実装しない（保留）。判定窓は初版では固定値とし、変更用 SETTING_KEY は設けない。

### LOGIN_FAILURE

T_LOGIN_LOG を対象に、直近60分で同一ユーザーのログイン失敗が LOGIN_FAIL_LIMIT（既定5）回以上。

### MANY_DEVICES

T_ACCESS_LOG.DEVICE_ID を対象に、直近24時間で同一ユーザーの異なる DEVICE_ID が MANY_DEVICES_LIMIT（既定4）台以上。NULL/空の DEVICE_ID は端末数に数えない。

### MANY_REGIONS

T_ACCESS_LOG を対象に、直近24時間で同一ユーザーの異なる地域（COUNTRY_CODE + REGION）が MANY_REGIONS_LIMIT（既定3）地域以上。COUNTRY_CODE / REGION が判定不能なログを独立地域として水増ししない。

### COUNTRY_CHANGE

T_ACCESS_LOG.COUNTRY_CODE を対象に、直近24時間で同一ユーザーの異なる有効な COUNTRY_CODE が2か国以上。NULL/空の COUNTRY_CODE は国数に数えない。固定条件のため国数用 SETTING_KEY は設けない。これ単独でユーザー停止等は行わない。

### IMPOSSIBLE_TRAVEL（初版保留）

短時間で現実的でない地域移動。現行ログに緯度経度がなく、距離・速度の客観判定仕様も未確定。CITY/REGION だけから推測実装しない。緯度経度カラム追加も WORK-009 では行わない。将来別途仕様化する。

### CONCURRENT_ACCESS（初版保留）

同時間帯に異なる地域・端末から継続利用。同時利用のセッション・時間差・DEVICE_ID 等の判定条件が未確定。初版では推測実装しない。将来別途仕様化する。

## 5. スコア

初期値はM_SYSTEM_SETTINGで管理する。既存のスコアキーは維持し、初版で新しいスコア値は追加しない。

```text
DEVICE_CHANGE_SCORE
REGION_CHANGE_SCORE
COUNTRY_CHANGE_SCORE
WARNING_SCORE
```

スコア値は運用データを見て調整する。初期値は確定的な不正判定を意味しない。

### 初版の検知とスコア方式（確定）

初版4種は、各Warning種別ごとの閾値を満たした時点で検知し、種別ごとに独立して T_WARNING へ登録する。通知条件に WARNING_SCORE の合算判定は用いない。複数種別が同時検知されてもスコアは合算せず、種別ごとに別レコードとして登録・通知する。

T_WARNING.WARNING_SCORE に記録する値:

- LOGIN_FAILURE: 0（初版では対応スコアキーを設けないため0を記録）
- MANY_DEVICES: DEVICE_CHANGE_SCORE の値
- MANY_REGIONS: REGION_CHANGE_SCORE の値
- COUNTRY_CHANGE: COUNTRY_CHANGE_SCORE の値

WARNING_SCORE（既定100）は削除・変更せず維持するが、WORK-009初版では通知可否の判定に使用しない。将来の優先度判定・スコア方式拡張のために残す。

検知・個別スコア・WARNING_SCORE のいずれも、自動停止には使用しない（停止・再開・最終判断は管理者のみ）。

## 6. 通知

## 6. 通知

Cloudflareの定期処理（Cron Trigger）でWarning候補を確認する。

初期頻度:

```text
1時間ごと
```

通知対象がある場合のみ管理者へメールする。Warning が0件ならメールを送信しない。

### 送信方式

- サービス: Resend。用途は Warning 発生時の管理者通知のみ。現在の想定送信量では Free プランの範囲で運用する。
- API Key: 既存 Secret MAIL_API_KEY を使用。新しい Key 用 SETTING_KEY / Secret は追加しない。実値は正本・コード・Git に保存しない。
- From: warning@shingo-camera.com（Resend でドメイン認証が必要。Production E2E 前に設定）。
- To: ADMIN_AUTH_USER_ID に対応する M_USER.LOGIN_MAIL。管理者メール用の新 SETTING_KEY / Secret は追加しない。ADMIN_AUTH_USER_ID は管理者認証・識別の基準であり、LOGIN_MAIL は通知先取得にのみ使う。メールアドレスだけで管理者判定はしない。

### 重複通知防止

新しい通知履歴テーブルは作らない。既存 T_WARNING と NOTIFIED_DATE、既存 WARNING_MAIL_INTERVAL_MIN=60 を使う。同一ユーザー・同一 WARNING_TYPE について、直近の NOTIFIED_DATE から WARNING_MAIL_INTERVAL_MIN 以内は同種メールを再送しない。送信できた場合は当該 T_WARNING の NOTIFIED_DATE を更新する。

## 7. 通知メール

件名例:

```text
【要確認】利用状況の確認が必要です
```

本文:

- 対象メールアドレス
- 商品
- Warning種別
- スコア
- 検知日時
- 端末数
- 地域数
- 管理画面の対象ユーザーURL

パスワード、JWT、秘密情報は含めない。

## 8. 管理者対応

### 問題なし

```text
T_WARNING.STATUS = 1
MEMO = 本人利用を確認
```

### 本人確認中

```text
T_WARNING.STATUS = 2
```

### 除外

VPN等で継続的に誤検知する場合:

```text
T_WARNING.STATUS = 9
```

除外はユーザー全体を永久にWarning対象外とする意味ではない。

### 一時停止

明らかに共有利用が疑われる場合:

```text
M_USER.STATUS = 2
```

または商品単位で:

```text
T_USER_PRODUCT.STATUS = 2
```

## 9. ユーザーへの連絡

Warning検知だけではユーザーへ自動送信しない。

管理者が停止または確認が必要と判断した時だけ連絡する。

文面は断定せず、セキュリティ確認として案内する。

## 10. 抑止効果

利用状況を確認していること自体を牽制として利用する。

停止後に正規利用者から説明があれば、管理画面から即時再開する。
