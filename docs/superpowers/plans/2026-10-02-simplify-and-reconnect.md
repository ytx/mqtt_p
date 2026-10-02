# MQTT Panel 整理（機能廃止・再接続・Topic ブラウザ） Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 5つの処理機能を削除して MQTT Panel を監視・表示・手動パブリッシュに絞り、自動再接続を確実にし、ブローカーから Topic を選んで追加できるようにし、ドキュメントから絵文字をなくす。

**Architecture:** 単一ファイル `index.html` のまま、不要なコードを削除して表示ロジックを1つの関数にまとめる。MQTT 接続はクライアントを1つだけ保持して mqtt.js 内蔵の再接続に任せ、状態表示は `setConnectionState` に集約する。Topic ブラウザはワイルドカードの一時購読で届いたトピックを Map に集めてモーダルに表示する。

**Tech Stack:** HTML / CSS / JavaScript（ビルドなし）、Bootstrap 5.3.0、Bootstrap Icons 1.10.0、MQTT.js 5.16.0（CDN）。検証は mosquitto と Chrome。

**Spec:** `docs/superpowers/specs/2026-10-02-simplify-and-reconnect-design.md`

## Global Constraints

- すべての機能は `index.html` 1ファイルに収める。ビルド工程やテスト基盤は追加しない。
- UI の文言は英語。
- 残す機能の挙動は変えない: List/Tile 表示、タイルクリックでの次値パブリッシュ、クイックパブリッシュ、インライン編集、JSON Path、ペイロード値の表示・色設定、ドラッグ並び替え、テーマ、クライアントステータス、リモート設定同期、エクスポート/インポート。
- `work/` 配下は触らない。
- mqtt.js は `https://unpkg.com/mqtt@5.16.0/dist/mqtt.min.js` に固定する。
- 接続オプションは `reconnectPeriod: 5000`、`connectTimeout: 10000`、`keepalive: 30`、`clean: true`、`reconnectOnConnackError: true`、`resubscribe: false`。
- Topic ブラウザの一覧は2000件で打ち切る。再描画の間引きは250ms。
- 新しく書く DOM 生成コードでは、ブローカー由来の文字列（トピック名、ペイロード）を `textContent` で入れる。`innerHTML` に入れない。
- コミットメッセージは英語で、末尾に `Co-Authored-By: Claude Fable 5.1 <noreply@anthropic.com>` を付ける。
- 行番号はコミット `b7d25d4` 時点の `index.html` のもの。編集で行がずれるので、識別子で検索して位置を確かめる。

## Review Focus

spec が暗に求めているが、そのままではどの手順でも確かめられない条件。各行のテストは担当タスクに入れてある。

1. ブローカーが接続を拒否し続ける場合（認証エラー）。黄色のまま再試行を続け、ブローカーが受け入れるようになったら接続する。→ Task 3 Step 9
2. 保存済みトピックが壊れている場合（JSON として不正、配列でない、要素が null）。起動が止まらず、空の一覧で立ち上がる。→ Task 2 Step 7
3. トピック名やペイロードに HTML が含まれる場合。Topic ブラウザで文字列としてそのまま表示され、要素として解釈されない。→ Task 4 Step 12
4. Topic ブラウザのフィルタが登録済みトピック名と同じ場合。ブラウザを閉じても、そのトピックの表示更新が止まらない。→ Task 4 Step 13
5. ブローカーに2000件を超えるトピックがある場合。2000件で打ち切り、その旨を表示し、操作できる状態を保つ。→ Task 4 Step 14

## File Structure

| ファイル | 変更 | 役割 |
|---|---|---|
| `index.html` | 修正 | アプリ本体。Task 1〜4 で変更する |
| `audio/sound2-1.wav` | 削除 | タイマー用の音声。Task 1 |
| `test-settings.json` | 修正 | サンプル設定。旧フィールドを除く。Task 2 |
| `README.md`、`README-ja.md` | 修正 | 絵文字の削除、廃止機能の記述削除、新機能の説明。Task 5 |
| `CLAUDE.md` | 修正 | プロジェクト説明を現状に合わせる。Task 5 |

## Verification Environment

全タスク共通の検証環境。作業ディレクトリは任意の一時ディレクトリ（以下 `$V`）。

```bash
export V="$(mktemp -d)"
cat > "$V/mosquitto.conf" <<'EOF'
listener 1883 127.0.0.1
listener 8083 127.0.0.1
protocol websockets
allow_anonymous true
EOF
cat > "$V/mosquitto-deny.conf" <<'EOF'
listener 1883 127.0.0.1
listener 8083 127.0.0.1
protocol websockets
allow_anonymous false
EOF
```

- ブローカー起動: `mosquitto -c "$V/mosquitto.conf" -v`（バックグラウンドで実行。停止は `pkill -f "$V/mosquitto"`）
- ページ配信: リポジトリのルートで `python3 -m http.server 8000 --bind 127.0.0.1`
- ページ: `http://127.0.0.1:8000/index.html`
- 構文チェック:

```bash
awk '/<script>$/{f=1;next}/<\/script>/{f=0}f' index.html > "$V/panel.js" && node --check "$V/panel.js"
```

期待: 何も出力されず、終了コード 0。

ブラウザのコンソールで実行する共通スニペット。

**Seed（旧形式の設定を入れて再読み込み）:**

```js
localStorage.clear();
localStorage.setItem('mqttPanelSettings', JSON.stringify({
  mqttHost: '127.0.0.1', mqttPort: '8083', mqttUsername: '', mqttPassword: '',
  statusTopic: 'clients/verify', onlinePayload: 'online', awayPayload: 'away',
  tileLabelFontSize: 18, tileDisplayFontSize: 14
}));
localStorage.setItem('mqttPanelTopics', JSON.stringify([
  { label: 'Light', name: 'verify/light', functionType: 'cycle', functionEnabled: true,
    payloadValues: [
      { value: 'on', display: 'ON', backgroundColor: '#20c997', textColor: '#ffffff', convertValue: '1' },
      { value: 'off', display: 'OFF', backgroundColor: '#ff0000', textColor: '#ffffff', convertValue: '0' }
    ],
    cycleNextPayload: 'next', cyclePrevPayload: 'prev', currentValue: 'on', currentPayload: null },
  { label: 'Bridge', name: 'verify/src', functionType: 'transfer', functionEnabled: true,
    payloadValues: [], transferTopic: 'verify/dst', currentPayload: null },
  { label: 'Timer', name: 'verify/timer', functionType: 'timer', functionEnabled: true,
    payloadValues: [], timerInterval: 1, timerCountdownBg: '#007bff', timerCountdownFg: '#ffffff',
    timerOvertimeBg: '#ff0000', timerOvertimeFg: '#ffffff', timerSound: 'beep', currentPayload: null },
  { label: 'Alarm', name: 'verify/alarm', functionType: 'schedule', functionEnabled: true,
    payloadValues: [], schedules: [{ days: [0,1,2,3,4,5,6], time: '00:00', payload: 'x', enabled: true }],
    currentPayload: null }
]));
location.reload();
```

**Rows（一覧の内容を取得）:**

```js
[...document.querySelectorAll('#topicTableBody tr')].map(r => [...r.cells].map(c => c.textContent.trim()))
```

**State（接続インジケータの状態）:**

```js
document.getElementById('connectionStatus').className
```

注意: 自動操作でブラウザを動かす場合、`alert` / `confirm` は操作を止めてしまう。検証の最初に次を実行する。

```js
window.alert = m => console.log('ALERT', m); window.confirm = () => true;
```

---

### Task 1: 処理機能の削除

**Files:**
- Modify: `index.html`
- Delete: `audio/sound2-1.wav`

**Interfaces:**
- Consumes: なし
- Produces:
  - `resolveTopicDisplay(topic) -> { displayText: string, backgroundColor: string, textColor: string }`
  - `renderCurrentView() -> void`（現在の表示モードに応じて `renderTileView()` か `renderTopics()` を呼ぶ）

- [ ] **Step 1: 削除後に残ってはいけない識別子を確認する（失敗することを確認）**

```bash
grep -cE "functionType|functionEnabled|handleTransfer|handleConvert|handleCycle|handleTimer|playTimerSound|formatTimerDisplay|toggleFunction|getFunctionDisplayName|showTimerColorPalette|testTimerSound|selectAllDays|deselectAllDays|startScheduleChecker|checkSchedules|scheduleInterval|timerIntervals|convert-value|convertValue|convertTopic|transferTopic|cycleNextPayload|timerSound|function-toggle|function-active|function-inactive|lastRetain|currentValue|audio/" index.html
```

期待: 0 より大きい数（現状は該当あり）。Step 9 で 0 になる。

- [ ] **Step 2: CSS を削除する**

`index.html` の `<style>` から次の2ブロックを削除する。

- `.function-toggle`、`.function-active`、`.function-inactive` の3ルール（29〜42行）
- `[data-theme="dark"] .function-toggle`、`[data-theme="dark"] .function-active`、`[data-theme="dark"] .function-inactive` の3ルール（668〜681行）

- [ ] **Step 3: HTML を削除する**

1. 一覧テーブルのヘッダから `<th>Function</th>`（757行）を削除する。
2. ペイロード値テーブルのヘッダ（968〜977行）を次に置き換える。

```html
                                <thead>
                                    <tr>
                                        <th style="width: 35%">Payload Value</th>
                                        <th style="width: 35%">Display Text</th>
                                        <th style="width: 60px; text-align: center">BG</th>
                                        <th style="width: 60px; text-align: center">Text</th>
                                        <th style="width: 40px"></th>
                                    </tr>
                                </thead>
```

3. `<!-- Function Settings Section -->` から、それを閉じる `</div>` まで（987〜1222行）を丸ごと削除する。削除後、`<!-- Payload Values Section -->` の `</div>` の直後に `</form>` が来る。

- [ ] **Step 4: グローバル変数、イベントリスナ、メッセージ処理を整理する**

1. `let timerIntervals = {};` の行（1241行）を削除する。
2. `initializeEventListeners` から次を削除する。
   - `document.getElementById('functionType').addEventListener('change', toggleFunctionSettings);`
   - `// Timer color pickers` のブロック（4つの `addEventListener`）
   - `// Test sound button` とその `addEventListener`
3. `connect` ハンドラから `startScheduleChecker();` を削除する。
4. `stripRuntimeFields` を次に置き換える。

```js
        function stripRuntimeFields(topicsArray) {
            return topicsArray.map(t => {
                const copy = Object.assign({}, t);
                delete copy.currentPayload;
                delete copy.rawPayload;
                delete copy.jsonDisplayText;
                return copy;
            });
        }
```

