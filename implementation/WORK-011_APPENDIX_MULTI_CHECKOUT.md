# WORK-011 追加対応資料: 複数商品 Checkout / 支払い試行ライフサイクル

本書は独立した WORK ではなく、**WORK-011 Public Site（`WORK-011_PUBLIC_SITE.md`）実装中に必要となった追加対応**の設計・実装・Production 実績をまとめた付録資料である。正式な WORK-011 は Public Site を指す。

Status: 追加対応 完了（WORK-011 Public Site の一部。Production E2E 完了 / operator 確認済み・2026-08-11）

購入基盤を「1 Checkout = 1 商品」から、複数の買い切り商品を 1 回の Stripe Checkout でまとめて購入できる構造へ拡張する。STORE UI（複数選択画面）は本 WORK では実装しない。3D_PREVIEW サブスクリプションも本 WORK では実装しない（将来追加を阻害しない設計とする）。

## 対象商品

- SUN_AND_MOON
- HANABI
- HANABI_GOOGLE_EARTH（HANABI PLANNER の追加機能）

## 注文モデル

- ORDER = 1 回の決済（T_ORDER）
- PURCHASE = その注文に含まれる各商品購入明細（T_PURCHASE）
- ENTITLEMENT = 商品利用権限（T_USER_PRODUCT）

## DB 変更（migration 0005）

- T_ORDER 新設（注文ヘッダ）。`UX_T_ORDER_EXTERNAL (PURCHASE_SOURCE, EXTERNAL_ORDER_ID)`・`IX_T_ORDER_USER_DATE`。
- T_PURCHASE に ORDER_ID（nullable）を追加。`IX_T_PURCHASE_ORDER`・`UX_T_PURCHASE_ORDER_PRODUCT (ORDER_ID, PRODUCT_ID)`。
- 追加型のみ（CREATE TABLE / ADD COLUMN / CREATE INDEX）。既存データを削除・再作成しない。
- 既存 T_PURCHASE / EXTERNAL_PURCHASE_ID / UX_T_PURCHASE_EXTERNAL は変更しない（note 移行の冪等の正本）。

Stripe 新方式は T_ORDER 1 行 + T_PURCHASE N 行（ORDER_ID あり）。既存 note 購入・既存 Stripe 単品は T_ORDER なし・ORDER_ID=NULL（現状維持）。

## Checkout API

- Request は `{ "productCodes": ["HANABI", ...] }`（配列）。後方互換で単数 `{ "productCode": "..." }` も受理。
- サーバー検証: 配列・非空・重複拒否（除去しない）・商品存在・販売可否（Price 設定）・二重購入防止・依存条件・SORT_NO 正規化・Price ID サーバー解決・複数 line_items 生成（mode=payment・quantity=1）。
- 金額・Price ID・PRODUCT_ID・商品順をブラウザから受け取らない/信用しない。
- metadata は auth_user_id / product_codes（監査用途。権限付与の正本にしない）。

## 依存条件（HANABI_GOOGLE_EARTH）

HANABI_GOOGLE_EARTH は、HANABI を保有済み または 同一注文に HANABI を含む場合のみ購入可。バックエンド（purchase 共有ロジック）で必須検証し、UI 依存にしない。

## Webhook

WORK-007 の安全機構を維持:

- Stripe 署名検証（constructEventAsync + createSubtleCryptoProvider）
- raw body 保持
- checkout.session.completed かつ payment_status=paid のみ
- Stripe から Session 再取得（line_items / line_items.data.price を expand）
- Price 照合・金額照合・quantity 照合
- 不正署名拒否・Webhook 再送耐性

複数商品対応:

- 権限付与対象は再取得した実 line_items を正本とする（metadata を正本にしない）。
- 各 line_item: quantity===1 / price.id / currency jpy / unit_amount / サーバー既知 Price。
- Price ID → PRODUCT_CODE 逆引き（Secret から構築）。想定外 Price・Price 設定重複は付与せず拒否。
- 全 unit_amount 合計 == amount_total を照合。

