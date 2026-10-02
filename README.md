# MQTT Panel

A modern, single-file MQTT monitoring panel with dark mode, drag & drop reordering, and real-time color-coded status display.

![MQTT Panel Interface](https://img.shields.io/badge/Interface-Modern%20UI-blue)
![Single File](https://img.shields.io/badge/Architecture-Single%20HTML-green)
![No Build Required](https://img.shields.io/badge/Setup-No%20Build-orange)

## Features

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

## Quick Start

1. **Download** the `index.html` file
2. **Open** it in any modern web browser
3. **Configure** your MQTT broker settings
4. **Start** monitoring and controlling your MQTT topics

No installation, no dependencies, no build process required!

## Step-by-Step Setup

### 1. Initial Configuration

1. Open `index.html` in your web browser
2. Click the **Settings button** in the control bar
3. Go to the **"MQTT Connection"** tab
4. Enter your broker details:
   - **Host**: Your MQTT broker address (e.g., `broker.example.com`)
   - **Port**: WebSocket port (typically `8083`)
   - **Username**: Your MQTT username (if required)
   - **Password**: Your MQTT password (if required)
5. (Optional) Configure **"Client Status"** tab for online/offline reporting:
   - **Status Topic**: Topic to publish status (e.g., `client/status`)
   - **Online Payload**: Message when connected (e.g., `online`)
   - **Away Payload**: Message when disconnected (e.g., `offline`)
6. Click **"Connect"**

> The app connects automatically on startup when settings are saved, and reconnects automatically after the connection is lost.
> Client status messages are published with retain flag for persistent status.

### 2. Adding Topics

1. Click the **Add Topic button** in the control bar
2. Fill in the basic information:
   - **Label**: A friendly name for your topic
   - **Topic Name**: The MQTT topic path (e.g., `home/sensors/temperature`)
   - **Show in Tile View**: Toggle whether to display in tile mode
3. (Optional) Configure **JSON Path** for structured payloads:
   - **JSON Path for Value**: Extract specific element as payload value (e.g., `data.status`)
   - **JSON Path for Display**: Extract specific element for display text (e.g., `data.label`)

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
> Brokers may limit how many retained messages they send at once (Mosquitto sends about 1000 by default, see `max_queued_messages`). If topics are missing, narrow the filter.

### 4. Configuring Payload Values

Payload values are displayed in a compact table format for easy configuration.

1. In the topic configuration dialog, click **"Add Payload Value"**
2. For each expected payload value:
   - **Payload Value**: The actual MQTT payload (e.g., `"on"`, `"23.5"`)
   - **Display Text**: Human-readable text (e.g., `"Light On"`, `"Room Temperature"`)
   - **BG** (Background Color): Click the color preview to select from 28 colors
   - **Text** (Text Color): Choose contrasting text color for readability

> **Wildcard (`*`)**: Set payload value to `*` to define default colors for any unmatched payloads

> **Default colors**: First payload uses Teal/White (#20c997/#ffffff), subsequent payloads use Red/White (#ff0000/#ffffff)

### 5. Save and Monitor

1. Click **"Save"** to create the topic
2. Watch real-time updates in the main table
3. Rows will automatically change colors based on current payload values

## Using the Interface

### View Modes

#### List View
- Traditional table layout with full topic details
- **Click Payload Cell**: Edit and publish values instantly
- **Right-click Row**: Access context menu for edit/duplicate/delete/quick publish
- **Drag Grip Handle**: Reorder topics by dragging the ≡ icon
- **Duplicate Topic**: Right-click and select "Duplicate" to create a copy

#### Tile View
- Grid-based layout optimized for 4:3 aspect ratio
- Shows only Label and Display text
- **Click Tile**: Publish next payload value (cycles through values)
- Automatically calculates optimal grid size
- Per-topic visibility control
- Dynamic font sizing based on tile dimensions
- Automatic text overflow handling (shrinks font, then shows "..." if needed)

### Control Bar (Right Side)

- **Fullscreen**: Enter/exit fullscreen mode
- **Theme**: Toggle dark/light theme
- **View**: Switch between list and tile views
- **Settings**: Open settings modal
- **Add Topic**: Create new topic
- **Browse Topics**: Discover topics on the broker and add them
- **GitHub**: Repository link (bottom)

### Status Indicators

The indicator in the top-right corner and the control bar background show the connection state:

- **Green indicator**: Connected
- **Yellow indicator and control bar**: Connecting or reconnecting; the panel retries every 5 seconds
- **Red indicator and control bar**: Disconnected; no connection settings, or **Disconnect** was clicked

## Configuration Examples

### Light Switch (Tile Click)

```
Label: Living Room Light
Topic: home/living-room/light

Payload Values:
- "off" → Display: "Off" → Background: Light Red
- "on" → Display: "On" → Background: Light Green
```

**Behavior**: The tile shows the current state. Each click on the tile publishes the next payload value: off → on → off

### JSON Payload Extraction

```
Label: IoT Sensor
Topic: sensors/device1
JSON Path for Value: data.status
JSON Path for Display: data.label

Payload Values:
- "active" → Display: "Active" → Background: Green
- "idle" → Display: "Idle" → Background: Gray
```

**Example JSON Payload**:
```json
{"data": {"status": "active", "label": "Sensor Online", "temp": 25.5}}
```

**Behavior**: Extracts `data.status` ("active") as payload value for color matching, and `data.label` ("Sensor Online") as display text.

## Retain Flags

Every message published by MQTT Panel uses the retain flag, so the broker keeps the latest value:

- **Tile View Click**: Published values are retained
- **Quick Publish** (Context Menu): Published values are retained
- **Inline Payload Edit**: Published values are retained
- **Client Status**: Both online and away messages are retained
- **Remote Topics Config**: Configuration JSON, status, and errors are retained

## Data Management

### Export Configuration
1. **Settings** → **"Data Management"** tab
2. Click **"Export"**
3. Copy the JSON configuration
4. Save to file for backup

### Import Configuration
1. **Settings** → **"Data Management"** tab
2. Paste JSON configuration in text area
3. Click **"Import"**
4. Configuration is automatically applied

### Backup Strategy
- Export regularly to preserve configurations
- Store JSON files in version control
- Share configurations between team members

### Font Size Settings
1. **Settings** → **"Font Size"** tab
2. Configure tile view font sizes:
   - **Label Font Size (%)**: Percentage of tile size for label text (default: 18%)
   - **Display Font Size (%)**: Percentage of tile size for display text (default: 14%)
3. Range: 5% to 50%
4. Changes apply immediately to tile view

### Remote Topics Configuration Sync

Topic configuration can be read and written via MQTT, enabling external tools to manage topics.

#### Prerequisites
- **Client Status** must be configured with a Status Topic (e.g., `clients/livingroom`)
- The Status Topic is used directly as the prefix (e.g., `clients/livingroom/topics`)

#### MQTT Topics

| Topic | Purpose | Retain |
|---|---|---|
| `clients/<hostname>/topics` | Topic configuration JSON (pretty format) | Yes |
| `clients/<hostname>/topics/status` | `success` or `error` after external update | Yes |
| `clients/<hostname>/topics/errors` | Error details when status is `error` | Yes |

#### How It Works

**Publishing (MQTT Panel → External)**:
- Configuration is published on MQTT connect and after any config change (add/edit/delete/duplicate/reorder/import/topic browser)
- Runtime state (`currentPayload`) is excluded from the published JSON
- Only publishes when configuration has actually changed

**Receiving (External → MQTT Panel)**:
- Subscribe to `clients/<hostname>/topics` and publish a JSON array of topic objects
- Each topic object must have at least a `name` field
- On success: topics are replaced, UI is re-rendered, `success` is published to status topic
- On error: `error` is published to status topic, details to errors topic
- If the received config is identical to current config, no action is taken
- Settings of the removed functions (`functionType`, `convertValue`, and so on) in incoming configuration are ignored

#### Example: Update Topics Externally

```bash
# Publish new configuration via mosquitto_pub
mosquitto_pub -h broker.example.com -t "clients/livingroom/topics" -r -f topics.json

# Check result
mosquitto_sub -h broker.example.com -t "clients/livingroom/topics/status" -C 1
```

## MQTT Broker Setup

### Public Test Brokers
- `broker.mqttdashboard.com:8000` (WebSocket)
- `test.mosquitto.org:8080` (WebSocket)

### Local Mosquitto Setup
```bash
# Install Mosquitto
sudo apt install mosquitto mosquitto-clients

# Enable WebSocket listener in /etc/mosquitto/mosquitto.conf
listener 8083
protocol websockets

# Restart service
sudo systemctl restart mosquitto
```

### Docker Setup
```bash
docker run -it -p 1883:1883 -p 8083:8083 \
  -v mosquitto.conf:/mosquitto/config/mosquitto.conf \
  eclipse-mosquitto
```

## Browser Compatibility

- Chrome 88+
- Firefox 85+
- Safari 14+
- Edge 88+

**Required Features**:
- WebSocket support
- Drag & Drop API
- CSS Custom Properties
- localStorage API

## Troubleshooting

### Connection Issues
- Check WebSocket port (usually 8083, not 1883)
- Verify broker supports WebSocket protocol
- Check firewall and network connectivity
- Try without username/password first
- A yellow indicator means the panel is still retrying; check the host, port, and credentials

### Color Not Showing
- Ensure payload values match exactly (case-sensitive)
- Check that payload value is configured in topic settings
- Verify MQTT messages are being received

### Drag & Drop Not Working
- Use the grip handle (≡) icon on the left
- Ensure browser supports Drag & Drop API
- Try refreshing the page

### Theme Not Persisting
- Check that localStorage is enabled
- Verify browser allows local storage for file:// URLs
- Try opening in http://localhost if needed

## Removed Functions

The Transfer, Convert, Cycle, Schedule, and Timer functions have been removed. MQTT Panel now monitors topics and publishes only on user action.

Existing configurations (localStorage, exported JSON, remote topics configuration) still load. Settings that belonged to those functions are discarded.

## Contributing

This is a single-file application for simplicity. When contributing:

1. **Preserve single-file architecture**
2. **Test in multiple browsers**
3. **Ensure both light and dark themes work**
4. **Maintain backward compatibility for data formats**
5. **Update CLAUDE.md documentation**

## License

MIT License - Feel free to use, modify, and distribute.

## Links

- **Repository**: https://github.com/ytx/mqtt_p
- **Issues**: https://github.com/ytx/mqtt_p/issues
- **MQTT.js Documentation**: https://github.com/mqttjs/MQTT.js
- **Bootstrap Documentation**: https://getbootstrap.com/docs/5.3/

---

**Made for the MQTT community**