5. `handleMessage` の `const topicConfig = topics.find(...)` 以降を次に置き換える。

```js
            const topicConfig = topics.find(t => t.name === topic);
            if (!topicConfig) return;

            // Store original payload
            topicConfig.rawPayload = payload;

            // Extract value from JSON if path is configured
            let effectivePayload = payload;
            if (topicConfig.jsonPathValue) {
                const extracted = extractJsonValue(payload, topicConfig.jsonPathValue);
                if (extracted !== null) {
                    effectivePayload = extracted;
                }
            }

            // Extract display text from JSON if path is configured
            let jsonDisplayText = null;
            if (topicConfig.jsonPathDisplay) {
                jsonDisplayText = extractJsonValue(payload, topicConfig.jsonPathDisplay);
            }
            topicConfig.jsonDisplayText = jsonDisplayText;

            updateTopicDisplay(topic, effectivePayload);
        }
```

6. `handleTopicsConfigMessage` 内の、受信データから実行時フィールドを消している `topics = data.map(incoming => { ... });` を次に置き換える。

```js
            topics = stripRuntimeFields(data);
```

7. 次の関数を丸ごと削除する: `handleTransfer`、`handleConvert`、`handleCycle`、`handleTimer`、`playTimerSound`、`formatTimerDisplay`（1682〜1844行）。

- [ ] **Step 5: Topic モーダルの処理を書き換える**

`showTopicModal` を次に置き換える。

```js
        // Topic management
        function showTopicModal(topicData = null) {
            const modal = bootstrap.Modal.getOrCreateInstance(document.getElementById('topicModal'));

            // Reset form
            document.getElementById('topicForm').reset();
            document.getElementById('payloadValues').innerHTML = '';

            if (topicData) {
                // Edit mode
                document.getElementById('topicModalTitle').textContent = 'Edit Topic';
                document.getElementById('topicLabel').value = topicData.label || '';
                document.getElementById('topicName').value = topicData.name || '';
                document.getElementById('showInTileView').checked = topicData.showInTileView !== false;

                // Load JSON path settings
                document.getElementById('jsonPathValue').value = topicData.jsonPathValue || '';
                document.getElementById('jsonPathDisplay').value = topicData.jsonPathDisplay || '';

                // Load payload values
                (topicData.payloadValues || []).forEach(pv => {
                    addPayloadEntry(pv);
                });
            } else {
                // Add mode
                document.getElementById('topicModalTitle').textContent = 'New Topic';
                addPayloadEntry(); // Add one empty entry
            }

            modal.show();
        }
```

`addPayloadEntry` の `entry.innerHTML` から、`convert-value` の入力を含む `<td>` を削除する。

```html
                <td>
                    <input type="text" class="form-control form-control-sm convert-value" value="${data?.convertValue || ''}">
                </td>
```

`toggleFunctionSettings` を丸ごと削除する。

`saveTopic` を次に置き換える。

```js
        function saveTopic() {
            const form = document.getElementById('topicForm');
            if (!form.checkValidity()) {
                form.reportValidity();
                return;
            }

            const topicData = {
                label: document.getElementById('topicLabel').value,
                name: document.getElementById('topicName').value,
                showInTileView: document.getElementById('showInTileView').checked,
                jsonPathValue: document.getElementById('jsonPathValue').value.trim() || null,
                jsonPathDisplay: document.getElementById('jsonPathDisplay').value.trim() || null,
                payloadValues: [],
                currentPayload: null
            };

            // Collect payload values
            const payloadEntries = document.querySelectorAll('.payload-entry');
            payloadEntries.forEach(entry => {
                const value = entry.querySelector('.payload-value').value;
                const display = entry.querySelector('.payload-display').value;
                const backgroundColor = entry.querySelector('.payload-background-color').value;
                const textColor = entry.querySelector('.payload-text-color').value;

                if (value) {
                    topicData.payloadValues.push({
                        value,
                        display: display || value,
                        backgroundColor,
                        textColor
                    });
                }
            });

            if (currentTopicIndex >= 0) {
                topics[currentTopicIndex] = topicData;
            } else {
                topics.push(topicData);
            }

            saveTopics();
            publishTopicsConfig();
            renderCurrentView();

            if (mqttClient && mqttClient.connected) {
                mqttClient.subscribe(topicData.name);
            }

            bootstrap.Modal.getInstance(document.getElementById('topicModal')).hide();
        }
```

- [ ] **Step 6: 表示ロジックを `resolveTopicDisplay` にまとめる**

`// UI rendering` コメントの直後、`renderTopics` の前に次の2関数を追加する。

```js
        function resolveTopicDisplay(topic) {
            const payloadValues = topic.payloadValues || [];
            const currentPayloadValue = payloadValues.find(pv => pv.value === topic.currentPayload);
            // Fallback to wildcard (*) if no exact match
            const effectivePayloadValue = currentPayloadValue || payloadValues.find(pv => pv.value === '*');

            // Use jsonDisplayText if available, otherwise use payload value display or current payload
            let displayText;
            if (topic.jsonDisplayText) {
                displayText = topic.jsonDisplayText;
            } else {
                displayText = currentPayloadValue ? currentPayloadValue.display : (topic.currentPayload || '');
            }

            return {
                displayText,
                backgroundColor: effectivePayloadValue ? effectivePayloadValue.backgroundColor : '#ffffff',
                textColor: effectivePayloadValue ? effectivePayloadValue.textColor : '#000000'
            };
        }

        function renderCurrentView() {
            if (currentView === 'tile') {
                renderTileView();
            } else {
                renderTopics();
            }
        }
```

`renderTopics` の中で、`let displayText, backgroundColor, textColor;` から `const functionText = ...;` の文の終わりまで（2151〜2200行）を次の1行に置き換える。

```js
                const { displayText, backgroundColor, textColor } = resolveTopicDisplay(topic);
```

同じく `row.innerHTML` を次に置き換える。

```js
                row.innerHTML = `
                    <td class="drag-handle" title="Drag to reorder">
                        <i class="bi bi-grip-vertical"></i>
                    </td>
                    <td>${topic.label}</td>
                    <td>${displayText}</td>
                    <td>${topic.name}</td>
                    <td class="payload-cell" data-index="${index}">
                        <span class="payload-value">${topic.currentPayload || ''}</span>
                    </td>
                `;
```

`renderTopics` の末尾近くにある `const functionToggle = row.querySelector('.function-toggle');` とそれに続く `if (functionToggle) { ... }` を削除する。

`renderTileView` の中で、`let displayText, backgroundColor, textColor;` から機能分岐の `if / else` の終わりまで（3017〜3063行）を次の1行に置き換える。

```js
                const { displayText, backgroundColor, textColor } = resolveTopicDisplay(topic);
```

- [ ] **Step 7: 残りの関数を削除し、再描画を `renderCurrentView` に寄せる**

1. 次の関数を丸ごと削除する: `getFunctionDisplayName`、`toggleFunction`、`showTimerColorPalette`（`// Timer color palette functions` コメントを含む）、`testTimerSound`（`// Test timer sound` コメントを含む）、`selectAllDays`、`deselectAllDays`（`// Day selection helper functions` コメントを含む）。
2. ファイル末尾の `// Schedule checker` から `checkSchedules` の終わりまで（3186〜3242行）を削除する。
3. `duplicateSelectedTopic` から `duplicatedTopic.currentValue = null;` を削除する。
4. 次の3か所にある下記のブロックを `renderCurrentView();` に置き換える: `handleTopicsConfigMessage`、`updateTopicDisplay`、`publishNextValue`。

```js
            if (currentView === 'tile') {
                renderTileView();
            } else {
                renderTopics();
            }
```

5. `importData` の `renderTopics();` を `renderCurrentView();` に置き換える。

- [ ] **Step 8: `audio/` を削除する**

```bash
git rm -r audio
```

- [ ] **Step 9: 識別子と構文を確認する**

Step 1 の grep を再実行する。期待: `0`。

構文チェック（Verification Environment 参照）を実行する。期待: 出力なし。

- [ ] **Step 10: ブラウザで動作を確認する**

ブローカーとページ配信を起動し、別の端末で `mosquitto_sub -h 127.0.0.1 -t 'verify/#' -v` を実行しておく。ページを開いて Seed を実行する。

1. Rows を実行する。期待: 4行で、各行は5セル（ハンドル、Label、Display、Topic、Payload）。ヘッダに Function がない。
2. `mosquitto_pub -h 127.0.0.1 -t verify/light -m on -r` を実行する。期待: Light の行が `["", "Light", "ON", "verify/light", "on"]` になり、背景が `#20c997`。
3. `mosquitto_pub -h 127.0.0.1 -t verify/src -m hello` を実行する。期待: `mosquitto_sub` に `verify/src hello` だけが出る。`verify/dst` は出ない（転送されない）。
4. `mosquitto_pub -h 127.0.0.1 -t verify/timer -m 5` を実行して5秒待つ。期待: Timer の行の Display と Payload が `5` のまま。`mosquitto_sub` に `verify/timer 4` などが出ない。
5. `mosquitto_pub -h 127.0.0.1 -t verify/light -m next` を実行する。期待: `mosquitto_sub` に `verify/light next` だけが出る（循環のパブリッシュがない）。
6. Light の行を右クリックして Edit を選ぶ。期待: モーダルに Function Settings がなく、ペイロード値テーブルに Convert Value 列がない。Save を押すと閉じる。
7. コントロールバーの表示切り替えでタイル表示にする。期待: 4タイルが表示される。Light のタイルをクリックすると `mosquitto_sub` に `verify/light off` が出て、タイルが赤になる。
8. コンソールにエラーがない。

- [ ] **Step 11: コミットする**

```bash
git add -A index.html audio
git commit -m "Remove transfer, convert, cycle, schedule, and timer functions

Co-Authored-By: Claude Fable 5.1 <noreply@anthropic.com>"
```

---

### Task 2: 旧データの正規化

**Files:**
- Modify: `index.html`
- Modify: `test-settings.json`

**Interfaces:**
- Consumes: `stripRuntimeFields(topicsArray)`、`validateTopicsConfig(data) -> string | null`、`renderCurrentView()`（Task 1）
- Produces: `normalizeTopic(topic) -> object`（旧フィールドを除いたコピーを返す。`payloadValues` は必ず配列）

- [ ] **Step 1: 正規化されていないことを確認する（失敗することを確認）**

ページで Seed を実行し、コンソールで次を実行する。

```js
typeof normalizeTopic
```

期待: `"undefined"`。

- [ ] **Step 2: `normalizeTopic` を追加する**

`// Color palette definition` の `colorPalette` 定義の直後に追加する。

