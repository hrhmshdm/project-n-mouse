# Project N-Mouse (ZMK Firmware)

Custom ultra-compact wireless mouse firmware powered by **ZMK Firmware (v0.3)** and Zephyr RTOS.

Designed around the **Seeed Studio XIAO nRF52840 (BLE)**, featuring a **PixArt PAW3395** optical sensor, **TTC Green Core mechanical rotary encoder**, and **Kailh Silent Micro Switches**.

---

## 💡 Concept & Design Inspiration

The physical design and transparent aesthetic of this project were inspired by the charging case of **Nothing Ear**. It aims to bring that minimalist, transparent industrial design language into an open-source, ultra-compact wireless mouse.

> ⚠️ **Disclaimer**: This is an independent, non-commercial DIY maker project. It is **not affiliated with, endorsed by, or associated with Nothing Technology Limited** in any way.

---

## 🛠 Hardware Specifications & Pinout

### MCU & Indicators
- **MCU**: Seeed Studio XIAO nRF52840 (BLE)
- **Status LED**: Integrated 3-color RGB LED mapped via `zmk-rgbled-widget` (`led-red`, `led-green`, `led-blue`)
- **Battery**: 3.7V 200mAh Li-Po (601230, JST 2.0 2-Pin)[cite: 4]
- **MCU Interconnect**: MAC8 Spring Pin Header / Conthrough (12-Pin, H=3.5mm)[cite: 6]

### Pin Assignment Table

| Pin | Function / Target | Component / Description | Type |
| :--- | :--- | :--- | :--- |
| **D0** | Left Click (SW1) | Kailh Silent Micro Switch (6x6x7.3mm)[cite: 3] | Direct GPIO (Active Low) |
| **D1** | Right Click (SW2) | Kailh Silent Micro Switch (6x6x7.3mm)[cite: 3] | Direct GPIO (Active Low) |
| **D2** | Middle Click (SW3) | 4.5x4.5x4.3mm Tactile Switch[cite: 2] | Direct GPIO (Active Low) |
| **D3** | Side Button (SW4) | 4.5x4.5x5.0mm Tactile Switch[cite: 5] | Direct GPIO (Active Low) |
| **D4** | Mode Button (SW5) | 4.5x4.5x4.3mm Tactile Switch[cite: 2] | Direct GPIO (Active Low) |
| **D5** | Encoder Phase A | TTC Green Core Rotary Encoder | `alps,ec11` |
| **D6** | Encoder Phase B | TTC Green Core Rotary Encoder | `alps,ec11` |
| **D7** | SPI CS | PAW3395 Chip Select | SPI2 (`nordic,nrf-spim`) |
| **D8** | SPI SCK | PAW3395 Clock | Hardware SPI |
| **D9** | SPI MISO | PAW3395 Data In | Hardware SPI |
| **D10** | SPI MOSI | PAW3395 Data Out | Hardware SPI |
| **P0.09** | IRQ / MOTION | PAW3395 Motion Interrupt | Direct GPIO (Active Low) |

---

## 📦 Key Hardware Components (BOM Highlights)

### Active Components & Sensors
- **Sensor**: PixArt PAW3395 (1000 CPI native configuration)
- **Power LDO (U3)**: Texas Instruments TLV74318PDBVR (1.8V Low-Dropout Regulator, SOT-23-5)[cite: 1]
- **Primary Switches (L/R)**: Kailh Silent Micro Switch (6x6x7.3mm, 2-pin)[cite: 3]
- **Tactile Switches (Aux)**: 4.5x4.5mm Tactile Switches (TS-4501 series, H=4.3mm / H=5.0mm)[cite: 2, 5]
- **Power Switch**: Subminiature Slide Switch (MSK12C01 series)[cite: 2]
- **Encoder**: TTC Green Core Mechanical Rotary Encoder (12 Pulses / 24 Detents)
- **Battery**: 3.7V 200mAh 601230 Rechargeable Li-Po Battery[cite: 4]
- **MCU Header**: MAC8 Conthrough / Spring Pin Header (12-Pin, Height: 3.5mm)[cite: 6]

### Recommended Passives (0805 SMD)
- **Resistors**: 10kΩ ±1% (R1), 4.7Ω ±1% (R2), 2MΩ ±1% (R3)[cite: 1]
- **Capacitors**: 100nF 50V X7R (C2, C4, C5), 10µF 35V X5R (C3, C6), 47µF 10V X5R (C8), 1µF 50V X7R (C9, C11), 4.7µF 50V X5R (C1)[cite: 1]

---

## ⚠️ Hardware Assembly Notes

### MCU Mounting (Conthrough / Spring Pin Header)
- **Footprint Requirement**: The mainboard uses specific footprints optimized for **MAC8 Conthrough 3.5mm spring pins** (smaller hole diameter compared to standard 2.54mm pin headers)[cite: 6]. Standard pin headers may not fit or establish reliable contact.
- **Assembly Workflow**:
  1. Insert the XIAO BLE module into the mainboard using the 3.5mm Conthrough header without soldering[cite: 6].
  2. Flash the firmware and perform complete hardware/sensor operational checks.
  3. Once verified, permanently solder the pins to ensure mechanical stability against drops and long-term switch actuation force.

