---
name: embassy-ch32-py32-mpfs
description: Community HALs for WCH CH32 RISC-V (ch32-hal), Puya PY32 (py32-hal), and Microchip PolarFire FPGA RISC-V SoC (mpfs-hal).
---

# Embassy CH32, PY32 & PolarFire (MPFS) HALs Skill

This skill details using Embassy with community-maintained HALs for WCH RISC-V microcontrollers, Puya low-cost ARM Cortex-M0+ chips, and Microchip PolarFire FPGA RISC-V SoCs.

---

## 1. WCH CH32 32-Bit RISC-V Series (`ch32-hal`)

`ch32-hal` provides Embassy-compatible async HAL drivers for WCH CH32V series microcontrollers (e.g. CH32V003, CH32V203, CH32V307).

- **Repository:** `https://github.com/ch32-rs/ch32-hal`
- **Target Triple:** `riscv32ec-unknown-none-elf` (for CH32V003) / `riscv32imac-unknown-none-elf`
- **Supported Peripherals:** GPIO, USART, SPI, I2C, Timers, ADC, DMA.

---

## 2. Puya Semiconductor PY32 Series (`py32-hal`)

`py32-hal` provides Embassy support for ultra-low-cost Puya PY32 ARM Cortex-M0+ microcontrollers (e.g. PY32F002A, PY32F030).

- **Repository:** `https://github.com/py32-rs/py32-hal`
- **Target Triple:** `thumbv6m-none-eabi`
- **Supported Peripherals:** GPIO, USART, SPI, I2C, ADC.

---

## 3. Microchip PolarFire SoC (`mpfs-hal`)

`mpfs-hal` supports Microchip PolarFire SoC FPGA chips featuring a 64-bit multi-core RISC-V CPU subsystem.

- **Repository:** `https://github.com/AlexCharlton/mpfs-hal`
- **Target Triple:** `riscv64gc-unknown-none-elf`