```js
        // Fields left over from removed processing functions
        const LEGACY_TOPIC_FIELDS = [
            'functionType', 'functionEnabled',
            'transferTopic', 'convertTopic', 'convertDefault',
            'cycleNextPayload', 'cyclePrevPayload', 'schedules',
            'timerInterval', 'timerCountdownBg', 'timerCountdownFg',
            'timerOvertimeBg', 'timerOvertimeFg', 'timerSound',
            'currentValue', 'lastRetain', 'timerRemaining'
        ];

        function normalizeTopic(topic) {
            const copy = Object.assign({}, topic);
            LEGACY_TOPIC_FIELDS.forEach(field => delete copy[field]);

            const payloadValues = Array.isArray(copy.payloadValues) ? copy.payloadValues : [];
            copy.payloadValues = payloadValues.map(pv => {
                const pvCopy = Object.assign({}, pv);
                // Convert old color format to new format
                if (pvCopy.color && !pvCopy.backgroundColor) {
                    pvCopy.backgroundColor = pvCopy.color;
                    pvCopy.textColor = '#000000';
                }
                delete pvCopy.convertValue;
                return pvCopy;
            });
            return copy;
        }
```

- [ ] **Step 3: `loadTopics` に適用する**

`loadTopics` を次に置き換える。

```js
        function loadTopics() {
            const saved = localStorage.getItem('mqttPanelTopics');
            if (!saved) return;

            try {
                const parsed = JSON.parse(saved);
                if (Array.isArray(parsed)) {
                    topics = parsed.filter(t => t && typeof t === 'object').map(normalizeTopic);
                }
            } catch (e) {
                console.error('Failed to load saved topics:', e);
            }
        }
```

- [ ] **Step 4: `importData` に適用する**

`importData` を次に置き換える。

```js
        function importData() {
            try {
                const data = JSON.parse(document.getElementById('dataTextArea').value);
                if (data.topics) {
                    const validationError = validateTopicsConfig(data.topics);
                    if (validationError) {
                        throw new Error(validationError);
                    }
                    topics = data.topics.map(normalizeTopic);
                    saveTopics();
                    publishTopicsConfig();
                    renderCurrentView();
                    subscribeToTopics();
                }
                if (data.settings) {
                    localStorage.setItem('mqttPanelSettings', JSON.stringify(data.settings));
                    loadSettings();
                }
                alert('Data imported successfully');
            } catch (error) {
                alert('Invalid data format');
            }
        }
```

`subscribeToTopics()` は未接続のとき何もしない。インポートしたトピックを再読み込みなしで購読するために呼ぶ。

- [ ] **Step 5: リモート設定の受信に適用する**

`handleTopicsConfigMessage` の `// Compare with current config` 以降、`// Save without publishing back` の前までを次に置き換える。

```js
            // Compare with current config (ignoring runtime and legacy fields)
            const incoming = stripRuntimeFields(data.map(normalizeTopic));
            const incomingJson = JSON.stringify(incoming);
            const currentJson = JSON.stringify(stripRuntimeFields(topics));
            if (incomingJson === currentJson) return;

            topics = incoming;
            subscribeToTopics();
```

- [ ] **Step 6: `test-settings.json` から旧フィールドを除く**

```bash
node -e '
const fs = require("fs");
const legacy = ["functionType","functionEnabled","transferTopic","convertTopic","convertDefault","cycleNextPayload","cyclePrevPayload","schedules","timerInterval","timerCountdownBg","timerCountdownFg","timerOvertimeBg","timerOvertimeFg","timerSound","currentValue","lastRetain","timerRemaining"];
const d = JSON.parse(fs.readFileSync("test-settings.json", "utf8"));
d.topics = d.topics.map(t => {
  legacy.forEach(k => delete t[k]);
  (t.payloadValues || []).forEach(pv => delete pv.convertValue);
  return t;
});
fs.writeFileSync("test-settings.json", JSON.stringify(d, null, 2) + "\n");
'
grep -cE "functionType|functionEnabled|convertValue|transferTopic|convertTopic|cycleNextPayload|schedules|timerInterval|currentValue" test-settings.json
```

期待: `0`。`git diff --stat test-settings.json` で削除行だけであることを確かめる。

- [ ] **Step 7: ブラウザで確認する**

構文チェックを実行する。期待: 出力なし。ページで Seed を実行する。

1. 関数単体:

```js
JSON.stringify(normalizeTopic({ name: 'a', functionType: 'cycle', schedules: [], lastRetain: true,
  payloadValues: [{ value: 'x', convertValue: '1', color: '#fff' }] }))
```

期待: `{"name":"a","payloadValues":[{"value":"x","color":"#fff","backgroundColor":"#fff","textColor":"#000000"}]}`

```js
JSON.stringify(normalizeTopic({ name: 'a', payloadValues: 'oops' }))
```

期待: `{"name":"a","payloadValues":[]}`

2. localStorage からの読み込み:

```js
topics.length + ':' + topics.some(t => LEGACY_TOPIC_FIELDS.some(f => f in t) || t.payloadValues.some(pv => 'convertValue' in pv))
```

期待: `"4:false"`

3. リモート設定（旧フィールド付き）:

```bash
mosquitto_pub -h 127.0.0.1 -t clients/verify/topics -r -m '[{"label":"Remote","name":"verify/remote","functionType":"transfer","transferTopic":"x"}]'
mosquitto_sub -h 127.0.0.1 -t clients/verify/topics/status -C 1
mosquitto_pub -h 127.0.0.1 -t verify/remote -m hi
```

期待: status は `success`。Rows は `[["", "Remote", "hi", "verify/remote", "hi"]]`。`JSON.stringify(topics[0])` に `functionType` と `transferTopic` が含まれず、`"payloadValues":[]` が含まれる。

4. リモート設定（不正）:

```bash
mosquitto_pub -h 127.0.0.1 -t clients/verify/topics -r -m 'not json'
mosquitto_sub -h 127.0.0.1 -t clients/verify/topics/status -C 1
```

期待: `error`。Rows は 3 と変わらない。

5. インポート: Settings → Data Management のテキストエリアに次を入れて Import を押す。

```json
{"topics":[{"label":"Imp","name":"verify/imp","functionType":"timer","timerSound":"beep","payloadValues":[{"value":"1","display":"One","color":"#ffc107","convertValue":"z"}]}]}
```

期待: コンソールに `ALERT Data imported successfully`。`JSON.stringify(topics)` が `[{"label":"Imp","name":"verify/imp","payloadValues":[{"value":"1","display":"One","color":"#ffc107","backgroundColor":"#ffc107","textColor":"#000000"}]}]`。`mosquitto_pub -h 127.0.0.1 -t verify/imp -m 1` で行の Display が `One` になる（再読み込みなしで購読されている）。

6. インポート（不正）: テキストエリアに `{"topics":{"a":1}}` を入れて Import を押す。期待: `ALERT Invalid data format`。`topics.length` は 1 のまま。

7. 壊れた保存データ（Review Focus 2）:

```js
localStorage.setItem('mqttPanelTopics', '{bad'); location.reload();
```

期待: ページが表示され、Rows は `[]`、コンソールに `Failed to load saved topics:` が出る。未捕捉の例外はない。

```js
localStorage.setItem('mqttPanelTopics', '[null, 5, {"name":"verify/ok","label":"OK"}]'); location.reload();
```

期待: Rows は1行（`OK`）。

```js
localStorage.setItem('mqttPanelTopics', '{"a":1}'); location.reload();
```

期待: Rows は `[]`、コンソールにエラーなし。

- [ ] **Step 8: コミットする**

```bash
git add index.html test-settings.json
git commit -m "Normalize legacy topic fields on load, import, and remote config

Co-Authored-By: Claude Fable 5.1 <noreply@anthropic.com>"
```

---

### Task 3: 自動再接続

**Files:**
- Modify: `index.html`

**Interfaces:**
- Consumes: `subscribeToTopics()`、`subscribeTopicsConfig()`、`publishTopicsConfig()`、`handleMessage(topic, payload, retain)`
- Produces:
  - `setConnectionState(state)`。`state` は `'connected' | 'reconnecting' | 'disconnected'`
  - `connectMQTT()`、`disconnectMQTT()`（シグネチャは従来と同じ）
  - `connect` ハンドラ（Task 4 がここに1行追加する）

- [ ] **Step 1: 現状では起動時にブローカーが落ちていると復帰しないことを確認する（失敗することを確認）**

ブローカーを停止する。ページで Seed を実行する。10秒後にブローカーを起動し、40秒待つ。次を実行する。

```js
!!(mqttClient && mqttClient.connected)
```

期待: `false`。現状のコードは `close` で `mqttClient = null` にするので、捨てられたクライアントが裏で再接続しても、パネルは接続を持っていない（インジケータが緑になっても行は更新されない）。`mosquitto_pub -h 127.0.0.1 -t verify/light -m on -r` を実行しても Light の行が変わらないことも確かめる。Step 8 の 1 で直る。

- [ ] **Step 2: mqtt.js のバージョンを固定する**

```html
    <script src="https://unpkg.com/mqtt/dist/mqtt.min.js"></script>
```

を次に置き換える。

```html
    <script src="https://unpkg.com/mqtt@5.16.0/dist/mqtt.min.js"></script>
```

- [ ] **Step 3: 再接続中の表示用 CSS を追加する**

`.status-disconnected` ルールの直後に追加する。

```css
        .status-reconnecting {
            background-color: #ffc107;
        }
```

`[data-theme="dark"] #tileControlBar.disconnected` ルールの直後に追加する。

```css
        #tileControlBar.reconnecting {
            background: rgba(255, 215, 100, 0.95);
        }

        [data-theme="dark"] #tileControlBar.reconnecting {
            background: rgba(130, 100, 0, 0.95);
        }
```

- [ ] **Step 4: グローバル変数と初期状態を書き換える**

次の2行を削除する。

```js
        let reconnectInterval = null; // Store reconnect check interval ID
        let isManualDisconnect = false; // Track if disconnect was manual
```

同じ位置に次を追加する。

```js
        let reconnectWaiting = false; // True while mqtt.js waits between reconnect attempts
```

`DOMContentLoaded` ハンドラ内の

```js
            // Set initial disconnected state
            document.getElementById('tileControlBar').classList.add('disconnected');
```

を次に置き換える。

```js
            setConnectionState('disconnected');
```

- [ ] **Step 5: 接続管理を書き換える**

`// MQTT connection management` コメントから `stopReconnectCheck` の終わりまで（`connectMQTT`、`disconnectMQTT`、`startReconnectCheck`、`stopReconnectCheck`）を次に置き換える。

