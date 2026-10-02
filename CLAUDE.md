# MQTT Panel

A single HTML file MQTT monitoring panel with modern UI and dark mode support.

## Overview

MQTT Panel is a web application that monitors MQTT topics and displays their current state. It shows configured topics in a table or tile view, colors them by payload value, and publishes only on user action (tile click, quick publish, inline edit).

## Key Features

### Topic Management
- **Table Display**: Topics displayed in sortable table format
  - Label: Custom identifier
  - Display: Display text corresponding to payload values
  - Topic: MQTT topic name
  - Payload: Current payload value (editable by clicking)

- **CRUD Operations**:
  - Add Topic: Control bar button
  - Browse Topics: Control bar button to discover topics on the broker and add them in bulk
  - Edit/Duplicate/Delete: Right-click on rows for context menu
  - Quick Publish: Right-click to publish predefined payload values
  - Duplicate: Creates a copy of the topic with "(Copy)" suffix

- **JSON Path Extraction**: Extract values from JSON payloads
  - JSON Path for Value: Extract specific element as payload value (dot notation, e.g., `data.status`)
  - JSON Path for Display: Extract specific element for display text (e.g., `data.label`)
  - Useful for processing structured JSON messages from IoT devices

### Drag & Drop Reordering
- **Row Sorting**: Drag rows by the grip handle to reorder topics
- **Visual Feedback**: Smooth animations during drag operations
- **Auto-save**: Order changes are automatically saved

### Display & Color Settings
- **Payload Value Configuration**: For each payload value
  - Display Text: Custom display name in table
  - Background Color: Table row background color (28-color palette)
  - Text Color: Table row text color (28-color palette)
