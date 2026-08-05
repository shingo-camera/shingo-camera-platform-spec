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

### LOGIN_FAILURE

一定期間内のログイン失敗が多い。

### MANY_DEVICES

短期間に多数のDEVICE_IDから利用。

### MANY_REGIONS

短期間に多数地域から利用。

### COUNTRY_CHANGE

短時間に異なる国から利用。

### IMPOSSIBLE_TRAVEL

短時間で現実的でない地域移動。

### CONCURRENT_ACCESS

同時間帯に異なる地域・端末から継続利用。

## 5. スコア

初期値はM_SYSTEM_SETTINGで管理する。

```text
DEVICE_CHANGE_SCORE
REGION_CHANGE_SCORE
COUNTRY_CHANGE_SCORE
WARNING_SCORE
```

スコア値は運用データを見て調整する。

初期値は確定的な不正判定を意味しない。

## 6. 通知

Cloudflareの定期処理でWarning候補を確認する。

初期頻度:

```text
1時間ごと
```

通知対象がある場合のみ管理者へメールする。

同一ユーザー・同一Warning種別の短時間重複通知を避ける。

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