```js
        // MQTT connection management
        function setConnectionState(state) {
            // state: 'connected' | 'reconnecting' | 'disconnected'
            document.getElementById('connectionStatus').className = 'status-indicator status-' + state;
            const controlBar = document.getElementById('tileControlBar');
            controlBar.classList.toggle('disconnected', state === 'disconnected');
            controlBar.classList.toggle('reconnecting', state === 'reconnecting');
        }

        function connectMQTT() {
            const host = document.getElementById('mqttHost').value.trim();
            const port = document.getElementById('mqttPort').value;
            const username = document.getElementById('mqttUsername').value;
            const password = document.getElementById('mqttPassword').value;
            const statusTopic = document.getElementById('statusTopic').value;
            const onlinePayload = document.getElementById('onlinePayload').value;
            const awayPayload = document.getElementById('awayPayload').value;

            if (!host) {
                alert('Please enter a host');
                return;
            }

            // Only one client may exist; drop the previous one before reconnecting
            if (mqttClient) {
                mqttClient.end(true);
                mqttClient = null;
            }

            // Determine protocol based on page protocol
            // If page is HTTPS, use wss://, otherwise use ws://
            const protocol = window.location.protocol === 'https:' ? 'wss://' : 'ws://';

            // mqtt.js retries every reconnectPeriod until the broker accepts the connection.
            // Subscriptions are re-issued in the connect handler, so resubscribe is off.
            const options = {
                username: username || undefined,
                password: password || undefined,
                clean: true,
                keepalive: 30,
                connectTimeout: 10000,
                reconnectPeriod: 5000,
                reconnectOnConnackError: true,
                resubscribe: false
            };

            // Add will message if status topic is configured
            if (statusTopic && awayPayload) {
                options.will = {
                    topic: statusTopic,
                    payload: awayPayload,
                    qos: 0,
                    retain: true
                };
            }

            // Build connection URL with appropriate protocol
            const url = `${protocol}${host}:${port}/mqtt`;

            const client = mqtt.connect(url, options);
            mqttClient = client;
            reconnectWaiting = false;
            setConnectionState('reconnecting');

            // Events from a replaced client are ignored
            client.on('connect', function() {
                if (client !== mqttClient) return;
                console.log('MQTT connected successfully');
                reconnectWaiting = false;
                setConnectionState('connected');

                // Publish online status if configured
                if (statusTopic && onlinePayload) {
                    client.publish(statusTopic, onlinePayload, { retain: true });
                    console.log(`Published online status: ${statusTopic} = ${onlinePayload}`);
                }

                subscribeToTopics();
                subscribeTopicsConfig();
                publishTopicsConfig();
            });

            client.on('message', function(topic, message, packet) {
                if (client !== mqttClient) return;
                handleMessage(topic, message.toString(), packet.retain);
            });

            client.on('reconnect', function() {
                if (client !== mqttClient) return;
                console.log('MQTT reconnecting...');
                reconnectWaiting = false;
                setConnectionState('reconnecting');
            });

            client.on('close', function() {
                if (client !== mqttClient) return;
                console.log('MQTT connection closed');
                reconnectWaiting = true;
                setConnectionState('reconnecting');
            });

            client.on('error', function(error) {
                if (client !== mqttClient) return;
                console.error('MQTT Error:', error);
            });
        }

        function disconnectMQTT() {
            if (mqttClient) {
                mqttClient.end();
                mqttClient = null;
            }
            reconnectWaiting = false;
            setConnectionState('disconnected');
        }

        // Skip the wait between retries when the network or the tab comes back
        function reconnectNow() {
            if (!mqttClient || mqttClient.connected || !reconnectWaiting) return;
            console.log('Reconnecting immediately...');
            mqttClient.reconnect();
        }
```

`reconnectNow` が `reconnectWaiting` を見る理由: mqtt.js の `reconnect()` は接続試行の最中に呼ぶと、進行中のストリームを閉じずに新しいストリームを作る。待ち時間中（`close` の後、次の `reconnect` イベントの前）にだけ呼ぶ。

- [ ] **Step 6: 即時再接続のイベントを登録する**

`initializeEventListeners` の `// Window resize handler for tile font size` の前に追加する。

```js
            // Reconnect without waiting when the network or the tab comes back
            window.addEventListener('online', reconnectNow);
            document.addEventListener('visibilitychange', function() {
                if (document.visibilityState === 'visible') {
                    reconnectNow();
                }
            });

```

- [ ] **Step 7: 古い識別子が残っていないことを確認する**

```bash
grep -cE "reconnectInterval|isManualDisconnect|startReconnectCheck|stopReconnectCheck|unpkg.com/mqtt/dist" index.html
```

期待: `0`。構文チェックを実行する。期待: 出力なし。

- [ ] **Step 8: ブラウザで確認する**

1. 起動時にブローカーが停止している場合: ブローカーを停止し、ページで Seed を実行する。State は `status-indicator status-reconnecting`。ブローカーを起動して15秒以内に State が `status-indicator status-connected` になる。`mosquitto_pub -h 127.0.0.1 -t verify/light -m on -r` で Light の行が ON になる。
2. ブローカーの再起動: ブローカーを停止する。State が `status-indicator status-reconnecting` になり、コントロールバーが黄色になる。ブローカーを起動して15秒以内に `status-indicator status-connected` に戻る。`mosquitto_pub -h 127.0.0.1 -t verify/light -m off -r` で Light の行が OFF になる（再購読されている）。`mosquitto_sub -h 127.0.0.1 -t clients/verify -C 1` が `online` を返す。
3. 手動 Disconnect: Settings → Disconnect を押す。State は `status-indicator status-disconnected`、`mqttClient === null`。15秒待っても変わらない。Connect を押すと `status-indicator status-connected` になる。
4. Connect の連打: ほかの `mosquitto_sub` をすべて止めてから、Connect を続けて3回押す。

```bash
mosquitto_sub -h 127.0.0.1 -t '$SYS/broker/clients/connected' -W 25 | tail -1
```

期待: `2`（ページと、この `mosquitto_sub` 自身。ブローカーは約10秒ごとに値を更新するので、25秒間受信した最後の値を見る）。

5. 即時再接続: ブローカーを停止し、State が reconnecting になったらコンソールで `window.dispatchEvent(new Event('online'))` を数回実行する。期待: 例外が出ない。ブローカーを起動すると15秒以内に接続し、4 と同じコマンドの結果が `2`（接続が二重になっていない）。
6. コンソールに、接続失敗時の `MQTT Error:` 以外のエラーがない。

- [ ] **Step 9: 接続を拒否するブローカーで確認する（Review Focus 1）**

ブローカーを停止し、`mosquitto -c "$V/mosquitto-deny.conf" -v` で起動する。ページを再読み込みして20秒待つ。

期待: State は `status-indicator status-reconnecting` のまま。ブローカーのログに、約5秒ごとに接続の拒否が出続ける（再試行が止まらない）。`alert` は出ない。

拒否するブローカーを停止し、`mosquitto -c "$V/mosquitto.conf" -v` で起動する。期待: 15秒以内に `status-indicator status-connected` になる。

- [ ] **Step 10: コミットする**

```bash
git add index.html
git commit -m "Rely on mqtt.js reconnect with a single client and three-state indicator

Co-Authored-By: Claude Fable 5.1 <noreply@anthropic.com>"
```

---

### Task 4: Topic ブラウザ

**Files:**
- Modify: `index.html`

**Interfaces:**
- Consumes: `setConnectionState(state)` と `connect` ハンドラ（Task 3）、`renderCurrentView()`（Task 1）、`getTopicsConfigTopic() -> string | null`、`saveTopics()`、`publishTopicsConfig()`、`handleMessage(topic, payload, retain)`
- Produces: `openTopicBrowser(mode)`。`mode` は `'multi' | 'single'`

- [ ] **Step 1: ブラウザがないことを確認する（失敗することを確認）**

```js
document.getElementById('topicBrowserModal') === null && typeof openTopicBrowser
```

期待: `"undefined"`。

- [ ] **Step 2: CSS を追加する**

`#topicModal .modal-dialog, #settingsModal .modal-dialog` ルールのセレクタに `#topicBrowserModal .modal-dialog` を加える。

```css
        #topicModal .modal-dialog,
        #settingsModal .modal-dialog,
        #topicBrowserModal .modal-dialog {
            max-width: 90%;
            width: 90%;
        }
```

`[data-theme="dark"] .form-select` ルールの直後に追加する。

```css
        [data-theme="dark"] .input-group-text {
            background-color: #3a3a3a;
            border-color: #555;
            color: #e0e0e0;
        }

        /* Topic browser */
        .browser-row {
            cursor: pointer;
        }

        .browser-row-registered {
            cursor: default;
            opacity: 0.5;
        }

        .browser-payload {
            max-width: 40vw;
            overflow: hidden;
            text-overflow: ellipsis;
            white-space: nowrap;
        }
```

- [ ] **Step 3: HTML を追加する**

コントロールバーの `tileAddBtn` ボタンの直後に追加する。

```html
        <button class="tile-control-btn" id="tileBrowseBtn" title="Browse Topics">
            <i class="bi bi-search"></i>
        </button>
```

Topic モーダルの Topic Name 入力（`<input type="text" class="form-control" id="topicName" required>`）を次に置き換える。

```html
                                <div class="input-group">
                                    <input type="text" class="form-control" id="topicName" required>
                                    <button type="button" class="btn btn-outline-secondary" id="topicBrowseBtn" title="Browse topics on the broker">
                                        <i class="bi bi-search"></i> Browse
                                    </button>
                                </div>
```

Topic モーダル（`<div class="modal fade" id="topicModal" ...>`）を閉じる `</div>` の直後、`<script src=...bootstrap...>` の前に追加する。

```html

    <!-- Topic Browser Modal -->
    <div class="modal fade" id="topicBrowserModal" tabindex="-1">
        <div class="modal-dialog modal-xl modal-dialog-scrollable">
            <div class="modal-content">
                <div class="modal-header">
                    <h5 class="modal-title">Browse Topics</h5>
                    <button type="button" class="btn-close" data-bs-dismiss="modal"></button>
                </div>
                <div class="modal-body">
                    <div class="row g-2 mb-3">
                        <div class="col-md-5">
                            <div class="input-group">
                                <span class="input-group-text">Filter</span>
                                <input type="text" class="form-control" id="browserFilter" value="#">
                                <button type="button" class="btn btn-outline-primary" id="browserApplyFilterBtn">Apply</button>
                            </div>
                        </div>
                        <div class="col-md-7">
                            <input type="search" class="form-control" id="browserSearch" placeholder="Search topics">
                        </div>
                    </div>
                    <div class="alert alert-warning d-none" id="browserNotice"></div>
                    <table class="table table-sm">
                        <thead>
                            <tr>
                                <th style="width: 40px" class="browser-check-col"></th>
                                <th>Topic</th>
                                <th>Last Payload</th>
                                <th style="width: 90px">Retained</th>
                            </tr>
                        </thead>
                        <tbody id="browserTableBody">
                            <!-- Discovered topics will be inserted here -->
                        </tbody>
                    </table>
                </div>
                <div class="modal-footer">
                    <span class="me-auto text-muted small" id="browserCount"></span>
                    <button type="button" class="btn btn-secondary" data-bs-dismiss="modal">Close</button>
                    <button type="button" class="btn btn-primary" id="browserAddBtn">Add selected</button>
                </div>
            </div>
        </div>
    </div>
```

