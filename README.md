# Orvibo AllOne (WiWo-R1) — ESPHome

**First public ESPHome integration for the Orvibo AllOne 433MHz RF hub.**

Reverse engineered from stock firmware v3.0.5 via Ghidra. Full Si446x (SX716) RF transceiver initialization replayed from extracted config tables.

> **Note:** This specific unit is an **RF-only model** shipped for controlling 433MHz blinds and curtains. It does **not** have IR hardware (no IR LED or IR receiver populated on the PCB). Other AllOne variants (S20/S25) may include IR — those would need GPIO3 (TX) and GPIO14 (RX) testing.

---

## What Works

- ✅ **RF TX** — Transmit 433MHz OOK signals via `remote_transmitter` (hardware-timed, proven reliable)
- ✅ **RF TX (bit-bang)** — Dynamic replay of learned signals via `sx716_transmit_raw()`
- ✅ **RF RX** — Capture signals via `remote_receiver` (500µs filter, quantization)
- ✅ **Si446x full init** — SPI config replay from stock firmware (56 CFG1 blocks + 26 CFG2 packets)
- ✅ **Boot stability** — Delayed init avoids GPIO0/GPIO15 strapping conflicts
- ✅ **WiFi** — Fast connect with manual IP, stable operation
- ✅ **Physical button** — GPIO4

## What Doesn't Work (Yet)

- ⚠️ **RF RX quality** — The Si446x `RX_RAW_DATA` output has no hardware squelch. Noise mixes with signal. The 500µs filter removes sub-500µs noise but also destroys protocols with short pulses (e.g., 280µs EV1527). Protocols with ≥500µs pulses capture and replay fine.
- ❌ **IR TX/RX** — This unit is an RF-only model. No IR hardware populated. Other AllOne variants may include IR on GPIO3 (TX) and GPIO14 (RX) — untested.
- ❌ **Dynamic replay via `remote_transmitter`** — `code: !lambda` fires instantly on ESP8266 with shared TX/RX pin. Bit-bang `sx716_transmit_raw()` works as alternative.

## Workaround for Short-Pulse Protocols

For remotes with <500µs pulse widths (common blinds, some fans), capture the code with a **CC1101 transceiver** or **RTL-SDR**, then paste it as a hardcoded button in the YAML. The Orvibo transmits these codes perfectly — it's only the *capture* that has limitations.

---

## Hardware

**Board:** US20RB V1.1
**WiFi Module:** CW8266-02Z (ESP8266-based, 1MB flash)
**RF Transceiver:** Si446x (SX716) on SPI, 433.92MHz OOK
**Stock Firmware:** v3.0.5 (Sep 11 2018), model "VS20RB"

### Pin Map

| ESP8266 GPIO | Function | Si446x Pin | Notes |
|-------------|----------|------------|-------|
| GPIO0 | SPI CS | NSEL | ⚠️ Boot strapping — use delayed init |
| GPIO2 | SPI MOSI | SDI | ⚠️ Boot strapping — must be HIGH at boot |
| GPIO13 | SPI SCK | SCLK | |
| GPIO15 | SPI MISO | SDO | ⚠️ Boot strapping — must be LOW at boot |
| GPIO12 | Enable/Reset | SDN | Active HIGH = shutdown |
| GPIO5 | TX/RX Data | GPIO0 | Shared pin: TX_DATA or RX_RAW_DATA |
| GPIO4 | Physical button | — | Active LOW, internal pullup |
| GPIO3 | IR TX (not populated) | — | May exist on other AllOne variants |
| GPIO14 | IR RX (not populated) | — | May exist on other AllOne variants |

### Si446x Configuration

The stock firmware initializes the Si446x with:
- **Modulation:** OOK, async TX direct mode
- **Frequency:** 433.92 MHz
- **TX Power:** 0x7F (127, maximum)
- **Modem:** RSSI jump threshold 0x28, spike detection 0x0B, AGC normal mode
- **Channel filter:** Custom coefficients for 433MHz OOK

All config data was extracted from the stock firmware binary using Ghidra and is replayed identically in `sx716_init.h`.

