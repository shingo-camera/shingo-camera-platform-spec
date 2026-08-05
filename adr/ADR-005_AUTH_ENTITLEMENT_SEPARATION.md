# ADR-005 認証と商品権限を分離

Status: Approved

## 決定

Supabase Authは本人認証のみを担当する。

商品利用可否はD1のT_USER_PRODUCTで管理する。

## 許可する状態

```text
Supabase Auth: あり
M_USER: あり
T_USER_PRODUCT: 0件
```

商品未購入でもログイン可能とする。

## 理由

- 商品追加で認証基盤を変更しない
- note移行、Stripe購入、管理者付与を同じ権限テーブルへ集約できる
- 将来の無料ユーザー・新商品にも対応できる
