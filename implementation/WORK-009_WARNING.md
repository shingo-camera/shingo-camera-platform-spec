# WORK-009 Warning Notification

Status: Approved

## 目的
不審な利用を定期判定し、対象がある場合だけ管理者へメール通知する。

## 実装
- Cloudflare Cron Trigger
- ログ集計
- M_SYSTEM_SETTING読込
- スコア計算
- T_WARNING登録
- 重複通知防止
- 管理者メール

## 完了条件
同一事象の重複メールがなく、自動停止を行わず、閾値変更で再デプロイ不要。