### Board Photo

![PCB Front](PCB_FRONT_ORVIBO.jpg)

Key components visible:
- **CW8266-02Z** WiFi module (center top, blue shielded module labeled "ORVIBO CW8266-02Z V.3.0.5")
- **PRGM header** (upper right, 6 pins: 3U3, GP0, GND, TX, RST, RX)
- **Physical button** (center, tactile switch)
- **433MHz coil antenna** (bottom right, spring coil)
- **Si446x (SX716)** RF transceiver IC (bottom right, near XTAL crystal)
- **Micro USB** power connector (left edge)
- **Board marking:** US20RB V1.1, 2017/02/15, FR-4, KB-6160

---

## Flashing

### PRGM Header

The board has a 6-pin **PRGM** header (upper right of the PCB, near the CW8266-02Z module) with the following pinout:

```
  PRGM Header (top view, pins labeled on silkscreen)
  ┌─────────────────┐
  │ 3U3  GP0  GND   │
  │  TX  RST   RX   │
  └─────────────────┘
```

| PRGM Pin | Function | Description |
|----------|----------|-------------|
| 3U3 | 3.3V | Power (do NOT connect if powering via USB) |
| GP0 | GPIO0 | Pull LOW during boot to enter flash mode |
| GND | Ground | Connect to USB-UART adapter GND |
| TX | UART TX | ESP8266 TX → adapter RX |
| RST | Reset | Pulse LOW to reset (optional, can power-cycle instead) |
| RX | UART RX | ESP8266 RX → adapter TX |

### Requirements

- **USB-UART adapter** (3.3V logic, e.g., DSD TECH SH-U09C5, FTDI FT232, CP2102)
- **esptool.py** (`pip install esptool`)
- **ESPHome** (`pip install esphome`)

> ⚠️ The ESP8266 is a **3.3V device**. Do not use a 5V UART adapter without level shifting.

### Wiring

Connect your USB-UART adapter to the PRGM header:

| Adapter Pin | → | PRGM Pin |
|-------------|---|----------|
| GND | → | GND |
| TX | → | RX |
| RX | → | TX |

Power the board via its **USB micro connector** (left side of PCB). Do NOT connect the adapter's 3.3V to the 3U3 pin if you're powering via USB — it creates a power conflict.

### Enter Flash Mode

1. Connect the UART adapter wiring (GND, TX→RX, RX→TX)
2. **Bridge GP0 to GND** (hold a jumper wire or use a momentary connection)
3. Power the board via USB (plug in the micro USB cable)
4. Release GP0 after 1 second — the ESP8266 is now in flash/download mode
5. Verify with: `esptool.py --port /dev/ttyUSB0 chip_id`

### Backup Stock Firmware (Recommended)

Always back up before flashing. The stock firmware is 1MB:

```bash
esptool.py --port /dev/ttyUSB0 --baud 115200 read_flash 0x00000 0x100000 orvibo_allone_backup.bin
```

### Flash ESPHome (First Time — Serial)

```bash
esphome run orvibo-allone.yaml
```

ESPHome will compile and flash via the serial connection. After the first flash, disconnect the UART adapter — all subsequent updates use **OTA** (over-the-air) via WiFi.

### Restore Stock Firmware

If you need to go back to stock:

```bash
esptool.py --port /dev/ttyUSB0 --baud 115200 write_flash 0x00000 orvibo_allone_backup.bin
```

Remember to enter flash mode again (GP0 → GND during power-on).

---

## Files

| File | Description |
|------|-------------|
| `orvibo-allone.yaml` | ESPHome configuration (fully commented) |
| `sx716_init.h` | Si446x driver — SPI init, RX/TX control, bit-bang TX, pulse quantization |
| `PCB_FRONT_ORVIBO.jpg` | PCB front photo showing PRGM header and component layout |

Copy both files to your ESPHome config directory. The H file must be in the same directory as the YAML.

---

## Usage

### Learning a Signal