---

## ⚙️ Key Technical Implementations

### 1. Direct Integer Wheel Scroll (`zmk,behavior-encoder-mouse-scroll`)
- **Profile**: Configured for 12 pulses / 24 detents matching the TTC Green Core hardware spec (`steps = <12>; triggers-per-rotation = <24>;`).
- **Low-Latency Logic**: Direct integer HID report dispatch (`±1` per physical notch), eliminating latency and notch misreads caused by floating-point software inertia layers.
- **Reference**: The direct scroll implementation references the approach from **elekit's (`elek1t`) encoder scroll behavior**, adapted and ported for ZMK v0.3 compatibility.

### 2. Sensor & Motion Processing (PAW3395 Driver)
- **Driver**: Powered by and adapted from **[badjeff/zmk-paw3395-driver](https://github.com/badjeff/zmk-paw3395-driver)**[cite: 7].
- **Clock**: SPI2 running at 4 MHz (`CONFIG_SPI=y`, `CONFIG_INPUT=y`, `CONFIG_PAW3395=y`).
- **Interrupt Routing**: Routed through NFC1 pin (`P0.09`) using `CONFIG_NFCT_PINS_AS_GPIOS=y`.
- **Drag Scroll (Layer 2)**: Re-routes sensor XY delta coordinates to horizontal/vertical scroll events (`zip_xy_to_scroll_mapper`) with axis inversion for rapid omnidirectional navigation.

### 3. Bluetooth & Power Management (`prj.conf`)
- **Dual Profile Pairing**: Limited to 2 paired devices (`CONFIG_BT_MAX_PAIRED=2`) for fast switching.
- **Low Latency BLE**: Tuned connection interval (7.5ms–11.25ms) with 2M PHY enabled.
- **BLE Congestion Mitigation**: Throttled PAW3395 report interval to 12ms (~83Hz) via `CONFIG_PAW3395_REPORT_INTERVAL_MIN=12`.
- **Power Optimization**:
  - Idle state after 10 seconds of inactivity.
  - Deep sleep triggered after 10 minutes (`CONFIG_ZMK_IDLE_SLEEP_TIMEOUT=600000`), wake-up on click/motion.
  - Reduced BLE transmit power to -4dBm (`CONFIG_BT_CTLR_TX_PWR_MINUS_4=y`).
- **Status Indicator**: Integrated RGB LED for connection and battery level feedback (`zmk-rgbled-widget`).

---

## 🖱 Keymap & Operation Layer Guide

### Layer 0: Default
- **LCLK / RCLK / MCLK**: Standard mouse buttons
- **LCLK + RCLK (Combo)**: Windows Copy (`Ctrl + C`)
- **Side Button (Tap)**: Navigate Back (`MB4`)
- **Side Button (Hold)**: Switch to **Layer 1**
- **Mode Button (Tap)**: Next Bluetooth profile (`BT_NXT`)
- **Mode Button (Hold)**: Switch to **Layer 2** (Bluetooth management)
- **Wheel**: Vertical scroll

### Layer 1: Navigation Layer (Hold Side Button)
- **Wheel**: Horizontal scroll (`SCRL_LEFT` / `SCRL_RIGHT`)
- **LCLK + RCLK (Combo)**: Windows Paste (`Ctrl + V`)
- **Mode Button**: Bootloader mode (`&bootloader`)

### Layer 2: BT & System Management (Hold Mode Button)
- **Left Click**: Select Profile 0 (`&bt BT_SEL 0`)
- **Right Click**: Select Profile 1 (`&bt BT_SEL 1`)
- **Middle Click**: Clear pairing for current profile (`&bt BT_CLR`)

---

## 🚀 Building & Flashing

1. Fork or clone this repository.
2. Ensure `config/` matches your board layout.
3. Push to `main` branch to trigger automated GitHub Actions build.
4. Download the generated `zmk.uf2` firmware artifact.
5. Put the XIAO BLE into bootloader mode and drop the `.uf2` file.

---

## 🙏 Acknowledgments & Credits

- **[badjeff/zmk-paw3395-driver](https://github.com/badjeff/zmk-paw3395-driver)**: The core PAW3395 optical sensor driver implementation for ZMK[cite: 7].
- **[elek1t](https://github.com/elek1t)**: Reference implementation for the direct 1:1 integer encoder scrolling behavior.
- **[zmk-rgbled-widget](https://github.com/caksoylar/zmk-rgbled-widget)**: On-board RGB LED battery and Bluetooth status indicator widget by caksoylar.
- **[ZMK Firmware](https://github.com/zmkfirmware/zmk)**: The underlying open-source keyboard and pointing device firmware project.
- Special thanks to the ZMK Firmware and custom input device communities.

---

## 📄 License
MIT License