## DB 反映（原子的・adr/ADR-008）

1 回の db.batch() に T_ORDER INSERT → T_PURCHASE INSERT×N → T_USER_PRODUCT INSERT/UPDATE×N を含め、全件成立または全件ロールバックとする（Local D1 で実測確認）。部分成功を許容して Stripe 再送で収束させる方式は採らない。

- last_insert_rowid() を複数明細で使わない。
- ORDER_ID は (PURCHASE_SOURCE, EXTERNAL_ORDER_ID) から、PURCHASE_ID は (ORDER_ID, PRODUCT_ID) から後続 SQL で参照。
- 冪等キーは T_ORDER の (PURCHASE_SOURCE, EXTERNAL_ORDER_ID)。再送時は既存 T_ORDER で処理済みと判定。

## 金額履歴

- T_ORDER.TOTAL_AMOUNT に注文合計、T_PURCHASE.AMOUNT に各商品の購入時点金額を保存。将来価格変更されても過去購入価格を保持。

## Secrets

```text
STRIPE_SECRET_KEY
STRIPE_WEBHOOK_SECRET
STRIPE_PRICE_SUN_AND_MOON
STRIPE_PRICE_HANABI
STRIPE_PRICE_HANABI_GOOGLE_EARTH
```

Price ID はソースにハードコードせず Secret を正とする。

## 将来 3D_PREVIEW（本 WORK では未実装）

- T_ORDER / T_PURCHASE / T_USER_PRODUCT 構造は subscription でも流用可能（START/END/STATUS/GRANT_TYPE で表現）。
- 買い切り + subscription を同一 Checkout で扱うかは 3D 実装時に Stripe 現行仕様を再確認して決定する（「必ず別 Checkout」とは確定しない）。
- 3D 実装時に追加想定: subscription 用 GRANT_TYPE/STATUS 運用、customer/subscription ID 保持、更新/解約 Webhook、3D×Planner 保有ゲート。

## 購入反映確認（status API・成功画面）

複数商品Checkout後の反映確認も複数商品対応とする（Checkoutだけ複数対応でstatusが単品、という矛盾を残さない）。

status API `GET /api/purchases/status`:

- `?productCodes=A,B,C`（カンマ区切り）を受理。後方互換で `?productCode=A`（単数）も受理し配列へ正規化。
- 判定は既存 isProductAvailable（entitlement正本。T_USER_PRODUCT の状態・期間）。T_ORDER/T_PURCHASE の存在では判定しない。ログイン中ユーザーの権限のみ（他ユーザー参照不可・URL改ざんで偽装不可）。
- Response は `{ allGranted, products:[{productCode, granted}...] }`。単数リクエスト時のみ従来形 `productCode`/`granted` も返す。
- 正規化ロジック（parseStatusProductCodes）と成功条件（computeAllGranted・全商品grantedのときのみtrue／0件はfalse）は純関数として切り出し、再実行可能テスト（node --test）で検証する。

成功画面 `public/purchase/success/`:

- URLの productCodes（単数 productCode も受理）を trim・空除外・重複一意化してから status API で全商品を確認。
- 全商品 granted のときのみ購入完了表示。一部未反映は polling 継続（maxAttempts=5・delay=2000ms、既存単品画面を踏襲）。上限到達しても決済失敗と断定せず反映確認中の安全文言を表示。
- 商品ごとの反映状態を表示（商品名は /api/products の name を再利用。取得失敗時は PRODUCT_CODE にフォールバック）。
- 権限付与・DB更新はしない（表示・確認のみ）。未ログイン時はログインを促す（ログインUX全体は改修しない）。

Checkout成功URL: `success_url=/purchase/success/?productCodes=<SORT_NO正規化順・カンマ連結・encodeURIComponent>`（重複はprecheckで排除済み）。

## 購入者メール固定 / 決済手段整理（WORK-011 追加修正）

Stripe Checkout でのメール再入力と Link 表示を解消する。

購入者メール:

