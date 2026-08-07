# ADR-012 JST日時保存とMigration不変方針

Status: Approved

## 決定

D1へ保存する業務日時は、日本時間（JST / UTC+09:00）のISO 8601 TEXTへ統一する。

例:

```text
2026-08-07T16:14:10.956+09:00
```

Workers側の現在日時生成は共通関数 `nowIso()` を使用する。

また、一度LocalまたはProductionへ適用したmigrationファイルは後から変更しない。

日時仕様の修正は `0001_initial_schema.sql` を書き換えず、`0002_fix_jst_datetime.sql` を追加して対応する。

## 理由

- 管理画面・問い合わせ時に日本時間で直接確認できる
- UTC、JST、タイムゾーンなし日時の混在を防げる
- 同一migration名で環境ごとにSQL内容が異なる事故を防げる
- 適用済みmigrationを履歴として信頼できる
- 新規環境でもmigrationを順番に適用すれば既存環境と同じ最終状態になる

## DDL.sqlとの関係

`database/DDL.sql` は現在の正しい最終仕様を示す。

そのため、最新の `DDL.sql` では初期データ日時も最初からJST形式で生成する。

一方、適用履歴である `migrations/0001_initial_schema.sql` は変更しない。

## 実装ルール

業務日時保存では、以下を直接使用しない。

```text
new Date().toISOString()
datetime('now')
CURRENT_TIMESTAMP
```

日時生成が必要な場合は、Workers側では `nowIso()`、DDL側ではJST `+09:00` 付きISO 8601を生成するSQLを使用する。
