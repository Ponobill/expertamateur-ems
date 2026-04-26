# CYD Thermostat — Onboarding Guide

This doc covers the bench-to-wall workflow for the ExpertAmateur ESP32-2432S028R ("Cheap Yellow Display" / CYD) radiant thermostat. Use this any time you're flashing a fresh board or recovering from a confused state.

The configuration file referenced throughout is `radiant-thermostat-zone1-cyd.yaml`.

---

## Hardware checklist

Per zone:

- 1× ESP32-2432S028R CYD board
- 1× DHT22 / AM2302 temperature + humidity sensor
- 1× single-channel optoisolated relay module (3.3V trigger, active LOW)
- 1× LM2596-type buck converter calibrated to 5.0V output
- 24VAC wall feed from the existing thermostat cable (R + C wires)
- Hookup wire, dupont jumpers for prototype, soldered leads for wall install

For prototyping you also want:

- USB-A to micro-USB or USB-C data cable (the 2USB CYD revision has both ports; either works for power and programming)
- Multimeter
- A small breadboard or perfboard to mount the relay and sensor
- Molex PicoBlade 1.25mm 4-pin cables for CN1 / P3 (these often don't ship with the board — order separately if missing)

---

## CYD connector reference

The CYD has two USB ports (top edge) and three or four small Molex PicoBlade connectors around the edges. Pin labels are silkscreened on the back of the board in white-on-yellow, which is genuinely hard to read — keep this section nearby during install.

Connector locations are described from the perspective of the board lying display-down with USB ports at the top.

Note: several manufacturers produce this board to the same reference design (Sunton, Guition, etc.). Silkscreen logos and minor cosmetic markings vary — the board may be labeled "ESP32-2432S028" without the trailing R, or carry a Guition logo instead of Sunton. Pinout and ESPHome config are identical across all variants.

### USB1 / USB2 (top edge)

USB1 is micro-USB (older revisions only have this). USB2 is USB-C (added on the 2USB revision). Either port handles power and programming. Use whichever cable you have. The CH340 USB-to-serial chip handles both.

### P1 — Serial / power injection (top right, 4-pin PicoBlade)

Pin labels on silkscreen, top to bottom: `VIN, TX, RX, GND`

| Pin | Signal | Use |
| --- | --- | --- |
| VIN | 5V | 5V power input/output — this is where you wire your buck supply |
| TX | GPIO1 | Serial TX (shared with CH340 USB-to-UART) |
| RX | GPIO3 | Serial RX (shared with CH340 USB-to-UART) |
| GND | Ground | |

**For wall installs, P1 is the cleanest power injection point.** Wire 5V from your calibrated buck supply to VIN and ground to GND. No need to splice into USB.

Note: TX and RX are tied directly to the USB-to-serial converter, so don't try to use them for other serial peripherals — you'll get garbage during programming and serial debug.

### CN1 — GPIO + I2C (left side upper, 4-pin PicoBlade)

Pin labels on silkscreen, top to bottom: `GND, IO27, IO22, 3.3V`

| Pin | Signal | Use in this build |
| --- | --- | --- |
| GND | Ground | Shared ground for sensor + relay |
| IO27 | GPIO27 | DHT22 data line |
| IO22 | GPIO22 | Relay IN signal (also exposed on P3) |
| 3.3V | 3.3V | DHT22 VCC |

**This is the primary connector for the thermostat build.** All four signals you need are on one cable.

For future expansion, this is also the natural I2C bus: GPIO27 = SDA, GPIO22 = SCL. Useful if you swap to a BME280 or SHT40 sensor later, but you'd lose the GPIO22 relay output and need to relocate it.

### P3 — Additional GPIO (left side lower, 4-pin PicoBlade)

Pin labels on silkscreen, top to bottom: `GND, IO35, IO22, IO21`

| Pin | Signal | Notes |
| --- | --- | --- |
| GND | Ground | |
| IO35 | GPIO35 | Input-only, no internal pullups, useful for analog sensors |
| IO22 | GPIO22 | Same pin exposed on CN1, just available here too |
| IO21 | GPIO21 | Tied to display backlight PWM — **do not use externally** |

P3 is mostly unused in this build. The future use case is wiring a DS18B20 floor temperature probe to IO35 for slab over-temp protection.

### P4 — Speaker (right side, 2-pin PicoBlade, labeled SPEAK)

Wired through the onboard audio amplifier to GPIO26. Not usable as a general-purpose GPIO. Skip.

### Wiring plan summary

For the prototype, you'll typically run:

```
CN1 (one 4-wire PicoBlade cable):
  GND  → relay GND  + DHT22 GND
  IO27 → DHT22 DATA
  IO22 → relay IN
  3.3V → DHT22 VCC

P1 (or solder to 5V pad):
  VIN  → relay VCC (5V — the relay coil needs 5V, not 3.3V)
  GND  → already shared via CN1
```

For the wall install, swap the USB cable for the buck supply and 24VAC source:

```
Buck supply input:  24VAC from R/C terminals on the SR501
Buck supply output: 5V to P1 VIN/GND
                    5V to relay VCC
```

### Connector cable note

The CYD doesn't always ship with matching PicoBlade cables — they're sold separately in 10-packs on Amazon or AliExpress for ~$5-10. Search "1.25mm 4-pin PicoBlade cable" or "JST 1.25mm 4P" (the latter is technically wrong but commonly used).

If cables are missing and you don't want to wait, soldering hookup wire directly to the connector pads works fine for permanent wall installs. The PicoBlade housings are friction-fit and can vibrate loose over years; soldered connections are arguably more reliable for in-wall hardware.

### Onboard items good to know about

- **RGB LED** (next to the SD card slot, labeled LED1) — wired to GPIO4/16/17, used in the config to flash red on heat call
- **LDR / ambient light sensor** (front side, near display) — wired to GPIO34, used in config for diagnostics, can drive auto-dimming later
- **microSD slot** — not used in this build, but consumes several GPIOs internally
- **BOOT and RESET buttons** (right side, between USB and the ESP32 module) — only needed if you can't enter download mode automatically during flashing

---

## Buck supply calibration

Do this once per power supply, before connecting it to a CYD.

1. Wire 24VAC (or 24VDC bench supply) to the buck's input terminals.
2. Put a multimeter on the buck's output terminals, set to DC volts. **Do not connect a load yet.**
3. Turn the trim pot slowly while watching the meter. The pot is multi-turn (10–25 turns end-to-end), so don't expect fast response — be patient and walk the voltage down to your target.
4. Set output to **5.0V exactly**. Tolerance matters here — the CYD's TFT backlight has a fixed forward voltage and behaves badly outside 4.9–5.1V.
5. Connect the load (a CYD or a 5V test load with a known current draw). Voltage should sag less than 0.1V. If it sags more, the buck is undersized for the load.
6. Mark the trim pot position with sharpie or nail polish. Trim pots drift over time and during install handling.

---

## First flash (one-time, USB-required)

The CYD ships with no firmware. The first flash must happen over USB. After that, all updates are OTA over Wi-Fi.

### Path A: ESPHome Add-on within Home Assistant (recommended for first build)

1. In `guignardha.local`, install the ESPHome Builder add-on: Settings → Add-ons → Add-on Store → ESPHome → Install → Start.
2. Open the add-on web UI.
3. Click "+ New Device". Name it (e.g. `radiant-zone1`), pick ESP32, accept defaults.
4. ESPHome generates a stub config — replace it entirely with the contents of `radiant-thermostat-zone1-cyd.yaml`.
5. Populate `secrets.yaml` with `wifi_ssid`, `wifi_password`, `fallback_ap_password`, `api_encryption_key`, and `ota_password`. Generate the API encryption key with `openssl rand -base64 32` if you don't have one already.
6. Click "Install" → "Plug into the computer running ESPHome Dashboard" — this produces a downloadable `.factory.bin` file.
7. In Chrome or Edge, open https://web.esphome.io. Plug the CYD into your laptop via the data USB port. Click "Connect" → select the serial device → upload the `.factory.bin`.
8. After flashing, unplug and replug the CYD. It should boot, connect to Wi-Fi, and appear in HA as `climate.zone1_thermostat`.

### Path B: ESPHome CLI on your dev machine

For Git-tracked workflow against the `expertamateur-ems` repo:

```bash
pip install esphome --break-system-packages
cd ~/Desktop/expertamateur-ems
esphome compile radiant-thermostat-zone1-cyd.yaml
esphome upload radiant-thermostat-zone1-cyd.yaml --device /dev/cu.SLAB_USBtoUART
```

Find the device path with `ls /dev/cu.*` before and after plugging in the CYD — the new entry is the right one. On macOS this is usually `cu.SLAB_USBtoUART` (CP2102 driver) or `cu.usbserial-XXXX`.

---

## Bench bring-up sequence

Do these in order. Don't skip ahead — each step proves a layer of the system.

### Step 1: Bare CYD on USB power

Just the CYD plugged into your laptop, no external wiring.

- Display backlight on
- Thermostat UI visible (zone label, "--.-" for temp, setpoint, OFF mode)
- HA shows `climate.zone1_thermostat` entity
- Tap +/- buttons on screen — setpoint should change in HA
- Tap mode strip — mode should toggle between HEAT and OFF in HA

If any of this fails, debug before adding hardware. Touch calibration may need tuning — uncomment the `on_touch` lambda in the touchscreen block and watch the logs to see raw vs. mapped coordinates.

### Step 2: Add the DHT22 sensor

Power off the CYD before wiring. Connect the DHT22:

- VCC → 3.3V on the CYD extended connector
- GND → GND on the CYD extended connector
- DATA → GPIO27

Re-power. Within 30 seconds, the display should show the actual room temperature and HA should report temp + humidity. If it shows "--.-" indefinitely, the sensor isn't connected or the pullup resistor is missing. Most AM2302 breakout boards include the pullup; if yours doesn't, add a 10K resistor between VCC and DATA.

### Step 3: Add the relay

Power off the CYD. Wire the relay module:

- VCC → 5V on the CYD extended connector (or to your buck's 5V output)
- GND → GND
- IN → GPIO22

Re-power. In HA, lower the setpoint below current temp.

- The thermostat should switch to HEATING action
- Relay should audibly click
- Built-in RGB LED on the CYD should turn red
- Display background should tint red

Verify the relay's COM-NO contacts close: with the relay energized, put a multimeter in continuity mode across COM and NO. It should beep / show ~0Ω. With the relay de-energized, no continuity.

### Step 4: Switch to wall power

Disconnect USB. Wire the calibrated buck supply:

- 24VAC R/C from a test cable → buck input terminals
- Buck 5V output → CYD's USB power pins (5V and GND on the extended connector)

The CYD should boot normally and behave identically to USB power. If it browns out or reboots randomly, your buck output is sagging under load — recheck calibration with the load attached.

### Step 5: Wire to the SR501

This is the only step that touches the actual heating system. **Power off the boiler at the breaker before doing this.**

- Relay COM and NO contacts wire in parallel with the SR501's existing thermostat terminal screws
- The existing red/white wires from the thermostat cable stay attached to the SR501 — your relay is just an additional path between those two terminals
- Don't disconnect the existing wires until you've verified the smart thermostat works end-to-end

Power the boiler back up. Lower the setpoint to call for heat. Within a few seconds you should hear:

1. The CYD's relay click
2. The SR501's relay click in response
3. The boiler firing (or the circulator engaging, depending on your sequence)

If you don't get the SR501 click, your relay's COM-NO isn't actually closing OR the wires aren't landed correctly. Re-verify with a multimeter before troubleshooting elsewhere.

---

## Dev mode vs. production timers

The thermostat platform's anti-short-cycle timers in the production config are real wall-clock waits:

```yaml
min_heating_off_time: 1800s   # 30 min minimum between heat calls
min_heating_run_time: 900s    # 15 min minimum once heat is called
min_idle_time: 300s           # 5 min between mode changes
```

These are correct for hydronic radiant in production, but they make bench testing tedious — you toggle off and can't toggle on for 30 minutes.

For prototyping, override temporarily:

```yaml
min_heating_off_time: 30s    # PROD: 1800s — restore before wall install
min_heating_run_time: 30s    # PROD: 900s
min_idle_time: 10s           # PROD: 300s
```

**Add a TODO comment so you don't forget to restore production values before deploying to a wall install.** A thermostat with 30-second cycle protection in a real radiant system will short-cycle the boiler and shorten its life.

---

## OTA updates after first flash

Once a CYD is on the network, all subsequent updates happen over Wi-Fi:

- ESPHome Add-on: edit YAML in the web UI, click "Install" → "Wirelessly"
- ESPHome CLI: `esphome run radiant-thermostat-zone1-cyd.yaml` (auto-detects whether USB or OTA is appropriate)

If a CYD ever gets bricked or stuck (bad config, can't reach Wi-Fi), unplug it from wall power, plug into USB, and reflash from scratch. The 8MB flash is hard to brick permanently.

---

## Per-zone deployment

For each new zone, copy the YAML and change three things:

1. `device_name: radiant-zone2` (or 3, 4, 5, 6)
2. `friendly_name: Radiant Zone 2`
3. The `name:` fields on the climate, sensor, and switch entities — these become the HA entity names

Optional but recommended: customize the zone label drawn on screen ("ZONE 1" → "MIDDLE BED", etc.) by changing the relevant `it.print` line in the display lambda.

The Tekmar 356, ZVC406, and SR501 don't care that the thermostats are different brands or different generations — they all just see a dry contact closure on their thermostat input terminals.

---

## Troubleshooting cheat sheet

| Symptom | Likely cause |
| --- | --- |
| Display blank, backlight off | Power supply not delivering 5V, or USB cable is power-only |
| Display works, touch doesn't | XPT2046 not initialized — check SPI pins in YAML match physical board |
| Touch is offset / inverted | Calibration values need adjustment — uncomment the `on_touch` lambda and tune |
| Temp shows "--.-" indefinitely | DHT22 not wired or missing pullup, check GPIO27 |
| Setpoint changes but relay doesn't click | Cycle timer hasn't expired (waiting period from last cycle) |
| Relay clicks but SR501 doesn't respond | COM-NO not actually closing, or wires not landed correctly on SR501 |
| Random reboots | Buck supply sagging under load, or USB cable is unreliable |
| Wi-Fi drops every few hours | ESP32 known issue with some routers — try setting `power_save_mode: NONE` |

---

## CYD connector reference

The ESP32-2432S028R has four 1.25mm Molex PicoBlade connectors on the back. The silkscreen labels are small and hard to read on the yellow PCB. This reference uses the labels printed on the board, oriented with the USB ports at the top.

### USB ports (top edge)

The 2-USB variant has both **USB1** (micro-USB) and **USB2** (USB-C). Either works for power and programming. The single-USB variant has only the micro-USB. No functional difference — use whichever cable is convenient.

### P1 — Serial / 5V power input (top-right, 4-pin)

Silkscreen labels (top to bottom): **VIN, TX, RX, GND**

| Pin | Signal | Notes |
|-----|--------|-------|
| VIN | 5V in/out | **Wall-power injection point.** Wire buck supply 5V here. |
| TX  | GPIO1 | Connected to CH340 USB-serial; avoid for general use |
| RX  | GPIO3 | Connected to CH340 USB-serial; avoid for general use |
| GND | Ground | |

For wall installs, this is the cleanest way to inject 5V from your buck supply without touching the USB pads.

### CN1 — GPIO + 3.3V power (left side, upper, 4-pin)

Silkscreen labels (top to bottom): **GND, IO27, IO22, 3.3V**

| Pin | Signal | Used for |
|-----|--------|----------|
| GND | Ground | Shared by DHT22 and relay GND |
| IO27 | GPIO27 | DHT22 data line |
| IO22 | GPIO22 | Relay IN signal (active LOW) |
| 3.3V | 3.3V out | DHT22 VCC |

**This is the primary connector you'll use for the thermostat build.** All four signals needed for the DHT22 and relay logic are here.

### P3 — Additional GPIO (left side, lower, 4-pin)

Silkscreen labels (top to bottom): **GND, IO35, IO22, IO21**

| Pin | Signal | Notes |
|-----|--------|-------|
| GND | Ground | |
| IO35 | GPIO35 | **Input only**, no pullup — useful for additional sensors |
| IO22 | GPIO22 | Same pin as on CN1, exposed twice for convenience |
| IO21 | GPIO21 | Display backlight — **do not use for anything else** |

For the prototype, P3 is mostly unused. Future floor temperature sensor (DS18B20) would land on IO35.

### P4 — Speaker (right side, 2-pin, labeled "SPEAK")

Connected through the onboard amplifier to GPIO26. Not usable as general GPIO at this connector. Ignore for thermostat work.

### Practical wiring plan

For the prototype with DHT22 + single-channel optoisolated relay:

```
CN1 (one 4-wire PicoBlade cable):
  GND  → relay GND + DHT22 GND
  IO27 → DHT22 DATA
  IO22 → relay IN
  3.3V → DHT22 VCC

P1 (separate, for relay coil power):
  VIN/5V → relay VCC
  GND    → relay GND (shares ground with CN1 via the relay board)
```

The relay's coil needs 5V for reliable operation, which is why VCC comes from P1 (or USB) rather than CN1's 3.3V rail.

### Cable sourcing

The CYD doesn't always ship with matching PicoBlade cables. If yours didn't include them:

- Search Amazon or AliExpress for "1.25mm 4-pin Molex PicoBlade cable" — typically $1-2 each in 10-packs
- Alternatively, solder hookup wire directly to the connector pads for permanent installs (more reliable than push-fit connectors over a 10+ year lifespan in a wall)
- For bench prototyping, soldering is fine and avoids waiting on cable orders

### Hardware variants

Multiple manufacturers produce this reference design (Sunton, Guition, others). The PCB color is consistently yellow and the layout is identical across vendors — only the silkscreen logo varies. Some boards omit the "R" suffix from "ESP32-2432S028" but are functionally identical. The ESPHome config in this repo works for all variants of the 2.8" CYD.

---

## Files

- `radiant-thermostat-zone1-cyd.yaml` — main config
- `secrets.yaml` — Wi-Fi, API key, OTA password (never commit this)
- `radiant_thermostat_wiring_schematic.svg` — wiring diagram (if exported)

## References

- ESP32-Cheap-Yellow-Display GitHub org (canonical reference): https://github.com/witnessmenow/ESP32-Cheap-Yellow-Display
- CYD pinout file with all pin assignments: https://github.com/witnessmenow/ESP32-Cheap-Yellow-Display/blob/main/PINS.md
- Random Nerd Tutorials annotated pinout (with photos): https://randomnerdtutorials.com/esp32-cheap-yellow-display-cyd-pinout-esp32-2432s028r/
- ESPHome CYD community thread: https://community.home-assistant.io/t/esp32-2432s028r-sunton-2-8-cyd-config/780600
- Ryan Ewen's ESPHome configs repo: https://github.com/RyanEwen/esphome-lvgl
- ESPHome thermostat platform docs: https://esphome.io/components/climate/thermostat.html
- Mischianti detailed CYD specs and schema: https://mischianti.org/esp32-2432s028-cheap-yellow-display-high-resolution-pinout-datasheet-schema-and-specs/
