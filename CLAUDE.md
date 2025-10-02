# MQTT Panel

A single HTML file MQTT message processing application with modern UI and dark mode support.

## Overview

MQTT Panel is a web application that monitors MQTT topics and processes received messages through transfer, conversion, or cycling operations. It displays configured topics in a table format and allows real-time monitoring of message states.

## Key Features

### Topic Management
- **Table Display**: Topics displayed in sortable table format
  - Label: Custom identifier
  - Display: Display text corresponding to payload values
  - Topic: MQTT topic name
  - Payload: Current payload value (editable by clicking)
  - Function: ON/OFF status of configured functions (monitoring always continues)

- **CRUD Operations**:
  - Add Topic: Control bar button
  - Edit/Duplicate/Delete: Right-click on rows for context menu
  - Quick Publish: Right-click to publish predefined payload values
  - Duplicate: Creates a copy of the topic with "(Copy)" suffix

### Drag & Drop Reordering
- **Row Sorting**: Drag rows by the grip handle to reorder topics
- **Visual Feedback**: Smooth animations during drag operations
- **Auto-save**: Order changes are automatically saved

### Processing Functions (Mutually Exclusive)

#### 1. Transfer Function
- Forwards received payload as-is to another topic
- Settings: Target topic

#### 2. Convert Function
- Publishes corresponding values to another topic based on received payload
- Conversion values can be set for each payload value
- Settings: Target topic, default value (for undefined payloads)

#### 3. Cycle Function
- Advances/reverses through a list of values when specific payloads (next/prev) are received
- Publishes with retain flag (saved to MQTT server)
- Uses first element if no initial value
- Settings: Next payload (required), previous payload (optional)

#### 4. Schedule Function
- Publishes messages at scheduled times based on day of week and time
- Supports up to 2 independent schedules per topic
- Each schedule can be individually enabled/disabled
- Publishes with retain flag
- Settings: Days of week, time, payload, enabled/disabled per schedule

#### 5. Timer Function
- Countdown timer that publishes decreasing values at specified intervals
- Starts countdown when receiving an integer payload
- Stops when receiving a non-numeric payload
- Displays time in HH:MM:SS or MM:SS format (hours shown only when > 0)
- Supports overtime (negative values) with different color scheme
- Color priority: Payload value settings take precedence over timer colors
- Settings:
  - Publish interval (seconds)
  - Countdown color (background/foreground) - used when no payload value match
  - Overtime color (background/foreground) - used when no payload value match
  - Sound when reaching 0 (beep/bell/chime/custom audio file)
