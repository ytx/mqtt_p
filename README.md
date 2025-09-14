# MQTT Patcher

A modern, single-file MQTT message processing application with dark mode, drag & drop reordering, and real-time color-coded monitoring.

![MQTT Patcher Interface](https://img.shields.io/badge/Interface-Modern%20UI-blue)
![Single File](https://img.shields.io/badge/Architecture-Single%20HTML-green)
![No Build Required](https://img.shields.io/badge/Setup-No%20Build-orange)

## ✨ Features

- 🎯 **Single HTML File** - Complete application in one file, no build process required
- 🌓 **Dark/Light Theme** - Automatic theme switching with persistent preferences
- 🎨 **28-Color Palette** - Custom background and text colors for visual status indication
- 🔄 **Drag & Drop Reordering** - Intuitive topic organization
- ⚡ **Real-time Updates** - Live MQTT message monitoring with instant visual feedback
- 📱 **Responsive Design** - Works seamlessly on desktop and mobile
- 🔧 **MQTT Functions** - Transfer, Convert, and Cycle operations with individual on/off control
- 🎛️ **Inline Editing** - Click to edit payloads and publish instantly
- 🚀 **Auto-connect** - Automatic MQTT connection on startup
- 💾 **Auto-save** - All settings saved automatically to localStorage
- 👁️ **Monitor Mode** - Disable functions to monitor only without processing messages

## 🚀 Quick Start

1. **Download** the `index.html` file
2. **Open** it in any modern web browser
3. **Configure** your MQTT broker settings
4. **Start** monitoring and controlling your MQTT topics

No installation, no dependencies, no build process required!

## 📋 Step-by-Step Setup

### 1. Initial Configuration

1. Open `index.html` in your web browser
2. Click the **Settings button** (⚙️) in the bottom-right corner
3. Go to the **"MQTT Connection"** tab
4. Enter your broker details:
   - **Host**: Your MQTT broker address (e.g., `broker.example.com`)
   - **Port**: WebSocket port (typically `8083`)
   - **Username**: Your MQTT username (if required)
   - **Password**: Your MQTT password (if required)
5. Click **"Connect"**

> 💡 The app will automatically reconnect on future visits if settings are saved.

### 2. Adding Topics

1. Click the **Add Topic button** (➕) in the top-right corner
2. Fill in the basic information:
   - **Label**: A friendly name for your topic
   - **Topic Name**: The MQTT topic path (e.g., `home/sensors/temperature`)

### 3. Configuring Payload Values

1. In the topic configuration dialog, click **"Add Payload Value"**
2. For each expected payload value:
   - **Payload Value**: The actual MQTT payload (e.g., `"on"`, `"23.5"`)
   - **Display Text**: Human-readable text (e.g., `"Light On"`, `"Room Temperature"`)
   - **Background Color**: Click the color preview to select from 28 colors
   - **Text Color**: Choose contrasting text color for readability
   - **Convert Value**: Value to send when using Convert function (optional)

### 4. Setting Up Functions

Choose one function type per topic:

#### 🔄 **Transfer Function**
- Forwards received payloads to another topic unchanged
- **Use case**: Bridging between different MQTT networks

#### 🔧 **Convert Function**
- Maps payload values to different output values
- **Use case**: Converting sensor readings to status messages
- **Example**: Temperature `"25.0"` → Status `"comfortable"`

#### 🔁 **Cycle Function**
- Cycles through predefined values when triggered
- **Use case**: Controlling multi-state devices (off → dim → bright → off)
- **Settings**:
  - **Next Payload**: Trigger to advance (e.g., `"next"`)
  - **Previous Payload**: Trigger to go back (e.g., `"prev"`)

### 5. Save and Monitor

1. Click **"Save"** to create the topic
2. Watch real-time updates in the main table
3. Rows will automatically change colors based on current payload values
4. **Toggle Functions**: Click the function button to enable/disable processing
   - **Green (ON)**: Messages are processed and functions execute
   - **Gray (OFF)**: Messages are monitored and displayed only, no processing

## 🎛️ Using the Interface

### Main Table Operations

- **🖱️ Click Payload Cell**: Edit and publish values instantly
- **🖱️ Right-click Row**: Access context menu for edit/delete/quick publish
- **🖱️ Drag Grip Handle**: Reorder topics by dragging the ≡ icon
- **🖱️ Click Function Button**: Toggle function execution on/off (monitoring continues)

### Floating Action Buttons

- **➕ Top-right**: Add new topic
- **🌓 Bottom-right-left**: Toggle dark/light theme
- **⚙️ Bottom-right**: Open settings
- **📁 Bottom-left**: GitHub repository link

### Status Indicators

- **🟢 Top-right corner**: Connected to MQTT broker
- **🔴 Top-right corner**: Disconnected from broker
- **Function Buttons**:
  - **Green (ON)**: Function active - processes and forwards messages
  - **Gray (OFF)**: Function inactive - monitors only, no processing

## 🔧 Configuration Examples

### Smart Light Control (Cycle Function)

```
Label: Living Room Light
Topic: home/living-room/light
Function: Cycle

Payload Values:
- "off" → Display: "Off" → Background: Light Red
- "dim" → Display: "Dimmed" → Background: Light Yellow
- "bright" → Display: "Bright" → Background: Light Green

Next Payload: "button_press"
Previous Payload: "long_press"
```

**Behavior**: Each button press cycles: Off → Dim → Bright → Off

**Function Control**: Toggle the function button to enable/disable cycling while continuing to monitor the current state.

### Temperature Monitor (Convert Function)

```
Label: Room Temperature
Topic: sensors/temperature
Function: Convert
Target Topic: home/status/comfort

Payload Values:
- "18.0" → Display: "Cold" → Convert: "heat_on"
- "22.0" → Display: "Perfect" → Convert: "auto"
- "28.0" → Display: "Hot" → Convert: "cool_on"

Default Value: "unknown"
```

**Behavior**: Temperature readings trigger HVAC commands

**Function Control**: Disable the function to monitor temperature without triggering HVAC commands.

### Sensor Bridge (Transfer Function)

```
Label: Motion Sensor
Topic: zigbee/motion/sensor1
Function: Transfer
Target Topic: home-assistant/binary_sensor/motion1

Payload Values:
- "true" → Display: "Motion Detected" → Background: Orange
- "false" → Display: "No Motion" → Background: Light Gray
```

**Behavior**: Forwards motion events between MQTT systems

**Function Control**: Turn off forwarding while still monitoring motion detection status.

## 📊 Data Management

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

## 🔗 MQTT Broker Setup

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

## 🌐 Browser Compatibility

- ✅ Chrome 88+
- ✅ Firefox 85+
- ✅ Safari 14+
- ✅ Edge 88+

**Required Features**:
- WebSocket support
- Drag & Drop API
- CSS Custom Properties
- localStorage API

## 🔧 Troubleshooting

### Connection Issues
- ✅ Check WebSocket port (usually 8083, not 1883)
- ✅ Verify broker supports WebSocket protocol
- ✅ Check firewall and network connectivity
- ✅ Try without username/password first

### Color Not Showing
- ✅ Ensure payload values match exactly (case-sensitive)
- ✅ Check that payload value is configured in topic settings
- ✅ Verify MQTT messages are being received

### Drag & Drop Not Working
- ✅ Use the grip handle (≡) icon on the left
- ✅ Ensure browser supports Drag & Drop API
- ✅ Try refreshing the page

### Theme Not Persisting
- ✅ Check that localStorage is enabled
- ✅ Verify browser allows local storage for file:// URLs
- ✅ Try opening in http://localhost if needed

## 🤝 Contributing

This is a single-file application for simplicity. When contributing:

1. **Preserve single-file architecture**
2. **Test in multiple browsers**
3. **Ensure both light and dark themes work**
4. **Maintain backward compatibility for data formats**
5. **Update CLAUDE.md documentation**

## 📄 License

MIT License - Feel free to use, modify, and distribute.

## 🔗 Links

- **Repository**: https://github.com/ytx/mqtt_p
- **Issues**: https://github.com/ytx/mqtt_p/issues
- **MQTT.js Documentation**: https://github.com/mqttjs/MQTT.js
- **Bootstrap Documentation**: https://getbootstrap.com/docs/5.3/

---

**Made with ❤️ for the MQTT community**