- [ ] **Step 4: 状態変数を追加する**

`let lastPublishedTopicsJson = null;` の行の直後に追加する。

```js

        // Topic browser state
        const BROWSER_MAX_TOPICS = 2000;
        let browserActive = false; // True while the browser modal is open
        let browserMode = 'multi'; // 'multi' (bulk add) or 'single' (pick for the topic form)
        let browserFilter = '#'; // Subscribed topic filter
        let browserTopics = new Map(); // topic name -> { payload, retain }
        let browserSelected = new Set();
        let browserTruncated = false;
        let browserError = null;
        let browserRenderTimer = null;
```

- [ ] **Step 5: ブラウザの関数を追加する**

`// Message handling` コメント（`extractJsonValue` の前）の直前に追加する。

```js
        // Topic browser: MQTT has no topic listing, so subscribe to a wildcard
        // filter while the modal is open and collect whatever arrives
        function defaultLabelForTopic(name) {
            const segments = name.split('/').filter(s => s !== '');
            return segments.length > 0 ? segments[segments.length - 1] : name;
        }

        function openTopicBrowser(mode) {
            browserMode = mode;
            const browserEl = document.getElementById('topicBrowserModal');
            const show = () => bootstrap.Modal.getOrCreateInstance(browserEl).show();

            if (mode === 'single') {
                // Bootstrap does not support stacked modals: hide the topic form
                // while browsing and bring it back when the browser closes
                const topicEl = document.getElementById('topicModal');
                topicEl.addEventListener('hidden.bs.modal', show, { once: true });
                bootstrap.Modal.getInstance(topicEl).hide();
            } else {
                show();
            }
        }

        function startTopicBrowser() {
            browserActive = true;
            browserTopics.clear();
            browserSelected.clear();
            browserTruncated = false;
            browserError = null;
            document.getElementById('browserFilter').value = browserFilter;
            document.getElementById('browserSearch').value = '';
            subscribeBrowserFilter();
            renderTopicBrowser();
        }

        function stopTopicBrowser() {
            unsubscribeBrowserFilter();
            browserActive = false;
            browserTopics.clear();
            browserSelected.clear();
            if (browserRenderTimer) {
                clearTimeout(browserRenderTimer);
                browserRenderTimer = null;
            }

            if (browserMode === 'single') {
                bootstrap.Modal.getOrCreateInstance(document.getElementById('topicModal')).show();
            }
        }

        function subscribeBrowserFilter() {
            if (!browserActive || !mqttClient || !mqttClient.connected) return;

            const filter = browserFilter;
            mqttClient.subscribe(filter, function(err, granted) {
                if (!browserActive || filter !== browserFilter) return;
                if (err) {
                    browserError = 'Cannot subscribe to "' + filter + '": ' + err.message;
                } else if (granted && granted.some(g => g.qos === 128)) {
                    browserError = 'The broker refused the subscription to "' + filter + '".';
                } else {
                    browserError = null;
                }
                scheduleBrowserRender();
            });
        }

        function unsubscribeBrowserFilter() {
            if (!mqttClient || !mqttClient.connected) return;
            // A failed subscription has nothing to unsubscribe
            if (browserError) return;
            // Keep the subscription when the panel itself relies on the same filter
            if (topics.some(t => t.name === browserFilter) || browserFilter === getTopicsConfigTopic()) return;
            mqttClient.unsubscribe(browserFilter);
        }

        function applyBrowserFilter() {
            const filter = document.getElementById('browserFilter').value.trim();
            if (!filter) return;

            unsubscribeBrowserFilter();
            browserFilter = filter;
            browserTopics.clear();
            browserSelected.clear();
            browserTruncated = false;
            browserError = null;
            subscribeBrowserFilter();
            renderTopicBrowser();
        }

        function collectBrowserMessage(topic, payload, retain) {
            // Config sync topics are internal to the panel
            const configTopic = getTopicsConfigTopic();
            if (configTopic && (topic === configTopic || topic.startsWith(configTopic + '/'))) return;

            const known = browserTopics.get(topic);
            if (!known && browserTopics.size >= BROWSER_MAX_TOPICS) {
                if (!browserTruncated) {
                    browserTruncated = true;
                    scheduleBrowserRender();
                }
                return;
            }
            if (known && known.payload === payload && known.retain === retain) return;

            browserTopics.set(topic, { payload, retain });
            scheduleBrowserRender();
        }

        function scheduleBrowserRender() {
            if (!browserActive || browserRenderTimer) return;
            browserRenderTimer = setTimeout(function() {
                browserRenderTimer = null;
                renderTopicBrowser();
            }, 250);
        }

        function renderTopicBrowser() {
            const tbody = document.getElementById('browserTableBody');
            const notice = document.getElementById('browserNotice');
            const addBtn = document.getElementById('browserAddBtn');
            const search = document.getElementById('browserSearch').value.trim().toLowerCase();
            const single = browserMode === 'single';
            const registered = new Set(topics.map(t => t.name));

            document.querySelectorAll('.browser-check-col').forEach(el => el.classList.toggle('d-none', single));
            addBtn.classList.toggle('d-none', single);

            const messages = [];
            if (!mqttClient || !mqttClient.connected) {
                messages.push('Not connected to the broker. Topics are listed once the connection is established.');
            }
            if (browserError) {
                messages.push(browserError);
            }
            if (browserTruncated) {
                messages.push('Showing the first ' + BROWSER_MAX_TOPICS + ' topics. Narrow the filter to see others.');
            }
            notice.textContent = messages.join(' ');
            notice.classList.toggle('d-none', messages.length === 0);

            const names = Array.from(browserTopics.keys())
                .filter(name => name.toLowerCase().includes(search))
                .sort();

            tbody.innerHTML = '';
            names.forEach(name => {
                const info = browserTopics.get(name);
                const isRegistered = registered.has(name);

                const row = document.createElement('tr');
                row.className = isRegistered ? 'browser-row browser-row-registered' : 'browser-row';
                row.dataset.topic = name;

                let checkbox = null;
                if (!single) {
                    const checkCell = document.createElement('td');
                    checkbox = document.createElement('input');
                    checkbox.type = 'checkbox';
                    checkbox.className = 'form-check-input';
                    checkbox.disabled = isRegistered;
                    checkbox.checked = browserSelected.has(name);
                    checkCell.appendChild(checkbox);
                    row.appendChild(checkCell);
                }

                // Broker-supplied strings are inserted as text, never as HTML
                const nameCell = document.createElement('td');
                nameCell.textContent = isRegistered ? name + ' (added)' : name;
                row.appendChild(nameCell);

                const payloadCell = document.createElement('td');
                payloadCell.className = 'browser-payload';
                payloadCell.textContent = info.payload.length > 200 ? info.payload.slice(0, 200) + '...' : info.payload;
                payloadCell.title = info.payload.slice(0, 500);
                row.appendChild(payloadCell);

                const retainCell = document.createElement('td');
                retainCell.textContent = info.retain ? 'Yes' : '';
                row.appendChild(retainCell);

                if (!isRegistered) {
                    row.addEventListener('click', function(e) {
                        if (single) {
                            pickBrowserTopic(name);
                            return;
                        }
                        // A click on the checkbox has already toggled it
                        if (e.target !== checkbox) {
                            checkbox.checked = !checkbox.checked;
                        }
                        if (checkbox.checked) {
                            browserSelected.add(name);
                        } else {
                            browserSelected.delete(name);
                        }
                        updateBrowserFooter(names.length);
                    });
                }

                tbody.appendChild(row);
            });

            updateBrowserFooter(names.length);
        }

        function updateBrowserFooter(shownCount) {
            let text = shownCount + ' of ' + browserTopics.size + ' topics';
            if (browserMode === 'multi') {
                text += ', ' + browserSelected.size + ' selected';
            }
            document.getElementById('browserCount').textContent = text;
            document.getElementById('browserAddBtn').disabled = browserSelected.size === 0;
        }

        function addSelectedBrowserTopics() {
            const registered = new Set(topics.map(t => t.name));
            const names = Array.from(browserSelected).filter(name => !registered.has(name)).sort();
            if (names.length === 0) return;

            names.forEach(name => {
                const info = browserTopics.get(name);
                topics.push({
                    label: defaultLabelForTopic(name),
                    name: name,
                    showInTileView: true,
                    jsonPathValue: null,
                    jsonPathDisplay: null,
                    payloadValues: [],
                    currentPayload: info ? info.payload : null
                });
                if (mqttClient && mqttClient.connected) {
                    mqttClient.subscribe(name);
                }
            });

            saveTopics();
            publishTopicsConfig();
            renderCurrentView();

            bootstrap.Modal.getInstance(document.getElementById('topicBrowserModal')).hide();
        }

        function pickBrowserTopic(name) {
            document.getElementById('topicName').value = name;
            const labelInput = document.getElementById('topicLabel');
            if (!labelInput.value.trim()) {
                labelInput.value = defaultLabelForTopic(name);
            }
            bootstrap.Modal.getInstance(document.getElementById('topicBrowserModal')).hide();
        }

```

- [ ] **Step 6: 受信メッセージをブラウザに渡す**

`handleMessage` の先頭（`// Handle topics config sync` の前）に追加する。

```js
            if (browserActive) {
                collectBrowserMessage(topic, payload, retain);
            }

```

- [ ] **Step 7: 接続状態の変化をブラウザに伝える**

`setConnectionState` の末尾に追加する。

```js
            // The browser shows a notice while not connected
            scheduleBrowserRender();
```

`connect` ハンドラの `publishTopicsConfig();` の直後に追加する。

```js
                subscribeBrowserFilter();
```

- [ ] **Step 8: イベントリスナを登録する**

`initializeEventListeners` の `// Export/Import` の前に追加する。

