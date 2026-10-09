# LinknLink eMotion Max 2 → ESPHome

Fully local ESPHome firmware for the **LinknLink eMotion Max 2** 60 GHz mmWave presence sensor. No LinknLink account, no cloud, nothing phoning home.

Everything the stock firmware did is supported, plus a few things it didn't:

- **Presence and zones**: 4 rectangle or polygon zones plus 2 exclusion zones, set from Home Assistant
- **Floor-relative height window**: e.g. only detect people above bed height
- **Radar settings**: sensitivity, trigger speed, hold delay, interference learning, low-power mode
- **IR blaster and receiver** as a native Home Assistant infrared proxy (HA 2026.4+), plus learn/replay and NEC/raw send actions
- **Ambient light** (OPT3004)
- **Status LED** (red/blue) and pinhole button
- **Temperature/humidity** via a USB sensor cable *(work in progress, see below)*

The whole config is a single YAML file with lambdas: no external components and no LibreTiny fork.

> **Use at your own risk.** Opening the case and flashing third-party firmware voids your warranty and can brick the device. **Back up the stock firmware first.**

---

## Hardware

| Part | Details |
|---|---|
| Wi-Fi/BLE module | LinknLink **LL8720-P** (LL8720-P_1V3, P/N 60001): Realtek **RTL8720CF** (AmebaZ2), Cortex-M33 @ 100 MHz, 256 KB RAM, 2 MB flash |
| Radar | **ADTM6101PJDM41P04** (V1.0-221223): Andar **ADT6101P** 60 GHz 2T2R, up to 3 tracked targets. It speaks the same TinyFrame protocol as the Hi-Link LD6002B, so ESPHome's native [`ld6002b`](https://esphome.io/components/sensor/ld6002b/) component works unchanged |
| Light sensor | TI OPT3004 (I²C, 0x44) |
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
| PA19 | Radar wake pin | Radar header P19; needed for low-power mode |
| PA7 | Pinhole button | Low at rest, high when pressed |
| PA11 / PA14 | OPT3004 SCL / SDA | Bit-banged; PA14 has no hardware I²C function |
| PA12 | LED red | PWM, active low |
| PA17 | LED blue | PWM, active low |
| PA15 / PA16 | Log UART2 RX / TX | Used for flashing. Serial logging is off in the config (see Known issues) |
| PA0 | Download-mode strap | High at power-on = download mode. Placeholder IR receiver pin; its interrupt is detached at boot (see Known issues) |
| USB D+ / D− | Temperature cable I²C | Not mapped yet. The config's scan button finds them |

Free GPIOs: PA8, PA9, PA10, PA13, PA20, PA23 (not all are necessarily broken out).

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

**Radar Low Power** makes the radar sleep between short measurements. It runs cooler and uses less power, but it's slower to respond and can lose still people, so it suits "asleep" or "away". It always turns off at boot.

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

### Temperature / humidity *(work in progress)*

LinknLink's THS cable (and Broadlink's HTS2, from the same family) puts an SHT3x-class sensor on the USB data lines. Which GPIOs the eMotion's D+/D− reach hasn't been mapped yet. With the cable plugged in, press **Temperature Sensor Scan**: it probes the free GPIOs and saves the bus it finds. The result shows in **Temperature Sensor Bus**.

The scan briefly drives unknown pins, so it only runs when you press the button. If the radar stops responding afterwards, power-cycle the device.

### Diagnostics

- **Radar Data Rate** (B/s): bytes per second arriving from the radar. 0 means the radar has gone silent.
- **Radar Parser Resyncs**: how many times since boot the parser watchdog has unstuck the radar parser (see Known issues).
- **Uptime** and **Wi-Fi Signal**: Uptime dropping back to ~0 means the device rebooted.

The log also warns when the radar stops sending data and says how long the silence lasted.

---

## Known issues and workarounds

