# MQTT Panel 整理: 処理機能の廃止、再接続の強化、Topic ブラウザ

日付: 2026-10-02

## 目的

MQTT Panel を「監視・表示・手動パブリッシュ」のパネルに絞り込み、実装を整理する。
そのうえで次の3点を追加する。

- 切断後の自動再接続を確実にする。
- Topic をブローカーから取得した一覧から選んで設定できるようにする。
- ドキュメントから絵文字をなくす。

## 対象外

- 単一ファイル構成（`index.html`）の変更。ビルド工程やテスト基盤は追加しない。
- 残す機能の挙動変更。List/Tile 表示、タイルクリックでの次値パブリッシュ、クイックパブリッシュ、
  インライン編集、JSON Path、ペイロード値の表示・色設定、ドラッグ並び替え、テーマ、
  クライアントステータス、リモート設定同期、エクスポート/インポートはそのまま残す。
- `work/` 配下のファイル（git 管理外）。

## 1. 処理機能の削除

転送・変換・循環・スケジュール・タイマーの5機能を完全に削除する。

### 削除するもの

- ハンドラ: `handleTransfer`、`handleConvert`、`handleCycle`、`handleTimer`、
  `playTimerSound`、`formatTimerDisplay`、`startScheduleChecker`、`checkSchedules`。
- `handleMessage` 内の機能分岐。受信時は値の抽出と表示更新だけを行う。
- Topic モーダルの Function 選択と機能別設定欄、`toggleFunctionSettings`、
  `selectAllDays` / `deselectAllDays`、タイマー色パレット（`showTimerColorPalette`）、
  サウンドテスト（`testTimerSound`）と、それらのイベントリスナ。
- ペイロード値設定テーブルの Convert Value 列。
- 一覧の Function 列と ON/OFF トグル（`toggleFunction`、`getFunctionDisplayName`）。
  一覧は Label / Display / Topic / Payload の4列になる。
- 表示側のタイマー分岐（HH:MM:SS 表示、カウントダウン色・超過色）。List と Tile の両方。
- グローバル変数 `timerIntervals`、localStorage の `schedule_*` キーの読み書き。
- `audio/` ディレクトリ。
- 上記に対応する CSS。

### 旧データの扱い

正規化関数 `normalizeTopic(topic)` を1つ用意し、次の旧フィールドを取り除く。

- トピック: `functionType`、`functionEnabled`、`transferTopic`、`convertTopic`、
  `convertDefault`、`cycleNextPayload`、`cyclePrevPayload`、`schedules`、
  `timerInterval`、`timerCountdownBg`、`timerCountdownFg`、`timerOvertimeBg`、
  `timerOvertimeFg`、`timerSound`
- ペイロード値: `convertValue`

適用する入口は3つ。

- localStorage からの読み込み（`loadTopics`）
- JSON インポート（`importData`）
- リモート設定の受信（`handleTopicsConfigMessage`）

旧フィールドを含む設定はエラーにしない。リモート設定の比較は正規化後の値で行うため、
旧フィールドだけが違う設定を受信した場合は「変更あり」として取り込み、
正規化済みの設定を保持する。

## 2. 自動再接続

mqtt.js 内蔵の再接続を使い、クライアントを1つだけ保持する。

### 現状の問題

- `error` / `close` のたびに `mqttClient = null` にしており、古いクライアントを `end()` せずに
  捨てている。mqtt.js 内蔵の再接続と30秒ポーリングの `connectMQTT()` が二重に動きうる。
- 再接続チェックは初回接続の成功後にしか始まらないため、起動時にブローカーが落ちていると
  再試行されない。

### 新しい挙動

- 接続オプション: `reconnectPeriod: 5000`、`connectTimeout: 10000`、`keepalive: 30`、
  `clean: true`。5秒間隔で無期限に再試行する。
- `error` / `close` / `offline` ではクライアントを破棄しない。状態表示だけを更新する。
- `connectMQTT()` は、既存クライアントがあれば `end(true)` で終了してから新しく作る。
  イベントハンドラは、自分が現在のクライアントである場合だけ処理する（古いクライアントの
  遅延イベントで状態が上書きされないようにする）。
- `connect` イベントのたびに、online ステータスの発行、登録トピックの購読、設定同期トピックの
  購読、設定の発行を行う。
- 手動 Disconnect は `end()` して再接続を止める。Connect を押すかページを再読み込みするまで
  再接続しない。
