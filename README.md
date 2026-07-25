# Hot Tub Knob

ESPHome firmware for a [Waveshare ESP32-S3-Knob-Touch-LCD-1.8](https://www.waveshare.com/wiki/ESP32-S3-Knob-Touch-LCD-1.8)
(1.8" round knob display, capacitive touch, CNC metal case) that shows and
controls a hot tub heat pump exposed in Home Assistant as a `climate` entity.

## What it does

- Displays the current water temperature (from a dedicated HA sensor) and the
  heat pump's own `current_temperature` on a circular gauge.
- Shows the target setpoint, adjustable by turning the physical knob in
  **0.5°C steps**.
- Shows the heat pump's HVAC action (Heating / Idle / Off) and its mode
  (Auto / Off).
- Turning the knob calls `climate.set_temperature` (debounced 400ms so rapid
  turning doesn't spam Home Assistant).
- Tapping the touchscreen toggles the heat pump between `auto` and `off`
  (`climate.set_hvac_mode`).
- The knob's position is synced from Home Assistant once at boot, then left
  alone so local turns aren't fought/overwritten by HA state updates.
- Backlight dims/sleeps after an idle timeout and wakes on touch or knob turn.

## Hardware

This board is unusual in two ways that caused most of the setup friction:

1. **No BOOT button.** There's a single small switch and a recessed reset
   pinhole, but no exposed BOOT/GPIO0 button. It doesn't matter, though —
   flashing works fine via plain `esptool`/`esphome` over USB without any
   button gymnastics.
2. **Dual MCU, single bidirectional USB-C port.** The board has two chips:
   - **ESP32-S3** (QFN56, 8MB PSRAM) — drives the display, touch, and knob.
     This is the one this firmware targets.
   - **ESP32-U4WDH** (classic ESP32, embedded flash) — a secondary chip,
     likely used for audio/mic/vibration-motor duties in Waveshare's stock
     voice-assistant demos. Not used by this project.

   **Which chip your computer talks to depends on which way the USB-C plug is
   inserted.** If `esptool`/`esphome` can't find an ESP32-S3, unplug the
   cable, flip it 180°, and try again. On macOS the two chips enumerate as
   different serial devices (e.g. `/dev/cu.usbmodem*` for the S3's native
   USB-Serial/JTAG, `/dev/cu.usbserial-*` for the U4WDH via its UART bridge).

- Display: ST77916 (round, 360x360, driven via `mipi_spi` in ESPHome core)
- Touch: CST816
- Rotary encoder: plain quadrature on GPIO7/GPIO8, read by the custom
  `rotary_encoder_custom` component (int-only, no fractional steps — see
  below for how we get 0.5°C per detent anyway)

## Repo layout

- `hot-tub-knob.yaml` — the whole ESPHome config (kept as a single file for
  now; small enough not to need splitting into packages)
- `secrets.yaml` — Wi-Fi credentials, API encryption key, OTA password
  (**gitignored** — never commit this)
- `secrets.yaml.example` — placeholder template for the above, safe to
  commit; copy it to `secrets.yaml` and fill in real values
- `.ha_token.secret` — a Home Assistant long-lived access token, used only
  for ad-hoc debugging via `curl` (**gitignored**)

## Configuration

Copy the secrets template and fill in real values:

```bash
cp secrets.yaml.example secrets.yaml
```

Edit the `substitutions:` block at the top of `hot-tub-knob.yaml`:

| Substitution | Meaning |
|---|---|
| `climate_entity` | Your heat pump's `climate.*` entity ID |
| `water_temp_entity` | A dedicated physical water-temp `sensor.*` entity (separate from the climate entity's own `current_temperature` attribute) |
| `screensaver` | How long the backlight stays on after the last touch/turn |
| `setpoint_min` / `setpoint_max` | Allowed setpoint range, in actual °C |
| `setpoint_min_units` / `setpoint_max_units` | Same range but ×2 (see "Half-degree steps" below) — keep these in sync manually if you change the range |

`secrets.yaml` needs:

```yaml
wifi_ssid: "..."
wifi_password: "..."
ap_password: "..."          # fallback hotspot password if Wi-Fi fails
api_encryption_key: "..."   # 32 random bytes, base64-encoded
ota_password: "..."
```

Generate a valid encryption key with:

```bash
python3 -c "import secrets,base64;print(base64.b64encode(secrets.token_bytes(32)).decode())"
```

## Building and flashing

Using `uv`/`uvx` (no need to install ESPHome globally):

```bash
# Validate config only
uvx --from esphome esphome config hot-tub-knob.yaml

# Compile only
uvx --from esphome esphome compile hot-tub-knob.yaml

# Compile + flash over USB
uvx --from esphome esphome run hot-tub-knob.yaml --device /dev/cu.usbmodem2101

# Compile + flash over the network (once it's on Wi-Fi)
uvx --from esphome esphome run hot-tub-knob.yaml --device hot-tub-knob.local

# Watch logs
uvx --from esphome esphome logs hot-tub-knob.yaml --device /dev/cu.usbmodem2101
```

**Important:** don't run `esphome logs` and `esphome run`/`upload` against the
same serial port at the same time — the log process holding the port open
while a flash is in progress can corrupt the write and brick-loop the board
(recoverable: just reflash cleanly with nothing else touching the port).

If the port isn't found or reports "No serial data received", flip the
USB-C cable 180° (see Hardware section above) rather than looking for a BOOT
button.

## Home Assistant setup — the one non-obvious step

**You must explicitly allow this device to control Home Assistant, or every
knob turn and tap will silently do nothing.**

By default, Home Assistant's ESPHome integration blocks devices from calling
services (`climate.set_temperature`, `climate.set_hvac_mode`, etc.). The
device connects fine, sensors flow in from HA fine, but every outgoing
service call gets silently rejected — with **no error visible from the
device side**. The only trace is in Home Assistant's own log:

```
Service call climate.set_temperature: with data {...} rejected;
If you trust this device and want to allow access for it to make Home
Assistant service calls, you can enable this functionality in the options flow
```

To fix it:

1. In Home Assistant: **Settings → Devices & Services → Integrations**.
2. Find the **ESPHome** entries — there's one card per physical ESPHome
   device (not just one for "ESPHome" overall). Locate the one for this
   board (e.g. "Hot Tub Knob").
3. Click the **⋮ menu → Configure** on that specific card.
4. Check **"Allow the device to perform Home Assistant actions"**.
5. Click **Submit**.

If you have multiple ESPHome devices, make sure you're configuring the right
one — it's easy to toggle a different device's entry by mistake.

## Design notes / tradeoffs

- **Half-degree knob steps.** The `rotary_encoder_custom` component only
  counts whole integers (no built-in step/scale option). To get 0.5°C per
  detent, the encoder's `min_value`/`max_value` are set to the real range
  ×2 (e.g. 26–40°C becomes 52–80), and every place that reads or writes the
  encoder's raw value divides or multiplies by 2 to convert to/from actual
  °C. Look for `pending_setpoint_units` / `ha_target_units` in the config —
  "units" always means half-degree steps, never actual °C.
- **One-way boot sync, not continuous sync.** The knob's position is set
  from HA's current target temperature once, right after boot
  (`setpoint_synced` latches `true` after the first sync). After that,
  changes made directly in Home Assistant do **not** move the knob — this
  is intentional, so the display doesn't fight your hand mid-turn. If you
  want live two-way sync instead, remove the `setpoint_synced` guard in the
  `ha_target_temp` sensor's `on_value` handler (accepting that a HA-side
  change could visibly jump the knob's displayed value while you're not
  touching it).
- **Debounced setpoint commits.** `commit_setpoint` uses `mode: restart`
  with a 400ms delay, so a burst of knob turns collapses into a single
  `climate.set_temperature` call after you stop turning, rather than one
  call per detent.
- **`current_temp` vs `water_temp`.** The big number on the main gauge is
  driven by the climate entity's own `current_temperature` attribute, which
  may or may not be the same physical measurement as `water_temp_entity`'s
  standalone sensor. Both are shown rather than assuming they're identical.

## Troubleshooting

- **Nothing happens on knob turn or tap, no errors anywhere** → almost
  always the "Allow Home Assistant actions" permission above. Confirm by
  checking Home Assistant's log (Settings → System → Logs, or
  `/api/error_log` via the REST API) right after interacting with the
  device — a silent rejection shows up there, not in the ESPHome logs.
- **`esptool`/`esphome` can't find the ESP32-S3** → flip the USB-C cable.
- **Board boot-loops after a flash** ("no bootable app partitions") → almost
  always caused by something else holding the serial port during the write
  (e.g. a leftover `esphome logs` process). Kill any process using the port
  (`lsof /dev/cu.usbmodem*`) and reflash.
- **Want to see exactly what value/action is being sent** → temporarily add
  a `logger.log:` step before the `homeassistant.action:` in the relevant
  script, reflash, and watch `esphome logs`. This is how the permission
  issue above was actually diagnosed — the ESP side logs confirmed correct
  values were being sent well before we found the HA-side rejection.
