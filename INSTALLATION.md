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
