# Admin API

Status: Approved

## 1. 方針

管理画面はPC専用。

通常運用を管理画面だけで完結させ、SQL直接更新を避ける。

管理者は初期1人。

複雑なロール・承認フローは作らない。

## 2. 管理者認証

共通`requireAdmin()`を使用する。

初期の管理者判定方式は実装時に以下のいずれかで固定する。

推奨:

```text
Cloudflare Secretまたは環境設定に管理者AUTH_USER_IDを登録
```

管理者メールアドレスだけを判定キーにしない。

## 3. GET /api/admin/dashboard

### Response項目

- 総ユーザー数
- 商品別有効ユーザー数
- 当日新規登録数
- 当日購入数
- note未移行件数
- 未対応Warning件数
- 直近Warning一覧

グラフは初期実装しない。

### 商品別有効ユーザー数の条件

WORK-005 の available 条件と一致させる。次をすべて満たすユーザーを商品ごとに数える。

- M_USER.STATUS = 1
- M_USER.DEL_FLG = 0
- M_PRODUCT.STATUS = 1
- M_PRODUCT.DEL_FLG = 0
- T_USER_PRODUCT.STATUS = 1
- T_USER_PRODUCT.DEL_FLG = 0
- START_DATE <= 現在日時 <= END_DATE

START_DATE / END_DATE と現在日時の比較は ISO 文字列の辞書順ではなく Date 変換で行う
（ミリ秒有無の混在に影響されないため。DATABASE.md 4.6）。
状態条件で候補を絞り込んだうえで期間を Date 判定し、商品ごとの N+1 にはしない。

### 「当日」集計（当日新規登録・当日購入）

JST を基準とし、範囲比較で数える。

- JST 当日 00:00:00 以上、翌日 00:00:00 未満
  （当日開始 <= 対象 < 翌日開始）
- Prepared Statement + bind による `>= ?` / `< ?` の範囲比較を使う
- prefix LIKE は使用しない

境界は 00:00:00（秒固定・ミリ秒なし）で生成するため、保存値のミリ秒有無に関わらず
日境界が正しく判定され、日跨ぎ・月跨ぎ・年跨ぎでも桁ズレしない。

## 4. GET /api/admin/users

### Query

```text
email
status
productCode
limit
offset
```

### 初期値

```text
limit = 100
offset = 0
```

### 一覧項目

- メールアドレス
- ユーザー状態
- 保有商品
- 登録日時
- 最終ログイン日時
- 最終アクセス日時

### Query parameter の検証方針（管理系 API 共通）

「未指定」と「指定されたが不正」を区別する。

- 未指定: 仕様上の default を使用する。
- 指定された値が不正: 400 VALIDATION_ERROR を返す。
- 不正値を黙って丸める・default へ置換することはしない。

各 parameter の検証:

- limit: 整数。1〜200。default 100。範囲外・非整数は 400。
- offset: 整数。0 以上。default 0。負数・非整数は 400。
- status: 対象テーブルの正本コード値のみ（users は M_USER.STATUS、warnings は T_WARNING.STATUS）。それ以外は 400。
- productCode: 形式検証（英数字・アンダースコア・ハイフン）。不正形式は 400。
- authUserId: UUID 形式。不正は 400。
- from / to: 有効な ISO 日時。不正は 400。from > to は 400。
- warningType: 合理的な最大長（TEXT 自由値）。超過は 400。
- email: 合理的な最大長（部分一致検索用。完全なメール形式である必要はない）。超過は 400。

SQL へ渡す値は常に Prepared Statement + bind とする。

## 5. GET /api/admin/users/{authUserId}

### 返却内容

- M_USER
- T_USER_PRODUCT
- T_PURCHASE
- T_NOTE_PURCHASE
- T_LOGIN_LOG
- T_ACCESS_LOG
- T_WARNING

ログは新しい順。

初期取得件数を制限し、追加読込を可能にする。

### 履歴取得の件数制限

次の履歴はすべて件数制限の対象とする。無制限取得は禁止。

- T_PURCHASE
- T_NOTE_PURCHASE
- T_LOGIN_LOG
- T_ACCESS_LOG
- T_WARNING

- 初期 50 件、最大 200 件
- 新しい順
- limit / offset でページング（SQL に必ず LIMIT を付与）

## 6. PUT /api/admin/users/{authUserId}/status

### Request

```json
{
  "status": 2
}
```

許可:

```text
1: 有効
2: 一時停止
9: 退会
```

退会は初期画面に出さず、必要時のみ使用してもよい。

### 操作

- 停止
- 再開