```js
            // Topic browser
            document.getElementById('tileBrowseBtn').addEventListener('click', function() {
                openTopicBrowser('multi');
            });
            document.getElementById('topicBrowseBtn').addEventListener('click', function() {
                openTopicBrowser('single');
            });
            document.getElementById('browserApplyFilterBtn').addEventListener('click', applyBrowserFilter);
            document.getElementById('browserFilter').addEventListener('keydown', function(e) {
                if (e.key === 'Enter') {
                    e.preventDefault();
                    applyBrowserFilter();
                }
            });
            document.getElementById('browserSearch').addEventListener('input', renderTopicBrowser);
            document.getElementById('browserAddBtn').addEventListener('click', addSelectedBrowserTopics);
            document.getElementById('topicBrowserModal').addEventListener('show.bs.modal', startTopicBrowser);
            document.getElementById('topicBrowserModal').addEventListener('hidden.bs.modal', stopTopicBrowser);

```

- [ ] **Step 9: 構文を確認する**

構文チェックを実行する。期待: 出力なし。

- [ ] **Step 10: 一括追加を確認する**

ブローカーを再起動して retain を空にし、ページで Seed を実行する。次を実行する。

```bash
for t in verify/a verify/b/c other/x; do mosquitto_pub -h 127.0.0.1 -t "$t" -m "v-$t" -r; done
mosquitto_pub -h 127.0.0.1 -t verify/light -m on -r
```

ブラウザ一覧を取得するスニペット:

```js
[...document.querySelectorAll('#browserTableBody tr')].map(r => r.dataset.topic)
```

1. コントロールバーの Browse Topics を押す。1秒後にスニペットを実行する。期待: `["clients/verify", "other/x", "verify/a", "verify/b/c", "verify/light"]`（昇順。`clients/verify/topics` を含まない）。
2. `verify/light` の行は `browser-row-registered` クラスを持ち、チェックボックスが `disabled`、名前の表示が `verify/light (added)`。
3. Search に `other` と入力する。期待: スニペットは `["other/x"]`。件数表示は `1 of 5 topics, 0 selected`。Search を空に戻す。
4. Filter を `verify/#` にして Apply を押す。1秒後、期待: `["verify/a", "verify/b/c", "verify/light"]`。
5. `mosquitto_pub -h 127.0.0.1 -t verify/live -m 1` を実行する（retain なし）。1秒後、期待: 一覧に `verify/live` が加わり、Retained 列が空。
6. `verify/a` と `verify/b/c` の行をクリックする。期待: 件数表示が `4 of 4 topics, 2 selected`、Add selected が有効。
7. Add selected を押す。期待: モーダルが閉じる。Rows の末尾2行が `["", "a", "v-verify/a", "verify/a", "v-verify/a"]` と `["", "c", "v-verify/b/c", "verify/b/c", "v-verify/b/c"]`。

```bash
mosquitto_sub -h 127.0.0.1 -t clients/verify/topics -C 1 | grep -c '"name": "verify/'
```

期待: `6`（元の4件と追加の2件）。

8. `mosquitto_pub -h 127.0.0.1 -t verify/a -m changed` を実行する。期待: `a` の行の Payload が `changed` になる（追加時に購読されている）。
9. 閉じた後の購読解除: `mosquitto_pub -h 127.0.0.1 -t verify/unseen -m 1` を実行し、コンソールで `browserActive + ':' + browserTopics.size` を実行する。期待: `"false:0"`。ブローカーのログに、ページのクライアントからの `UNSUBSCRIBE`（`verify/#`）が出ている。

- [ ] **Step 11: 1件選択、未接続、再接続を確認する**

1. コントロールバーの Add Topic を押し、Label を空のまま Topic Name の横の Browse を押す。期待: Topic モーダルが隠れ、ブラウザが開く。チェックボックス列と Add selected がない。
2. フィルタは前回の `verify/#` のまま。`verify/live` は retain されていないので出ない。`mosquitto_pub -h 127.0.0.1 -t verify/pick -m p -r` を実行し、`verify/pick` の行をクリックする。期待: ブラウザが閉じて Topic モーダルが再表示され、Topic Name が `verify/pick`、Label が `pick`。
3. Label を `Mine` に変えてもう一度 Browse を押し、Close で閉じる。期待: Topic モーダルが再表示され、Label は `Mine`、Topic Name は `verify/pick` のまま。Cancel で閉じる。
4. 未接続: Settings → Disconnect を押し、Browse Topics を開く。期待: `Not connected to the broker.` で始まる注意書きが出て、一覧が空。
5. 開いたまま接続: ブラウザを閉じ、Settings → Connect を押して接続する。Browse Topics を開き、ブローカーを停止する。期待: 注意書きが出る。ブローカーを起動する。期待: 15秒以内に注意書きが消える。`mosquitto_pub -h 127.0.0.1 -t verify/after -m 1` を実行すると一覧に `verify/after` が出る（再購読されている）。
6. 不正なフィルタ: Filter を `a/#/b` にして Apply を押す。期待: `Cannot subscribe to "a/#/b"` で始まる注意書きが出る。Filter を `#` に戻して Apply を押すと注意書きが消える。

- [ ] **Step 12: HTML を含むトピックで確認する（Review Focus 3）**

```bash
mosquitto_pub -h 127.0.0.1 -t 'verify/<b>x</b>' -m '<img src=x onerror=alert(1)>' -r
```

Browse Topics を開き、次を実行する。

```js
document.querySelectorAll('#browserTableBody b, #browserTableBody img').length + ':' +
  [...document.querySelectorAll('#browserTableBody tr')].some(r => r.dataset.topic === 'verify/<b>x</b>')
```

期待: `"0:true"`。コンソールに `ALERT` が出ない。ブラウザを閉じる（このトピックは追加しない）。

- [ ] **Step 13: 登録済みトピックと同じフィルタで確認する（Review Focus 4）**

Browse Topics を開き、Filter を `verify/light` にして Apply を押し、Close で閉じる。

```bash
mosquitto_pub -h 127.0.0.1 -t verify/light -m off -r
```

期待: Light の行が OFF になる（ブラウザを閉じてもパネル自身の購読が残っている）。ブローカーのログに `verify/light` の `UNSUBSCRIBE` が出ていない。

確認後、Browse Topics を開いて Filter を `#` に戻し、Apply を押して閉じる。

- [ ] **Step 14: 2000件を超えるトピックで確認する（Review Focus 5）**

```bash
for i in $(seq 1 2100); do mosquitto_pub -h 127.0.0.1 -t "bulk/t$i" -m "$i" -r; done
```

Browse Topics を開き、3秒待って次を実行する。

```js
browserTopics.size + ':' + browserTruncated + ':' + document.getElementById('browserNotice').textContent.includes('first 2000 topics')
```

期待: `"2000:true:true"`。Search に `bulk/t19` と入力すると一覧が絞り込まれ、入力に対して固まらない。チェックボックスをクリックすると件数表示の selected が増える。

確認後、ブローカーを再起動して retain を空にする。

- [ ] **Step 15: コミットする**

```bash
git add index.html
git commit -m "Add topic browser for discovering and adding broker topics

Co-Authored-By: Claude Fable 5.1 <noreply@anthropic.com>"
```

---

### Task 5: ドキュメント

**Files:**
- Modify: `README.md`
- Modify: `README-ja.md`
- Modify: `CLAUDE.md`

**Interfaces:**
- Consumes: Task 1〜4 で確定した挙動と UI の名称（Browse Topics、Add selected、Filter、3状態のインジケータ）
- Produces: なし

- [ ] **Step 1: 絵文字が残っていることを確認する（失敗することを確認）**

```bash
perl -CSD -ne 'print "$ARGV:$.: $_" if /[\x{1F000}-\x{1FAFF}\x{2300}-\x{23FF}\x{2600}-\x{27BF}\x{2B00}-\x{2BFF}\x{FE0F}]/; close ARGV if eof' README.md README-ja.md CLAUDE.md | wc -l
```

期待: 0 より大きい数。

- [ ] **Step 2: 絵文字を機械的に削除する**

```bash
perl -CSD -i -pe 's/[\x{1F000}-\x{1FAFF}\x{2300}-\x{23FF}\x{2600}-\x{27BF}\x{2B00}-\x{2BFF}]\x{FE0F}?[ \t]?//g; s/\x{FE0F}//g' README.md README-ja.md
```

`→`（U+2192）と `≡`（U+2261）は範囲外なので残る。

削除で崩れた箇所を手で直す。

| ファイル | 直す箇所 | 直した後 |
|---|---|---|
| `README.md` | `Click the **Settings button** () in the bottom-right corner` | `Click the **Settings button** in the control bar` |
| `README-ja.md` | `右下の**設定ボタン**（）をクリック` | `コントロールバーの**設定ボタン**をクリック` |
| `README.md` | `Click the **Add Topic button** ()` に相当する箇所（空の括弧） | 空の括弧を削除 |
| `README-ja.md` | `コントロールバーの**トピック追加ボタン**（）をクリック` | `コントロールバーの**トピック追加ボタン**をクリック` |
| `README.md` | `**Made with for the MQTT community**` | `**Made for the MQTT community**` |
| `README-ja.md` | `**MQTTコミュニティのために を込めて作成**` | `**MQTTコミュニティのために作成**` |

```bash
grep -nE "\(\)|（）" README.md README-ja.md
```

期待: 出力なし。

- [ ] **Step 3: `README.md` の内容を更新する**

1. 冒頭の説明文を次に置き換える。

```markdown
A modern, single-file MQTT monitoring panel with dark mode, drag & drop reordering, and real-time color-coded status display.
```

2. `## Features` の箇条書きを次に置き換える。

```markdown
- **Single HTML File** - Complete application in one file, no build process required
- **Dark/Light Theme** - Automatic theme switching with persistent preferences
- **28-Color Palette** - Custom background and text colors for visual status indication
- **Drag & Drop Reordering** - Intuitive topic organization
- **Real-time Updates** - Live MQTT message monitoring with instant visual feedback
- **Responsive Design** - Works seamlessly on desktop and mobile
- **Inline Editing** - Click to edit payloads and publish instantly
- **Auto-connect** - Automatic MQTT connection on startup
- **Auto-reconnect** - Retries every 5 seconds until the broker is reachable, even if it was down at startup
- **Topic Browser** - Discover topics on the broker and add them in bulk
- **Auto-save** - All settings saved automatically to localStorage
- **Tile View** - Grid-based display optimized for 4:3 aspect ratio with click-to-publish
- **Fullscreen Mode** - Immersive fullscreen experience with dedicated control bar
- **Client Status** - Online/offline status reporting with will message support
- **JSON Path** - Extract values from JSON payloads using dot notation
- **Dynamic Font Size** - Configurable tile font sizes with automatic overflow handling
- **Remote Config Sync** - Read/write topic configuration via MQTT for external management
```

