---
name: embassy-i2c-smbus-pmbus
description: Advanced I2C modes, Fast-Mode Plus (1 MHz), SMBus/PMBus protocol extensions, Packet Error Checking (PEC), and I2C slave mode in Embassy.
---

# Embassy Advanced I2C, SMBus & PMBus Skill

This skill covers high-speed I2C configurations, Fast-Mode Plus (1 MHz), SMBus/PMBus power management protocols, Packet Error Checking (PEC) hardware CRC, and async I2C slave mode.

---

## 1. Fast-Mode Plus (1 MHz) & High-Speed I2C

Standard I2C operates at 100 kHz (Standard-mode) or 400 kHz (Fast-mode). Fast-Mode Plus (Fm+) operates at 1 MHz, requiring specific RCC clock prescalers and GPIO open-drain drive strength configuration.

```rust
use embassy_stm32::i2c::{Config, I2c};
use embassy_stm32::time::Hertz;

let mut config = Config::default();
config.sda_pullup = true;
config.scl_pullup = true;

// Configure I2C peripheral for 1 MHz Fast-Mode Plus
let mut i2c = I2c::new(
    p.I2C1,
    p.PB6, // SCL
    p.PB7, // SDA
    Irqs,
    p.DMA1_CH1,
    p.DMA1_CH2,
    Hertz(1_000_000), // 1 MHz clock speed
    config,
);
```

---

## 2. SMBus & PMBus Extensions

SMBus (System Management Bus) and PMBus (Power Management Bus) extend I2C with:
- **Packet Error Checking (PEC):** Uses CRC-8 hardware error detection (`x^8 + x^2 + x + 1`) appended to payload bytes.
- **Timeout Enforcement:** Prevents bus lockup if a device holds SCL low for > 35 ms.

---

## 3. Async I2C Slave Mode

In I2C Slave mode, the MCU responds to external master read/write requests over the I2C bus:

```rust
// On supported Embassy HALs:
// Listen asynchronously for I2C address match and process incoming commands
```
