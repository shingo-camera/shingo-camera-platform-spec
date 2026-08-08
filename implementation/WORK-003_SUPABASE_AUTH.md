# WORK-003 Supabase Authentication

Status: Completed

## 目的
共通アカウントの新規登録、メール認証、ログイン、ログアウト、パスワード再設定を実装する。

## 画面
`/login` `/signup` `/forgot-password` `/reset-password`

## 実装
- Supabase Client共通化
- 新規登録
- メール認証案内
- ログイン・ログアウト
- パスワード再設定
- セッション取得
- X-Device-Id生成・保存

## 完了条件
Productionで登録、認証、ログイン、ログアウト、再設定が成功し、秘密キーがフロントに存在しない。
