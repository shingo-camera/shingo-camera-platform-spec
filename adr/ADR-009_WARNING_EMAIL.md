# ADR-009 Warningはメール通知を基本運用とする

Status: Approved

## 決定

管理者が毎日ダッシュボードを確認する運用にしない。

Warning検知時だけ管理者へメール通知する。

## 理由

- 手のかからない運用を優先
- 想定人数ではWarning頻度は低い見込み
- 問題発生時だけ詳細確認すればよい
- 通知が多い場合は閾値を調整できる

## 実装

Cloudflareの定期処理で判定する。

同一事象の重複通知はT_WARNINGで抑止する。

## メールサービス（WORK-009で確定）

- サービスは Resend を採用する。用途は Warning 発生時の管理者通知のみ。現在の想定送信量では Free プランの範囲で運用する（料金プランの具体的な上限値は変動しうるため正本には固定しない）。
- Cloudflare Workers から Resend API を呼び出す。API Key は既存 Secret MAIL_API_KEY を使用し、新しい Key 用 SETTING_KEY / Secret は追加しない。実値は正本・コード・Git に保存しない。
- 送信元は warning@shingo-camera.com。Resend でのドメイン認証を Production 運用前に行う。
- 送信先は ADMIN_AUTH_USER_ID に対応する M_USER.LOGIN_MAIL。管理者メール用の新 SETTING_KEY / Secret は追加しない。ADMIN_AUTH_USER_ID は管理者認証・識別の基準のままとし、LOGIN_MAIL は通知先取得にのみ使う（メールアドレスだけで管理者判定はしない）。
- 本文に Password / JWT / Secret / API Key 等の秘密情報を含めない。