- **Lost radar bytes can stall the `ld6002b` parser for minutes.** The parser recovers by itself eventually, so it looks like the radar going quiet and then coming back.
  - **Why it happens:** after a lost byte, the parser can lock onto a fake frame start whose 1-byte header checksum happens to match (1 in 256). The bogus length is usually over its 1 KB limit, so it logs `Frame too large` and silently skips that many bytes, up to 64 KB. That skip swallows every target report and command reply.
  - **The fix:** a 10-second watchdog in the config detects a large bogus skip, resets the parser and counts it in **Radar Parser Resyncs**. It reaches the component's protected state through a small `Peek` helper struct.
  - This is an ESPHome issue rather than a radar fault, and is worth fixing upstream with a byte-gap timeout or resync.
- **LibreTiny: the serial receive buffer is fixed at 256 bytes, and `uart: rx_buffer_size` is ignored.** 256 bytes is only about 22 ms of radar data at 115200 baud, so any longer stall in the main loop drops bytes, which then triggers the parser stall above. The config raises it to 2048 bytes (about 180 ms) with `rtl87xx: framework: options: LT_SERIAL_BUFFER_SIZE: 2048`.
- **LibreTiny: serial logging blocks the main loop.** Each log line is written to UART2 while the loop waits, about 1 ms per 11 characters at 115200, and that starves the radar UART. The config sets `logger: baud_rate: 0`. Logs over the API (the ESPHome dashboard, `esphome logs`) still work; set it back to 115200 for bench debugging.
- **The radar sometimes stops responding after an OTA update.** The log shows every `ld6002b` command timing out. An OTA only restarts the Wi-Fi chip; the radar stays powered, and stray data on its RX line during the reboot can wedge it. ESPHome's `ld6002b` docs note the module has no reset pin or reset command. Some of these cases may actually be the parser stall above, so try the **Restart** button first. If that doesn't clear it, unplug the device for 10 s.
- **LibreTiny: `generic-rtl8720cf` doesn't define the UART1 pins** (only `PIN_SERIAL1_RX_0/_1`), so ESPHome silently falls back to software serial. The config's `-DPIN_SERIAL1_RX=2u` / `-DPIN_SERIAL1_TX=3u` build flags fix this.
- **LibreTiny: calling `digitalRead()` inside an interrupt handler tears down the pin interrupt on RTL8720C**, which breaks ESPHome's `remote_receiver`. The config receives IR with its own ISR that only timestamps edges (CHANGE fires on both edges on this chip). The decoded data goes to the `ir_rf_proxy` receiver.
- **LibreTiny: freed pin objects are reused** ([#404](https://github.com/libretiny-eu/libretiny/issues/404)). On RTL8720C, attaching an interrupt frees the pin's GPIO object but leaves a dangling pointer. Any later `pinMode()` or `digitalRead()` on that pin then corrupts the heap. That makes the placeholder `remote_receiver` on PA0 (which exists only because `ir_rf_proxy` requires one) a hazard: its interrupt handler calls `digitalRead()`, and the pin floats because attaching the interrupt drops the pull-down. The config detaches its interrupt at boot, and the temperature scan never touches PA0. Never switch a `libretiny_pwm` pin to another mode either.
- **LibreTiny: RTL8720C support is still young.** LibreTiny's own support table rates it 2/5 for stability, with Wi-Fi, PWM, interrupts and the watchdog marked untested. Also open: [#396](https://github.com/libretiny-eu/libretiny/issues/396), where a restart or OTA can hang the chip itself until it's power-cycled.
- **Power supply:** the radar peaks at about 600 mA through the board's linear regulator. Use a decent 5 V 2 A supply and cable; brownouts can upset the radar.
- **No hardware I²C** for the light sensor or temperature cable: their pins don't have I²C functions, and ESPHome's `i2c:` on LibreTiny is hardware-only. Both are bit-banged in lambdas, which costs well under 1 ms per read.
- **Bluetooth**: the RTL8720CF has BLE 4.2, but LibreTiny has no Bluetooth support for it, so it's unused.
- **Wi-Fi power save** defaults to `none` on RTL87xx. The config sets `power_save_mode: light`.

---

## Credits

- [ESPHome](https://esphome.io) and its [`ld6002b`](https://esphome.io/components/sensor/ld6002b/) component
- [LibreTiny](https://github.com/libretiny-eu/libretiny) and ltchiptool
- [Flipper-IRDB](https://github.com/Lucaslhm/Flipper-IRDB) for IR codes

## License

MIT
