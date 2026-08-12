# WORK-011 Public Site and Legal Navigation

Status: Done（Production 運用開始・operator 確認済み・2026-08-11）

## 目的
販売開始に必要な公開トップ、商品詳細、サポート・法務導線を実装する。

## 画面
`/` `/products` `/products/sun-and-moon` `/support` `/terms` `/privacy` `/commercial-transactions` `/contact`

## 方針
ミニマル、製品中心、受賞歴・プロフィール前面表示なし、SNSリンク、アプリ画面と実写結果を中心にする。

## 実装中に実施した追加対応

WORK-011 Public Site 実装の過程で、販売開始に必要な以下を追加対応として実施した。いずれも本 WORK-011 の一部であり、独立した WORK 番号は付与しない。

- ユーザー動線再編: HOME / STORE / MY PAGE（旧 `/home/` → `/mypage/`）、共通ヘッダー（未ログイン=HOME/STORE/SUPPORT/LOGIN、ログイン=+MY PAGE/LOGOUT、ADMIN のみ ADMIN 表示）、スマホ用ハンバーガーメニュー、LOGIN/SIGNUP の redirect 中継、商品カード UI の HOME 基準統一、商品説明文・note 導線の site-config 一元化。
- 複数商品 Checkout / 支払い試行ライフサイクル: `WORK-011_APPENDIX_MULTI_CHECKOUT.md`（追加対応資料）を参照。設計・実装・Production E2E 実績はそちらに集約する。
- Platform 共通 favicon（`shingo_camera LABO` 用・商品固有アイコン非流用）、HOME title を `shingo_camera LABO` へ変更。

## Production 実績（operator 確認・2026-08-11）

運用者（repo オーナー）が Production 環境で確認した完了状態として記録する（コード・自動テスト範囲外の実環境確認）。

- Cloudflare custom domain `shingo-camera.com` で Production 運用開始。`APP_BASE_URL = "https://shingo-camera.com"` へ正式切替。
- 公開サイト・LOGIN・STORE・MY PAGE・SUPPORT・法務導線の表示、未ログイン閲覧、購入時 LOGIN 誘導・復帰、MY PAGE 導線を実機確認。
- 複数商品 Checkout・購入済み表示・権限反映を含む購入系スモークに合格（詳細は追加対応資料の Production 実績）。
- HOME title / 共通 favicon の Production 反映を確認。
- 最終 Production スモーク合格。

以上により WORK-011 Public Site を Done（Production 運用中）とする。
