# Orvibo AllOne (WiWo-R1) — ESPHome

**First public ESPHome integration for the Orvibo AllOne 433MHz RF hub.**

Si446x (SX716) RF transceiver fully reverse engineered from stock firmware v3.0.5 via Ghidra. SPI init, frequency config, modem parameters — all extracted and replayed.

---

## ⚠️ Current State: TX Works, RX Needs Help

**Let's be upfront:** RF transmit works great. RF receive does not work well. This project needs community help to crack the RX side.

### What works reliably

- **RF TX with hardcoded codes** — Paste a code array captured from a CC1101 transceiver or RTL-SDR into the YAML, and the Orvibo transmits it perfectly via the Si446x at full power. Multiple devices confirmed responding. This is solid and production-ready.

### What doesn't work well

- **RF RX / Learn mode** — The Si446x `RX_RAW_DATA` output has **no hardware squelch**. GPIO5 outputs constant RF noise when in RX mode. ESPHome's `remote_receiver` captures noise mixed with signal, producing jittery, truncated, and often unusable codes. Replaying captured codes back does not reliably trigger target devices.

### What was tried (and failed)

| Approach | Result |
|----------|--------|
| `remote_receiver` with `filter: 500µs` | Stable but destroys protocols with <500µs pulses (most 433MHz remotes use 280µs). Captured codes too corrupted for replay. |
| `remote_receiver` with `filter: 200-250µs` | Crashes the ESP8266 — noise floods the interrupt handler |
| `RX_DATA` (0x14) instead of `RX_RAW_DATA` (0x15) | Worse — RX_DATA needs the packet handler's sync word detector |
| GPIO ISR capture (`attachInterrupt`) | Si446x noise = 100k+ interrupts/sec → watchdog crash |
| Tight polling loop with `yield()` | Starves WiFi stack → API disconnect → crash |
| Frame comparison engine | Noise never produces consistent frames to compare |
| Pulse quantization | Cleans up jitter but can't fix fundamentally corrupted captures |

### Where help is needed

The stock Orvibo firmware **does** learn and replay RF codes successfully on this exact hardware. It uses a GPIO interrupt-based pulse capture approach (we found `pluseData`, `pluseNum`, `timeArray` strings in the firmware), but the exact algorithm hasn't been fully reverse engineered from the Xtensa assembly.

**If you have Ghidra/IDA experience** and want to trace the stock firmware's RF study function, the firmware dump is reproducible via `esptool.py read_flash`. Key offsets:

- `0x001960` — Complete RF code JSON example with clean `[5000, -600, 280, -600, 600, -280]` timing
- `0x00167C` — `rf_cmd_start` string reference
- `0x0016C8` — `"send or study clashed"` (mutual exclusion between TX and study modes)
- `0x001648` / `0x001654` — `pluseNum` / `pluseData` fields
- `0x001298` — Si446x MODEM config (SET_PROPERTY group 0x20)
- `0x001261` — Stock GPIO_PIN_CFG: `GPIO0=TX_DATA(0x11), GPIO1=RX_DATA(0x14)`

**If you have Si446x experience** — the modem has OOK demodulation, spike detection, AGC, and RSSI thresholding built in. There may be a configuration that produces cleaner `RX_RAW_DATA` output, or a way to use the packet handler for raw OOK capture. The current config is a byte-for-byte replay of the stock firmware's init sequence, but the stock firmware may send additional commands during study mode.

**If you have ESP8266 low-level experience** — the core challenge is capturing microsecond-precision GPIO edges on a pin that produces constant noise transitions. The stock firmware somehow manages this (likely by disabling WiFi during study, or using a hardware timer ISR at a fixed sample rate). ESPHome's architecture makes both approaches difficult.

---

## Hardware

**Board:** US20RB V1.1 (2017/02/15)
**WiFi Module:** CW8266-02Z (ESP8266, 1MB flash)
**RF Transceiver:** Si446x (SX716) on SPI, 433.92MHz OOK
**Stock Firmware:** v3.0.5 (Sep 11 2018), model "VS20RB"

