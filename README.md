# MeshCore Companion Radio on TTGO T-Beam SX1276 (no AXP PMU)

MeshCore v1.15.0 companion radio firmware, patched to run on the early TTGO T-Beam that ships **without an AXP192/AXP2101 power management IC**. Two connection modes are supported:

- **BLE** — pairs with the MeshCore companion app on Android/iOS
- **WiFi/TCP** — connects to your LAN via WiFiManager captive portal; works with the [MeshCore Home Assistant integration](https://github.com/spmfte/hass-meshcore)

This variant is often mislabelled as "T-Beam v1.1" in reseller listings but is identifiable by the absence of the AXP chip and the LoRa radio powered directly from the 3.3 V rail.

| Front | Back |
|-------|------|
| ![Front](images/tbeam_front.jpg) | ![Back](images/tbeam_back.jpg) |

---

## Hardware

| Component | Details |
|-----------|---------|
| MCU | ESP32-D0WDQ6 rev 1.0, dual-core 240 MHz |
| LoRa radio | Semtech SX1276 (TTGO eLORa32 module, 868 MHz) |
| GPS | u-blox NEO-6M |
| Power management | **None** — no AXP192/AXP2101 PMU |
| Flash | 4 MB |
| Battery | 18650 holder (back), JST connector |
| USB | Micro-USB (CP2104 UART bridge) |

### How to identify this variant

- **AXP absent**: no I2C response at `0x34` on Wire (SDA=21, SCL=22)
- **LoRa confirmed SX1276**: Semtech RF95 — RadioLib sees it as `SX1276`; DIO2 on GPIO 32
- **No display header**: this board has no OLED populated
- The PCB silkscreen reads **TTGO** on the bottom and **TTGO eLORa Model LORA32** on the LoRa module

---

## Why standard MeshCore firmware doesn't work

MeshCore's `TBeamBoard` driver unconditionally tries to initialise an AXP2101 then AXP192 over I2C before touching the radio. On this board:

1. Both `XPowersAXP2101::init()` and `XPowersAXP192::init()` fail — the IC simply isn't there.
2. The failed I2C probes leave the ESP32 `Wire` bus in a corrupted state (`i2cSetClock(): bus is not initialized`).
3. The original code called `return false` when no PMU was found — but `TBeamBoard::begin()` ignores the return value. The real hang was `rtc_clock.begin(Wire)` in `radio_init()` blocking on the corrupted bus.
4. The bootloader shipped with the PlatformIO toolchain uses a higher brownout threshold than the release binary, causing an immediate reset before `setup()` even runs on a battery-only board.
5. After BLE connected and a client asked for battery voltage, `getBattMilliVolts()` dereferenced the null `PMU` pointer → `LoadProhibited` panic.

---

## Patches applied to MeshCore v1.15.0

All changes are in `patches/`. Apply them on top of the upstream `meshcore-dev/MeshCore` source.

### 4. `examples/companion_radio/main.cpp`

Adds WiFiManager support alongside the existing BLE and hardcoded-WiFi paths. When `WIFI_MANAGER=1` is defined:

1. After boot, calls `WiFiManager::autoConnect("MeshCore-TBeam")`
2. If no credentials are saved → creates AP `MeshCore-TBeam`, portal at `192.168.4.1` (3-minute timeout then restart)
3. Once connected → prints IP over serial and starts TCP server on port 5000
4. Home Assistant MeshCore integration connects via TCP to `<T-Beam-IP>:5000`

To reset saved WiFi credentials, uncomment `wm.resetSettings()` in `main.cpp`, flash once, then comment it out again.

---

### 1. `src/helpers/esp32/TBeamBoard.cpp`

**a) Disable brownout detector at start of `begin()`**

The PlatformIO bootloader's brownout threshold causes a reset before `setup()` on battery power. Disabling it in software is the correct fix for this hardware.

```cpp
// Added at top of file:
#include "soc/soc.h"
#include "soc/rtc_cntl_reg.h"

void TBeamBoard::begin() {
    WRITE_PERI_REG(RTC_CNTL_BROWN_OUT_REG, 0);  // disable brownout on boards without AXP power management
    ESP32Board::begin();
    power_init();
    ...
}
```

