# LinknLink eMotion Max 2 → ESPHome

Fully local ESPHome firmware for the **LinknLink eMotion Max 2** 60 GHz mmWave presence sensor. No LinknLink account, no cloud, nothing phoning home.

Everything the stock firmware did is supported, plus a few things it didn't:

- **Presence and zones**: 4 rectangle or polygon zones plus 2 exclusion zones, set from Home Assistant
- **Floor-relative height window**: e.g. only detect people above bed height
- **Radar settings**: sensitivity, trigger speed, hold delay, interference learning, low-power mode
- **IR blaster and receiver** as a native Home Assistant infrared proxy (HA 2026.4+), plus learn/replay and NEC/raw send actions
- **Ambient light** (OPT3004)
- **Status LED** (red/blue) and pinhole button
- **Temperature/humidity** via LinknLink's THS or Broadlink's HTS2 USB sensor cable (SHTC3)

The whole config is a single YAML file with lambdas: no external components and no LibreTiny fork.

**Requires ESPHome 2026.11 or later.** 2026.11 moved `ld6002b` onto a shared `ld600x` base, and the radar watchdog is written against it. It also brings LibreTiny 1.14.0, which fixes several RTL8720C bugs (see Known issues).

> **Use at your own risk.** Opening the case and flashing third-party firmware voids your warranty and can brick the device. **Back up the stock firmware first.**

---

## Hardware

| Part | Details |
|---|---|
| Wi-Fi/BLE module | LinknLink **LL8720-P** (LL8720-P_1V3, P/N 60001): Realtek **RTL8720CF** (AmebaZ2), Cortex-M33 @ 100 MHz, 256 KB RAM, 2 MB flash |
| Radar | **ADTM6101PJDM41P04** (V1.0-221223): Andar **ADT6101P** 60 GHz 2T2R, up to 3 tracked targets. It speaks the same TinyFrame protocol as the Hi-Link LD6002B, so ESPHome's native [`ld6002b`](https://esphome.io/components/sensor/ld6002b/) component works unchanged |
| Light sensor | TI OPT3004 (I²C, 0x44) |
| Temperature/humidity *(optional)* | LinknLink THS / Broadlink HTS2 USB cable: Sensirion SHTC3 (I²C, 0x70) on the USB D+/D− lines |
| IR | IR LED blaster + 38 kHz IR receiver |
| LED | Red/blue bi-colour, common anode |
| Power | 5 V USB, 1117-type 3.3 V LDO (the radar can peak at ~600 mA) |

The radar board is mounted **upside down** in the enclosure, so the radar's X and Z axes are inverted. The config has an **Upside Down Mounting** switch, on by default, that corrects both.

<!-- Add photos: board top, board bottom with UART pads, radar module -->

### Pin map

| GPIO | Function | Notes |
|---|---|---|
| PA2 / PA3 | Radar UART RX / TX | Hardware UART1 (needs the build flags in the config) |
| PA4 | IR LEDs | `remote_transmitter`, hardware PWM carrier |
| PA18 | IR receiver | Custom edge-timestamp ISR (see Known issues) |
| PA7 | Pinhole button | Low at rest, high when pressed |
| PA11 / PA14 | OPT3004 SCL / SDA | Bit-banged; PA14 has no hardware I²C function |
| PA12 | LED red | PWM, active low |
| PA17 | LED blue | PWM, active low |
| PA15 / PA16 | Log UART2 RX / TX | Used for flashing. Serial logging is off in the config (see Known issues) |
| PA0 | Download-mode strap | High at power-on = download mode. Placeholder IR receiver pin (`ir_rf_proxy` requires a `remote_receiver`); its interrupt is detached at boot |
| PA19 / PA20 | USB D+ / D− → temperature cable I²C | Bit-banged; SDA/SCL orientation is detected at boot. These are the chip's I²C0 pins |

Free GPIOs: PA8, PA9, PA10, PA13, PA23 (not all are necessarily broken out).

The radar header's wake pin (P19) isn't connected to the Wi-Fi module, so there's no wake line.

---

## Flashing (macOS)

You'll need a 3.3 V USB-UART adapter and [ltchiptool](https://github.com/libretiny-eu/ltchiptool):

```sh
brew install pipx
pipx ensurepath          # then reopen Terminal
pipx install ltchiptool
```

