# WORK-008 note Migration

Status: Approved

## 目的
既存HANABI購入者を共通アカウントへ移行する。

## 実装
- CSVアップロード
- 有料記事のみ取込
- チップ除外
- 重複取込防止（NOTE_TRANSACTION_IDを冪等キーとする）
- NOTE_TRANSACTION_ID照合
- T_PURCHASE/T_USER_PRODUCT/T_NOTE_PURCHASE一括更新
- 管理画面手動補正（link/unlink/flag/invalidate/reset）
- 管理note一覧のページング（limit/offset、ORDER BY PURCHASE_DATE DESC, NOTE_PURCHASE_ID DESC）

## Local E2E で確定した仕様

以下は NOTE_MIGRATION_API.md / ADMIN_API.md / operation/NOTE_MIGRATION.md に反映済み。

- 利用者applyの事前条件: 同一商品の T_USER_PRODUCT.DEL_FLG=0 行を保持している場合は移行拒否（STATUS・GRANT_TYPE 不問）。拒否時は副作用なし・共通エラー。
- T_PURCHASE復活ルール: 同一note取引の既存T_PURCHASEが DEL_FLG=1 なら物理DELETEせず既存PURCHASE_IDをUPDATEで復活。DEL_FLG=0 は拒否、不存在は新規INSERT。
- T_USER_PRODUCTの物理行判定: PK=(AUTH_USER_ID, PRODUCT_ID)、DEL_FLGはPK外。DEL_FLG=0有効行あり→上書きしない / DEL_FLG=0なし+物理行あり→UPDATE再有効化 / 物理行なし→INSERT。
- batch順序: `[insertPurchase, updateNote, upsertUserProduct]`（last_insert_rowid() 汚染対策）。applyNoteMigration / adminLinkNotePurchase 双方で統一。
- unlink→利用者reapply→旧PURCHASE_ID復活のフローを確認済み。
- 管理者手動linkは利用者制限の対象外（既存権限を上書きせず購入履歴のみ紐付け）。
- CSV再取込の冪等性（同一CSV再投入で取込0・重複スキップ計上）を実CSVで確認済み。
- 管理note画面/利用者移行画面のGoTrueClient多重生成を解消（単一Supabase client共有）。

## 完了条件
HANABI・Google Earth移行、同一商品重複移行の拒否、商品違い拒否、移行済み拒否、ゲスト購入、テスター例外、unlink→reapply復活、管理者link非上書き、CSV冪等性を確認する。

## 状態
Local E2E 合格。Production E2E は未実施（承認待ち）。
