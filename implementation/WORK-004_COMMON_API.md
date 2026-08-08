# WORK-004 Common API Foundation

Status: Completed

## 目的
全APIで利用する認証、管理者判定、レスポンス、入力検証、D1アクセスを共通化する。

## 共通関数
`requireUser()` `requireAdmin()` `jsonOk()` `jsonError()` `validateJson()` `getDeviceId()` `nowIso()` `getSystemSetting()` `writeLoginLog()` `writeAccessLog()`

## 完了条件
- 正常JWTでAUTH_USER_ID取得
- 不正JWTで401
- 管理者・一般ユーザー分離
- 共通レスポンス統一
- 入力エラー形式統一
