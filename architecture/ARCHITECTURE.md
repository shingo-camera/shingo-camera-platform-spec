# Platform Architecture

Status: Approved

## 1. 全体構成

```text
ブラウザ
├─ 公開トップページ
├─ アカウント画面
├─ ログイン後ホーム
├─ 管理画面
└─ 各アプリ
    ├─ SUN AND MOON
    └─ HANABI
        ↓
Cloudflare Workers / Pages Functions
├─ 認証トークン検証
├─ M_USER同期
├─ 商品権限確認
├─ 購入処理
├─ note移行
├─ 管理者API
├─ ログ記録
└─ Warning判定
        ↓
Cloudflare D1
├─ M_USER
├─ M_PRODUCT
├─ T_USER_PRODUCT
├─ T_PURCHASE
├─ T_NOTE_PURCHASE
├─ T_LOGIN_LOG
├─ T_ACCESS_LOG
└─ M_SYSTEM_SETTING
```

外部サービス:

```text
Supabase Auth
└─ メールアドレス、パスワード、メール認証、セッション

Stripe
└─ 決済、Webhook通知

note
└─ 既存購入履歴CSV
```

## 2. 責務分担

### 2.1 Supabase Auth

担当:

- 新規登録
- ログイン
- ログアウト
- メール認証
- パスワード再設定
- パスワード変更
- AUTH_USER_ID発行
- セッション管理

担当しない:

- 商品権限
- 購入履歴
- note移行
- 管理者による商品付与

### 2.2 Cloudflare Workers / Pages Functions

担当:

- Supabaseの認証トークン検証
- D1アクセス
- M_USER同期
- 商品権限判定
- Stripe Checkout開始
- Stripe Webhook受信
- note移行
- 管理者API
- アクセスログ
- Warning抽出・通知

### 2.3 Cloudflare D1

担当:

- 業務データの正本
- 商品マスタ
- 商品権限
- 購入履歴
- note移行元データ
- ログ
- 運用設定

パスワードは保存しない。

### 2.4 Stripe

担当:

- 決済
- 決済結果通知
- 将来のサブスクリプション管理

Stripeは商品利用権限を管理しない。

### 2.5 アプリ

担当:

- アプリ固有の画面
- アプリ固有の計算
- アプリ固有の描画
- 共通APIを使った認証・商品権限確認

## 3. 共通化するもの

- 認証トークン取得
- APIクライアント
- APIレスポンス形式
- D1接続
- 商品権限確認
- 管理者権限確認
- DEVICE_ID発行
- 入力チェック
- 日時・金額表示
- 共通ボタン、フォーム、ダイアログ

## 4. 共通化しないもの

- SUN AND MOONの天体計算
- HANABIの花火計算
- 風補正
- アプリ固有の描画
- アプリ固有の画面状態
- アプリ固有の撮影計画データ

共通化は、実際に2箇所以上から使うものを優先する。

## 5. URL方針

初期案:

```text
/                     公開トップ
/login                ログイン
/signup               新規登録
/forgot-password      パスワード再設定
/home                 ログイン後ホーム
/account              アカウント設定
/products             商品一覧
/admin                管理画面
/apps/sun-and-moon    SUN AND MOON
/apps/hanabi          HANABI
```

既存アプリのURL構成を大きく変える必要がある場合は、段階的に移行する。

## 6. リポジトリ方針

設計書リポジトリ:

```text
shingo-camera-platform-spec
```

コードは別リポジトリまたは既存アプリリポジトリで管理する。

初期実装では既存SUN AND MOONコードを無理にモノレポへ移動しない。

まず共通アカウント基盤を作成し、その後各アプリへ段階的に接続する。

## 7. 将来拡張

新しいアプリ追加時の基本作業:

1. M_PRODUCTへ商品追加
2. 商品詳細・販売画面追加
3. 商品権限判定を追加
4. ログイン後ホームへカード追加
5. 必要に応じてStripe商品を追加

認証基盤とM_USERは変更しない。
