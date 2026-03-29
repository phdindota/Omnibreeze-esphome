# OmniBreeze Costco Fan — ESPHome Local Control

Cloud-free local control of the **OmniBreeze DC2313R Tower Fan with Wi-Fi** (sold at Costco) using ESPHome and an ESP32-S3 microcontroller.

This fan ships with a proprietary **Uascent/uHome+** Wi-Fi module (UAM086, BK7238 chip) that communicates with the fan MCU using standard **Tuya MCU protocol over inverted UART** on a single bidirectional data line.

## Features

Full bidirectional control from Home Assistant:

- ✅ Power ON/OFF
- ✅ Fan Speed (5 levels)
- ✅ Oscillation ON/OFF
- ✅ Mode (Normal, Natural, Sleep, Auto)
- ✅ Timer (1–12 hours)
- ✅ Beep ON/OFF
- ✅ Display ON/OFF
- ✅ Room Temperature sensor (°C)
- ✅ Real-time status feedback from the fan MCU

## ⚠️ Compatibility Warning — Check Your Board Revision First

There are at least **two known board revisions** of the OmniBreeze DC2313R. **Open your fan and check which board you have before purchasing any parts.**

### ✅ Compatible — DC2205-WiFi V1.3 (2023 batch)

- **PCB label:** DC2205-WiFi V1.3, dated 2023.06.06
- **Wi-Fi module:** UAM086 (Beken BK7238) on a **removable 4-pin JST connector**
- **Cloud app:** uHome+ (by Shenzhen Uascent Technology)
- **This is the board this project was developed and tested on.** The JST connector makes the ESP32 swap straightforward — unplug the UAM086, wire in the ESP32.

### ❌ Not directly compatible — DC2313R FR-1 V1.0 (2025 batch)

- **PCB label:** DC2313R FR-1 V1.0, dated 2025.07.15
- **Wi-Fi module:** FCM242D (FCM242DACM0-OP-02, FCC ID: XMR2023FCM242D) **soldered directly to the PCB** via castellated pads — **no JST connector**
- **Cloud app:** Reported to use the "Landbook" app instead of uHome+
- **JST pin labels:** RF, ZA, QG, **QX** (vs ZO on the 2023 board)
- **Temperature display:** °F (US market variant, vs °C on the 2023 Canadian batch)

The 2025 board likely uses the same Tuya MCU protocol internally (the JST pin labels are nearly identical), but the soldered-on module makes the ESP32 swap **significantly harder**. You would need to either desolder the FCM242D with a hot air station to access the UART pads, or trace and tap the UART lines on the back of the PCB. This is an advanced modification and not covered by this guide.

**How to tell which board you have without opening the fan:** If you purchased from Costco in 2023–2024, you likely have the compatible 2023 batch. Purchases from late 2025 onward are more likely to have the newer revision. The QC sticker date on the PCB is the definitive way to check.

If you have the 2025 board and successfully get this working, please open a GitHub issue or PR with your findings!

## Hardware Required

