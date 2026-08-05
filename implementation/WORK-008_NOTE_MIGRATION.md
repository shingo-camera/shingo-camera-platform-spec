# WORK-008 note Migration

Status: Approved

## 目的
既存HANABI購入者を共通アカウントへ移行する。

## 実装
- CSVアップロード
- 有料記事のみ取込
- チップ除外
- 重複取込防止
- NOTE_TRANSACTION_ID照合
- T_PURCHASE/T_USER_PRODUCT/T_NOTE_PURCHASE一括更新
- 管理画面手動補正

## 完了条件
HANABI・Google Earth移行、再入力拒否、商品違い拒否、ゲスト購入、テスター例外を確認する。
