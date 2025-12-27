# Marstek Venus Bridge

**Local MQTT Bridge for Marstek Venus A Battery Systems**

Monitor your **Marstek Venus A** battery system in real-time using MQTT. This bridge connects to your Venus A energy storage system via the local UDP API and publishes battery data (SOC, temperature, power, energy statistics) to an MQTT broker running on a Raspberry Pi.

Perfect for **home automation**, **Home Assistant integration**, **solar energy monitoring**, and **smart home** setups. Works with any MQTT client on Android, iOS, web, or command-line tools.

---

## 🎯 Problem

The Marstek Venus A does not send real-time MQTT data to the cloud. Client apps must constantly poll the Cloud REST API (30s interval), which:
- Drains battery
- Generates unnecessary network traffic
- Has higher latency in home network (Venus A is on same LAN)

## 💡 Solution

A Raspberry Pi bridge in your home network:
- Polls Venus A locally via **UDP API** every 60 seconds
- Publishes data to local MQTT broker
- Client apps subscribe to MQTT (push instead of pull)
- Optional secure remote access via TLS (port forwarding)

---

## 🏗️ Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                         HOME NETWORK                            │
│                                                                 │
│  ┌──────────────┐         ┌─────────────────────────┐         │
│  │  Venus A     │◄────────│   Raspberry Pi          │         │
│  │              │  UDP    │                         │         │
│  │  Port 30000  │  JSON   │  ┌──────────────────┐  │         │
│  │  (API ON)    │  -RPC   │  │ Venus Poller     │  │         │
│  └──────────────┘  60s    │  │ (Python + UDP)   │  │         │
│                           │  └────────┬─────────┘  │         │
│                           │           │ publish     │         │
│                           │           ▼             │         │
│                           │  ┌──────────────────┐  │         │
│                           │  │ Mosquitto MQTT   │  │         │
│                           │  │ Broker :1883     │  │         │
│                           │  └────────┬─────────┘  │         │
│                           └───────────┼────────────┘         │
│                                       │                       │
└───────────────────────────────────────┼───────────────────────┘
                                        │ subscribe
                            ┌───────────┴───────────┐
                            │                       │
                    ┌───────▼────────┐      ┌──────▼───────┐
                    │  Mobile App    │      │  Tablet      │
                    │  (on WiFi)     │      │  (on WiFi)   │
                    │                │      │              │
                    │  MQTT Client   │      │  MQTT Client │
                    └────────────────┘      └──────────────┘
```

---

## 🚀 Features

### ✅ Implemented
- **UDP JSON-RPC Communication** with Venus A Local API (Marstek Device Open API Rev 1.0)
- **Mosquitto MQTT Broker** running on Raspberry Pi
- **Python Venus Poller** with 60-second interval
- **Docker Compose** setup for easy deployment
- **Real-time Data Publishing**:
  - Battery: SOC, temperature, capacity, charge/discharge permissions
  - Energy System: PV power, grid power, battery power, energy statistics
- **Device Control Functions**:
  - Manual Mode (scheduled power control)
  - Passive Mode (immediate power control with countdown)
  - Auto Mode (automatic operation)
  - AI Mode (AI-driven optimization)
- **Error Handling** and reconnection logic
- **Debug Logging** for troubleshooting

### 📋 Planned
- 🔄 Web dashboard for monitoring (optional)
- 🔄 Historical data logging (InfluxDB/Grafana)
- 🔄 Home Assistant auto-discovery

---

## 📋 Components

### 1. Raspberry Pi (Home Network)

**Hardware:**
- Raspberry Pi 3/4/5 (or any Linux server)
- Network connection (LAN or WiFi)

**Software:**
- Docker + Docker Compose
- Python 3.9+
- Mosquitto MQTT Broker

### 2. Venus Poller (Python)

**Function:**
- Polls Venus A via **UDP JSON-RPC API** (Port 30000, every 60s)
- Calls `Bat.GetStatus` and `ES.GetStatus` methods
- Publishes combined data to MQTT
- Automatic retry on timeout/errors
- Runs as Docker container with auto-restart

**Communication Protocol:**
- **Protocol:** UDP JSON-RPC (Marstek Device Open API Rev 1.0)
- **Port:** 30000 (configurable)
- **Request Format:**
  ```json
  {
    "id": 1,
    "method": "Bat.GetStatus",
    "params": {"id": 0}
  }
  ```

**MQTT Topic Schema:**
```
marstek/venus/{mac}/data        # Real-time data
marstek/venus/{mac}/control     # Control commands (future)
```

**Payload Example (Real Data):**
```json
{
  "timestamp": "2025-12-27T09:40:48.559450Z",
  "soc": 90,
  "battery_temp": 4.0,
  "battery_capacity": 1886.0,
  "rated_capacity": 2080.0,
  "charging_allowed": true,
  "discharging_allowed": true,
  "pv_power": 0,
  "grid_power": 0,
  "offgrid_power": 0,
  "battery_power": 0,
  "total_pv_energy": 0,
  "total_grid_output": 7444,
  "total_grid_input": 9682,
  "total_load_energy": 0
}
```

### 3. Client Applications (Platform-Independent)

Any MQTT client can subscribe to the bridge:

**Supported Platforms:**
- Android / iOS
- Web browsers
- Home automation systems (Home Assistant, Node-RED, etc.)
- Command line tools (mosquitto_sub, MQTT Explorer)

**Local Access (in home network):**
- Connect to Raspberry Pi MQTT broker (port 1883, unencrypted)
- Subscribe to `marstek/venus/+/data`
- Push-based updates, no polling overhead
- Low latency (<1s)

**Remote Access (away from home):**
- Connect via TLS-encrypted MQTT (port 8883)
- Requires one-time port forwarding setup in router
- Bank-level security (TLS 1.3, AES-256)
- Trust-on-First-Use (TOFU) certificate pinning

---

## 📊 Benefits

| Criterion | Before (REST Polling) | After (MQTT Bridge) |
|-----------|----------------------|----------------------|
| **Latency (Home)** | 0-30s | <1s |
| **Power Consumption (Client)** | High (constant HTTP) | Low (push) |
| **Network Traffic** | 120 requests/h | ~1 MQTT subscribe |
| **Remote Access** | ✅ Via Cloud API | ✅ Via TLS (port forward) |
| **Offline Capability** | ❌ No | ✅ Last data cached |

---

## 🔧 Installation

### Prerequisites

1. **Venus A Configuration:**
   - Connected to local network (WiFi or Ethernet)
   - Static IP address recommended (e.g. 192.168.1.100)
   - **Open API enabled** via Marstek App (Settings → Open API)
   - UDP Port 30000 (default, configurable in app)

2. **Raspberry Pi Setup:**
   - Raspberry Pi 3/4/5 or any Linux server
   - Docker + Docker Compose installed
   - Same network as Venus A
   - At least 1GB RAM, 8GB storage

3. **Client Application (Optional):**
   - MQTT client library (platform-dependent)
   - Access to home WiFi network
   - For mobile: Background service permission

### Quick Start

```bash
# Clone repository
git clone https://github.com/IvanKablar/marstek-venus-bridge.git
cd marstek-venus-bridge

