# Security Policy

Status: Approved

## 1. 基本方針

- 正規ユーザーを過度に疑わない
- 不正利用を抑止する
- 誤検知を前提とする
- 自動永久停止は行わない
- 最終判断は管理者が行う
- 管理者操作で復旧可能にする
- 不要な外部SaaSを追加しない

## 2. セキュリティ境界

### 認証

Supabase Authを使用する。

### API

Cloudflare Workers / Pages FunctionsでJWTを検証する。

### DB

Cloudflare D1を利用し、ブラウザから直接操作させない。

### 決済

Stripe Checkoutを利用し、カード情報を自サイトで保持しない。

### 環境変数と秘密情報

Cloudflare の通常環境変数（Dashboard 管理・非秘密）:

```text
APP_ENV
SUPABASE_URL
SUPABASE_ANON_KEY
ADMIN_AUTH_USER_ID
```

- SUPABASE_URL / SUPABASE_ANON_KEY はフロントへ配布可（/api/config 経由）。
- ADMIN_AUTH_USER_ID は秘密鍵ではないが、フロント / API レスポンスへ値を露出しない
  （管理者判定にのみサーバー側で使用する）。
- これらは Cloudflare Dashboard の通常環境変数として管理し、wrangler.toml へ実値を書かない。
- wrangler.toml の `keep_vars = true` により、deploy 時に Dashboard 管理の通常環境変数を
  削除・上書きしない。

Cloudflare Secrets へ保存する秘密情報（`wrangler secret put`、deploy で消えない）:

```text
SUPABASE_SERVICE_ROLE_KEY
STRIPE_SECRET_KEY
STRIPE_WEBHOOK_SECRET
MAIL_API_KEY
```

Service Role Key、Stripe Secret Key、Webhook Secret、メール送信用秘密情報をフロントへ渡さない。

## 3. 認証トークン

- Authorization Bearerで送信
- API側で署名・有効期限を検証
- 検証済みsubをAUTH_USER_IDとして利用
- リクエスト本文のAUTH_USER_IDは信用しない
- 完全なJWTをログ出力しない

## 4. 商品権限

画面表示だけでなく、各アプリ固有APIでも商品権限を確認する。

```text
画面で利用可能
≠
APIを無条件で利用可能
```

商品API側で毎回`requireProduct()`を通す。

## 5. 管理者権限

初期管理者は1人。

管理者AUTH_USER_IDをサーバー側環境設定で管理する。

管理者メールアドレスのみで判定しない。

管理者APIはすべて`requireAdmin()`を通す。

### 管理者自身のSTATUS変更制限

`ADMIN_AUTH_USER_ID` と対象 AUTH_USER_ID が同一の場合、管理不能を避けるため
STATUS=2（停止）・STATUS=9（退会）への変更を禁止する（STATUS=1 の有効化は許可）。
禁止時は HTTP 400 / code=ADMIN_SELF_STATUS_CHANGE_NOT_ALLOWED を返す。

### ADMIN_AUTH_USER_ID の取り扱い

- サーバー側環境変数として保持し、管理者判定にのみ使用する。
- 値そのものをフロント / API レスポンス / /api/config へ露出しない。
- 秘密鍵ではないが、通常環境変数として Cloudflare Dashboard で管理する。

## 6. 入力

- Prepared Statementを使用
- 文字数上限を設ける
- UUID・日時・コード値を検証
- CSVは形式、列、ファイルサイズを検証
- 想定外ファイル形式は受け付けない
- HTMLを保存・表示する場合はエスケープする
- MEMOはテキストとして扱う

## 7. ログ

保存しない情報:

- パスワード
- パスワードハッシュ
- 完全なJWT
- Stripe秘密キー
- Supabase Service Role Key
- カード情報

IP、DEVICE_ID、User-Agentは不正利用確認に必要な範囲で保持する。

## 8. Warning

Warningは不正確な可能性がある補助情報とする。

自動停止しない。

管理者へ通知し、アクセス履歴を確認したうえで判断する。

## 9. レート制限

初期はCloudflare標準機能とアプリ側の簡易制御を使用する。

優先対象:

- ログイン失敗記録API
- パスワード再設定依頼
- note移行試行
- Stripe Checkout作成
- 管理者CSV取込

必要になった場合のみ専用の有料機能を追加する。

## 10. エラー表示

外部サービスやDBの詳細を利用者へ返さない。

利用者向けは短い業務メッセージとする。

内部詳細はCloudflareログへ記録する。

## 11. バックアップと復旧

バックアップは機密情報として扱う。

CSV・DBダンプを公開リポジトリへ置かない。

個人情報やIPを含むバックアップはローカルまたは非公開ストレージに保管する。
