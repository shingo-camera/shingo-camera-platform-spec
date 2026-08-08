# WORK-008 note Migration

Status: Completed

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

## Production E2E 実績

Production deploy 済み（Worker: shingo-camera-platform、Version ID: ca47b347-975b-4be8-9a4c-426f9e6780a8）。/api/health は result=OK / environment=production。

- CSV初期取込: 6月CSV（読込151・取込141・対象外10・重複0・エラー0）、7月CSV（読込32・取込32）を投入し、T_NOTE_PURCHASE 合計173件。8月分は月途中で日々増加するため初期確定取込には含めず、今後は最新CSVを追加投入し既存分は冪等性で重複スキップ、新規分のみ追加する運用とする。
- CSV冪等性: 6月・7月CSVの再投入で取込0・重複スキップ（141・32）を確認。NOTE_TRANSACTION_ID単位の冪等取込をProductionで確認済み。
- 管理note一覧ページング: 173件を1ページ目100件（NOTE_PURCHASE_ID 173降順）＋2ページ目73件で全件到達。ORDER BY PURCHASE_DATE DESC, NOTE_PURCHASE_ID DESC の決定的順序を実画面確認済み。
- 正常apply: HANABI（PRODUCT_ID=2）を移行し、T_PURCHASE / T_NOTE_PURCHASE / T_USER_PRODUCT の3テーブルでPURCHASE_IDの整合を確認済み。
- 同一商品保有時の別note拒否: 正常apply後、同じユーザーが別HANABI取引をapplyすると共通エラーで拒否。T_PURCHASE 0件・対象note未移行のまま・既存T_USER_PRODUCT不変（副作用なし）を確認済み。
- 移行済みnoteの別ユーザー拒否: 移行済み取引を別ユーザーがapplyすると共通エラーで拒否されることを確認済み。
- 商品不一致拒否: HANABI_GOOGLE_EARTH取引（PRODUCT_ID=3）をHANABIとしてapplyすると共通エラーで拒否。T_PURCHASE 0件・対象note未移行のままを確認済み。
- unlink: 正常移行を管理画面から解除し、T_PURCHASE.DEL_FLG=1・note由来T_USER_PRODUCT無効化（STATUS=2・DEL_FLG=1・PURCHASE_ID=NULL）・T_NOTE_PURCHASE未移行化を確認済み。
- E2E後クリーンアップ: 販売前・公開前のため、E2Eで作成したT_PURCHASE/T_USER_PRODUCTのHANABI行を物理削除し、CSV初期取込直後（T_NOTE_PURCHASEの実購入履歴は残し、対象note取引は未移行）の状態へ復元済み。

## 状態
Local E2E 合格。Production E2E 合格。WORK-008 完了。