- 購入には Platform 認証が必須。購入者メールは認証情報（AuthContext.email、検証済み JWT）から確定する。ブラウザ request body の email は使用しない（body に email を含めても parseProductCodes が productCodes/productCode のみ参照するため無視される）。
- email が取得できない場合は `AUTH_EMAIL_REQUIRED`（403）で Checkout 開始を拒否（Stripe 画面で補完させない）。
- Checkout Session に `customer_email`＝認証済み email を渡す。Stripe 公式仕様により email は事前入力かつ編集不可（readonly）になり、別メールでの購入を防ぐ。
- entitlement 付与先は AUTH_USER_ID（metadata.auth_user_id）を正本のまま。Stripe email は権限判定キーにしない。metadata へ email を重複保持しない。

決済手段:

- payment_method_types をコードで固定しない（Dynamic Payment Methods。Stripe Dashboard の Payment methods 設定を正とする）。将来 PayPay 等追加時も Worker/STORE の大規模改修を避けられる。
- Apple Pay は Hosted Checkout の標準機能で追加コード不要（Platform 側で独自 Apple Pay 実装をしない）。
- Link は Stripe Dashboard の payment method settings で無効化する（コード変更不要）。
- 金額は Stripe Price 側で管理（コードは合計照合のみ）。3 商品同時購入で SUN_AND_MOON 13,000 + HANABI 4,000 + HANABI_GOOGLE_EARTH 10,000 = 27,000 円を維持。

将来 Stripe Customer:

- 今回は買い切りのみのため Stripe Customer を 1:1 で恒久管理しない（新規 T_CUSTOMER を作らない）。customer_email のみ渡し customer（Customer ID）は渡さない。
- 将来 3D_PREVIEW subscription 実装時に Customer ID の保持が必要になるが、今回の customer_email 追加はそれを阻害しない（Customer ID 保持列の追加で拡張可能）。

security 確認:

- 未ログイン → Checkout 開始不可（401）。
- email 取得不能 → Checkout 開始不可（AUTH_EMAIL_REQUIRED 403）。
- request body の偽 email → 無視（認証 email を使用）。
- Stripe Checkout 購入者メール = 認証済み Platform email。
- Webhook entitlement 先 = AUTH_USER_ID（Stripe email で別ユーザーに権限が付かない）。

Dashboard 設定（ユーザー作業）:

- Link → OFF（payment method settings）。
- カード → ON。
- Apple Pay → 利用可能環境で表示を許可（デフォルト有効）。

## 変更しないもの

完成済みトップページ（public/index.html・Hero・レイアウト等）、STORE UI、SUN AND MOON 本体、HANABI 本体、ログイン UX、/home 統合、認証方式。既存 STORE（site.js）は後方互換の単数 productCode 受理で無変更のまま動作する。

## テスト（再実行可能）

`test/purchase_logic.test.mjs`（node --test）: Checkout の productCodes 検証（parseProductCodes・body の email は無視されること含む）／status の URL 正規化（parseStatusProductCodes）／成功条件（computeAllGranted）を検証。`npm test` で実行（pretest が esbuild で純関数をバンドル）。

Local D1 統合・純ロジック検証（開発時）: 複数商品 fulfill / rollback / 依存 / Price 逆引き / Checkout API / Webhook 検証 / note 回帰 / status 複数商品 / 購入者メール固定（customer_email 設定・email 欠損拒否・偽 email 無視）。

## 支払い試行層の堅牢化（migration 0006）

複数商品 Checkout の運用堅牢化として、支払い試行層を追加した（database/DDL.sql・database/TABLES.md・api/PURCHASE_API.md「支払い試行ライフサイクル」を正本）。

