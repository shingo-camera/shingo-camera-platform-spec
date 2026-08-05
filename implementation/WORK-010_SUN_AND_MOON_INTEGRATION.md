# WORK-010 SUN AND MOON Integration

Status: Approved

## 目的
既存SUN AND MOONへ共通認証・商品権限確認を追加する。

## 前提
最新`portrait.html`を唯一の実装基準とし、Moonは統合済みとして扱う。

## 実装
- 未ログイン時ログインへ
- `SUN_AND_MOON`権限確認
- 権限なし時商品案内
- 計算APIでも権限確認
- アクセスログ

## 禁止
天体計算、APIレスポンス、既存UIの無関係な変更。

## 完了条件
未ログイン・未購入・停止を拒否し、購入者が既存機能を問題なく利用できる。
