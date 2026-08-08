# note Purchase Migration Operation

Status: Approved

## 1. 目的

既存HANABI購入者を、新しい共通認証・商品権限方式へ移行する。

## 2. CSVの確認結果

取得可能な主な情報:

- 決済/返金日時
- 購入者名
- 決済種別
- 決済方法
- コンテンツ名
- 販売額
- 取引ID

取込対象は有料記事。

チップは無視する。

## 3. 初期取込手順

1. noteから販売履歴CSVを取得
2. 管理画面からCSVアップロード
3. 取込結果確認
4. HANABIとGoogle Earthの件数確認
5. 重複・エラー確認
6. 未移行状態で登録

### CSV再取込の冪等性

取込は NOTE_TRANSACTION_ID を冪等キーとする。同一CSVの再投入・複数月の分割投入・追加分の投入で二重登録は起きない。既に取り込んだ取引は「重複スキップ」に計上され、取込件数は0となる（読込件数・対象外件数は変わらない）。月次で追加CSVを重ねて投入する運用を前提とする。

## 4. 利用者移行手順

1. 共通アカウント作成
2. メール認証
3. ログイン
4. note移行画面を開く
5. 商品を選択
6. 取引ID入力
7. 照合成功
8. 商品権限付与
9. 商品利用開始

### 利用者移行の制約

- 同一商品を既に保有している場合の追加移行は不可。対象商品について T_USER_PRODUCT.DEL_FLG=0 の行を1件でも保持しているユーザーは、その商品の別note取引IDを移行できない（共通エラー）。STATUS・権限由来（Stripe/note/テスター/管理者付与）を問わない。これは、権限が増えないのに別取引IDを無駄に消費するのを防ぐため。
- 別商品の移行は可能。例えばHANABIを保有していても、HANABI_GOOGLE_EARTHは移行できる。
- 拒否時は購入履歴の作成・復活・移行済み化を一切行わず、既存権限も変更しない。

## 5. 問い合わせ対応

確認可能な情報:

- 共通アカウントのメールアドレス
- NOTE_ID
- NOTE_TRANSACTION_ID
- 商品
- note購入日時
- 金額
- 移行日時
- 移行先AUTH_USER_ID

NOTE_IDはSNSアカウント名と一致する利用者もいるため、本人確認の補助として利用できる。

NOTE_IDだけで権限付与しない。

## 6. 誤移行

管理画面から対象を確認し、必要に応じて以下を修正する。

- T_NOTE_PURCHASE
- T_PURCHASE
- T_USER_PRODUCT

人数が少ないため、初期は手動対応を許容する。

### unlink / reapply / relink 運用

- unlink（紐付け解除）: 影響を画面で確認のうえ実行する。T_PURCHASEは論理削除（DEL_FLG=1、PURCHASE_IDは保持）、note由来T_USER_PRODUCTのみ無効化（DEL_FLG=1・STATUS=2・PURCHASE_ID=NULL）、T_NOTE_PURCHASEは未移行へ戻す（MATCH_STATUS=0・MATCH_AUTH_USER_ID=NULL・MATCH_DATE=NULL・PURCHASE_ID=NULL）。Stripe等の別経路権限（DEL_FLG=0）は不変。
- 利用者による再apply: unlink後は対象商品のnote由来T_USER_PRODUCTがDEL_FLG=1になるため、同じユーザーが同じ取引IDを再applyできる。このとき新しいT_PURCHASEは作らず、DEL_FLG=1の既存T_PURCHASEを元のPURCHASE_IDのまま復活し、note由来T_USER_PRODUCTをUPDATEで再有効化する。
- 管理者による再link: unlink後の取引を管理者が別ユーザーへ手動linkできる。元のPURCHASE_IDを復活させ、新ユーザーへ付与する。link先の既存権限は上書きしない。

### 管理者補正（手動link）

管理者手動linkは利用者移行の制約（同一商品の権限保持時は移行不可）の対象外。既存有効権限がある場合でも、その権限を上書きせず購入履歴のみを紐付ける。特殊事情の補正用途で用いる。

## 7. Earth先行購入テスター

Google Earthだけ購入済みの場合、その購入権限を移行する。

HANABI本体は管理者から手動付与する。
