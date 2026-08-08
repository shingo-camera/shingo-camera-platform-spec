# WORK-001 Platform Foundation

Status: Completed

## 目的
共通基盤コードを置く新規プロジェクトを作成し、Cloudflareへ非公開Productionとしてデプロイできる状態にする。

## 実装
- プロジェクト初期化
- TypeScript推奨
- Local起動
- Productionデプロイ
- `/api/health`追加
- 共通JSONレスポンスとエラーハンドラの骨格
- 環境変数例
- `.gitignore`
- README

## 完了条件
- Localで起動する
- Productionへデプロイできる
- `/api/health`が200を返す
- 秘密情報がGitへ含まれない
- 既存SUN AND MOONへ変更を加えていない
