# WORK-013 HANABI Integration

Status: Implemented / DEV verified / Production not published

## 位置づけ

本WORKは「これからHANABIを統合する計画」ではない。実装正本 `shingo-camera/shingo-camera-platform` の `develop`（本Spec更新時点のHEAD `227858d`）で既に実施済みのHANABIアプリ統合・公開前調整を、実装事実とDEV実機確認結果に基づいて正本化するWORKである。WORK-010（SUN AND MOON Integration）と並列の、アプリ単位の統合WORKとして管理する。

Production公開は未了。本WORKの「Implemented / DEV verified」と「Production公開完了」を混同しない。

## 目的
既存HANABI PLANNERをPlatformへアプリ統合し、共通認証・商品権限・公開URLへ集約したうえで、Production公開可能な状態へ仕上げる。今回は大きな計算ロジックの作り直しではなく、UI・PC/スマホ状態同期・マップ操作・表示refresh・保存データ復元・stale state除去・レスポンス改善・回帰テストを中心とした公開前調整である。

## 前提
HANABI本体の唯一の実装正本は `public/apps/hanabi/index.html` とする。scene計算・terrain・風補正はserver側（`/api/apps/hanabi/*` 計算API群）をauthoritativeとし、clientは結果をconsumeする。HANABI固有の秘密計算式・内部係数をclientへ戻さない。

HANABIの計算結果（風以外）はAPI化前HANABIをGolden Masterとする。風はworld-fixed実装をGolden Masterとする。

## 実装
- アプリ画面: `/apps/hanabi/`（静的配信）
- HANABI固有API: `/api/apps/hanabi/*`（Worker、`run_worker_first=["/api/*"]`）
- 各計算APIは商品権限確認（`HANABI`）を通す。新しい認証方式は追加しない。
- scene計算はshort-term LRUを持つrequest managerを経由してserver結果を再利用する（`scene-request-manager.js`）。framing系操作（上下左右・焦点距離・拡大縮小・縦横・sensor・compMode）はserver再計算不要としてcache reuseする。

## 禁止
HANABI計算core（scene / hanabi_calc / terrain / terrain_tiles / png / validate、compute / earth / app_start）の変更、scene計算結果・terrain計算結果・world-fixed挙動・authoritative payloadの意味/単位/丸め/精度の変更、server-authoritative境界の変更、秘密計算ロジックのclient移動。SUN AND MOON本体・`src/shared`・migrations・wrangler設定の変更。

## 発売前 実機修正（PC / スマホ）

DEV（https://shingo-camera.com/dev/）でPC・スマホ実機確認を実施し、発見した主要問題を修正済み。ユーザーによるPC最終確認まで進み、今回確認された主要不具合は修正済み。以下を現行仕様として確定する。

### 1. 撮影地点情報表示のrefresh（state lifecycle）

撮影地点の条件が変更された場合、地図上の撮影地点情報表示（撮影地点マーカーのラベル）も現在のPlanner Stateへ追従して更新される。表示は現在のstateから遅れて古い値（「取得中…」等）に留まらない。

対象とする変更操作と確定挙動:

- 筒場移動モードで撮影地点を移動して更新し、移動モードを解除した場合、撮影地点情報表示は現在の撮影地点へ追従して確定表示へ戻る。
- 三脚高さを変更した場合も、撮影地点情報表示は現在のstateへ追従して更新される。
- 標高オフセットを変更した場合も、同様に更新される。

これはHANABIの計算そのものの問題ではなく、撮影地点情報表示のrefresh / state lifecycleの問題として整理する。プレビュー（撮影結果）は従来どおり現在のstateへ更新される。撮影地点の座標・標高取得・scene / terrainの計算内容は変更しない。

実装コミット: `dabb13f Fix HANABI viewpoint refresh and load failure state`。

### 2. 保存データ呼び出し失敗時の未選択化

保存済みの撮影計画等を呼び出す際、保存時に参照していた実体が現在存在しない場合は、以前の現在選択値を維持しない。UIと内部stateの双方を未選択へ収束させる。

- 筒場を呼び出せない → 未選択。現在選択中の筒場を保持しない。
- 対象を呼び出せない → 未選択。現在選択中の対象を保持しない。

「呼び出しに失敗したから現在の選択を残す」ではない。保存データが指す実体を解決できなければ、表示名だけでなく内部の選択状態もクリアする。複数対象のstale状態も残さない。正常に呼び出せた場合の既存動作は変更しない。

実装コミット: `dabb13f Fix HANABI viewpoint refresh and load failure state`。

### 3. 対象未選択時のstale target除去

見た目上「対象なし」であるにもかかわらず、地図に以前の対象を示す青丸が残る／Google Map（対象）が押せる／Google Mapを開くと以前の対象地点が表示される／対象編集を押すと対象なしなのに以前の対象データが呼び出される、という状態が存在した。