1. Press **Learn Signal** in Home Assistant
2. Press your remote within a few seconds (hold close to the Orvibo)
3. Check ESPHome logs for the quantized code:
   ```
   [rf] Quantized: 196 pulses
   [rf] [1511, -888, 1511, -888, 2706, -6896, ...]
   ```
4. Copy the code array from the logs

### Adding a Hardcoded Button

Paste the captured code (or a code captured from CC1101) into the YAML:

```yaml
  - platform: template
    name: "My Device"
    icon: mdi:remote
    on_press:
      - lambda: sx716_start_tx();
      - remote_transmitter.transmit_raw:
          transmitter_id: rf_tx
          carrier_frequency: 0Hz
          code: [PASTE_CODE_HERE]
          repeat:
            times: 10
            wait_time: 0s
      - lambda: sx716_stop_rx();
```

### Replaying a Learned Signal

Press **Replay Learned Signal** to transmit the last learned code via bit-bang TX (5 repeats). Press **Replay x3** for 15 repeats.

---

## Pulse Quantization

The stock Orvibo firmware stores RF codes with perfectly clean timing values like `[5000, -600, 280, -600, 600, -280]`. The Si446x RX produces jittery timing. This project replicates the stock firmware's approach:

1. Capture raw pulse timings
2. Sort absolute values
3. Cluster adjacent values within 25% tolerance (28% max drift from cluster start)
4. Replace each pulse with its cluster's median value

**Before quantization:** `[598, 614, 631, 1175, 1186, 1210, 5098, 5121]`
**After quantization:** `[614, 614, 614, 1186, 1186, 1186, 5110, 5110]`

---

## Stock Firmware RF Protocol

The stock firmware uses a JSON-based cloud protocol for RF codes:

```json
{
  "timeArray": [
    {"array": [5000, -600, 280, -600, 600, -280, ...], "count": 6},
    {"array": [-1000], "count": 1},
    {"array": [5000, -600, ...], "count": 6}
  ],
  "modulation": "OOK",
  "id": "0x5b1520"
}
```

Each sub-array is a frame with a repeat count. The `-1000` entries are inter-frame gaps. This format was extracted from a complete example embedded in the firmware at offset `0x001960`.

---

## Technical Notes

### Boot Strapping

GPIO0 (CS), GPIO2 (MOSI), and GPIO15 (MISO) are ESP8266 boot strapping pins. The Si446x pulls these during boot, causing potential boot failures. Solution: initialize the Si446x via a delayed `interval:` block (5 seconds after boot), not `on_boot:`.

### Shared TX/RX Pin

GPIO5 serves as both TX_DATA and RX_RAW_DATA on the Si446x GPIO0. Before TX, call `sx716_start_tx()` to configure GPIO0 as TX_DATA (0x11) and switch to TX mode. Before RX, call `sx716_start_rx()` to configure GPIO0 as RX_RAW_DATA (0x15) and switch to RX mode.

### Si446x RX Noise

The Si446x `RX_RAW_DATA` (0x15) output has no squelch — GPIO5 outputs demodulated RF noise constantly when in RX mode. This is fundamentally different from transceivers like the CC1101 which have hardware-level signal conditioning. The stock firmware handles this with GPIO interrupt-based capture and software frame comparison. ESPHome's `remote_receiver` on ESP8266 struggles with the constant noise due to interrupt overhead.

### Why Not Use ISR/Polling Capture?

Multiple approaches were tested to bypass `remote_receiver`:
- **GPIO ISR** (`attachInterrupt`) — Si446x noise generates 100k+ interrupts/sec, overwhelming the ESP8266 CPU and triggering watchdog resets
- **Tight polling loop** — Starves WiFi stack even with `yield()` every 20ms, causing API disconnects
- **Hardware timer sampling** — Not enough ESP8266 API support in Arduino framework

The `remote_receiver` with 500µs filter is the most stable approach on this platform.

---

## Credits

- Reverse engineered by examining stock firmware v3.0.5 with Ghidra
- Si446x SPI init tables, GPIO configuration, and RF protocol format extracted from firmware binary
- Inspired by the [ESPHome CC1101 component](https://esphome.io/components/cc1101/) architecture

## License

MIT