# Configure Venus A IP and MAC address
cp config.example.json config.json
nano config.json  # Update "ip" and "mac" fields

# Start Docker stack
docker-compose up -d

# Monitor logs (should show real data every 60s)
docker-compose logs -f venus-poller

# Test MQTT subscription
mosquitto_sub -h localhost -p 1883 -t "marstek/venus/#" -v
```

**Expected Output:**
```
marstek/venus/aabbccddeeff/data {
  "soc": 90,
  "battery_temp": 4.0,
  "battery_capacity": 1886.0,
  ...
}
```

---

## 📁 Project Structure

```
marstek-venus-bridge/
├── README.md                 # Project documentation
├── INSTALLATION.md           # Detailed installation guide
├── LICENSE                   # MIT License
├── docker-compose.yml        # Docker orchestration
├── config.example.json       # Configuration template
├── config.json               # Actual config (git-ignored)
├── .gitignore
│
├── venus-poller/            # Python UDP→MQTT bridge
│   ├── Dockerfile
│   ├── requirements.txt     # Python dependencies
│   ├── poller.py            # Main polling loop
│   └── venus_api.py         # UDP JSON-RPC client
│
└── mqtt-broker/             # Mosquitto configuration
    └── mosquitto.conf       # MQTT broker settings
```

---

## 🛠️ Technology Stack

- **Backend:** Python 3.11
- **Communication:** UDP JSON-RPC (Marstek Device Open API Rev 1.0)
- **MQTT Broker:** Eclipse Mosquitto 2.0.22
- **Container:** Docker / Docker Compose
- **Protocol:** MQTT 3.1.1

---

## 📝 Development Status

### ✅ Completed
- [x] UDP JSON-RPC client implementation
- [x] Python poller with MQTT publishing
- [x] Docker Compose setup
- [x] Battery status monitoring (Bat.GetStatus)
- [x] Energy system monitoring (ES.GetStatus)
- [x] Control functions (ES.SetMode: Manual, Passive, Auto, AI)
- [x] Error handling and retry logic
- [x] Documentation (README, INSTALLATION)

### 🔄 In Progress
- [ ] MQTT client integration (platform-specific)
- [ ] Auto-discovery for Venus A in network
- [ ] Automated testing

### 💡 Future Ideas
- [ ] Web dashboard for monitoring
- [ ] Historical data logging (InfluxDB/Grafana)
- [ ] Home Assistant integration
- [ ] Energy cost calculation
- [ ] Smart charging/discharging automation

---

## 🎛️ Device Control API

The bridge supports full control of Venus A operating modes:

### Manual Mode
Schedule-based power control with time windows:
```python
venus_client.set_manual_mode(
    power=100,              # Watts (+ = charge, - = discharge)
    start_time="08:00",
    end_time="20:00",
    week_set=127            # Bitmask: 127 = all days
)
```

### Passive Mode
Immediate power control with countdown:
```python
venus_client.set_passive_mode(
    power=-200,             # Discharge 200W
    countdown=300           # For 5 minutes
)
```

### Auto / AI Mode
Automatic system-controlled operation:
```python
venus_client.set_auto_mode()    # Auto mode
venus_client.set_ai_mode()      # AI optimization
```

### Query Current Mode
```python
mode_info = venus_client.get_mode()
# Returns: {"mode": "Passive", "ongrid_power": 100, "bat_soc": 90}
```

---

## 🤝 Related Projects

- [hame-relay](https://github.com/tomquist/hame-relay) - MQTT relay for Venus E/Jupiter

---

## 📄 License

MIT License - see [LICENSE](LICENSE) file

---

## 👤 Author

Ivan Kablar

---

## 💖 Support

This project is developed and maintained in my free time. If it helps you monitor your battery system and you'd like to support continued development:

[![PayPal](https://img.shields.io/badge/PayPal-Donate-00457C?style=flat&logo=paypal&logoColor=white)](https://paypal.me/ivankablar)

Your support helps with ongoing maintenance, new features, and hardware testing.

---

## 🙏 Acknowledgments

- [Eclipse Mosquitto](https://mosquitto.org/) - MQTT broker
- [hame-relay](https://github.com/tomquist/hame-relay) - Inspiration for MQTT integration