原因: PC側の対象selectを未選択にしても、スマホ側対象selectに古い対象IDが残る場合があり、`getEffectiveTargetId()` が `pcVal || mobVal` で古い `mobVal` を拾っていた。また複数対象のmain IDについても、削除済み等で実体が存在しないIDをeffective targetとして返し得た。

確定仕様:

- 対象未選択時には、過去の対象stateをeffective targetとして一切使用しない。
- `onTargetChange()` でPC/スマホの対象selectを、未選択を含め必ず同値へ同期する。
- PC/スマホ双方の対象編集ボタンを、effective targetが存在しない場合はdisabledにする。
- `getEffectiveTargetId()` は、複数対象のmain IDについて `db.targets` に実体が存在する場合のみ返す。存在しなければ空文字を返して未選択扱いとする。
- その結果、対象青丸・Google Map（対象）・対象編集・対象依存処理は、すべて現在のeffective targetと一致する。

計算結果は不変。scene payloadは全対象を渡し `targetId` はeffective ID（未選択はnull）であり、実体のないID（削除済み）も未選択も、serverでは「対象なし」として同一sceneになる。正常な単一/複数対象選択は従来どおり。

実装コミット: `258a1ec Fix HANABI stale target state`。

### 4. Google Map（対象）の確定挙動

対象が現在有効に選択されている場合のみ、Google Map（対象）はその対象地点を利用できる。対象未選択の場合は、過去の対象座標を使わない・stale targetを使わない・対象編集を使えない・地図上に対象青丸を残さない。見た目のselect表示だけでなく、effective targetを正本として関連UIを一致させる。

## PC / スマホ

PC・スマホの状態同期を現行仕様とする。対象select・撮影地点情報表示・対象編集ボタン状態はPC/スマホで整合させ、片側の操作結果が他方に古い状態として残らない。上記1〜4の修正はPC/スマホ双方に適用する。

## レスポンス改善（発売前品質要件）

一度取得したserver authoritativeなscene結果を、authoritative inputが完全一致する場合に限り再利用する（short-term LRU）。framing系操作はserver再計算せずclient側で反映する。同一入力への重複request抑止・stale response破棄・地図クリック時のUI先行反映（標高非依存の水平表示を標高取得前に反映）・撮影地点確定の世代ガードを行う。これらはrequest制御・cache・rendering・並列化の範囲であり、計算結果は変更しない。

## 回帰テスト（HANABI）

HANABI統合／公開前調整に対する回帰テストを実装リポジトリ `develop` へ反映済み。反映コミット: `227858d Add HANABI integration regression tests`（`test/` 配下のみ、計14ファイル）。

反映ファイル:

```text
test/_fixtures/scene_golden_fixture.mjs
test/frontend_fixes.test.mjs
test/hanabi_ar_sensor.test.mjs
test/hanabi_gmap_ui.test.mjs
test/hanabi_mapclick_perf.test.mjs
test/hanabi_mobile_ui_fixes.test.mjs
test/hanabi_move_step.test.mjs
test/hanabi_move_viewpoint.test.mjs
test/hanabi_plan_load.test.mjs
test/hanabi_scene_golden.test.mjs
test/hanabi_scene_key_perf.test.mjs
test/hanabi_scene_manager.test.mjs
test/hanabi_target_clear.test.mjs
test/hanabi_view_refresh_loadfail.test.mjs
```

主な回帰観点: 対象未選択への収束、PC/スマホ対象select同期、対象編集ボタン状態、対象青丸の消去、Google Map（対象）のstale参照防止、正常な単一/複数対象選択の回帰、撮影地点情報refresh、三脚高さ変更時の表示更新、標高オフセット変更時の表示更新、保存データロード失敗時の未選択化、撮影計画ロード、移動モード、scene manager、scene Golden Master、scene key/performance、地図クリックperformance、スマホUI、AR sensor。

テスト結果の正本は `implementation/TEST_STATUS.md`。

## DEV反映状況

HANABI実装はDEVへ反映済み。直近の実装DEV deployでは `/apps/hanabi/index.html` を更新した。その後 `227858d` で追加したのは `test/` 配下のみのため、そのcommit後にDEV再deployはしていない（テストファイルはpublic実行資産ではなく再deploy不要）。現在DEV上のHANABI実装は `258a1ec` までの実装内容と一致する。

## 完了条件

Platformへのアプリ統合・DEV反映・PC/スマホ実機確認・今回発見した主要不具合修正・HANABI専用回帰テストのGit反映が完了していること。本WORKの完了はProduction公開完了を意味しない。Production公開はWORK-012で扱う。

## 現在ステータス

- 実装済み（`develop` HEAD `227858d`）
- DEV反映済み（実装は `258a1ec` 相当、`227858d` は `test/` のみ）
- PC/スマホ実機確認済み（DEV）
- 今回発見した主要不具合修正済み（`dabb13f` / `258a1ec`）
- 回帰テストGit反映済み（`227858d`）
- Production未公開

## 残作業

1. 本Spec正本反映
2. 実装正本とSpec正本の最終整合
3. Production deploy
4. 公開後スモーク確認