> **Note:** This unit is an RF-only model shipped for 433MHz blind/curtain control. No IR hardware is populated on the PCB. Other AllOne variants (S20/S25) may include IR on GPIO3 (TX) and GPIO14 (RX).

### Pin Map

| ESP8266 GPIO | Function | Si446x Pin | Notes |
|-------------|----------|------------|-------|
| GPIO0 | SPI CS | NSEL | ⚠️ Boot strapping — use delayed init |
| GPIO2 | SPI MOSI | SDI | ⚠️ Boot strapping — must be HIGH at boot |
| GPIO13 | SPI SCK | SCLK | |
| GPIO15 | SPI MISO | SDO | ⚠️ Boot strapping — must be LOW at boot |
| GPIO12 | Enable/Reset | SDN | Active HIGH = shutdown |
| GPIO5 | TX/RX Data | GPIO0 | Shared: TX_DATA (0x11) or RX_RAW_DATA (0x15) |
| GPIO4 | Physical button | — | Active LOW, internal pullup |

### Board Photo

![PCB Front](PCB_FRONT_ORVIBO.jpg)

Key components:
- **CW8266-02Z** — Blue shielded WiFi module (center top)
- **PRGM header** — 6-pin flash header (upper right): 3U3, GP0, GND, TX, RST, RX
- **Physical button** — Tactile switch (center)
- **433MHz coil antenna** — Spring coil (bottom right)
- **Si446x (SX716)** — RF transceiver IC (bottom right, near XTAL)
- **Micro USB** — Power input (left edge)

---

## Flashing

### PRGM Header Pinout

```
  PRGM Header (top view, labeled on silkscreen)
  ┌─────────────────┐
  │ 3U3  GP0  GND   │
  │  TX  RST   RX   │
  └─────────────────┘
```

| Pin | Function | Description |
|-----|----------|-------------|
| 3U3 | 3.3V | Do NOT connect if powering via USB |
| GP0 | GPIO0 | Bridge to GND during boot → flash mode |
| GND | Ground | Connect to UART adapter GND |
| TX | UART TX | ESP8266 TX → adapter RX |
| RST | Reset | Pulse LOW to reset (or just power-cycle) |
| RX | UART RX | ESP8266 RX → adapter TX |

### Requirements

- USB-UART adapter (3.3V logic — e.g., DSD TECH SH-U09C5, FTDI, CP2102)
- `esptool.py` (`pip install esptool`)
- ESPHome (`pip install esphome`)

> ⚠️ The ESP8266 is a 3.3V device. Do not use a 5V UART adapter without level shifting.

### Wiring

| Adapter | → | PRGM Header |
|---------|---|-------------|
| GND | → | GND |
| TX | → | RX |
| RX | → | TX |

Power the board via its **micro USB connector** (left side of PCB).

### Enter Flash Mode

1. Wire GND, TX→RX, RX→TX
2. **Bridge GP0 to GND** with a jumper wire
3. Plug in USB power (or pulse RST LOW if already powered)
4. Release GP0 after 1 second
5. Verify: `esptool.py --port /dev/ttyUSB0 chip_id`

> **Tip:** The ESP8266 boot ROM outputs at **74880 baud**. Connect a serial monitor at this baud rate to see the boot log and verify your UART wiring before flashing:
> ```bash
> screen /dev/ttyUSB0 74880
> ```
> Power-cycle the board — you should see the ESP8266 boot header. Press Ctrl+A then K to exit screen.

### Backup Stock Firmware

Always back up before flashing. The flash is 1MB:

```bash
esptool.py --port /dev/ttyUSB0 --baud 115200 read_flash 0x00000 0x100000 orvibo_allone_backup.bin
```

> **Note:** `esptool.py` uses 115200 baud for flash operations (not 74880). The 74880 rate is only for the boot ROM log.

### Flash ESPHome

```bash
esphome run orvibo-allone.yaml
```

First flash via serial. All subsequent updates via OTA over WiFi.

### Restore Stock Firmware