- ブラウザの `online` イベントと、タブが再表示されたとき（`visibilitychange`）に、
  未接続かつ手動切断でなければ `mqttClient.reconnect()` を呼んで待ち時間なしで再接続を試みる。
- 30秒ポーリング（`startReconnectCheck` / `stopReconnectCheck` / `reconnectInterval`）は削除する。
- 起動時の自動接続は従来どおり、ホスト設定があれば行う。

### 状態表示

接続インジケータとコントロールバーを3状態にする。

| 状態 | 条件 | 色 |
|---|---|---|
| connected | `connect` 受信後 | 緑 |
| reconnecting | クライアントがあり未接続（初回接続中を含む） | 黄 |
| disconnected | クライアントなし、または手動切断 | 赤 |

状態の更新は `setConnectionState(state)` の1か所にまとめる。

### ライブラリ

mqtt.js の CDN URL は現在バージョン未指定。`mqtt@5.16.0` に固定する。

## 3. Topic ブラウザ

MQTT にはトピック一覧を返す API がないため、ワイルドカードで一時的に購読し、
届いたトピックを一覧にする。

### 画面

- コントロールバーに「Browse Topics」ボタンを追加し、専用モーダルを開く。
- モーダルの構成:
  - フィルタ入力（既定 `#`）と適用ボタン
  - 検索入力（トピック名の部分一致で絞り込み）
  - 一覧: チェックボックス、トピック名、最新ペイロード（省略表示）、retain の有無
  - 件数表示と「Add selected」ボタン
- 一覧はトピック名の昇順。登録済みのトピックは選択不可で表示する。
- 未接続のときは一覧の代わりに、接続が必要である旨を表示する。

### 動作

- モーダルを開いたときにフィルタで購読し、閉じたときに購読解除する。
  フィルタを変更したときは、旧フィルタを解除して一覧を空にし、新フィルタで購読する。
- モーダルを開いている間に再接続が起きた場合は、フィルタを再購読する。
- 受信メッセージは `handleMessage` の先頭でブラウザ用の収集処理にも渡す。
  収集は `Map<topic, {payload, retain}>` に保持し、一覧の再描画は間引く（250ms）。
- 一覧は2000件で打ち切り、打ち切った旨を表示する。
- 設定同期用のトピック（`<Status Topic>/topics` とその配下）は一覧に出さない。
- `$SYS` 配下は `#` では届かない。見たい場合はフィルタに `$SYS/#` を指定する。
- ワイルドカード購読と登録トピックの購読が重なると、ブローカーによっては同じメッセージが
  2回届く。受信処理は表示更新だけなので、重複しても結果は変わらない。

### 追加モード

- 一括追加（コントロールバーから開いた場合）:
  チェックしたトピックをまとめて追加する。ラベルはトピック末尾のセグメント、
  `payloadValues` は空、`showInTileView` は true。追加後に保存、購読、設定発行、再描画を行う。
- 1件選択（Add/Edit Topic モーダルの Topic 入力欄横の「Browse」ボタンから開いた場合）:
  行をクリックするとトピック名を入力欄に反映してブラウザを閉じる。チェックボックスと
  「Add selected」は表示しない。

## 4. ドキュメント

- `README.md` と `README-ja.md`: 絵文字をすべて削除する。廃止機能の記述を削除し、
  自動再接続と Topic ブラウザの説明を追加する。
- `CLAUDE.md`: 廃止機能の記述を削除し、再接続、Topic ブラウザ、データ形式を現状に合わせる。
- `test-settings.json`: 旧フィールドを取り除く。

## 5. 検証

ローカルの mosquitto（WebSocket リスナを有効にした一時設定で起動）に対して、
ブラウザで実際に動かして確認する。

- 旧フィールドを含む設定を localStorage / インポート / リモート設定から読み込める。
- 受信メッセージで表示と色が更新される（List / Tile）。
- タイルクリック、クイックパブリッシュ、インライン編集でパブリッシュできる。
- ブローカー停止中に起動し、ブローカー起動後に自動で接続する。
- 接続中にブローカーを再起動すると自動で再接続し、トピックの更新が再開する。
- 手動 Disconnect 後は再接続しない。Connect で再接続する。
- インジケータが3状態を正しく表示する。
- Topic ブラウザ: 一覧表示、検索、フィルタ変更、一括追加、1件選択、登録済みの選択不可、
  閉じたときの購読解除。
- ブラウザのコンソールにエラーが出ない。
- README に絵文字が残っていない（grep で確認）。