- Default colors:
  - Countdown: Blue background (#007bff), White text (#ffffff)
  - Overtime: Red background (#ff0000), White text (#ffffff)

### Display & Color Settings
- **Payload Value Configuration**: For each payload value
  - Display Text: Custom display name in table
  - Background Color: Table row background color (28-color palette)
  - Text Color: Table row text color (28-color palette)
  - Convert Value: Value used in convert function
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
- **Auto-connect**: Automatically connects on startup if settings exist
- **Auto-subscribe**: Automatically subscribes to configured topics
- **Status Indicator**: Fixed position indicator (top-right corner)
- **Real-time Updates**: Live payload value updates and color changes
- **Function Control**: Individual on/off control for each topic's processing functions
  - ON (Green): Messages are processed and functions execute
  - OFF (Gray): Messages are monitored and displayed only, no processing

### Data Management
- **Auto-save**: All settings saved to localStorage automatically
- **Export**: Full configuration export to JSON format
- **Import**: JSON configuration import with backward compatibility
- **Payload Editing**: Click-to-edit payload values with direct MQTT publish

## Technical Specifications

### File Structure
- `index.html`: Single file application (HTML/CSS/JavaScript)
- `CLAUDE.md`: Project documentation and instructions

### Dependencies (CDN)
- Bootstrap 5.3.0: UI framework with dark mode support
- Bootstrap Icons: Icon library
- MQTT.js: MQTT client library

### Browser Requirements
- Modern web browser with WebSocket support
- Drag & Drop API support
- CSS custom properties support
- localStorage support

### MQTT Broker Requirements
- WebSocket support (typically port 8083)
- Optional: Username/password authentication

## Usage Guide

### 1. Initial Setup
1. Open `index.html` in a modern web browser
2. Click the settings button (bottom-right) → "MQTT Connection" tab
3. Enter broker information (host, port, username, password)
4. Click "Connect" to establish MQTT connection

### 2. Topic Configuration
1. Click "Add Topic" button in the control bar
2. Enter basic information (label, topic name)
3. Configure payload values with display text and colors (not required for Timer/Schedule functions)
4. Select and configure a function (Transfer/Convert/Cycle/Schedule/Timer)
5. Click "Save" to complete setup

### 3. Operation
- **Real-time Monitoring**: View live message updates in the table
- **Function Toggle**: Click function buttons to enable/disable processing (monitoring continues)
  - Green (ON): Functions execute and process messages
  - Gray (OFF): Monitor only, no message processing
- **Quick Publish**: Right-click rows to publish predefined values
- **Edit Payloads**: Click payload cells to edit and publish directly
- **Reorder Topics**: Drag rows using the grip handle

### 4. Data Management
- **Export/Import**: Use Settings → "Data Management" tab
- **Auto-backup**: All changes saved automatically
- **Theme Toggle**: Click theme button (bottom-right area)

## Configuration Examples

### Cycle Function Example
```
Label: Light Control
Topic: home/light/status
Payload Values:
- value: "off", display: "Off", background: "#ffcccc", text: "#000000"
- value: "dim", display: "Dimmed", background: "#ffffcc", text: "#000000"
- value: "bright", display: "Bright", background: "#ccffcc", text: "#000000"
Function: Cycle
Next Payload: "next"
Previous Payload: "prev"
```

When current value is "dim" and "next" is received, "bright" will be published.

**Function Control**: Toggle OFF to monitor light status without affecting the cycling behavior.

### Convert Function Example
```
Label: Temperature Sensor
Topic: sensors/temperature
Payload Values:
- value: "20.5", display: "Room Temp", convert: "comfortable"
- value: "25.0", display: "Warm", convert: "warm"
Function: Convert
Target Topic: home/status/comfort
Default Value: "unknown"

**Function Control**: Disable to monitor temperature readings without triggering comfort control messages.
```

## Data Format

Configuration data is saved to localStorage in the following format:

```json
{
  "topics": [
    {
      "label": "Topic Name",
      "name": "mqtt/topic",
      "functionType": "cycle",
      "functionEnabled": true,
      "payloadValues": [
        {
          "value": "on",
          "display": "On",
          "backgroundColor": "#ccffcc",
          "textColor": "#000000",
          "convertValue": "1"
        }
      ],
      "cycleNextPayload": "next",
      "cyclePrevPayload": "prev",
      "currentPayload": "on",
      "currentValue": "on"
    }
  ],
  "settings": {
    "mqttHost": "broker.example.com",
    "mqttPort": "8083",
    "mqttUsername": "user",
    "mqttPassword": "pass"
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

### Control Bar (Right Side)
- **Fullscreen Toggle**: Enter/exit fullscreen mode
- **Theme Toggle**: Switch between dark/light themes
- **View Toggle**: Switch between list and tile views
- **Settings**: Open settings modal
- **Add Topic**: Create new topic
- **GitHub Link**: Bottom of control bar

### Additional UI Elements
- **Connection Status Indicator**: Top-right corner (green=connected, red=disconnected)
- **Drag Handle**: Left column with grip icon for row reordering (list view)
- **Payload Cells**: Click to edit and publish values directly (list view)
- **Function Buttons**: Toggle function enable/disable
- **Context Menu**: Right-click for edit/delete/quick publish options (list view)
- **Color Palette**: 28-color selection for background and text colors

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
- Context menu with quick publish options
- Dark/light theme with complete UI coverage
- Drag & drop row reordering
- Auto-connect MQTT on startup
- 28-color palette for background and text colors
- Real-time row color updates based on payload values
- Schedule function with day/time-based publishing
- Timer function with countdown and audio alerts
- English UI throughout the application