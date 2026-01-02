# ESP32-ETH01-Presence-Sensor
A rock solid wired (AC and Ethernet) mmWave presence sensorusing:

- **WT32-ETH01** (ESP32 + LAN8720)
- **LD2410C** mmWave radar
- **Hi-Link HLK-10M05** isolated AC→5V PSU
- **ESPHome** (Docker build, OTA updates only)

It is designed specifically to avoid:

- Wi-Fi instability
- ESP32 boot-pin traps
- UART conflicts
- False presence triggers in bathrooms

This setup has been built, powered, flashed, and run continuously.

---

## ⚠️ Safety Warning — Mains Voltage

**This project involves 230/240V AC mains wiring.**

- The Hi-Link HLK-10M05 is connected directly to live mains via the green screw terminal block.
- There is **no fuse** on this board — the AC supply is unfused between the terminal and the PSU.
- If you replicate this, consider adding an inline fuse (e.g., 0.5A slow-blow) or mounting inside a fused enclosure.
- **Never work on this board while powered.**
- **Ensure adequate creepage distance** between mains-side and low-voltage traces.
- Double-check your wiring before applying power — brown = live, blue = neutral in most EU/UK flex.

If you're not confident working with mains voltage, don't. Get someone qualified to help.

---

## Why This Hardware Combo

### WT32-ETH01

- Native Ethernet → deterministic latency
- No Wi-Fi dropouts
- OTA works flawlessly once initial flash is done

### LD2410 mmWave

- Detects presence, not just motion
- Works through steam, darkness, and partial obstructions
- Ideal for bathrooms where PIR fails

### Hi-Link HLK-10M05

- Isolated AC → 5V
- Enough current headroom (2A)
- Small, certified, predictable thermal behavior
---
## Power Architecture

- HLK-10M05 provides 5V from mains
- WT32-ETH01 gets 5v from central DC distribution block, regulates internally to 3.3V
- LD2410C gets power from 5V rail but TX and RX connect to ESP32 (on 3.3V logic level and UART-safe)


> ⚠️ **Check your LD2410 variant**  
> Some LD2410 boards want 5V on VCC.  
> UART is still 3.3V — **do not feed 5V into ESP32 GPIOs.**

---

## Assembly Notes

This build uses a standard **double-sided perfboard** as the carrier, with components mounted and wired point-to-point.

### Layout

- **Hi-Link HLK-10M05** occupies the left side of the board. It's a chunky module — give it room and keep mains traces short.
- **Green screw terminal** (bottom-left) accepts AC live and neutral. Brown and blue flex wires visible in the build.
- **WT32-ETH01** is mounted on the right, with the Ethernet jack facing outward for easy cable access.
- **LD2410** (small blue PCB, bottom-right) is positioned with its antenna facing into the room — orientation matters for detection pattern.

### Wiring
on the underside of the perfboard are soldered wires to and from the HiLink's AC and DC outputs to the AC terminal block and the DC 5v and ground rail distribution  block respectively.  the surface connections use dupont connectors to provide power to and interface the LD2410 and the WT32-ETH01 as below: 

## UART Pin Choice

Known-good, repeatable wiring uses the **RXD / TXD header pins** on the WT32-ETH01:

| LD2410 Signal | WT32-ETH01 Pin | ESP32 GPIO | Why |
|--------|----------------|------------|-----|
| TX (LD2410 → ESP32) | RXD | GPIO5 | Not a strapping pin |
| RX (ESP32 → LD2410) | TXD | GPIO17 | Not a strapping pin |
| VCC | 5V  | — | 5V rail |
| GND | GND | — | Common ground |

### Why not GPIO4 / GPIO0 / GPIO12?

They are boot strapping pins. They **will** bite you eventually.

### Why not RX0 / TX0?

They are fine electrically, but collide with flashing and boot logs. RXD/TXD avoids that completely.

This wiring is used successfully by multiple ETH01 + ESPHome builds and has zero boot side-effects.

---

- **Orange and red wires** connect the LD2410 to the WT32-ETH01 UART (TX/RX as per table above).
- **Green and yellow wires** route 5V and GND from the Hi-Link to the WT32-ETH01 power input.
- Keep UART wires short and away from the PSU to minimise noise pickup.

### Mounting Considerations

- The perfboard can be mounted in a flush backbox or surface enclosure.
- Ensure the LD2410 sensor has a clear line of sight — it can see through plastic enclosures (and ceiling plasterboard!) but not metal.
- Allow ventilation around the Hi-Link; it runs warm under continuous load.

---

---

## Flashing ESPHome

The YAML configuration is provided in [`presence-sensor.yaml`](presence-sensor.yaml) (adjust filename as needed).

### Initial Flash (Serial — Required Once)

The WT32-ETH01 must be flashed via serial the first time - it doesn't have a USB port like other ESP32 boards. You'll need a USB-to-TTL adapter (3.3V logic).

1. **Disconnect from mains** — flash the board before it's wired to AC.
2. **Connect the serial adapter:**
   - `TX` (adapter) → `RX0` (WT32-ETH01)
   - `RX` (adapter) → `TX0` (WT32-ETH01)
   - `GND` → `GND`
   - `3.3V` → `3V3` (or power the board separately via 5V)
3. **Enter flash mode:** Hold `IO0` low while powering on, or briefly pull `IO0` to GND and press EN/reset.  a dupont connector shortinh GPIO0 and GND does the trick for me.
4. **Compile and upload:**
   ```bash
   esphome run presence-sensor.yaml
---

## Bathroom-Specific Tuning Notes

- **Delayed OFF (30s)** prevents lights switching off mid-shower
- **Delayed ON (3s)** suppresses spurious reflections from fans, towels, etc.
- Use the **HLK app** (Bluetooth) to tune still/move gate thresholds — ESPHome exposes them, but HLK's tooling is faster for initial calibration

---

## Reliability Notes (Real-World)

- Ethernet eliminates presence "flapping"
- No UART contention
- No boot failures
- OTA works indefinitely
- Survives power cycling cleanly

This is the least fragile ESP32 + mmWave setup I've used to date.

---

## Final Thoughts

If you want:

- Actual presence detection
- Wired reliability
- No boot pin voodoo

This combination works — not theoretically, but on the bench and on the wall.