**b) Reset Wire bus and continue when no PMU is found**

Without this, a corrupted I2C state causes `radio_init()` to hang indefinitely.

```cpp
if (!PMU) {
    // No AXP found — LoRa powered directly from 3.3V rail (no PMU switching).
    // Reset Wire so radio_init()'s RTC scan doesn't block on the corrupted I2C state.
    Wire.end();
    Wire.begin(PIN_BOARD_SDA, PIN_BOARD_SCL);
    return true;   // <-- was: return false
}
```

### 2. `src/helpers/esp32/TBeamBoard.h`

**Null-guard on `getBattMilliVolts()`**

When BLE connects and the companion app queries battery voltage with `PMU == NULL`, the original code causes a `LoadProhibited` hard fault.

```cpp
uint16_t getBattMilliVolts() {
    if (!PMU) return 0;   // <-- added guard
    return PMU->getBattVoltage();
}
```

### 3. `variants/lilygo_tbeam_SX1276/platformio.ini`

`MESH_DEBUG` disabled for production (the debug build spams `noise_floor` every few seconds over serial).

---

## Build

Requires [PlatformIO](https://platformio.org/).

```bash
git clone https://github.com/meshcore-dev/MeshCore.git
cd MeshCore

# Clone patches alongside
git clone https://github.com/hdcasey/meshcore-tbeam-sx1276-no-axp.git patches_repo

# Apply patches
cp patches_repo/patches/TBeamBoard.cpp src/helpers/esp32/TBeamBoard.cpp
cp patches_repo/patches/TBeamBoard.h   src/helpers/esp32/TBeamBoard.h
cp patches_repo/patches/platformio.ini variants/lilygo_tbeam_SX1276/platformio.ini
cp patches_repo/patches/main.cpp       examples/companion_radio/main.cpp

# Build — choose one:
pio run -e Tbeam_SX1276_companion_radio_ble   # BLE (pairs with app)
pio run -e Tbeam_SX1276_companion_radio_wifi  # WiFi/TCP (for Home Assistant)
```

---

## Flash

Use the **release bootloader** from the official MeshCore release binary — the PlatformIO default bootloader has a higher brownout threshold that resets this board at boot.

```bash
# Set ENV to whichever build you compiled:
ENV=Tbeam_SX1276_companion_radio_wifi   # or _ble

# Extract release bootloader (from the official merged binary):
python3 -c "
with open('${ENV}-v1.15.0-merged.bin','rb') as f:
    f.seek(0x1000); bootloader = f.read(0x7000)
open('release_bootloader.bin','wb').write(bootloader)
"

# Flash
esptool.py --port /dev/ttyUSB0 --baud 460800 \
  write_flash --flash_mode dio --flash_freq 40m \
  0x1000  release_bootloader.bin \
  0x8000  .pio/build/${ENV}/partitions.bin \
  0x10000 .pio/build/${ENV}/firmware.bin
```

> On macOS the port is typically `/dev/tty.usbserial-XXXXXXXX`.

---

## Connect to the mesh

### BLE mode

Advertises over Bluetooth — pairs with the [MeshCore companion app](https://github.com/meshcore-dev/meshcore-flutter) on Android/iOS. Set node name and region (`EU_868` for 868 MHz) via the app on first boot.

### WiFi/TCP mode (Home Assistant)

On first boot the T-Beam creates a WiFi access point named **`MeshCore-TBeam`**:

1. Connect your phone to `MeshCore-TBeam`
2. A captive portal opens at `192.168.4.1` — enter your home WiFi SSID and password
3. The T-Beam saves the credentials, reboots, and connects to your network
4. Open a serial monitor (`pio device monitor`) to see the assigned IP address
5. In Home Assistant → **Settings → Devices & Services → Add Integration → MeshCore** → choose **TCP** → enter the IP and port `5000`

The T-Beam will reconnect automatically on subsequent reboots. Assign a static DHCP lease in your router to keep the IP stable.

---

## Pin reference (SX1276 variant)

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
- Hardware diagnosis via [Meshtastic](https://meshtastic.org/) to confirm SX1276 and absent AXP
