# ADR-008 複数テーブル更新はD1 batchを使用

Status: Approved

## 決定

以下の複数更新はD1のbatchで実行する。

- Stripe購入
- note移行
- 複数テーブルへ影響する管理者補正

## 理由

- ステートメントを順次実行できる
- 途中失敗時に一連の更新を成立させない
- ネットワーク往復を減らせる
- 購入履歴だけ登録され権限がない等の不整合を防ぐ

## 注意

実装時はPrepared Statementとbindを使用する。

## 複数商品Checkout（WORK-011）での原子的反映

複数商品Checkoutでは、1回の `db.batch()` に以下をすべて含めて実行する。

```text
T_ORDER INSERT               （注文ヘッダ 1行）
T_PURCHASE INSERT × 商品数     （購入明細）
T_USER_PRODUCT INSERT/UPDATE × 商品数（権限）
```

現行のCloudflare D1 Workers Bindingの `db.batch()` は、各statementを順次・非並行に実行し、途中のstatementが失敗した場合はシーケンス全体をロールバックする（Local D1エンジンで実測確認済み）。この原子性を用いて、注文・明細・権限を「全件成立、または全件ロールバック」で反映する。

一時的な部分成功を許容してWebhook再送で収束させる方式は採らない。成功時は全件成立、失敗時は全件ロールバックとする。

### last_insert_rowid() を複数明細で使わない

`last_insert_rowid()` は接続単位で「直近INSERT」を指すため、複数INSERTが並ぶbatchでは「どのT_PURCHASEに対応するPURCHASE_IDか」を安全に判定できない。したがって複数明細では使用しない。

代わりに、同一batch内で先行INSERT行を後続SQLから参照する（実測でbatch内の順次可視性を確認済み）。

- ORDER_IDは `(PURCHASE_SOURCE, EXTERNAL_ORDER_ID)` から参照する。
- PURCHASE_IDは `(ORDER_ID, PRODUCT_ID)` から参照する（`UX_T_PURCHASE_ORDER_PRODUCT` で1件に確定）。
- ORDER_IDをアプリ側で MAX+1 採番しない。

### 冪等性

注文単位の冪等キーは T_ORDER の `(PURCHASE_SOURCE, EXTERNAL_ORDER_ID)` 一意制約とする。batch成功時は全件成立するため、T_ORDER存在＝その注文の全DB反映済みと扱える。Webhook再送時は、既存T_ORDERを確認して処理済みなら何もせず200を返す。