- **OmniBreeze DC2313R Tower Fan** (Costco Item #4333021) — **2023 batch with JST connector** (see compatibility warning above)
- **ESP32-S3 development board** (tested with Waveshare ESP32-S3 Mini)
- **1KΩ resistor** (1/4W, through-hole)
- Hookup wire / jumper wires
- Soldering iron (for permanent installation)

## Fan JST Connector Pinout

The fan MCU has a 4-pin JST connector labeled on the PCB as:

| JST Pin | Label | Function | Voltage |
|---------|-------|----------|---------|
| 1 | RF | UART enable / presence detect | 0V idle (inverted) |
| 2 | ZA | Ground | GND |
| 3 | QG | 5V VCC | **5V — DO NOT CONNECT TO ESP32** |
| 4 | ZO | Bidirectional UART data | 0V idle (inverted) |

The wire markings on the JST cable:
- `+` wire → RF (pin 1)
- `-` wire → ZA / GND (pin 2)
- `X` wire → QG / 5V (pin 3) — **DO NOT CONNECT**
- `--` wire → ZO / data (pin 4)

## Wiring Diagram

```
                          ┌──────────────────┐
                          │   ESP32-S3 Mini   │
                          │                   │
Fan JST                   │                   │
┌─────┐                   │                   │
│ RF  │───────────────────│ GPIO6 (TX, inv)   │
│     │                   │       │           │
│ ZA  │───────────────────│ GND   │           │
│     │                   │       │           │
│ QG  │  DO NOT CONNECT   │       │ [1KΩ]     │
│     │                   │       │           │
│ ZO  │───┬───────────────│ GPIO5 (RX, inv)   │
│     │   │               │                   │
│     │   └───[1KΩ]───────│ (from GPIO6)      │
└─────┘                   └──────────────────┘

Summary:
  ZA (GND)  → ESP32 GND
  RF        → ESP32 GPIO6 (direct)
  ZO        → ESP32 GPIO5 (direct) AND ESP32 GPIO6 (through 1KΩ resistor)
  QG (5V)   → DO NOT CONNECT
```

**Key detail:** GPIO6 connects to **both** RF (direct wire) and ZO (through a 1KΩ resistor). This allows the Tuya component to send heartbeat/init frames to RF (keeping the MCU happy) while also sending weakened commands to ZO that don't overpower the MCU's own transmissions on that line.

## How It Works

### Protocol

The fan MCU communicates using **standard Tuya MCU protocol** (`55 AA` framing) but with **inverted UART** (idle LOW instead of standard idle HIGH). The data line (ZO) is **half-duplex bidirectional** — both the Wi-Fi module and MCU share the same wire for TX and RX.

The RF pin acts as a **presence/enable signal**. The MCU only begins communicating when it detects valid Tuya UART activity on RF, indicating a Wi-Fi module is connected.

### Tuya Datapoints

| DP ID | Type | Function | Values |
|-------|------|----------|--------|
| 1 | bool | Power | ON/OFF |
| 2 | enum | Mode | 0=Normal, 1=Natural, 2=Sleep, 3=Auto |
| 3 | int | Fan Speed | 1–5 |
| 5 | bool | Oscillation | ON/OFF |
| 13 | bool | Beep | ON/OFF |
| 15 | bool | Display | ON/OFF |
| 21 | int | Temperature | °C (read-only) |
| 22 | enum | Timer | 0=Cancel, 1–12=hours |
| 24 | bitmask | Unknown | 0 (read-only) |

### Original Wi-Fi Module

The stock Wi-Fi module is labeled **UAM086** and contains a **Beken BK7238** SoC. It connects to the **Uascent/uHome+** cloud platform (not Tuya cloud, despite using Tuya MCU protocol internally). This module is completely replaced by the ESP32-S3 in this project.

## Installation

### 1. Open the fan

Remove the top control panel cover to access the MCU PCB (labeled DC2205-WiFi V1.3). Disconnect the 4-pin JST cable from the UAM086 Wi-Fi module board.

### 2. Wire the ESP32-S3

Follow the wiring diagram above. The 1KΩ resistor is critical — without it, the ESP32's TX signal overpowers the MCU's status transmissions on the shared ZO line.

### 3. Flash ESPHome

Copy `omnibreeze-fan.yaml` to your ESPHome configuration directory. Update the `secrets.yaml` references for your Wi-Fi credentials, API key, and OTA password.

For the initial flash of the ESP32-S3, use the ESPHome Web Flasher (https://web.esphome.io) in Chrome/Chromium. Hold the BOOT button on the ESP32-S3 while plugging in USB, then flash.

### 4. Boot sequence

1. Power the fan ON using the physical power button
2. The ESP32-S3 will initialize the Tuya connection
3. All datapoints will register within a few seconds
4. Control is available from Home Assistant immediately

**Note:** The Tuya component initializes best when the fan MCU is already powered on and sending status frames during ESP32 boot.

### 5. Add to Home Assistant

The device should be auto-discovered by Home Assistant. If not, manually add the ESPHome integration and enter the device's IP address.

## Troubleshooting

### "Initialization failed at init_state X"
The fan MCU wasn't sending data during ESP32 boot. Turn the fan ON physically, then reboot the ESP32.

### Commands sent but fan doesn't respond
Check that GPIO6 is connected to **both** RF (direct) and ZO (through 1KΩ resistor). Without the RF connection, the MCU won't accept commands.

### Status updates not appearing
Check that GPIO5 is connected to ZO. Check that the 1KΩ resistor is in the GPIO6→ZO path (not in the GPIO6→RF path).

### Fan makes 5 beeps on every command
The Tuya component sends WiFi status updates that trigger beeps. Turn off beep via the Beep switch in HA, or add `beep_datapoint: 13` if supported.

### No data at all
Verify wiring with a multimeter:
- ZA should be GND (continuity with ESP32 GND)
- QG should read ~5V (do NOT connect to ESP32)
- ZO and RF should read ~0V when idle (inverted UART)

## Hardware Details

### Fan MCU PCB
- Board: **DC2205-WiFi V1.3** (dated 2023.06.06)
- Manufacturer: Ningbo Singfun Electric Appliance (OmniBreeze brand)
- MCU: SOP-16 package (exact part number unreadable)
- Display driver: SOP-16 package (likely TM-series LED driver)
- Temperature sensor: NTC thermistor
- Touch interface: Capacitive touch springs
- IR receiver: For remote control

### Original Wi-Fi Module
- Module: **UAM086-A0-V1.0**
- SoC: **Beken BK7238** (QFN package)
- Cloud platform: Uascent/uHome+ (NOT Tuya cloud)
- App: uHome+ (by Shenzhen Uascent Technology Co., Ltd.)
- MAC OUI: 8C:B7:D0 (unregistered)

## Reverse Engineering Notes

This project required extensive reverse engineering because:

1. The fan uses the **uHome+** platform, not Tuya — no existing community tools (CloudCutter, tuya-local) work
2. The Wi-Fi module (UAM086/BK7238) is **not a standard Tuya module** (not CB3S, WB3S, etc.)
3. The UART is **inverted** (idle LOW) — captured data appeared as XOR 0xFF encoded until this was discovered
4. The data line is **half-duplex bidirectional** on a single wire (ZO)
5. The RF pin serves as a **presence/enable** signal requiring active UART frames
6. The 1KΩ resistor solution for half-duplex was the final breakthrough

### Key discoveries
- Initial UART captures showed `55 95 7E 7C...` frames — these were actually `AA 6A 81 83...` (inverted), which is standard Tuya protocol `55 AA` read with wrong polarity
- The JST pin labels (RF, ZA, QG, ZO) on the PCB were the key to figuring out the correct pinout
- QG is labeled but blacked out on the PCB silkscreen — it carries 5V VCC
- The MCU only communicates when RF receives valid Tuya UART traffic

## License

MIT License — see [LICENSE](LICENSE) for details.

## Credits

Reverse engineered and developed by [@ea](https://github.com/ea) with assistance from Claude (Anthropic).

## Contributing

If you have an OmniBreeze fan and can test or improve this configuration, pull requests are welcome! Especially useful:
- Testing with different OmniBreeze model numbers
- Confirming the Auto mode (DP 2, value 3) behavior
- Identifying DP 24 (bitmask) function
- 3D printable enclosure for the ESP32-S3 inside the fan housing
