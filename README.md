# shingo-camera Platform Specification

Status: Active

このリポジトリは、shingo-camera Platformの設計書・実装指示書を管理する正本です。

## 正本の場所

実装WORKはすべて `implementation/` 配下を正本とします。

```text
implementation/
├─ IMPLEMENTATION_PLAN.md
├─ WORK-001_PLATFORM_FOUNDATION.md
├─ WORK-002_D1_SETUP.md
├─ WORK-003_SUPABASE_AUTH.md
├─ WORK-004_COMMON_API.md
├─ WORK-005_ACCOUNT_ENTITLEMENT.md
├─ WORK-006_ADMIN.md
├─ WORK-007_STRIPE.md
├─ WORK-008_NOTE_MIGRATION.md
├─ WORK-009_WARNING.md
├─ WORK-010_SUN_AND_MOON_INTEGRATION.md
├─ WORK-011_PUBLIC_SITE.md
├─ WORK-012_PUBLICATION.md
└─ CHECKLIST.md
```

ルート直下にはWORKファイルを置きません。重複管理を避けるため、Claudeへの指示でも必ず `implementation/...` のパスを指定してください。

## フォルダ構成

```text
architecture/   基盤・認証・セキュリティ設計
api/            API仕様
adr/            Architecture Decision Record
database/       DB設計・DDL・ERD・テーブル定義
implementation/ 実装順序・WORK指示・チェックリスト
operation/      運用・移行・決済・Warning・バックアップ
prompt/         Claude実装ルール・レビュー基準
screen/         公開画面・管理画面設計
```

## 実装時の読み方

各WORK開始時は、最低限以下を確認します。

1. `prompt/CLAUDE.md`
2. `prompt/REVIEW_RULE.md`
3. `implementation/IMPLEMENTATION_PLAN.md`
4. 対象の `implementation/WORK-xxx_*.md`
5. WORK内で指定された関連設計書

過去チャットとGit上の設計書に差異がある場合は、Gitの最新版を優先します。

## 現在の進捗

- WORK-001 Platform Foundation: 完了
- WORK-002 D1 Initial Setup: 完了
- WORK-003 Supabase Authentication: 完了
- WORK-004 Common API Foundation: 次工程

## 環境

```text
Local
Production（公開前は未告知で運用）
```

専用Stagingは初期構築では作成しません。
