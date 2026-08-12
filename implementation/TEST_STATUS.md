# Test Status（発売前 自動テスト現況）

Status: Recorded

実装正本 `shingo-camera-platform-current-20260813-v4`（HEAD: dbb777e）時点の `npm test` 現況と、既知の 12 fail の原因調査結論を記録する。品質管理上の記録であり、Production 実装の不具合台帳ではない。

## 現況

```text
215 tests
203 pass
12 fail
```

`npm test`（`node --test test/*.test.mjs`）実行時。typecheck / dry-run は成功。

## 12 fail の分類（調査済み）

- BLOCKER: 0
- WARNING: 0
- STALE TEST: 1
- TEST ENV / HARNESS ISSUE: 11

結論: **Production 実装の不具合 0 件・発売 BLOCKER 0 件。テスト更新のみ必要。**

## 原因

### TEST ENV / HARNESS ISSUE（11件）

対象テスト:

- `test/frontend_fixes.test.mjs`: site.js `cancelCheckout` / `startMultiCheckout` / `initStore`、active-checkout（`handleActiveCheckout`・initStore）系（6件）
- `test/sam_presale_fixes.test.mjs`: 固有建物UI（`openMobSubjectPopup` / `selectLandmarkBuilding` の Admin ガード）、`__smKmzDebug`、円錐台 `showTopW`、`subjectTotalElev`（targetOffset 廃止→elevOffset 一本化）系（5件）

原因: 実装ソースが CRLF 改行、テストの複数行正規表現が LF（`\n`）前提。Node の `readFileSync(..., "utf8")` は改行を変換しないため、`\n  }\n` 等の関数ブロック終端・複数行連結パターンが `\r\n` ファイルに一致せず、関数抽出が null になる／複数行マッチが外れる。単一行の `indexOf` / `assert.match` に依存する他 203 テストは通過している。

確認: `\r?\n` に対応させた正規表現ではいずれも一致し、抽出後の実装内容は現行仕様を満たす（active-checkout の completed→fulfill recovery・安全側終端、cancelCheckout の terminal 判定＋clearPending、initStore の未ログイン表示＋ログイン誘導、固有建物の非Admin遮断、`__smKmzDebug` 格納、`showTopW` frustum 判定、`subjectTotalElev = elev + elevOffset` 等）。

Production（Cloudflare Workers）は改行コードに依存しないため本番影響なし。

### STALE TEST（1件）

対象: `test/sam_presale_fixes.test.mjs` の「[項目5] スマホ地図上の太陽/月トグルが既存 setSunsetMode を再利用している」。

原因: スマホ太陽/月トグルは発売前実機修正で衛星ボタン直下の右端縦列へ移動済み（WORK-010 参照）。実装は項目5の実質要件（`setSunsetMode` 再利用・PC非表示／スマホのみ表示・状態一元・☀/☾ 判別）をすべて満たすが、テストが移動前の旧CSS字面（`#map-body-toggle{display:none;position...}` ＋ `@media(max-width:600px){#map-body-toggle{display:block;}}`）を期待しているため fail。

## 推奨対応（テスト側のみ・今回未実施）

Platform 実装は変更しない。次工程でテスト側を更新する。

- 11件: テスト正規表現の `\n` を `\r?\n` に統一、または正本の改行を LF 固定（`.gitattributes` に `* text=lf` 等）、または各テスト冒頭で `readFileSync(...).replace(/\r\n/g, "\n")` 正規化。
- 1件（項目5）: 旧CSS字面の期待を現行仕様（PC `display:none;`・スマホ `@media (max-width:600px)` で `display:flex`・衛星直下右端縦列・safe-area）へ更新。

## 結論

12 fail は【テスト更新のみ必要】。発売前に実装を修正すべき BLOCKER は無い。
