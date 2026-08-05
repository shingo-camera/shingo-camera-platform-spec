# ADR-003 M_SYSTEM_SETTINGを採用

Status: Approved

## 決定

Warning閾値やログ保持日数など、運用中に調整する値をM_SYSTEM_SETTINGで管理する。

## 理由

- 閾値変更のためにデプロイする必要がない
- 運営しながら値を調整できる
- 追加費用がない
- コード内への数値散在を防げる

## 制約

認証秘密情報、APIキー、パスワードは保存しない。

秘密情報はCloudflare Secrets等で管理する。