- T_CHECKOUT_ATTEMPT / T_CHECKOUT_ATTEMPT_ITEM: Checkout Session を「支払い試行」として追跡し、operationId により HTTP 再送を同一試行へ収束させる。
- T_PRODUCT_CHECKOUT_LOCK: `PRIMARY KEY (AUTH_USER_ID, PRODUCT_ID)` を並行 Checkout 排他の正本とし、同一商品の active 予約を最大 1 に制約する。
- T_PAYMENT_EVENT / T_ORDER.PAYMENT_INTENT_ID: duplicate paid / refund / dispute / fulfill failure / reconcile / server indeterminate を記録し、refund/dispute から注文を逆引きする土台とする。
- 追加した API: `GET /api/purchases/active-checkout`（再開可能な手続きの正本・別端末対応）、`POST /api/purchases/recover`（再開）、`POST /api/purchases/cancel`（取り消し）。checkout は restart 検出（`CHECKOUT_RESTART_CONFIRM` 等）と settle（Stripe 状態同期）を行う。
- Stripe 依存経路（complete 収束・expire 失敗時の resumable 非確定）は自動テスト不可のため、実機 E2E で確認する（下記 Production 実績）。

追加型のみ（CREATE TABLE / ADD COLUMN / CREATE INDEX）で既存データ非破壊。既存 T_ORDER(paid 注文)/T_PURCHASE/T_USER_PRODUCT/note 移行は無改変（adr/ADR-012）。

## Production 実績（operator 確認・2026-08-11）

以下は運用者（repo オーナー）が Production 環境で実際に確認した結果として記録する。コード・自動テストの範囲外の実環境確認であり、確認主体は運用者である。

- migration 0005 / 0006 の Production 未適用を検出し、`wrangler d1 migrations apply --remote` 相当で適用済み。
- Production Stripe Price Secret（SUN_AND_MOON / HANABI / HANABI_GOOGLE_EARTH）を設定。
- inactive Stripe Price による Checkout 失敗を実機で検出し、Active Price へ修正。
- SUN AND MOON 単品 Checkout 成功。
- 複数商品 Checkout 成功（3 商品同時購入 27,000 円を含む金額整合を確認）。
- STORE の購入済み表示・MY PAGE への権限反映を確認。
- テストユーザーの購入関連データを Admin から削除後、再購入試験を実施し正常。
- `APP_BASE_URL = "https://shingo-camera.com"` へ正式切替。Cloudflare custom domain `shingo-camera.com` で Production 運用開始。
- HOME title を `shingo_camera LABO` に変更。Platform 共通 favicon（商品固有アイコン非流用）を追加。
- 最終 Production スモーク合格。

補足（当該実機確認で検証済みの運用経路）: 二重 Checkout 時の既存購入手続き復元、購入手続き取り消し、Earth 依存関係、expired → 終端化 → lock 解放、expired 購入の再開時通知、pending checkout の復元。

## 追加対応の完了条件との対応

| 完了条件 | 対応 |
|---|---|
| 複数商品を 1 回の Checkout で購入 | checkout の productCodes 配列受理・複数 line_items・db.batch 原子反映（実装・自動テスト・実機） |
| 二重課金の事前防止 | T_PRODUCT_CHECKOUT_LOCK の PK 排他・restart 検出（実機で復元確認） |
| HTTP 再送収束 | operationId + UX_ATTEMPT_OPERATION（idempotency） |
| paid 未反映の救済 | active-checkout の complete→fulfillment 収束・success recovery |
| 購入者メール固定 | customer_email = 認証済み email・AUTH_EMAIL_REQUIRED |
| status / 成功画面の複数商品対応 | parseStatusProductCodes・computeAllGranted（純関数テスト） |
| DB 変更は追加型・非破壊 | migration 0005 / 0006（CREATE/ADD COLUMN/CREATE INDEX のみ） |
| Local E2E | 完了（自動テスト・Local D1 統合） |
| Production E2E | 完了（上記 Production 実績・operator 確認） |

以上により、本追加対応（複数商品 Checkout / 支払い試行ライフサイクル）は Production E2E まで完了した。これは WORK-011 Public Site の一部として実施したものであり、独立した WORK 番号は付与しない。WORK-011 全体の完了状態は `WORK-011_PUBLIC_SITE.md` を正本とする。