### 1. Wire up

Use the UART pads on the bottom of the board. <!-- photo -->

| Adapter | Board |
|---|---|
| GND | GND |
| RX | TXD (log TX, PA16) |
| TX | RXD (log RX, PA15) |

**Don't power the board from the adapter's 3.3 V pin.** The radar draws far more than it can supply. Power the eMotion from its own USB.

Find the adapter's port with `ls /dev/cu.*` (use the `cu.` device, not `tty.`).

### 2. Enter download mode

Hold **PA0 (IOA0) at 3.3 V** while powering the device on, then release it. The chip waits in download mode instead of booting.

### 3. Back up the stock firmware

```sh
ltchiptool flash read -d /dev/cu.usbserial-XXXX realtek-ambz2 emotion_stock.bin
```

If it fails partway through, add `-b 115200`. You should end up with a 2 MB file.

> **Keep this backup private.** The stock flash, and the stock boot log, contain your Wi-Fi credentials, MQTT credentials and the device's LinknLink licence key.

### 4. Flash ESPHome

1. Add `emotion-max-2.yaml` to ESPHome and fill in `secrets.yaml` (below). To rename the device, change `device_name` / `friendly_name` under `substitutions:` at the top.
2. Build it and download the **UF2** file (Dashboard → Install → Manual download).
3. Enter download mode again, then:

   ```sh
   ltchiptool flash write -d /dev/cu.usbserial-XXXX emotion-max-2.uf2
   ```

After this, updates go over the air.

### secrets.yaml

```yaml
wifi_ssid: "your-ssid"
wifi_password: "your-password"
api_encryption_key: "base64 key from `esphome` or the dashboard"
# ota_password: "..."   # recommended, then uncomment it in the config
```

---

## Using it

### Zones

Coordinates are in millimetres, with the sensor at the origin and **+Y pointing straight out into the room**. X runs left–right. The easiest way to find which side is positive in your room is to walk around and watch the **Target 1 X / Y** diagnostic sensors.

- **Polygon Zones** off: rectangle zones. Set **Zone 1–4 Begin/End X/Y** and **Occupancy Mask 1–2 Begin/End X/Y**. Begin and End can be in either order.
- **Polygon Zones** on: polygon zones. Each **Polygon Zone 1–4** and **Polygon Exclusion 1–2** text entity takes the polygon's corners as `x:y;x:y;x:y;…` in mm, with at least 3 and up to 20 points. For example, `-1500:500;1500:500;1500:3000;-1500:3000`.
- **Masks/exclusions:** a target inside one is ignored for overall occupancy and for every zone. Use them for fans, curtains or other things that move.
- **Max Distance** ignores anything further away. **Installation Angle** rotates the coordinate frame if the sensor is mounted at an angle.
- Each zone has its own **Occupancy Off Delay**, and there's a **Zone N Target Count**.

Zones 2–4 and Mask 2 are disabled by default in HA. Enable the entities you need.

### Height window

The radar has its own Z (height) filter. The config converts floor-relative heights into it:

- **Sensor Height**: how high the sensor is mounted, in metres.
- **Minimum / Maximum Detection Height**: only targets between these heights above the floor are tracked. The default minimum is 0 m (everything). Raising it, e.g. to 0.8 m with the sensor beside a bed, ignores someone lying in the bed.
- **Target 1 Height** (diagnostic) shows the live height of the first target, which helps with tuning.

Heights are measured from the centre of a person's radar reflection (roughly the torso), not the top of their head.

### Radar settings

These are stored in the radar's own flash: **Radar Sensitivity**, **Trigger Speed**, **Hold Delay**, **Installation Mode**, **Auto/Clear Interference** (learns static reflectors) and **Reset Detection Areas**.

**Radar Low Power** makes the radar sleep between short measurements. It runs cooler and uses less power, but it's slower to respond and can lose still people, so it suits "asleep" or "away". It always turns off at boot. There's no wake line to the radar (see Pin map), so if the switch occasionally fails to turn low-power off, toggle it again.

### Infrared

