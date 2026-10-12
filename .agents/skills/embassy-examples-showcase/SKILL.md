---
name: embassy-examples-showcase
description: Real-world Embassy production applications showcase (Embassy in the Wild) including RMK, Printhor, ProtoV MINI, Opentrig, and Card/IO.
---

# Embassy Examples & Production Showcase Skill

This skill documents real-world open-source production projects built on Embassy ("Embassy in the Wild") serving as architecture references.

---

## Real-World Production Projects

### 1. RMK (Rust Keyboard Firmware)
- **Repository:** `https://github.com/haobogu/rmk/`
- **Target MCUs:** STM32, RP2040, nRF52, ESP32
- **Key Features:** Wireless BLE keyboard firmware with layer support, Vial real-time key remapping, and debounced key matrix scanning driven by Embassy tasks.

### 2. Printhor (High-Reliability 3D Printer Firmware)
- **Repository:** `https://github.com/cbruiz/printhor/`
- **Target MCUs:** STM32
- **Key Features:** Highly reliable, concurrent 3D printer controller executing G-code parsing, motion planning, thermal PID loops, and stepper motor control via async Embassy tasks.

### 3. ProtoV MINI (USB-C Breadboard Power Supply)
- **Repository:** `https://github.com/thataquarel/protovolt`
- **Target MCU:** RP2040
- **Key Features:** Dual-channel USB Power Delivery (PD) lab power supply running `embedded-graphics` on an OLED display with real-time ADC voltage/current monitoring.

### 4. Opentrig (Particle Physics Event Trigger & DAQ)
- **Repository:** `https://github.com/Dawson-HEP/opentrig/`
- **Target MCU:** RP2040
- **Key Features:** High-speed particle event trigger and data acquisition system tested at DESY II Test Beam Facility using Embassy's async EXTI interrupt handling.

### 5. Card/IO (Open Source ECG Firmware)
- **Repository:** `https://github.com/card-io-ecg/card-io-fw`
- **Target MCUs:** ESP32-S3, ESP32-C6
- **Key Features:** Medical-grade Electrocardiogram (ECG) bio-potential sensor sampling, digital filtering, and BLE telemetry transmission.

### 6. Embassy Clock (Layered UI Architecture Reference)
- **Repository:** `https://github.com/CarlKCarlK/clock`
- **Target MCU:** Raspberry Pi Pico (RP2040)
- **Key Features:** Modular bare-metal clock demonstrating clean layered task separation (`Display -> Blinker -> Clock UI`) and Renode emulation support.
