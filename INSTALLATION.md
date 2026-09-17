# Installation Guide

This guide takes you from a bare Wemos D1 Mini to a working, Home Assistant–connected pump controller.

## 1. Wire it up

Follow the [wiring table in README.md](README.md#-wiring-guide). Double-check:
- The PZEM-004T is on the **load side** correctly (line/neutral through the current-sense coil, per the module's silkscreen).
- The relay module's Start/Stop dry contacts are wired **in parallel** with your existing panel's Start/Stop pushbuttons — this project pulses relays exactly like a human pressing those buttons, it does not replace your contactor.
- The Wemos D1 Mini is powered from an **isolated** 5V supply (e.g. HLK-PM01), not a USB charger sharing a ground with mains wiring, unless that charger is properly isolated.

> ⚠️ Work with the main breaker OFF. If you are not confident with 230V AC wiring, hire a licensed electrician.

## 2. Install ESPHome

Pick one:

- **Home Assistant users (recommended):** Settings → Add-ons → Add-on Store → search "ESPHome" → Install → Start. Open the ESPHome panel from your sidebar.
- **Standalone (any computer):**
  ```bash
  pip install esphome
  esphome dashboard config/
  ```
  Then open `http://localhost:6052`.

Both give you the same ESPHome Dashboard, which handles secrets, building, and flashing for you.

## 3. Get your files

In your ESPHome `config/` folder (or the Home Assistant `esphome` config directory):

1. Copy [`secrets.yaml.example`](secrets.yaml.example) → `secrets.yaml` and fill in:
   - `wifi_ssid` / `wifi_password` — your network.
   - `api_encryption_key` — generate one from the dashboard's "Secrets" helper, or run `openssl rand -base64 32`.
   - `ota_password` — any strong password.
2. Copy [`smart-waterpump.yaml`](smart-waterpump.yaml) into the dashboard and rename it to something meaningful, e.g. `shop-waterpump.yaml`.
3. Edit the two substitutions at the top (`name`, `friendly_name`) to match your device.

You do **not** need to copy any of the sensor/relay/dry-run logic — that's pulled automatically from this repository's `packages/pump-controller.yaml` at compile time via the `packages:` key.

## 4. First flash (via USB)

The very first flash must be over USB (ESPHome can't push OTA firmware to a device that has never run ESPHome before).

- In the ESPHome Dashboard, click **Install** next to your device → **Plug into this computer** → pick the serial port → wait for it to compile and flash.
- Or from the CLI: `esphome run shop-waterpump.yaml` and select the USB port when prompted.

Watch the log — you should see the device connect to WiFi, then the PZEM readings start appearing every couple of seconds.

## 5. Add to Home Assistant

If you're running the ESPHome add-on inside Home Assistant, the device is auto-discovered — go to **Settings → Devices & Services**, you should see a notification to add it. Otherwise: **Settings → Devices & Services → Add Integration → ESPHome**, enter the device's IP and the `api_encryption_key` from your secrets.

## 6. Add the dashboard card

Copy [`home-assistant/dashboard.yaml`](home-assistant/dashboard.yaml) into a new **Manual** card in your Lovelace dashboard, replacing every `shop_waterpump` with your own device's entity prefix (find it under **Settings → Devices & Services → Entities**, filter by your device).

## 7. Future updates

From now on, edits to the pump logic happen by bumping the `@main` ref in your device file's `packages:` line (or pinning to a tag/commit for stability) and re-flashing — which happens **over WiFi**, no more USB needed:

```bash
esphome run shop-waterpump.yaml   # OTA if the device is already online
```

---

## Remote Control Panel Setup (optional)

This walks through building the physical Green/Red button panel from scratch and connecting it to a pump controller you've already set up above. It doesn't wire to the pump directly — it calls the pump's Start/Stop switches **through Home Assistant**, so the pump controller must already be added to Home Assistant first.

### 1. Wire it up

Follow the [wiring diagram in README.md](README.md#-optional-physical-remote-control-panel): Green button → D1 (ESP8266) or GPIO27 (ESP32), Red button → D2 / GPIO26, WiFi LED → D5 / GPIO25, Status LED → D6 / GPIO33, all other legs to GND. Buttons need no resistor (internal pull-up); LEDs need a ~220–330Ω resistor in series.

### 2. Add secrets

Add to your `secrets.yaml` (use distinct `remote_panel_*` names so they never collide with your pump controller's own WiFi/API secrets):

```yaml
remote_panel_wifi_ssid: "YourWiFiName"
remote_panel_wifi_password: "YourWiFiPassword"
remote_panel_wifi_ssid_backup: "YourBackupWiFiName"      # optional
remote_panel_wifi_password_backup: "YourBackupPassword"  # optional
remote_panel_api_encryption_key: "PASTE_A_DIFFERENT_BASE64_32_BYTE_KEY_HERE"
remote_panel_ota_password: "choose-a-different-strong-ota-password"
```

### 3. Create the device file

Copy [`remote-control-panel.yaml`](remote-control-panel.yaml) (ESP8266/D1 Mini) or [`remote-control-panel-esp32.yaml`](remote-control-panel-esp32.yaml) (ESP32) into your dashboard. If your pump device isn't named `shop-waterpump`, uncomment and edit `start_switch_entity_id` / `stop_switch_entity_id` in the substitutions to match your actual switch entity IDs.

> If you update the package on GitHub and re-flash within 24 hours, ESPHome may serve a stale cached copy. Switch the `packages:` line to the long form with `refresh: 0s` to force a fresh pull — see the example device files for the exact syntax.

### 4. First flash (via USB)

Same as the pump controller: **Install → Plug into this computer** for the first flash, OTA from then on.

### 5. Add the panel to Home Assistant

**Settings → Devices & Services** — it should auto-discover as "Pump Remote Panel." If not, **Add Integration → ESPHome**, enter its IP and the `remote_panel_api_encryption_key`.

### 6. Enable Home Assistant actions — the step it's easy to miss

The buttons work by calling a Home Assistant service (`switch.turn_on`) from the ESP, and **Home Assistant blocks this by default** for security. Without this step, button presses register on the device (you'll see the "Start Button"/"Stop Button" sensors flicker) but nothing downstream happens — no error, just silence.

1. **Settings → Devices & Services → ESPHome**
2. Find the **Pump Remote Panel** entry (not the pump controller's) and open **Configure** (⚙️ / three-dot menu)
3. Enable **"Allow the device to perform Home Assistant actions"**
4. Save

### 7. Add a status/diagnostic card (optional but handy while testing)

Since the physical LEDs may not be wired yet, this lets you watch connectivity and button presses live from the dashboard:

```yaml
type: vertical-stack
cards:
  - type: heading
    heading: "🎛️ REMOTE PANEL STATUS"
    heading_style: title
  - type: grid
    columns: 2
    square: false
    cards:
      - type: tile
        entity: binary_sensor.pump_remote_panel_wifi_connected
        name: WiFi Link
        icon: mdi:wifi
      - type: tile
        entity: binary_sensor.pump_remote_panel_home_assistant_connected
        name: HA Link
        icon: mdi:home-assistant
  - type: grid
    columns: 2
    square: false
    cards:
      - type: tile
        entity: binary_sensor.pump_remote_panel_start_button
        name: Start Button (live)
        icon: mdi:gesture-tap-button
      - type: tile
        entity: binary_sensor.pump_remote_panel_stop_button
        name: Stop Button (live)
        icon: mdi:gesture-tap-button
```

### 8. Test it

Short the Start pin to GND (or press the physical button once wired) — "Start Button (live)" should flicker on, and `switch.shop_waterpump_start_pump` should actually toggle. If the button flickers but the pump switch never moves, go back to Step 6.

---

## Advanced overrides

Because `packages:` merges dictionaries with your device file taking precedence, you can override or extend anything the package defines by redeclaring that top-level key.

**Add a second (backup) WiFi network:**

```yaml
wifi:
  networks:
    - ssid: !secret wifi_ssid
      password: !secret wifi_password
      priority: 100
    - ssid: !secret wifi_ssid_backup
      password: !secret wifi_password_backup
      priority: 50
  ap:
    ssid: "${friendly_name} Fallback"
```
(add `wifi_ssid_backup` / `wifi_password_backup` to your `secrets.yaml`)

**Pin a static IP:**

```yaml
wifi:
  networks:
    - ssid: !secret wifi_ssid
      password: !secret wifi_password
  manual_ip:
    static_ip: 192.168.1.50
    gateway: 192.168.1.1
    subnet: 255.255.255.0
```

**Run two pumps on the same PZEM bus:** PZEM-004T supports addressable multi-device buses; wire a second PZEM in parallel on the same RX/TX lines and instantiate a second `pzemac` sensor block with `address: 2` in your device file (see the [ESPHome PZEM-AC docs](https://esphome.io/components/sensor/pzemac)).

---

## Troubleshooting

| Symptom | Fix |
| :--- | :--- |
| No PZEM readings / sensor stuck at `NaN` | Confirm `logger: baud_rate: 0` (UART0 must be free), and that RX/TX aren't swapped — try flipping `pzem_rx_pin`/`pzem_tx_pin`. |
| Device won't join WiFi | ESP8266 requires `min_auth_mode: WPA2` compatible networks (already set); confirm your router isn't WPA3-only, and that the SSID/password in `secrets.yaml` are correct. |
| Device unreachable after WiFi change | Connect to the `<friendly_name> Fallback` hotspot it broadcasts, a captive portal will let you reconfigure WiFi without re-flashing. |
| Dry-run trips too early/late | Tune the `dry_run_amps` (trip threshold) and `running_amps` (running-detection threshold) substitutions to match your motor's actual no-load vs loaded current draw — check the "Pump Amps" sensor history in Home Assistant to pick good values. |
| Relay clicks on power-up | Confirm `early_pin_init: false` and `restore_mode: RESTORE_DEFAULT_OFF` are present (they are, by default, in the package) — these prevent boot-time relay flicker. |
| `esphome config` fails in CI / locally | Make sure `secrets.yaml` exists with **all four** required keys (`wifi_ssid`, `wifi_password`, `api_encryption_key`, `ota_password`). |
| `[uart_id] is an invalid option for [sensor.pzemac]` or similar | Your ESPHome install predates the `pzemac` → `modbus` migration. Update ESPHome (`pip install --upgrade esphome`, or update the ESPHome add-on) to at least the version in the badge at the top of [README.md](README.md). |
| `packages/....yaml does not exist in repository` right after pushing a change | Stale package cache — ESPHome only re-fetches a `github://` package once every 24 hours by default. Use the long-form `packages:` syntax with `refresh: 0s` (see the remote panel device files for the exact syntax) to force a fresh pull. |
| Remote panel button flickers the "Start/Stop Button" sensor, but the pump switch never moves | Home Assistant blocks ESPHome devices from calling HA services by default. Go to **Settings → Devices & Services → ESPHome → (Pump Remote Panel) → Configure** and enable **"Allow the device to perform Home Assistant actions."** See [Remote Control Panel Setup § 6](#6-enable-home-assistant-actions--the-step-its-easy-to-miss). |
