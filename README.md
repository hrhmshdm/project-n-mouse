# Project N-Mouse (ZMK Firmware)

Custom ultra-compact wireless mouse firmware powered by **ZMK Firmware (v0.3)** and Zephyr RTOS.

Designed around the **Seeed Studio XIAO nRF52840 (BLE)**, featuring a **PixArt PAW3395** optical sensor, **TTC Green Core mechanical rotary encoder**, and **Kailh Silent Micro Switches**.

---

## 💡 Concept & Design Inspiration

The physical design and transparent aesthetic of this project were inspired by the charging case of **Nothing Ear**. It aims to bring that minimalist, transparent industrial design language into an open-source, ultra-compact wireless mouse.
<img width="3359" height="2142" alt="Image" src="https://github.com/user-attachments/assets/516aef2c-0d70-4ce6-85a7-541295b95843" />
> ⚠️ **Disclaimer & Project Notice**: 
> - This is an independent, non-commercial DIY maker project. It is **not affiliated with, endorsed by, or associated with Nothing Technology Limited** in any way.
> - **Beginner's Project Notice**: This is my very first custom hardware/firmware project. As a hobbyist build, please be aware that the repository file structure, control logic, and electronic circuit design patterns may deviate from standard engineering conventions or best practices. Use or reference at your own discretion.

---

## 🛠 Hardware Specifications & Pinout

Schematic diagrams and PCB assembly reference photos are available in the [`hardware/`](hardware/) directory.

### MCU & Indicators
- **MCU**: Seeed Studio XIAO nRF52840 (BLE)
- **Status LED**: Integrated 3-color RGB LED mapped via `zmk-rgbled-widget` (`led-red`, `led-green`, `led-blue`)
- **Battery**: 3.7V 200mAh Li-Po (601230, JST 2.0 2-Pin)

### Pin Assignment Table

| Pin | Function / Target | Component / Description | Type |
| :--- | :--- | :--- | :--- |
| **D0** | Left Click (SW1) | Kailh Silent Micro Switch (6x6x7.3mm) | Direct GPIO (Active Low) |
| **D1** | Right Click (SW2) | Kailh Silent Micro Switch (6x6x7.3mm) | Direct GPIO (Active Low) |
| **D2** | Middle Click (SW3) | Short Tactile Switch (4.5x4.5x4.3mm, TS-4501) | Direct GPIO (Active Low) |
| **D3** | Side Button (SW4) | Kailh Silent Micro Switch (6x6x7.3mm) | Direct GPIO (Active Low) |
| **D4** | Mode Button (SW5) | Tall Tactile Switch (4.5x4.5x5.0mm, TS-4501) | Direct GPIO (Active Low) |
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
- **Switches (L / R / Side)**: Kailh Silent Micro Switch (6x6x7.3mm, 2-pin) × 3
- **Middle Click Switch**: Short Tactile Switch (4.5x4.5x4.3mm, TS-4501)
- **Mode Switch**: Tall Tactile Switch (4.5x4.5x5.0mm, TS-4501)
- **Power Switch**: Subminiature Slide Switch (MSK12C01 series)
- **Encoder**: TTC Green Core Mechanical Rotary Encoder (12 Pulses / 24 Detents)
- **Battery**: 3.7V 200mAh 601230 Rechargeable Li-Po Battery

### Recommended Passives (0805 SMD)
- **Resistors**: 10kΩ ±1% (R1), 4.7Ω ±1% (R2), 2MΩ ±1% (R3)[cite: 1]
- **Capacitors**: 100nF 50V X7R (C2, C4, C5), 10µF 35V X5R (C3, C6), 47µF 10V X5R (C8), 1µF 50V X7R (C9, C11), 4.7µF 50V X5R (C1)[cite: 1]

---

## ⚙️ Key Technical Implementations

