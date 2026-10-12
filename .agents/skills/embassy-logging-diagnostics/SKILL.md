---
name: embassy-logging-diagnostics
description: Logging infrastructure, defmt-rtt, embassy-usb-logger, standard log crate bridging, panic handlers (panic-probe, panic-reset), and RTT buffer tuning.
---

# Embassy Logging & Diagnostics Skill

This skill details logging infrastructure, defmt-rtt transport, streaming logs over USB CDC-ACM, bridging standard `log` crate macros, and panic handlers.

---

## 1. Logging Framework Comparison

| Framework | Overhead | Deferred Formatting | Transport | Best For |
| :--- | :--- | :--- | :--- | :--- |
| **`defmt`** | Very Low (indexes only) | Yes (decoded on host) | RTT / USB / UART | Production `no_std` firmware |
| **`log`** | Medium (formats strings on MCU) | No | UART / RTT | Portable crates & host testing |
| **`env_logger`** | High | No | stdout / stderr | `std` target host binaries |

---

## 2. Streaming Logs Over USB (`embassy-usb-logger`)

Stream `defmt` or `log` messages directly over a Virtual COM Port (CDC-ACM) without needing an external debug probe.

```rust
use embassy_usb_logger::with_custom_style;

#[embassy_executor::main]
async fn main(spawner: Spawner) {
    let p = embassy_stm32::init(Default::default());

    // Initialize USB driver
    let driver = embassy_stm32::usb::Driver::new_fs(p.USB, Irqs, p.PA12, p.PA11);

    // Run USB logger task in background
    embassy_usb_logger::run!(1024, log::LevelFilter::Info, driver);

    log::info!("System booted! Logging active over USB serial port.");
}
```

---

## 3. Panic Handlers (`panic-probe` vs `panic-reset`)

### `panic-probe` (Development)
Prints panic location and backtrace over RTT to `probe-rs`, then halts the CPU:

```rust
use panic_probe as _;
```

### `panic-reset` (Production)
Triggers an immediate system microcontroller software reset (`SCB::sys_reset()`) upon panic:

```rust
use panic_reset as _;
```