3. `### 1. Initial Configuration` の末尾にある2行の引用を次に置き換える。

```markdown
> The app connects automatically on startup when settings are saved, and reconnects automatically after the connection is lost.
> Client status messages are published with retain flag for persistent status.
```

4. `### 2. Adding Topics` の節の直後に次の節を追加する。

```markdown
### 3. Adding Topics from the Broker

Instead of typing topic names, you can pick them from the broker.

1. Click the **Browse Topics button** in the control bar
2. The panel subscribes to the **Filter** (default `#`) while the dialog is open and lists every topic that arrives, with its latest payload
3. Narrow the list with **Search**, or change the **Filter** (e.g., `home/#`) and click **Apply**
4. Check the topics you want and click **Add selected**

Added topics use the last segment of the topic name as their label. Edit them afterwards to set payload values and colors.

The **Browse** button next to **Topic Name** in the topic dialog opens the same list to pick a single topic.

> MQTT has no command to list topics. Retained topics appear immediately; other topics appear when a message is published while the dialog is open.
> The list stops at 2000 topics. Narrow the filter to see others. To browse broker statistics, set the filter to `$SYS/#`.
```

5. `### 3. Configuring Payload Values` の見出しを `### 4. Configuring Payload Values` に変える。その節から `- **Convert Value**: ...` の行と、`> Payload values are not required for Timer and Schedule functions` の引用（前後の空行1つを含む）を削除する。
6. `### 4. Setting Up Functions` の節を、見出しから `### 5. Save and Monitor` の直前まで丸ごと削除する。
7. `### 5. Save and Monitor` の項目 4（`**Toggle Functions**` と、その下の2行）を削除する。
8. `#### List View` から `**Click Function Button**` の行を削除する。
9. `### Control Bar (Right Side)` の `**Add Topic**` の行の直後に次を追加する。

```markdown
- **Browse Topics**: Discover topics on the broker and add them
```

10. `### Status Indicators` の節の本文を次に置き換える。

```markdown
The indicator in the top-right corner and the control bar background show the connection state:

- **Green indicator**: Connected
- **Yellow indicator and control bar**: Connecting or reconnecting; the panel retries every 5 seconds
- **Red indicator and control bar**: Disconnected; no connection settings, or **Disconnect** was clicked
```

11. `## Configuration Examples` から `### Smart Light Control (Cycle Function)`、`### Temperature Monitor (Convert Function)`、`### Sensor Bridge (Transfer Function)` の3節を削除し、代わりに `### JSON Payload Extraction` の前に次を追加する。

````markdown
### Light Switch (Tile Click)

```
Label: Living Room Light
Topic: home/living-room/light

Payload Values:
- "off" → Display: "Off" → Background: Light Red
- "on" → Display: "On" → Background: Light Green
```

**Behavior**: The tile shows the current state. Each click on the tile publishes the next payload value: off → on → off
````

12. `## Retain Flags` の節の本文（小見出し3つを含む）を次に置き換える。

```markdown
Every message published by MQTT Panel uses the retain flag, so the broker keeps the latest value:

- **Tile View Click**: Published values are retained
- **Quick Publish** (Context Menu): Published values are retained
- **Inline Payload Edit**: Published values are retained
- **Client Status**: Both online and away messages are retained
- **Remote Topics Config**: Configuration JSON, status, and errors are retained
```

13. `#### How It Works` の Publishing の3行を次に置き換える。

```markdown
- Configuration is published on MQTT connect and after any config change (add/edit/delete/duplicate/reorder/import/topic browser)
- Runtime state (`currentPayload`) is excluded from the published JSON
- Only publishes when configuration has actually changed
```

Receiving の箇条書きの末尾に次を追加する。

```markdown
- Settings of the removed functions (`functionType`, `convertValue`, and so on) in incoming configuration are ignored
```

14. `### Connection Issues` の箇条書きの末尾に次を追加する。

```markdown
- A yellow indicator means the panel is still retrying; check the host, port, and credentials
```

15. `## Contributing` の前に次の節を追加する。

```markdown
## Removed Functions

The Transfer, Convert, Cycle, Schedule, and Timer functions have been removed. MQTT Panel now monitors topics and publishes only on user action.

Existing configurations (localStorage, exported JSON, remote topics configuration) still load. Settings that belonged to those functions are discarded.
```

- [ ] **Step 4: `README-ja.md` の内容を更新する**

Step 3 と同じ位置に、同じ番号の変更を加える。置き換えと追加の文面は次のとおり。

1. 冒頭の説明文:

```markdown
ダークモード、ドラッグ&ドロップ並び替え、リアルタイムの色分け表示を備えた、単一ファイルのMQTT監視パネル。
```

2. `## 機能` の箇条書き:

```markdown
- **単一HTMLファイル** - ビルド不要、1ファイルで完結するアプリケーション
- **ダーク/ライトテーマ** - 設定を保存する自動テーマ切り替え
- **28色パレット** - 視覚的なステータス表示のためのカスタム背景色・文字色
- **ドラッグ&ドロップ並び替え** - 直感的なトピック整理
- **リアルタイム更新** - 即座の視覚フィードバック付きライブMQTTメッセージ監視
- **レスポンシブデザイン** - デスクトップとモバイルでシームレスに動作
- **インライン編集** - クリックでペイロードを編集して即座にパブリッシュ
- **自動接続** - 起動時にMQTTへ自動接続
- **自動再接続** - ブローカーに接続できるまで5秒ごとに再試行（起動時にブローカーが停止していても復帰）
- **トピックブラウザ** - ブローカー上のトピックを一覧から選んでまとめて追加
- **自動保存** - すべての設定をlocalStorageに自動保存
- **タイル表示** - 4:3比率に最適化されたグリッド表示とクリックパブリッシュ
- **フルスクリーンモード** - 専用コントロールバー付き没入型表示
- **クライアントステータス** - Willメッセージサポート付きオンライン/オフライン状態レポート
- **JSON Path** - ドット記法でJSONペイロードから値を抽出
- **動的フォントサイズ** - タイルフォントサイズの設定とオーバーフロー自動処理
- **リモート設定同期** - MQTTを介したトピック設定の読み書きによる外部管理
```

3. `### 1. 初期設定` 末尾の引用:

```markdown
> 設定が保存されている場合、起動時に自動的に接続し、接続が切れた後も自動的に再接続します。
> クライアントステータスメッセージはリテインフラグ付きでパブリッシュされ、永続的なステータスとして保存されます。
```

4. `### 2. トピックの追加` の直後に追加する節:

```markdown
### 3. ブローカーからトピックを追加

トピック名を手入力する代わりに、ブローカー上のトピックから選べます。

1. コントロールバーの**Browse Topicsボタン**をクリック
2. ダイアログを開いている間、**Filter**（既定は `#`）で購読し、届いたトピックを最新のペイロードとともに一覧表示
3. **Search**で一覧を絞り込むか、**Filter**を変更（例：`home/#`）して**「Apply」**をクリック
4. 追加したいトピックにチェックを入れて**「Add selected」**をクリック

追加したトピックのラベルには、トピック名の末尾のセグメントが入ります。ペイロード値や色は、追加後に編集して設定します。

トピック設定ダイアログの**Topic Name**の横にある**「Browse」**ボタンからも同じ一覧を開けます。こちらは1件を選んで入力欄に反映します。

> MQTTにはトピック一覧を取得するコマンドがありません。リテインされているトピックはすぐに表示され、それ以外はダイアログを開いている間にメッセージがパブリッシュされると表示されます。
> 一覧は2000件で打ち切ります。それ以上ある場合はFilterで絞り込んでください。ブローカーの統計情報を見るにはFilterに `$SYS/#` を指定します。
```

5. `### 3. ペイロード値の設定` を `### 4. ペイロード値の設定` に変える。`- **Convert Value**: ...` の行と `> タイマー機能とスケジュール機能ではペイロード値の設定は不要です` の引用を削除する。
6. `### 4. 機能の設定` の節を丸ごと削除する。
7. `### 5. 保存と監視` の項目 4（`**機能の切り替え**` と、その下の2行）を削除する。
8. `#### リスト表示` から `**機能ボタンをクリック**` の行を削除する。
9. `### コントロールバー（右側）` の `**トピック追加**` の行の直後に追加:

```markdown
- **Browse Topics**: ブローカー上のトピックを探して追加
```

10. `### ステータスインジケーター` の本文:

```markdown
右上のインジケーターとコントロールバーの背景色で接続状態を示します。

- **緑のインジケーター**: 接続中
- **黄色のインジケーターとコントロールバー**: 接続または再接続を試行中（5秒ごとに再試行）
- **赤のインジケーターとコントロールバー**: 切断（接続設定がない、または「Disconnect」をクリックした）
```

11. `## 設定例` から `### スマートライト制御（循環機能）`、`### 温度モニター（変換機能）`、`### センサーブリッジ（転送機能）` を削除し、`### JSONペイロード抽出` の前に追加:

````markdown
### ライトスイッチ（タイルクリック）

```
ラベル: リビングルームライト
トピック: home/living-room/light

ペイロード値:
- "off" → 表示: "消灯" → 背景: ライトレッド
- "on" → 表示: "点灯" → 背景: ライトグリーン
```

**動作**: タイルに現在の状態を表示します。タイルをクリックするたびに次のペイロード値をパブリッシュします: 消灯 → 点灯 → 消灯
````

12. `## リテインフラグ` の本文:

```markdown
MQTT Panelがパブリッシュするメッセージにはすべてリテインフラグが付き、ブローカーが最新の値を保持します。

- **タイル表示クリック**: パブリッシュされた値は保持される
- **クイックパブリッシュ**（コンテキストメニュー）: パブリッシュされた値は保持される
- **インラインペイロード編集**: パブリッシュされた値は保持される
- **クライアントステータス**: オンラインメッセージとオフラインメッセージの両方が保持される
- **リモートトピック設定**: 設定JSON、ステータス、エラーが保持される
```

13. `#### 動作` のパブリッシュの3行:

```markdown
- MQTT接続時および設定変更時（追加/編集/削除/複製/並び替え/インポート/トピックブラウザ）に設定をパブリッシュ
- 実行時の状態（`currentPayload`）はJSONから除外
- 設定が実際に変更された場合のみパブリッシュ
```

受信の箇条書きの末尾に追加:

```markdown
- 受信した設定に廃止された機能の設定（`functionType`、`convertValue` など）が含まれていても無視する
```

14. `### 接続の問題` の末尾に追加:

```markdown
- インジケーターが黄色の間は再試行を続けています。ホスト、ポート、認証情報を確認してください
```

15. `## 貢献` の前に追加:

```markdown
## 廃止された機能

転送・変換・循環・スケジュール・タイマーの各機能は廃止されました。MQTT Panelはトピックを監視し、ユーザーの操作があったときだけパブリッシュします。

既存の設定（localStorage、エクスポートしたJSON、リモートトピック設定）はそのまま読み込めます。廃止された機能の設定は破棄されます。
```

- [ ] **Step 5: `CLAUDE.md` を更新する**

この手順の中で `\`` と書いてある箇所は、`CLAUDE.md` にはバッククォートだけを書く（`\` は書かない）。

1. 冒頭の `A single HTML file MQTT message processing application with modern UI and dark mode support.` を `A single HTML file MQTT monitoring panel with modern UI and dark mode support.` に置き換える。
2. `## Overview` の本文を次に置き換える。

```markdown
MQTT Panel is a web application that monitors MQTT topics and displays their current state. It shows configured topics in a table or tile view, colors them by payload value, and publishes only on user action (tile click, quick publish, inline edit).
```

3. `### Topic Management` の Table Display から `- Function: ...` の行を削除する。CRUD Operations の `- Add Topic: Control bar button` の直後に `  - Browse Topics: Control bar button to discover topics on the broker and add them in bulk` を追加する。
4. `### Processing Functions (Mutually Exclusive)` の節を、見出しから `### Display & Color Settings` の直前まで丸ごと削除する。
5. `### Display & Color Settings` から `  - Convert Value: Value used in convert function` の行を削除する。
6. `### MQTT Connection` の箇条書きを次のように変える。
   - `**Auto-subscribe**` の行の直後に追加:

```markdown
- **Auto-reconnect**: A single mqtt.js client retries every 5 seconds without limit (`reconnectPeriod: 5000`, `keepalive: 30`, `connectTimeout: 10000`, `reconnectOnConnackError: true`)
  - Works when the broker is down at startup and when the broker refuses the connection
  - On every connect: publishes online status, re-subscribes all topics, re-subscribes and publishes the topics config
  - Manual Disconnect stops reconnecting until Connect is clicked or the page is reloaded
  - Browser `online` event and tab becoming visible trigger an immediate retry
```

   - `**Status Indicator**` の行を次に置き換える: `- **Status Indicator**: Fixed position indicator (top-right corner) with three states: connected (green), reconnecting (yellow), disconnected (red). All updates go through \`setConnectionState\``
   - `**Function Control**` の行と、その下の2行（ON / OFF）を削除する。
7. `### Remote Topics Configuration Sync` の前に次の節を追加する。

```markdown
### Topic Browser
- **Discovery**: MQTT has no topic listing, so the browser subscribes to a wildcard filter (default `#`, editable) while its modal is open and collects the topics that arrive
- **List**: Topic name, latest payload, retained flag; sorted by name; text search; capped at 2000 topics
- **Bulk Add** (control bar button): Check topics and click "Add selected"; label is the last topic segment, payload values are empty
- **Single Pick** ("Browse" button next to Topic Name in the topic modal): Click a row to fill the Topic Name field; the topic modal is hidden while browsing and restored afterwards
- **Exclusions**: Topics already added are shown but not selectable; the topics config sync topics are not listed
- **Unsubscribe**: The filter is unsubscribed when the modal closes, unless a configured topic uses the same name
- **Safety**: Topic names and payloads from the broker are inserted with `textContent`
```

8. `### Remote Topics Configuration Sync` の中で次を変える。
   - `Config changes: topic add/edit/delete/duplicate, function toggle, reorder, import` を `Config changes: topic add/edit/delete/duplicate, reorder, import, topic browser add` に置き換える。
   - `Runtime state changes (currentPayload, currentValue) do NOT trigger publish` を `Runtime state changes (currentPayload) do NOT trigger publish` に置き換える。
   - `Runtime fields (\`currentPayload\`, \`currentValue\`) are stripped on publish and ignored on receive` を `Runtime fields (\`currentPayload\`, \`rawPayload\`, \`jsonDisplayText\`) are stripped on publish and ignored on receive` に置き換える。
   - その直後に `  - Legacy fields of removed functions are stripped by \`normalizeTopic\` before comparison` を追加する。
   - `**JSON Format**` の行の括弧内を `(\`currentPayload\`, \`rawPayload\`, \`jsonDisplayText\`)` に置き換える。
9. `### Data Management` の `**Import**` の行の直後に次を追加する。

```markdown
- **Legacy Data**: `normalizeTopic` removes fields of the removed functions (`functionType`, `functionEnabled`, `transferTopic`, `convertTopic`, `convertDefault`, `cycleNextPayload`, `cyclePrevPayload`, `schedules`, `timer*`, `convertValue`) on localStorage load, import, and remote config receive
```

10. `### Retain Flags` の本文を次に置き換える。

```markdown
All messages published by the panel use the retain flag:

- Tile View Click
- Quick Publish (Context Menu)
- Inline Payload Edit
- Client Status: Both online and away messages
- Remote Topics Config: Configuration JSON, status, and errors
```

11. `### File Structure` はそのまま。`### Dependencies (CDN)` の `- MQTT.js: MQTT client library` を `- MQTT.js 5.16.0: MQTT client library (version pinned)` に置き換える。
12. `### 2. Topic Configuration` の手順を次に置き換える。

```markdown
1. Click "Add Topic" button in the control bar, or "Browse Topics" to pick topics from the broker
2. Enter basic information (label, topic name)
3. Configure payload values with display text and colors
4. Click "Save" to complete setup
```

13. `### 3. Operation` から `**Function Toggle**` の行と、その下の2行を削除する。
14. `## Configuration Examples` の節を、見出しから `## Data Format` の直前まで丸ごと削除する。
15. `## Data Format` の JSON から次の行を削除する: `"functionType"`、`"functionEnabled"`、`"convertValue"`、`"cycleNextPayload"`、`"cyclePrevPayload"`、`"currentValue"`。`"textColor": "#000000",` の末尾のカンマと `"currentPayload": "on",` の末尾のカンマを外して JSON として正しい形にする。
16. `### Control Bar (Right Side)` の `**Add Topic**` の行の直後に `- **Browse Topics**: Discover and add topics from the broker` を追加する。
17. `### Additional UI Elements` の `**Connection Status Indicator**` の行を `- **Connection Status Indicator**: Top-right corner (green=connected, yellow=reconnecting, red=disconnected)` に置き換え、`**Function Buttons**` の行を削除する。
18. `# Important Instructions` の `## Key Features Implemented` から次の行を削除する: `Configurable retain flags for different publish operations`、`Schedule function with day/time-based publishing`、`Timer function with countdown and audio alerts`、`Timer color priority system (payload values override timer colors)`。`Auto-connect MQTT on startup ...` の行の直後に次の2行を追加する。

```markdown
- Automatic reconnect with a single mqtt.js client and three-state connection indicator
- Topic browser for discovering broker topics (bulk add and single pick)
```

- [ ] **Step 6: 確認する**

Step 1 のコマンドを再実行する。期待: `0`。

```bash
grep -nEi "transfer|convert|cycle function|schedule|timer|function button|function toggle|転送|変換|循環機能|スケジュール|タイマー|機能ボタン" README.md README-ja.md CLAUDE.md
```

期待: 該当するのは「Removed Functions / 廃止された機能」の節、受信時に無視する旨の行、`CLAUDE.md` の Legacy Data の行と `normalizeTopic` に触れた行だけ。それ以外が出たら直す。

```bash
grep -n "^### [0-9]" README.md README-ja.md
```

期待: 各ファイルで `### 1.` 〜 `### 5.` が番号順に1つずつ並ぶ。

`CLAUDE.md` の Data Format の JSON を取り出して `node -e` で `JSON.parse` し、例外が出ないことを確かめる。

```bash
awk '/^## Data Format/{s=1} s&&/^```json/{f=1;next} f&&/^```/{exit} f' CLAUDE.md | node -e 'JSON.parse(require("fs").readFileSync(0,"utf8")); console.log("ok")'
```

期待: `ok`。

- [ ] **Step 7: コミットする**

```bash
git add README.md README-ja.md CLAUDE.md
git commit -m "Update docs: remove emoji and removed functions, document reconnect and topic browser

Co-Authored-By: Claude Fable 5.1 <noreply@anthropic.com>"
```

---

### Task 6: 全体の最終確認

**Files:**
- 変更なし（問題が見つかった場合だけ、該当タスクのファイルを直す）

**Interfaces:**
- Consumes: Task 1〜5 の成果物すべて
- Produces: なし

- [ ] **Step 1: 静的な確認をまとめて実行する**

```bash
awk '/<script>$/{f=1;next}/<\/script>/{f=0}f' index.html > "$V/panel.js" && node --check "$V/panel.js"
grep -cE "handleTransfer|handleConvert|handleCycle|handleTimer|playTimerSound|formatTimerDisplay|toggleFunction|getFunctionDisplayName|showTimerColorPalette|testTimerSound|selectAllDays|deselectAllDays|startScheduleChecker|checkSchedules|timerIntervals|convert-value|function-toggle|reconnectInterval|isManualDisconnect|audio/" index.html
ls audio 2>&1
git status --short
```

期待: 構文チェックは出力なし。grep は `0`。`ls audio` は `No such file or directory`。`git status --short` は出力なし。

- [ ] **Step 2: 通しで動作を確認する**

ブローカーを再起動し、ページで Seed を実行する。spec の「5. 検証」の各項目を上から順に確かめる。

1. 旧設定が読み込まれ、一覧が4行・4列（ハンドルを除く）で表示される。
2. `mosquitto_pub -h 127.0.0.1 -t verify/light -m on -r` で List の行が ON（`#20c997`）になる。タイル表示に切り替えても Light のタイルが同じ色で表示される。
3. タイルクリックで `off` がパブリッシュされる。リスト表示に戻し、行の右クリック → `Publish: ON`、Payload セルのクリック編集 → `off` を Publish、のどちらも `mosquitto_sub -h 127.0.0.1 -t verify/light -v` に出る。
4. ブローカーを停止 → 黄色 → 起動 → 15秒以内に緑、の後で 2 が再度通る。
5. Disconnect → 赤のまま → Connect → 緑。
6. Browse Topics で1件を一括追加でき、Add Topic → Browse で1件を選べる。
7. ダークテーマに切り替え、Browse Topics と Topic モーダルの文字が読める（背景と文字のコントラストがある）。
8. コンソールに `MQTT Error:`（ブローカー停止中のもの）以外のエラーがない。

- [ ] **Step 3: 後片付けをする**

```bash
pkill -f "$V/mosquitto" ; pkill -f "http.server 8000" ; rm -rf "$V"
```

問題を直した場合は、該当タスクのコミット規則に従ってコミットする。