### 1. Direct Integer Wheel Scroll (`zmk,behavior-encoder-mouse-scroll`)
- **Profile**: Configured for 12 pulses / 24 detents matching the TTC Green Core hardware spec (`steps = <12>; triggers-per-rotation = <24>;`).
- **Low-Latency Logic**: Direct integer HID report dispatch (`±1` per physical notch), eliminating latency and notch misreads caused by floating-point software inertia layers.
- **Reference**: Ported and adapted from **elekit's (`elek1t`) encoder scroll behavior** for ZMK v0.3 compatibility.

### 2. Sensor & Motion Processing (PAW3395 Driver)
- **Driver**: Powered by and adapted from **[badjeff/zmk-paw3395-driver](https://github.com/badjeff/zmk-paw3395-driver)**.
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

> ℹ️ **Note**: This keymap layout is tailored specifically for personal office and document workflows on a Windows PC. It can be freely customized or remapped to suit your preferences using standard ZMK keymap behaviors.

### Layer 0: Default
- **LCLK / RCLK / MCLK**: Standard mouse buttons
- **LCLK + RCLK (Combo)**: Windows Copy (`Ctrl + C`)
- **Side Button (Tap)**: Navigate Back (`MB4`)
- **Side Button (Hold)**: Momentary switch to **Layer 1**
- **Mode Button (SW5 - Tap)**: Switch to next Bluetooth profile (`BT_NXT`)
- **Mode Button (SW5 - Hold 2s)**: Clear pairing for current profile (`BT_CLR`)
- **Wheel**: Vertical scroll (1:1 direct integer notch)

### Layer 1: Navigation Layer (Hold Side Button)
- **Wheel**: Horizontal scroll (`SCRL_LEFT` / `SCRL_RIGHT`)
- **LCLK**: Navigate Forward (`MB5`)
- **MCLK**: Task View (`Win + Tab`)
- **RCLK (Tap)**: Windows Paste (`Ctrl + V`)
- **RCLK (Hold)**: Momentary switch to **Layer 2 (System & BT)**
- **LCLK + RCLK (Combo)**: Snipping Tool (`Win + Shift + S`) via dedicated macro
- **Mode Button (SW5)**: Enter DFU Bootloader mode (`&bootloader`)

### Layer 2: BT & System Management (Hold Side + Hold Right Click)
- **Left Click**: Select Profile 0 (`&bt BT_SEL 0`)
- **Middle Click**: Select Profile 1 (`&bt BT_SEL 1`)
- **Sensor Motion**: Omnidirectional high-speed drag scroll

---

## 🚀 Building & Flashing

1. Fork or clone this repository.
2. Ensure `config/` matches your board layout.
3. Push to `main` branch to trigger automated GitHub Actions build.
4. Download the generated `zmk.uf2` firmware artifact.
5. Put the XIAO BLE into bootloader mode and drag-and-drop the `.uf2` file.

---

## 🙏 Acknowledgments & Credits

- **[badjeff/zmk-paw3395-driver](https://github.com/badjeff/zmk-paw3395-driver)**: The core PAW3395 optical sensor driver implementation for ZMK.
- **[elek1t](https://github.com/elek1t)**: Reference implementation for the direct 1:1 integer encoder scrolling behavior.
- **[zmk-rgbled-widget](https://github.com/caksoylar/zmk-rgbled-widget)**: On-board RGB LED battery and Bluetooth status indicator widget by caksoylar.
- **[ZMK Firmware](https://github.com/zmkfirmware/zmk)**: The underlying open-source keyboard and pointing device firmware project.
- Special thanks to the ZMK Firmware and custom input device communities.

---

## 📄 License
MIT License

<img width="3024" height="4032" alt="Image" src="https://github.com/user-attachments/assets/09af6861-6cd9-40f4-ad3c-32fd9120923d" />
<img width="2732" height="2685" alt="Image" src="https://github.com/user-attachments/assets/e35d8247-265b-4b80-95ef-54f8910618aa" />
