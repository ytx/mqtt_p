# MQTT Patcher

A single HTML file MQTT message processing application with modern UI and dark mode support.

## Overview

MQTT Patcher is a web application that monitors MQTT topics and processes received messages through transfer, conversion, or cycling operations. It displays configured topics in a table format and allows real-time monitoring of message states.

## Key Features

### Topic Management
- **Table Display**: Topics displayed in sortable table format
  - Label: Custom identifier
  - Display: Display text corresponding to payload values
  - Topic: MQTT topic name
  - Payload: Current payload value (editable by clicking)
  - Function: ON/OFF status of configured functions

- **CRUD Operations**:
  - Add Topic: Floating action button (top-right)
  - Edit/Delete: Right-click on rows for context menu
  - Quick Publish: Right-click to publish predefined payload values

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

### Display & Color Settings
- **Payload Value Configuration**: For each payload value
  - Display Text: Custom display name in table
  - Background Color: Table row background color (28-color palette)
  - Text Color: Table row text color (28-color palette)
  - Convert Value: Value used in convert function

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
1. Click "Add Topic" floating action button (top-right)
2. Enter basic information (label, topic name)
3. Configure payload values with display text and colors
4. Select and configure a function (Transfer/Convert/Cycle)
5. Click "Save" to complete setup

### 3. Operation
- **Real-time Monitoring**: View live message updates in the table
- **Function Toggle**: Click function buttons to enable/disable
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

### Floating Action Buttons
- **Top-right**: Add Topic (+)
- **Bottom-right**: Settings (gear icon)
- **Bottom-right-left**: Theme Toggle (sun/moon icon)
- **Bottom-left**: GitHub Link
- **Top-right corner**: Connection Status Indicator

### Interactive Elements
- **Drag Handle**: Left column with grip icon for row reordering
- **Payload Cells**: Click to edit and publish values directly
- **Function Buttons**: Toggle function enable/disable
- **Context Menu**: Right-click for edit/delete/quick publish options
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
- Floating action buttons with glassmorphism design
- Inline payload editing with direct MQTT publish
- Context menu with quick publish options
- Dark/light theme with complete UI coverage
- Drag & drop row reordering
- Auto-connect MQTT on startup
- 28-color palette for background and text colors
- Real-time row color updates based on payload values
- English UI throughout the application