# Repurposing an old TTGO T-Beam as a MeshCore node for Home Assistant

Got an early TTGO T-Beam collecting dust? This repo gives it a second life as a **stationary MeshCore LoRa node** integrated into Home Assistant — no app, no Bluetooth, just plug it in, connect it to your WiFi, and it sits on your network bridging the mesh to HA.

| Front | Back |
|-------|------|
| ![Front](images/tbeam_front.jpg) | ![Back](images/tbeam_back.jpg) |

The target hardware is the early T-Beam variant that ships **without an AXP192/AXP2101 power management IC** — a board that standard MeshCore firmware refuses to run on. This repo patches MeshCore v1.15.0 to make it work, and adds WiFi/TCP connectivity so the [MeshCore Home Assistant integration](https://github.com/spmfte/hass-meshcore) can talk to it directly.

---

## The hardware

This is the early TTGO T-Beam, often mislabelled "v1.1" in reseller listings. The easiest way to identify it: **no AXP chip on the board**.

| Component | Details |
|-----------|---------|
| MCU | ESP32-D0WDQ6 rev 1.0, dual-core 240 MHz |
| LoRa | Semtech SX1276 (TTGO eLORa32 module, 868 MHz) |
| GPS | u-blox NEO-6M |
| Power management | **None** — no AXP192/AXP2101 PMU |
| Flash | 4 MB |
| USB | Micro-USB (CP2104) |

Other identifiers:
- No I2C response at `0x34` (SDA=21, SCL=22)
- RadioLib detects the radio as `SX1276`, DIO2 on GPIO 32
- No OLED populated
- PCB silkscreen: **TTGO eLORa Model LORA32** on the LoRa module

---

## Why standard MeshCore firmware doesn't work on this board

MeshCore's `TBeamBoard` driver unconditionally tries to initialise an AXP2101 then AXP192 over I2C before it touches the radio. On this board there is no AXP chip, so:

1. Both `XPowersAXP2101::init()` and `XPowersAXP192::init()` fail — the IC isn't there.
2. The failed probes leave the ESP32 `Wire` bus in a corrupted state.
3. `rtc_clock.begin(Wire)` in `radio_init()` then blocks indefinitely on the corrupted bus.
4. The PlatformIO bootloader has a higher brownout threshold than the MeshCore release binary, causing an immediate reset on USB power.
5. If anything manages to start and queries battery voltage with `PMU == NULL`, it hits a `LoadProhibited` hard fault.

---

## Patches applied to MeshCore v1.15.0

All changed files are in `patches/`.

### `src/helpers/esp32/TBeamBoard.cpp`

**Disable brownout detector at boot**

```cpp
#include "soc/soc.h"
#include "soc/rtc_cntl_reg.h"

void TBeamBoard::begin() {
    WRITE_PERI_REG(RTC_CNTL_BROWN_OUT_REG, 0);
    ...
}
```

**Reset Wire bus and continue when no PMU is found**

```cpp
if (!PMU) {
    Wire.end();
    Wire.begin(PIN_BOARD_SDA, PIN_BOARD_SCL);
    return true;   // was: return false
}
```

### `src/helpers/esp32/TBeamBoard.h`

**Null-guard on `getBattMilliVolts()`**

```cpp
uint16_t getBattMilliVolts() {
    if (!PMU) return 0;
    return PMU->getBattVoltage();
}
```

### `examples/companion_radio/main.cpp`

Adds WiFiManager provisioning. On first boot the T-Beam creates a WiFi AP (`MeshCore-TBeam`). After credentials are saved it connects to your network and starts a TCP server on port 5000. Home Assistant connects to it there.

To wipe saved WiFi credentials: uncomment `wm.resetSettings()` in `main.cpp`, flash once, re-comment.

### `variants/lilygo_tbeam_SX1276/platformio.ini`

Adds the `Tbeam_SX1276_companion_radio_wifi` build environment with:
- `WIFI_MANAGER=1` and `TCP_PORT=5000`
- `tzapu/WiFiManager` (from GitHub master — the registry 0.16.0 is ESP8266-only)
- `lib_ignore = ESP32 BLE Arduino` — BLE not needed, freeing internal DRAM
- `MAX_CONTACTS=60`, `OFFLINE_QUEUE_SIZE=64` — ESP32 BLE stack BSS lives in a reserved DRAM region; without it, WebServer + DNS move into user DRAM and overflow without these reductions
- `board_build.partitions = partitions_wifi.csv`

### `partitions_wifi.csv`

Custom partition table: app 1.5 MB, SPIFFS **2.375 MB**. The default `min_spiffs.csv` only gives ~110 KB of storage which fills up immediately as MeshCore writes contacts and identity.

---

## Build

Requires [PlatformIO](https://platformio.org/).

```bash
git clone https://github.com/meshcore-dev/MeshCore.git
cd MeshCore

git clone https://github.com/hdcasey/meshcore-tbeam-sx1276-no-axp.git patches_repo

cp patches_repo/patches/TBeamBoard.cpp      src/helpers/esp32/TBeamBoard.cpp
cp patches_repo/patches/TBeamBoard.h        src/helpers/esp32/TBeamBoard.h
cp patches_repo/patches/platformio.ini      variants/lilygo_tbeam_SX1276/platformio.ini
cp patches_repo/patches/main.cpp            examples/companion_radio/main.cpp
cp patches_repo/patches/partitions_wifi.csv partitions_wifi.csv

pio run -e Tbeam_SX1276_companion_radio_wifi
```

---

## Flash

Use the **release binary** from this repo's [releases page](https://github.com/hdcasey/meshcore-tbeam-sx1276-no-axp/releases) — the PlatformIO-built bootloader has a higher brownout threshold and resets the board at startup.

```bash
esptool.py --port /dev/ttyUSB0 --baud 460800 \
  write_flash --flash_mode dio --flash_freq 40m \
  0x1000  bootloader.bin \
  0x8000  partitions.bin \
  0x10000 firmware.bin
```

Or flash the single merged binary at offset `0x0`:

```bash
esptool.py --port /dev/ttyUSB0 --baud 460800 \
  write_flash --flash_mode dio --flash_freq 40m \
  0x0 Tbeam_SX1276_companion_radio_wifi-v1.15.0-merged.bin
```

> macOS: port is typically `/dev/tty.usbserial-XXXXXXXX`

---

## Setup

On first boot the T-Beam creates a WiFi access point named **`MeshCore-TBeam`**:

1. Connect your phone or laptop to `MeshCore-TBeam`
2. A captive portal opens at `192.168.4.1` — enter your home WiFi SSID and password
3. The T-Beam saves the credentials, reboots, and joins your network
4. Check your router's DHCP leases (or open a serial monitor at 115200 baud) to find the assigned IP
5. Assign a **static DHCP lease** in your router so the IP doesn't change

In Home Assistant: **Settings → Devices & Services → Add Integration → MeshCore → TCP → IP → port `5000`**

The T-Beam reconnects automatically on every reboot and runs headlessly — just needs USB power.

---

## Pin reference

| Signal | GPIO |
|--------|------|
| LORA NSS | 18 |
| LORA RESET | 23 |
| LORA DIO0 | 26 |
| LORA DIO1 | 33 |
| LORA DIO2 | 32 |
| LORA SCLK | 5 |
| LORA MISO | 19 |
| LORA MOSI | 27 |
| I2C SDA | 21 |
| I2C SCL | 22 |
| GPS RX | 12 |
| GPS TX | 34 |
| User button | 38 |
| TX LED | 4 |

---

## Credits

- [MeshCore](https://github.com/meshcore-dev/MeshCore) by meshcore-dev
- [RadioLib](https://github.com/jgromes/RadioLib)
- [XPowersLib](https://github.com/lewisxhe/XPowersLib)
- [WiFiManager](https://github.com/tzapu/WiFiManager) by tzapu
- Hardware diagnosis via [Meshtastic](https://meshtastic.org/) to confirm SX1276 and absent AXP
