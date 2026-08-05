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
4. NOTE_TRANSACTION_ID検索
5. PRODUCT_ID一致確認
6. MATCH_STATUSが未移行か確認
7. T_PURCHASE登録
8. T_USER_PRODUCT登録または復活
9. T_NOTE_PURCHASEを移行済みに更新

7〜9はD1 batchで一括実行する。

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

## 8. ゲスト購入

NOTE_IDがゲストの場合はNULLまたはCSV上の表示値を参考情報として保持する。

移行判定はNOTE_TRANSACTION_IDを中心に行う。