- Shows up in HA as native **IR Transmitter** and **IR Receiver** entities (HA 2026.4+ infrared platform), usable by integrations such as HAIR.
- **IR Replay Last Code** re-sends the last code received. Point a remote at the sensor, press a button, then replay it.
- Received codes are logged. NEC codes appear as an address/command pair; anything else as raw timings you can paste into `send_ir_raw`.
- API actions for scripts and automations (the prefix follows `device_name`):

  ```yaml
  action: esphome.emotion_max_2_send_ir_nec
  data: { address: "0xBF00", command: "0x2FD0" }      # hex or decimal strings

  action: esphome.emotion_max_2_send_ir_raw
  data: { command: [9000, -4500, 560, -560, ...] }    # µs, +mark / −space
  ```

**Example: Hisense VIDAA TVs (NEC address `0xBF00`).** The config includes buttons for discrete power on `0x2FD0` (wakes the TV from deep standby), power off `0x2ED1` and Input `0xED12`. A fuller list of codes is in [Flipper-IRDB](https://github.com/Lucaslhm/Flipper-IRDB/tree/main/TVs/Hisense). To convert Flipper `NECext` bytes to ESPHome format, swap each pair: Flipper `address: 00 BF`, `command: D0 2F` becomes `0xBF00` / `0x2FD0`.

### Status LED and button

- **LED**: flashes blue while booting until Home Assistant connects, then goes off. It turns solid red if HA has been disconnected for 10 s. Otherwise it's yours to control from HA as an RGB light (green does nothing, because the LED has no green die).
- **Button**: hold 5–10 s to restart; hold for more than 10 s to factory reset.

### Temperature / humidity

LinknLink's THS cable, and Broadlink's HTS2 from the same family, put a Sensirion SHTC3 on the USB data lines. Those lines reach PA19/PA20 as an I²C bus. Plug the cable in and it's detected automatically. The config tries both SDA/SCL orientations and also recognises SHT3x/SHT4x sensors at 0x44/0x45.

- **Temperature** and **Humidity** are read every 15 s. Every reading is published.
- **Temperature Offset** calibrates against a reference thermometer.
- **Temperature Sensor Bus** (diagnostic) shows what was found, e.g. `SHTC3 at 0x70 – SDA PA20, SCL PA19`.
- Unplugging the cable makes the readings unavailable, and they come back when it's reconnected.

### Diagnostics

- **Radar Parser Resyncs**: how many times since boot the watchdog has unstuck the radar parser (see Known issues). It should normally stay at 0.
- **Uptime** and **Wi-Fi Signal**: Uptime dropping back to ~0 means the device rebooted.
- If the device crashed, ESPHome 2026.11+ prints the crash details in the log the next time you connect.

For deeper debugging, ESPHome's [`debug`](https://esphome.io/components/debug/) component adds Free Heap and Loop Time sensors.

---

## Known issues and workarounds

- **Lost radar bytes can stall the `ld6002b` parser for minutes.** The parser recovers by itself eventually, so it looks like the radar going quiet and then coming back.
  - **Why it happens:** after a lost byte, the parser can lock onto a fake frame start whose 1-byte header checksum happens to match (1 in 256). The bogus length is usually over its 1 KB limit, so it logs `Frame too large` and silently skips that many bytes, up to 64 KB. That skip swallows every target report and command reply.
  - **The fix:** in normal mode the radar sends frames continuously (about 750 B/s even in an empty room), so a watchdog resets the parser if no valid frame has arrived for 5 s, and counts it in **Radar Parser Resyncs** once frames resume. It's skipped in low-power mode, where the radar legitimately goes quiet. It reaches the component's protected state through a small `Peek` helper struct.
  - This is an ESPHome issue rather than a radar fault, and is still present in 2026.11. It's worth fixing upstream with a byte-gap timeout or resync.
- **LibreTiny: the serial receive buffer is fixed at 256 bytes, and `uart: rx_buffer_size` is ignored.** 256 bytes is only about 22 ms of radar data at 115200 baud, so any longer stall in the main loop drops bytes, which then triggers the parser stall above. The config raises it to 2048 bytes (about 180 ms) with `rtl87xx: framework: options: LT_SERIAL_BUFFER_SIZE: 2048`.
- **LibreTiny: serial logging blocks the main loop.** Each log line is written to UART2 while the loop waits, about 1 ms per 11 characters at 115200, and that starves the radar UART. The config sets `logger: baud_rate: 0`. Logs over the API (the ESPHome dashboard, `esphome logs`) still work; set it back to 115200 for bench debugging.
- **The radar sometimes stops responding after an OTA update.** The log shows every `ld6002b` command timing out. An OTA only restarts the Wi-Fi chip; the radar stays powered, and stray data on its RX line during the reboot can wedge it. ESPHome's `ld6002b` docs note the module has no reset pin or reset command. Some of these cases may actually be the parser stall above, so try the **Restart** button first. If that doesn't clear it, unplug the device for 10 s.
- **LibreTiny: `generic-rtl8720cf` doesn't define the UART1 pins** (only `PIN_SERIAL1_RX_0/_1`), so ESPHome silently falls back to software serial. The config's `-DPIN_SERIAL1_RX=2u` / `-DPIN_SERIAL1_TX=3u` build flags fix this.
- **LibreTiny: `micros()` can jump back ~1 ms on RTL8720C.** It combines the FreeRTOS tick count with the SysTick counter. Inside another interrupt, SysTick can wrap before its own interrupt has bumped the tick count, so `micros()` returns a time about 1 ms early. Anything timing edges in an ISR sees ±1000 µs errors. The config's IR receiver uses its own `now_us()`, which adds the missing millisecond when a SysTick interrupt is pending and never goes backwards. (ESPHome already carries its own clock for RTL8710B, but not RTL8720C.)
- **ESPHome's `remote_receiver` doesn't work reliably on RTL8720C.** Before 2026.11, reading the pin from its interrupt tore the interrupt down. 2026.11 fixed that (esphome#22400), but the `micros()` jump above still breaks it: a backwards step looks like a long gap with a short stored value, so the receiver's reader runs past the end of the frame and replays its whole buffer (including stale memory from the previous firmware), with bursts of `Buffer overflow`. The receiver idles high, so it would also need `inverted: true`. The config keeps its own ISR that only timestamps edges (CHANGE fires on both edges on this chip) and passes the frames to the `ir_rf_proxy` receiver. The placeholder `remote_receiver` on PA0 exists only because `ir_rf_proxy` requires one. PA0 floats (attaching the interrupt drops the pull-down), so its interrupt is detached at boot.
- **Fixed in LibreTiny 1.14.0 (ESPHome 2026.11):** a dangling GPIO pointer after a pin's mode changed, which could corrupt the heap ([#404](https://github.com/libretiny-eu/libretiny/issues/404), fixed by #405); a restart or OTA hanging the chip until it was power-cycled ([#396](https://github.com/libretiny-eu/libretiny/issues/396), fixed by #397); and a Wi-Fi event double free (#401). Even so, never switch a `libretiny_pwm` pin to another mode.
- **LibreTiny: RTL8720C support is still young.** LibreTiny's own support table rates it 2/5 for stability, with Wi-Fi, PWM, interrupts and the watchdog marked untested.
- **Faint glow from an "off" LED (fixed in the config):** ESPHome's `libretiny_pwm` writes "full" as 1023/1024, because it scales to 2^bits − 1 while LibreTiny divides by 2^bits. An inverted, common-anode LED that should be off therefore gets a 0.1 % pulse each period, which shows as a faint purple. The light drives small template outputs that write a true 100 % (`analogWrite(pin, 1 << 10)`) for off.
- **Power supply:** the radar peaks at about 600 mA through the board's linear regulator. Use a decent 5 V 2 A supply and cable; brownouts can upset the radar.
- **Bit-banged I²C:** the light sensor's SDA (PA14) has no I²C function. ESPHome's `i2c:` on LibreTiny is hardware-only, and RTL8720C I²C is untested in LibreTiny. So both buses are bit-banged in lambdas, at well under 1 ms of work per read. That includes the temperature cable, even though PA19/PA20 are hardware I²C0 pins.
- **Bluetooth**: the RTL8720CF has BLE 4.2, but LibreTiny has no Bluetooth support for it, so it's unused.
- **Wi-Fi power save** defaults to `none` on RTL87xx. The config sets `power_save_mode: light`.

---

## Credits

- [ESPHome](https://esphome.io) and its [`ld6002b`](https://esphome.io/components/sensor/ld6002b/) component
- [LibreTiny](https://github.com/libretiny-eu/libretiny) and ltchiptool
- [Flipper-IRDB](https://github.com/Lucaslhm/Flipper-IRDB) for IR codes

## License

MIT