```bash
esptool.py --port /dev/ttyUSB0 --baud 115200 write_flash 0x00000 orvibo_allone_backup.bin
```

---

## Files

| File | Description |
|------|-------------|
| `orvibo-allone.yaml` | ESPHome config (fully commented) |
| `sx716_init.h` | Si446x driver — SPI init, RX/TX, bit-bang TX, pulse quantization |
| `PCB_FRONT_ORVIBO.jpg` | PCB photo showing PRGM header and layout |

Copy `orvibo-allone.yaml` and `sx716_init.h` to your ESPHome config directory. They must be in the same folder.

---

## Usage

### Hardcoded TX Buttons (Recommended — Works Reliably)

Capture codes with a **CC1101 transceiver**, **RTL-SDR**, or any clean 433MHz receiver. Paste into the YAML:

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

### Learn Mode (Experimental — See Limitations Above)

1. Press **Learn Signal** in Home Assistant
2. Press your remote near the Orvibo
3. Check ESPHome logs for the captured code
4. Codes are quantized automatically (clusters jittery timings into clean values)
5. Try **Replay Learned Signal** to test

Works best with protocols that have ≥500µs pulse widths. Short-pulse protocols (280µs EV1527, common in blinds/curtains) are destroyed by the 500µs noise filter.

---

## Technical Details

### Si446x Init Sequence

The stock firmware initializes the Si446x with 56 encrypted POWER_UP blocks (CFG1), a frequency tune command, and 26 SET_PROPERTY packets (CFG2) configuring:

- OOK modulation, async TX direct mode
- 433.92 MHz center frequency
- TX power 0x7F (maximum, 127)
- RSSI jump threshold 0x28, spike detection 0x0B
- Custom channel filter coefficients
- AGC normal mode

All extracted via Ghidra from the firmware binary and replayed byte-for-byte in `sx716_init.h`.

### Boot Strapping

GPIO0 (CS), GPIO2 (MOSI), and GPIO15 (MISO) are ESP8266 boot strapping pins. The Si446x pulls these during power-on. Solution: delay Si446x init to 5 seconds after boot using an `interval:` block, not `on_boot:`.

### Shared TX/RX Pin

GPIO5 is the only data connection between the ESP8266 and Si446x. It serves as TX_DATA (0x11) during transmit and RX_RAW_DATA (0x15) during receive. Mode switching requires SPI commands to reconfigure the Si446x GPIO0 function.

### Stock Firmware RF Protocol

The firmware at offset `0x001960` contains a complete RF code example:

```json
{
  "timeArray": [
    {"array": [5000, -600, 280, -600, 600, -280, ...], "count": 6},
    {"array": [-1000], "count": 1}
  ],
  "modulation": "OOK",
  "id": "0x5b1520"
}
```

The stock firmware stores perfectly quantized timing values. It captures raw pulses and clusters them into clean values before storage — this project's `rf_quantize()` function replicates that approach.

### The RX Problem (For Future Contributors)

The fundamental challenge: the Si446x `RX_RAW_DATA` outputs demodulated OOK data with **no squelch**. When no signal is present, the pin outputs random noise transitions at rates exceeding 100kHz. Transceivers like the CC1101 have hardware squelch that silences the output when signal strength is below a threshold — the Si446x does not.

The stock firmware solves this on the same ESP8266 hardware. The solution likely involves one or more of:

1. **Different Si446x configuration during study mode** — additional MODEM settings not present in the init sequence
2. **Disabling WiFi during capture** — frees CPU for dedicated GPIO monitoring
3. **Hardware timer (FRC1) based sampling** — fixed-rate GPIO polling instead of edge-triggered interrupts
4. **RSSI-based gating** — only recording pulses when Si446x RSSI exceeds a threshold
5. **Multi-frame comparison** — capturing multiple noisy frames and extracting the common signal

---

## Credits

- Si446x configuration and RF protocol format reverse engineered from stock firmware v3.0.5 using Ghidra
- Inspired by the [ESPHome CC1101 component](https://esphome.io/components/cc1101/)

## License

MIT
