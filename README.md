# ☀️ Siseli Solar Cloud Home Assistant Bridge

[![Version](https://img.shields.io/badge/version-2.5.24--GPeV-blue.svg)](CHANGELOG.md)
[![HA Add-on](https://img.shields.io/badge/Home%20Assistant-Add--on-green.svg)](https://www.home-assistant.io/)

> **Acknowledgment:** This project is a fork of the excellent add-on originally created by [fadmaz/siseli-ha](https://github.com/fadmaz/siseli-ha), itself expanded from [yuraantonov11/siseli-ha](https://github.com/yuraantonov11/siseli-ha). Huge thanks to both authors!

Unleash your Siseli-compatible inverter into Home Assistant — **100% locally and instantly** — without relying on external clouds for HA data. The bridge intercepts MQTT traffic to the Siseli Cloud, decodes it locally, and creates sensors via MQTT Auto-Discovery.

> **🔒 Privacy Note:** Your Home Assistant instance intercepts the data for local use, but it simultaneously transparently forwards the traffic to the Siseli Cloud. This ensures your official mobile app continues to work flawlessly.

---

## 🆕 This Fork — Newer GPeV/PWDK/5yXy Inverter Family

**If you bought your Siseli-based inverter in 2024–2025, this fork is for you.**

Newer inverter firmware uses different block names (`GPeV`, `PWDK`, `5yXy`, `ucsX`, `6cuV`, `kbr1`, …) instead of the older `SUCV`, `WdRR`, `2ONL` blocks. The upstream add-on produces no sensors on these devices. This fork adds full decoder support for the new block family while keeping everything else (architecture, MQTT discovery, HA entity structure) identical to fadmaz's original.

### New block decoders added

| Block | Data decoded |
|-------|-------------|
| `GPeV` | Grid voltage, grid frequency |
| `5yXy` | Output V/Hz/VA/W/%, reactive power, uptime |
| `PWDK` | Battery voltage, charge current, series count, thresholds |
| `41Q7` | Battery capacity / SOC |
| `ucsX` | Discharge current, charging voltages, max charge, battery Ah |
| `O0MF` | Ambient temperature, charge/discharge current |
| `oh1V` | PV1 & PV2 voltages |
| `m50D` | PV temp, inverter temp, boost temp |
| `wIkm` | Working mode (UTI / Battery / Fault / SUB) |
| `dnwz` | Battery type |
| `6yCP` | Firmware version and build date |
| `6cuV` | Energy counters — PV today/total, grid import, load, battery charge |
| `kbr1` | Output priority, fault count, AC OVP/UVP limits, mains power, efficiency |
| `yqs1` | Grid V/Hz fallback, PV power, inverter lifetime hours |
| `RrrX` | Output current |
| `6NWg` | Battery hard OVP voltage |
| `zwWS` | Fan 1 speed |
| `NRwZ` / `Yzqw` | Fan 2 speed |
| `EJaH` | Lifetime event count |

---

## 🚀 Quick Setup

### Step 1: Prepare Home Assistant

Ensure the official **Mosquitto Broker** add-on is installed and configured:

1. Go to **Settings -> Add-ons -> Add-on Store**.
2. Install **Mosquitto Broker**.
3. Start it and ensure you have an MQTT user created.

### Step 2: Add This Repository

1. Copy this repository URL: `https://github.com/Mali-prod/siseli-ha`
2. In Home Assistant, go to **Settings -> Add-ons -> Add-on Store**.
3. Click the three dots in the top right -> **Repositories**.
4. Paste the URL and click **Add**.

### Step 3: Install & Configure

1. Find **Siseli Inverter Bridge** in the store and click **Install**.
2. Go to the **Configuration** tab.
3. Fill in the required fields:
   - **INVERTER_IP**: The local IP of your inverter (e.g., `192.168.30.31`).
   - **ROUTER_IP**: The local IP of your router (e.g., `192.168.1.1`).
   - **AUTO_INTERCEPT**: Keep `true` to use ARP Spoofing (automatic interception).

- Optional parallel-system fields:
  - **INVERTER_COUNT**: Number of parallel inverters.
  - **BATTERY_COUNT**: Number of batteries in the bank.
  - **BATTERY_CAPACITY_PER_BATTERY_AH**: Capacity per battery in Ah.

4. Go to the **Info** tab, enable **Watchdog**, and click **Start**.

### Parallel Inverter/Battery Scaling

When using multiple inverters in parallel, main summary power sensors are scaled with:

`c_scaled_power = raw_power * INVERTER_COUNT`

This is applied to:

- `c_generation_power_w`
- `c_mains_power_w`
- `c_load_w`

For battery-bank visibility, the bridge also publishes calculated BMS capacity helper sensors on the Main device:

- `c_bms_total_capacity_ah`

All calculated sensors use the `c_` prefix so they are easy to distinguish from raw inverter values.

---

## 🛠 How it Works (Technical)

The add-on uses multiple methods for traffic interception. For the inverter to start sending data to this add-on, it needs to "think" it is sending it to the Siseli cloud:

### Option A: ARP Spoofing (Auto-Intercept, Recommended)

With `AUTO_INTERCEPT` enabled, the add-on tricks the inverter into sending its data to Home Assistant instead of the router. The bridge parses the data and transparently forwards it to the Siseli cloud.

> **⚠️ WARNING:** You are using ARP spoofing, which is a sensitive network technique. It can trigger security alerts on advanced network setups or enterprise routers (like UniFi or pfSense).

### Option B: DNS Configuration

Configure your router so that requests to the Siseli cloud domain resolve to the local IP address of your Home Assistant.

### Option C: Manual Redirect / Static Route (Legacy)

Create a static route on your router that redirects traffic for the target IP `8.212.18.157` to the IP of your Home Assistant.

---

## 📊 Available Sensors

This bridge dynamically extracts and exposes **almost every single sensor and data point available in the official Siseli app** (100+ entities) directly into Home Assistant via MQTT Auto-Discovery.

Sensors are split across multiple Home Assistant devices:

- **Battery**
- **BMS**
- **Grid**
- **Load**
- **PV**
- **Diagnostics** (for non-functional or fallback settings)

The exposed data includes:

- **🔋 Battery & BMS Status**
  - Overall Voltage, Capacity (%), Charge/Discharge Currents, Battery Type
  - Nominal Capacity (Ah), Shutdown/Low-Alarm/Hard-OVP voltages
  - Remaining Capacity (Ah), Min/Max Cell Voltages, individual cell voltages (1–16)
- **⚡ Grid & Load Status**
  - AC Input Voltage & Mains Frequency, AC OVP/UVP limits
  - Active Load (W), Reactive Power (var), Apparent Power (VA), Output Voltage/Frequency/Current, Load Percentage
  - Energy counters: grid import today/total, load today/total (kWh)
- **☀️ PV Panel Status**
  - Daily and Total Electricity Generation (kWh)
  - PV, PV1 & PV2 Voltages, PV Temperature
  - PV Power and Generation Power
- **⚙️ Advanced Device Settings**
  - Working Mode (UTI, Battery, SUB, Fault), Charging Priority, Output Priority
  - Fan Speeds, Charging Voltages (Float/Bulk/Equalization)
  - Firmware Version and Build Date
  - Inverter Lifetime Hours, Uptime, Efficiency
  - Ambient Temperature, Inverter Temperature, Boost Temperature

---

## ❓ Compatibility

### This fork (GPeV/PWDK/5yXy blocks)
Tested on newer Siseli-based inverters purchased in 2024–2025 that send `GPeV`, `PWDK`, `5yXy`, `ucsX`, `6cuV` blocks.

### Original add-on (fadmaz/siseli-ha)
If your inverter sends `SUCV`, `WdRR`, `2ONL` blocks (older firmware), use the upstream repo at [fadmaz/siseli-ha](https://github.com/fadmaz/siseli-ha).

---

## 🧪 Troubleshooting

**No data appearing in Home Assistant?**

- **Check MQTT Connection:** Ensure your Mosquitto broker is running and the add-on logs show a successful connection.
- **Verify Inverter IP:** Double-check that `INVERTER_IP` and `ROUTER_IP` are exactly correct in the configuration.
- **Disable AUTO_INTERCEPT:** If ARP spoofing is blocked by your router, set `AUTO_INTERCEPT` to `false` and try the **DNS Configuration** or **Static Route** methods instead.
- **Wrong fork?** Check your add-on logs for block names. If you see `SUCV` or `WdRR`, use [fadmaz/siseli-ha](https://github.com/fadmaz/siseli-ha) instead.

**After upgrading, I see duplicate/stale entities in Home Assistant**

- Remove old retained discovery payloads from your broker, then restart the add-on:

```bash
mosquitto_pub -h core-mosquitto -t 'homeassistant/sensor/siseli_inverter_1/+/config' -n -r
```

---

## 📄 License

MIT License. Free to use and modify.
