---
name: embassy-mcx-mspm0-imxrt
description: HAL drivers and peripheral configurations for NXP MCX-A (embassy-mcxa), TI MSPM0 (embassy-mspm0), and NXP i.MX RT (embassy-imxrt).
---

# Embassy MCX, MSPM0 & i.MX RT HALs Skill

This skill details Embassy support for NXP MCX-A, Texas Instruments MSPM0, and NXP i.MX RT microcontrollers.

---

## 1. NXP MCX-A Series (`embassy-mcxa`)

`embassy-mcxa` provides async HAL drivers based on the `nxp-pac` peripheral crate for NXP MCX-A ARM Cortex-M33 microcontrollers.

### Supported Peripherals
- Clocks, GPIO, ADC, I2C, I3C, LPUart, OSTimer, RTC, SPI, TRNG, WWDT, CTIMER, CRC, DMA.

### LPUart Setup Example
```rust
use embassy_mcxa::lpuart::{Config, LpUart};

let p = embassy_mcxa::init(Default::default());

let mut uart = LpUart::new(p.LPUART0, p.P0_2, p.P0_3, Irqs, p.DMA0_CH0, Default::default());
uart.write(b"Hello NXP MCX-A!\r\n").await.unwrap();
```

---

## 2. Texas Instruments MSPM0 Series (`embassy-mspm0`)

`embassy-mspm0` provides lightweight HAL drivers for TI MSPM0 and MSPS003 ultra-low-power Arm Cortex-M0+ microcontrollers.

### Supported Peripherals
- Clocks, GPIO, UART, SPI, I2C, Timers, ADC.

```rust
use embassy_mspm0::gpio::{Level, Output, Speed};

let p = embassy_mspm0::init(Default::default());
let mut led = Output::new(p.PA0, Level::Low, Speed::Low);
led.toggle();
```

---

## 3. NXP i.MX RT Crossover Series (`embassy-imxrt`)

`embassy-imxrt` supports high-performance Cortex-M7 i.MX RT series (MIMXRT685S, MIMXRT633S).

### Supported Peripherals
- CRC, DMA, GPIO, RNG, UART.
