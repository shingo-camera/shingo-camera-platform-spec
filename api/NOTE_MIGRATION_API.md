# note Migration API

Status: Approved

## 1. 目的

noteでHANABIまたはGoogle Earth追加機能を購入済みの利用者が、新しい共通アカウントへ商品権限を移行できるようにする。

## 2. 前提

利用者は先に共通アカウントを作成し、ログインする。

note購入情報は認証ユーザー作成には使用しない。

note購入情報は商品権限付与にのみ使用する。

## 3. POST /api/migrations/note/apply

### 認証

必須。

### Request

```json
{
  "productCode": "HANABI",
  "transactionId": "note取引ID"
}
```

NOTE_IDの入力は初期必須としない。

### 処理

1. JWT検証
2. M_USER有効確認
3. PRODUCT_CODE確認
4. NOTE_TRANSACTION_ID検索（未移行かつ選択商品と一致するもの）
5. 同一note取引の既存T_PURCHASE確認（PURCHASE_SOURCE=1 + EXTERNAL_PURCHASE_ID、DEL_FLG込み）
   - 不存在 → 新規INSERT対象
   - DEL_FLG=0 → 既に有効な購入がある。共通エラーで拒否
   - DEL_FLG=1 → 物理DELETEせず、既存PURCHASE_IDをUPDATEで復活する対象
6. 利用者移行の事前条件（同一商品の権限保持チェック）
   - 同一AUTH_USER_ID + PRODUCT_IDについて、T_USER_PRODUCT.DEL_FLG=0の行が1件でも存在する場合は移行を拒否する。
   - STATUS（1/2/0/その他）・GRANT_TYPE（Stripe/note/テスター/管理者付与）を問わず、DEL_FLG=0なら拒否。
   - DEL_FLG=1のみ、または行なしの場合は拒否しない。
   - 拒否時はT_PURCHASE作成/復活なし・T_NOTE_PURCHASE変更なし・T_USER_PRODUCT変更なしで、共通エラーを返す。
7. T_USER_PRODUCT登録/復活/再有効化（下記「T_USER_PRODUCTの物理行判定」に従う）
8. T_NOTE_PURCHASEを移行済みに更新

T_PURCHASE登録/復活、T_USER_PRODUCT処理、T_NOTE_PURCHASE更新はD1 batchで一括実行する。

### T_PURCHASEの復活ルール

同一note取引（PURCHASE_SOURCE=1 + EXTERNAL_PURCHASE_ID）の既存T_PURCHASEをDEL_FLG込みで判定する。

- 不存在 → 新規INSERT
- DEL_FLG=0 → 共通エラーで拒否（重複）
- DEL_FLG=1 → 既存PURCHASE_IDをUPDATEで復活（新規INSERTしない）。復活時はnote購入データを正として AUTH_USER_ID / PRODUCT_ID / PURCHASE_DATE / AMOUNT / PAYMENT_STATUS=1 / DEL_FLG=0 / UPDATE_DATE を更新し、その既存PURCHASE_IDをT_NOTE_PURCHASEへ再紐付けする。

これにより、管理者unlink後（T_PURCHASE.DEL_FLG=1）に利用者が同じ取引IDを再applyすると、新しいPURCHASE_IDを作らず元のPURCHASE_IDが復活する。

### T_USER_PRODUCTの物理行判定

T_USER_PRODUCTのPKは (AUTH_USER_ID, PRODUCT_ID) であり、DEL_FLGはPKに含まれない。DEL_FLG=1の行が残る状態でINSERTするとPK衝突するため、次の2段階で判定する。

- DEL_FLG=0の有効行あり → 既存権限として扱い、一切上書きしない（購入履歴のみ登録/復活）。
  ※ 利用者applyでは事前条件（手順6）で拒否されるため通常この分岐には到達しない。管理者linkでは既存権限を上書きしない用途でこの分岐を使う。
- DEL_FLG=0行なし + 物理行あり（DEL_FLG=1の残骸） → UPDATEで再有効化（note由来として STATUS=1 / DEL_FLG=0 / GRANT_TYPE=1 / PURCHASE_ID設定）。
- 物理行なし → 新規INSERT。

### batch順序

T_USER_PRODUCTへのINSERTは通常テーブルのrowidを新規採番し last_insert_rowid() を書き換える。T_NOTE_PURCHASE更新が last_insert_rowid() でT_PURCHASE.IDを参照するため、batchは

```text
[insertPurchase, updateNote, upsertUserProduct]
```

の順とする（updateNoteはUPDATEでrowid不変のため、insertPurchase直後にT_PURCHASEのIDを正しく参照できる）。T_PURCHASE復活パス（DEL_FLG=1）では既存PURCHASE_IDを固定バインドするため last_insert_rowid() に依存しない。applyNoteMigration / adminLinkNotePurchase の双方で同じ順序・同じ判定を用いる。

### 登録値

T_PURCHASE:

```text
PURCHASE_SOURCE = 1
EXTERNAL_PURCHASE_ID = NOTE_TRANSACTION_ID
PURCHASE_DATE = note購入日時
AMOUNT = note購入金額
PAYMENT_STATUS = 1
```

T_USER_PRODUCT:

```text
START_DATE = note購入日時
END_DATE = 9999-12-31T23:59:59+09:00
GRANT_TYPE = 1
STATUS = 1
```

T_NOTE_PURCHASE:

```text
MATCH_STATUS = 1
MATCH_AUTH_USER_ID = ログイン中AUTH_USER_ID
MATCH_DATE = 現在日時
PURCHASE_ID = 作成した購入ID
```

## 4. 利用者向けエラー

以下を利用者向けに区別しない。

- 取引IDなし
- 商品違い
- 移行済み
- 無効
- 削除済み
- 同一note取引の有効なT_PURCHASEが既に存在（DEL_FLG=0）
- 同一商品のT_USER_PRODUCT.DEL_FLG=0行を既に保持（手順6の事前条件）

共通文言:

```text
購入情報を確認できませんでした。入力内容をご確認ください。
```

管理画面では実際の理由を確認可能とする。

## 5. GET /api/migrations/note/status

### 目的

ログイン中ユーザーのnote移行済み商品を返す。

### Response

```json
{
  "result": "OK",
  "data": {
    "migrations": [
      {
        "productCode": "HANABI",
        "matchedAt": "2026-08-05T18:00:00+09:00"
      }
    ]
  }
}
```

## 6. Earthだけ購入したテスター

Google Earth追加機能のみをそのまま移行可能とする。

HANABI本体権限は管理画面から手動付与する。

```text
GRANT_TYPE = 2
PURCHASE_ID = NULL
MEMO = Earth先行購入テスター
```

## 7. CSV取込

管理者APIでCSVを取り込む。

対象:

```text
決済種別 = 有料記事
```

除外:

```text
チップ
```

コンテンツ名からPRODUCT_IDへ変換する。

タイトル変更が少数で運営者が判別可能なため、初期はコードまたは取込設定で対応する。

商品名の重複保存は行わない。

### 冪等性

NOTE_TRANSACTION_IDを冪等キーとする。同一CSVの再投入・複数月の分割投入・順不同投入で二重登録は起きない。既に取り込み済みの取引はスキップして「重複」に計上し、取込件数は増えない。取込結果は 読込 / 取込 / 対象外（チップ等） / 重複スキップ / エラー を返す。

## 8. ゲスト購入

NOTE_IDがゲストの場合はNULLまたはCSV上の表示値を参考情報として保持する。

移行判定はNOTE_TRANSACTION_IDを中心に行う。