停止・再開は確認ダイアログ後、ボタン1回で完了する。

### 管理者自身のSTATUS変更制限

`env.ADMIN_AUTH_USER_ID` と対象 `authUserId` が同一の場合、管理不能を避けるため
STATUS の変更を次のとおり制限する。

- STATUS=2（停止）: 禁止
- STATUS=9（退会）: 禁止
- STATUS=1（有効化）: 許可

禁止時のレスポンス:

```text
HTTP 400
code = ADMIN_SELF_STATUS_CHANGE_NOT_ALLOWED
message = 管理者自身を停止または退会状態に変更できません。
```

`ADMIN_AUTH_USER_ID` の値そのものは API レスポンスに含めない。

## 7. PUT /api/admin/users/{authUserId}/products/{productCode}

### Request例: 付与・再開

```json
{
  "status": 1,
  "grantType": 3,
  "startAt": "2026-08-05T18:00:00+09:00",
  "endAt": "9999-12-31T23:59:59+09:00",
  "memo": "管理者手動付与"
}
```

### Request例: 停止

```json
{
  "status": 2
}
```

同一ユーザー・同一商品の既存レコードを更新する。

新規権限の場合のみINSERTする。

### 検証（新規付与）

新規レコードを作成する場合、次を必須とする。

- status: 必須
- grantType: 必須
- startAt: 必須（有効な日時）
- endAt: 必須（有効な日時）
- memo: 任意
- startAt <= endAt（Date 変換で比較）

手動付与（grantType=3）では PURCHASE_ID は NULL とし、T_PURCHASE を生成しない。

### 検証（既存権限更新）

既存レコードを更新する場合:

- status のみの更新（停止・再開）を許可する。
- startAt / endAt の片方だけを変更する場合も、変更後の実効値と既存値を
  組み合わせて startAt <= endAt を検証する（Date 変換）。
  - startAt のみ変更 → 新 startAt <= 既存 endAt
  - endAt のみ変更 → 既存 startAt <= 新 endAt
  - 両方変更 → 新 startAt <= 新 endAt
- 指定された日時は単独指定でも有効な日時であること。
- 停止時に既存の GRANT_TYPE / START_DATE / END_DATE / MEMO / PURCHASE_ID を破壊しない
  （status だけを更新し、未指定項目は現状維持）。

日時比較は ISO 文字列の辞書順ではなく Date 変換で行う（DATABASE.md 4.6）。
9999-12-31T23:59:59+09:00 を正常に扱う。

### 対象ユーザーの存在確認

対象 `authUserId` が M_USER に存在しない場合:

```text
HTTP 404
code = USER_NOT_FOUND
```

FK 制約任せにせず、更新前に存在を確認する。

### OPEN ISSUE: 停止・論理削除商品への付与可否（要決定）

現状の実装は、PRODUCT_CODE が存在すれば `M_PRODUCT.STATUS` / `DEL_FLG` に関係なく
権限操作が可能である。

停止商品（STATUS<>1）・論理削除商品（DEL_FLG=1）への**新規付与**を許可するかは
未確定である。確定仕様として書かず、要決定事項として残す。

## 8. GET /api/admin/warnings

### Query

```text
status
warningType
authUserId
from
to
limit
offset
```

### 一覧項目

- 対象メール
- 商品
- Warning種別
- スコア
- 検知日時
- 通知日時
- 対応状態

## 9. PUT /api/admin/warnings/{warningId}

### Request

```json
{
  "status": 1,
  "memo": "本人利用を確認"
}
```

ユーザー停止はこのAPIで自動実行しない。

必要なら別途ユーザー状態APIを実行する。

## 10. POST /api/admin/note/import

### 形式

```text
multipart/form-data
```

CSVファイルと取込モードを受け取る。

### 処理結果

```json
{
  "result": "OK",
  "data": {
    "read": 151,
    "imported": 141,
    "ignoredTips": 10,
    "duplicates": 0,
    "errors": []
  }
}
```

### 冪等性

NOTE_TRANSACTION_IDの一意制約により再取込を防ぐ。

同一行は重複として数え、エラー終了しない。

## 11. GET /api/admin/note/purchases

### Query

```text
noteId
transactionId
productCode
matchStatus
limit
offset
```

## 12. PUT /api/admin/note/purchases/{id}

用途:

- 手動紐付け
- 紐付け解除
- 要確認化
- 無効化

誤紐付け解除時は、T_PURCHASEとT_USER_PRODUCTへの影響を画面に表示し、確認後に実行する。

初期は複雑な自動巻戻しを作らず、対象データを明示した管理APIを用意する。
