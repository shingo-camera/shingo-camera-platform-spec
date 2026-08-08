# Product and Entitlement API

Status: Approved

## 1. GET /api/products

### 目的

公開商品一覧またはログイン後の商品一覧を返す。

### 認証

任意。

### 対象

```text
M_PRODUCT.STATUS = 1
M_PRODUCT.DEL_FLG = 0
```

### Response

```json
{
  "result": "OK",
  "data": {
    "products": [
      {
        "code": "SUN_AND_MOON",
        "name": "SUN AND MOON PLANNER",
        "sortNo": 10
      }
    ]
  }
}
```

販売価格やStripe Price IDは、決済設計の実装時にCloudflare Secretsまたは商品設定として管理する。

秘密キーをM_PRODUCTへ保存しない。

## 2. GET /api/products/{code}

### 目的

商品詳細画面用。

### Path

```text
code = PRODUCT_CODE
```

### Error

存在しない、停止、論理削除の場合:

```text
PRODUCT_NOT_FOUND
```

## 3. GET /api/entitlements/{code}

### 目的

アプリ起動直前に商品利用可否を判定する。

### 認証

必須。

### 処理

1. JWT検証
2. M_USER状態確認
3. M_PRODUCT確認
4. T_USER_PRODUCT確認
5. 日付範囲確認
6. T_ACCESS_LOGへ権限確認を記録
7. 結果返却

### Response: 利用可

```json
{
  "result": "OK",
  "data": {
    "productCode": "SUN_AND_MOON",
    "available": true,
    "startAt": "2026-08-05T12:00:00+09:00",
    "endAt": "9999-12-31T23:59:59+09:00"
  }
}
```

### Response: 権限なし

HTTP 403

```json
{
  "result": "NG",
  "error": {
    "code": "PRODUCT_NOT_GRANTED",
    "message": "この商品は利用できません。"
  }
}
```

## 4. requireProduct

各アプリ固有APIから共通関数として呼び出す。

例:

```text
requireProduct(request, env, "SUN_AND_MOON")
```

アプリ画面で判定済みでも、計算API側でも必ず権限を確認する。

画面の表示制御だけをセキュリティ境界にしない。

## 5. アクセスログ抑制

権限確認（`GET /api/entitlements/{code}`）は `ACCESS_TYPE = 1` で T_ACCESS_LOG へ記録する。
同一条件の短時間連続アクセスは毎回保存しない。

### 同一条件

次の4項目が一致するものを同一条件とみなす。

```text
AUTH_USER_ID
PRODUCT_ID
ACCESS_TYPE   （= 1）
DEVICE_ID
```

同一条件の最新 T_ACCESS_LOG の記録時刻から `ACCESS_LOG_INTERVAL_MIN` 分以内であれば、
新しいログを記録しない。間隔を超えていれば記録する。

### DEVICE_ID の扱い

- DEVICE_ID あり: 同一 UUID 単位で抑制する。
- DEVICE_ID なし: `DEVICE_ID IS NULL` 同士を同一グループとして扱い抑制する。
- DEVICE_ID が無い場合でも、抑制のための識別子を新規生成しない。

判定 SQL は Prepared Statement を用い、NULL 同士の一致を
`DEVICE_ID = ? OR (DEVICE_ID IS NULL AND ? IS NULL)` で表現する。

### 抑制間隔の設定

抑制間隔は M_SYSTEM_SETTING の `ACCESS_LOG_INTERVAL_MIN`（分）から取得する。

- 初期値: 60
- 0: 抑制なし（毎回記録する）
- 設定値が「不存在」「非整数」「負数」の場合は内部設定エラーとして扱う。
  利用者へは設定キーや内部値の詳細を返さない。

### 日時比較

START_DATE / END_DATE を用いた available 判定は、ISO 8601 文字列の辞書順比較ではなく
日時（Date）へ変換して比較する。

理由: 保存日時はミリ秒付き（例 `...T06:30:00.000+09:00`）とミリ秒なし
（例 `9999-12-31T23:59:59+09:00`）が混在し得るため、辞書順比較では秒直後の
`.` と `+` の並びにより同一時刻付近で誤判定する可能性がある。オフセットは全て
`+09:00` に統一（ADR-012）されており、Date 変換に曖昧さはない。
