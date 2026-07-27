# CLAUDE.md

Guidance for Claude Code when working in this repo.

## What this is

ESPHome firmware for a Waveshare ESP32-S3-Knob-Touch-LCD-1.8 round knob
display that shows/controls a hot tub heat pump via Home Assistant. See
`README.md` for full details — this file only covers things an agent needs
to know before touching the code or hardware.

## Key files

- `hot-tub-knob.yaml` — the entire firmware config
- `secrets.yaml` — real credentials, **gitignored**, never read/print/commit
  its contents unnecessarily
- `secrets.yaml.example` — placeholder template, safe to edit/commit
- `.ha_token.secret` — HA long-lived access token for debugging via curl,
  **gitignored**, treat as a live credential

## Toolchain

Use `uvx --from esphome esphome <command>` — don't assume `esphome` is
installed globally.

```bash
uvx --from esphome esphome config hot-tub-knob.yaml   # validate
uvx --from esphome esphome compile hot-tub-knob.yaml   # compile only
uvx --from esphome esphome run hot-tub-knob.yaml --device <port-or-host>
uvx --from esphome esphome logs hot-tub-knob.yaml --device <port-or-host>
```

`<port-or-host>` is either a serial device (`/dev/cu.usbmodem*`) or
`hot-tub-knob.local` for OTA.

## Hardware gotchas (don't rediscover these the hard way)

- **No BOOT button on this board.** Don't go looking for one or try
  GPIO0/manual-bootloader tricks — plain `esptool`/`esphome` flashing works
  fine over USB as-is.
- **Bidirectional USB-C, two chips.** The board has an ESP32-S3 (what this
  firmware targets — display/touch/knob) and a secondary ESP32-U4WDH.
  Which one the computer sees depends on **which way the USB-C cable is
  plugged in**. If `esptool`/`esphome` can't find an ESP32-S3, tell the user
  to flip the cable — don't assume the board is broken or unflashable.
- **Never run `esphome logs` and `esphome run`/`upload` against the same
  serial port concurrently.** A log process holding the port open during a
  flash can corrupt the write and boot-loop the device. Kill any lingering
  log process (`pkill -f "esphome logs"`) before flashing over USB.

## Home Assistant permission gotcha

Every `homeassistant.action:` call (`climate.set_temperature`,
`climate.set_hvac_mode`, etc.) is **silently dropped by Home Assistant** by
default — no error on the ESPHome/device side at all. If knob turns or taps
appear to do nothing even though logs show the ESP32 sending the right
values, this is almost certainly the cause. The fix lives entirely on the
HA side (Settings → Devices & Services → the specific ESPHome device's card
→ ⋮ → Configure → "Allow the device to perform Home Assistant actions"),
not in this repo's YAML. Check HA's own log/`/api/error_log` for a line
like `Service call ... rejected; ... enable this functionality in the
options flow` before assuming the firmware config is wrong.

## Conventions used in this config

- Any global/variable with `_units` in its name is in **half-degree units**
  (raw encoder value = actual °C × 2), because `rotary_encoder_custom` only
  counts whole integers. Convert with `/ 2.0f` or `* 2.0f` at the boundary —
  never mix units and actual °C without converting.
- The knob's position syncs from Home Assistant **once, at boot only**
  (guarded by the `setpoint_synced` global). This is intentional, not a bug
  to "fix" — see README's Design notes section before changing it.
- Debounce pattern: any script that shouldn't fire on every single physical
  tick (e.g. `commit_setpoint`) uses `mode: restart` plus a short `delay:`.
  Follow this pattern for any new debounced action rather than inventing a
  new one.
- Setpoint reads/writes go through `target_temp_entity`
  (`number.hot_tub_heat_pump_target_temp`, `number.set_value`), never
  `climate_entity`/`climate.set_temperature`. The heat pump has an
  independent setpoint register per operating mode, and the `climate`
  entity is hardwired to the Auto-mode register only — writing via it
  silently no-ops in Heating/Cooling. `climate_entity` is still used for
  `current_temperature` (inlet temp) and on/off (`climate.set_hvac_mode`).

## When testing changes

Prefer USB flashing + `esphome logs` for anything you need to debug live
(you can see boot output and instrumented `logger.log:` lines). OTA
(`hot-tub-knob.local`) is fine for routine updates once you're confident in
a change. If Home Assistant behavior is in question, use the HA REST API
with `.ha_token.secret` to inspect entity state/attributes or call services
directly — this was how the permission issue above was actually isolated
(direct `curl` calls to `climate.set_temperature` worked while the same
call via the device didn't, proving the break was HA-side, not firmware).