- **Wildcard Value (`*`)**: Set payload value to `*` to define default colors for unmatched payloads
- **Default Colors for New Payloads**:
  - First payload: Teal background (#20c997), White text (#ffffff)
  - Subsequent payloads: Red background (#ff0000), White text (#ffffff)

### Theme Support
- **Dark/Light Mode**: Toggle between themes with floating action button
- **Auto-save**: Theme preference saved to localStorage
- **Complete Coverage**: All UI elements properly themed
- **Custom Colors**: Row colors work in both light and dark modes

### MQTT Connection
- **Connection Settings**: Host, port, username, password
- **Protocol Auto-detection**: Automatically uses wss:// for HTTPS pages, ws:// for HTTP pages
- **Client Status**: Configurable online/offline status reporting
  - Status Topic: Topic to publish client status
  - Online Payload: Message published when connected
  - Away Payload: Message published via will when disconnected
  - Both messages use retain flag for persistent status
- **Auto-connect**: Automatically connects on startup if settings exist
- **Auto-subscribe**: Automatically subscribes to configured topics
- **Auto-reconnect**: A single mqtt.js client retries every 5 seconds without limit (`reconnectPeriod: 5000`, `keepalive: 30`, `connectTimeout: 10000`, `reconnectOnConnackError: true`)
  - Works when the broker is down at startup and when the broker refuses the connection
  - On every connect: publishes online status, re-subscribes all topics, re-subscribes and publishes the topics config
  - Manual Disconnect stops reconnecting until Connect is clicked or the page is reloaded
  - Browser `online` event and tab becoming visible trigger an immediate retry
- **Status Indicator**: Fixed position indicator (top-right corner) with three states: connected (green), reconnecting (yellow), disconnected (red). All updates go through `setConnectionState`
- **Real-time Updates**: Live payload value updates and color changes
- **HTTPS Compatibility**: When accessing via HTTPS, WebSocket connections automatically use WSS (secure WebSocket)
  - Note: MQTT broker must support WSS connections for HTTPS deployments
  - Alternative: Use reverse proxy (nginx/apache) to provide WSS endpoint

### Topic Browser
- **Discovery**: MQTT has no topic listing, so the browser subscribes to a wildcard filter (default `#`, editable) while its modal is open and collects the topics that arrive
- **List**: Topic name, latest payload, retained flag; sorted by name; text search; capped at 2000 topics
- **Bulk Add** (control bar button): Check topics and click "Add selected"; label is the last topic segment, payload values are empty
- **Single Pick** ("Browse" button next to Topic Name in the topic modal): Click a row to fill the Topic Name field; the topic modal is hidden while browsing and restored afterwards
- **Exclusions**: Topics already added are shown but not selectable; the topics config sync topics are not listed
- **Unsubscribe**: The filter is unsubscribed when the modal closes, unless a configured topic uses the same name
- **Safety**: Topic names and payloads from the broker are inserted with `textContent`
- **Broker Limits**: A broker may cap how many retained messages it sends at once (Mosquitto: `max_queued_messages`, default 1000), so a wide filter can return fewer topics than exist

### Remote Topics Configuration Sync
- **MQTT-based Configuration**: Read/write topic configuration via MQTT topics
  - Uses Status Topic directly as prefix (e.g., Status Topic `clients/livingroom` → `clients/livingroom/topics`)
  - Requires Status Topic to be configured; feature is disabled without it
- **Topics**:
  - `clients/<hostname>/topics`: Topic configuration JSON (pretty format, retain)
  - `clients/<hostname>/topics/status`: `success` or `error` (retain)
  - `clients/<hostname>/topics/errors`: Error details when status is `error` (retain)
- **Publishing (local → external)**:
  - Publishes current config on MQTT connect and after any config change
  - Config changes: topic add/edit/delete/duplicate, reorder, import, topic browser add
  - Runtime state changes (currentPayload) do NOT trigger publish
  - Skips publish if config hasn't changed (JSON comparison)
- **Receiving (external → local)**:
  - Subscribes to config topic on MQTT connect
  - Validates incoming JSON (must be array, each element must have `name` field)
  - On validation error: publishes `error` to status topic and details to errors topic
  - On success with changes: replaces topics array, saves to localStorage, re-renders UI, publishes `success`
  - If no changes detected: does nothing (no echo)
  - Runtime fields (`currentPayload`, `rawPayload`, `jsonDisplayText`) are stripped on publish and ignored on receive
  - Legacy fields of removed functions are stripped by `normalizeTopic` before comparison
- **Echo Prevention**: Ignores own publish echo using a flag
- **Infinite Loop Prevention**: When applying external config, saves directly to localStorage without re-publishing
- **JSON Format**: Array of topic objects excluding runtime fields (`currentPayload`, `rawPayload`, `jsonDisplayText`)

### Data Management
- **Auto-save**: All settings saved to localStorage automatically
- **Export**: Full configuration export to JSON format
- **Import**: JSON configuration import with backward compatibility
- **Legacy Data**: `normalizeTopic` removes fields of the removed functions (`functionType`, `functionEnabled`, `transferTopic`, `convertTopic`, `convertDefault`, `cycleNextPayload`, `cyclePrevPayload`, `schedules`, `timer*`, `convertValue`) on localStorage load, import, and remote config receive
- **Payload Editing**: Click-to-edit payload values with direct MQTT publish
- **Remote Sync**: Topic configuration synchronized via MQTT (see Remote Topics Configuration Sync)

### Retain Flags

All messages published by the panel use the retain flag:

- Tile View Click
- Quick Publish (Context Menu)
- Inline Payload Edit
- Client Status: Both online and away messages
- Remote Topics Config: Configuration JSON, status, and errors

## Technical Specifications

### File Structure
- `index.html`: Single file application (HTML/CSS/JavaScript)
- `CLAUDE.md`: Project documentation and instructions

### Dependencies (CDN)
- Bootstrap 5.3.0: UI framework with dark mode support
- Bootstrap Icons: Icon library
- MQTT.js 5.16.0: MQTT client library (version pinned)

### Browser Requirements
- Modern web browser with WebSocket support
- Drag & Drop API support
- CSS custom properties support
- localStorage support

### MQTT Broker Requirements
- WebSocket support (typically port 8083 for ws://)
- For HTTPS deployments: WSS (secure WebSocket) support required
  - Direct WSS: Configure broker with TLS/SSL certificates (port 8084 or 9001 typically)
  - Reverse Proxy: Use nginx/apache to provide WSS endpoint over existing WebSocket
- Optional: Username/password authentication

#### WSS Setup Examples

**Mosquitto Direct WSS:**
```conf
listener 8083
protocol websockets

listener 8084
protocol websockets
certfile /path/to/cert.pem
keyfile /path/to/key.pem
```

**Nginx Reverse Proxy:**
```nginx
location /mqtt {
    proxy_pass http://localhost:8083;
    proxy_http_version 1.1;
    proxy_set_header Upgrade $http_upgrade;
    proxy_set_header Connection "upgrade";
}
```

## Usage Guide

### 1. Initial Setup
1. Open `index.html` in a modern web browser
2. Click the settings button (bottom-right) → "MQTT Connection" tab
3. Enter broker information (host, port, username, password)
4. (Optional) Configure "Client Status" tab for online/offline status reporting
   - Status Topic: e.g., "client/status"
   - Online Payload: e.g., "online"
   - Away Payload: e.g., "offline"
5. Click "Connect" to establish MQTT connection

### 2. Topic Configuration

1. Click "Add Topic" button in the control bar, or "Browse Topics" to pick topics from the broker
2. Enter basic information (label, topic name)
3. Configure payload values with display text and colors
4. Click "Save" to complete setup

### 3. Operation
- **Real-time Monitoring**: View live message updates in the table
- **Quick Publish**: Right-click rows to publish predefined values
- **Edit Payloads**: Click payload cells to edit and publish directly
- **Reorder Topics**: Drag rows using the grip handle

### 4. Data Management
- **Export/Import**: Use Settings → "Data Management" tab
- **Auto-backup**: All changes saved automatically
- **Theme Toggle**: Click theme button (bottom-right area)

### 5. Font Size Settings
- **Settings** → **"Font Size"** tab
- **Label Font Size (%)**: Percentage of tile size for label text (default: 18%)
- **Display Font Size (%)**: Percentage of tile size for display text (default: 14%)
- Range: 5% to 50%
- Changes apply immediately to tile view

## Data Format

Configuration data is saved to localStorage in the following format:

```json
{
  "topics": [
    {
      "label": "Topic Name",
      "name": "mqtt/topic",
      "jsonPathValue": "data.status",
      "jsonPathDisplay": "data.label",
      "payloadValues": [
        {
          "value": "on",
          "display": "On",
          "backgroundColor": "#ccffcc",
          "textColor": "#000000"
        }
      ],
      "currentPayload": "on"
    }
  ],
  "settings": {
    "mqttHost": "broker.example.com",
    "mqttPort": "8083",
    "mqttUsername": "user",
    "mqttPassword": "pass",
    "statusTopic": "client/status",
    "onlinePayload": "online",
    "awayPayload": "offline",
    "tileLabelFontSize": 18,
    "tileDisplayFontSize": 14
  },
  "theme": "dark"
}
```

## UI Components

### View Modes
- **List View**: Traditional table display with all topic details
- **Tile View**: Grid-based display optimized for 4:3 aspect ratio
  - Shows only Label and Display text
  - Click tiles to publish next value
  - Per-topic visibility control
  - Dynamic font sizing based on tile dimensions
  - Automatic text overflow handling (shrinks font, then shows "..." if needed)

### Control Bar (Right Side)
- **Fullscreen Toggle**: Enter/exit fullscreen mode
- **Theme Toggle**: Switch between dark/light themes
- **View Toggle**: Switch between list and tile views
- **Settings**: Open settings modal
- **Add Topic**: Create new topic
- **Browse Topics**: Discover and add topics from the broker
- **GitHub Link**: Bottom of control bar

### Additional UI Elements
- **Connection Status Indicator**: Top-right corner (green=connected, yellow=reconnecting, red=disconnected)
- **Drag Handle**: Left column with grip icon for row reordering (list view)
- **Payload Cells**: Click to edit and publish values directly (list view)
- **Context Menu**: Right-click for edit/delete/quick publish options (list view)
- **Color Palette**: 28-color selection for background and text colors
- **Wide Modals**: Topic and settings modals use 90% screen width for better usability
- **Table-based Payload Settings**: Payload values displayed in compact table format with clickable color previews

## Development

- **Repository**: https://github.com/ytx/mqtt_p
- **Single File**: Complete application in one HTML file
- **No Build Process**: Direct browser execution
- **Responsive Design**: Works on desktop and mobile devices
- **Accessibility**: Keyboard navigation and screen reader support

# Important Instructions

This is a single-file MQTT application with the following development guidelines:

- **Single File Architecture**: All functionality contained in `index.html`
- **Modern UI**: Bootstrap 5 with custom dark/light theme support
- **Real-time Updates**: Live MQTT message processing and display
- **Drag & Drop**: Interactive row reordering with visual feedback
- **Color Customization**: 28-color palette for row styling
- **Responsive Design**: Mobile and desktop compatible
- **Auto-save**: All settings and changes saved automatically
- **Export/Import**: Full configuration backup and restore

## Key Features Implemented
- Control bar with fullscreen, theme, and view controls
- List and tile view modes with optimized layouts
- Inline payload editing with direct MQTT publish
- Context menu with quick publish and duplicate options
- Topic duplication feature for easy configuration reuse
- Dark/light theme with complete UI coverage
- Drag & drop row reordering
- Auto-connect MQTT on startup with protocol auto-detection (ws:// or wss://)
- Automatic reconnect with a single mqtt.js client and three-state connection indicator
- Topic browser for discovering broker topics (bulk add and single pick)
- HTTPS-compatible with automatic WSS protocol selection
- Client status reporting with will message support
- 28-color palette for background and text colors
- Smart default colors for new payload values (Teal for first, Red for subsequent)
- Real-time row color updates based on payload values
- JSON Path extraction for structured JSON payloads (dot notation)
- Configurable tile view font sizes (label and display percentages)
- Automatic text overflow handling in tile view (font shrinking and ellipsis)
- Table-based Payload Value Settings with compact color selection
- Remote topics configuration sync via MQTT
- English UI throughout the application