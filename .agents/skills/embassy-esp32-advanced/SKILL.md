---
name: embassy-esp32-advanced
description: Espressif ESP32 series (ESP32, ESP32-C3, C6, S2, S3), esp-hal integration, Xtensa vs RISC-V targets, esp-wifi, and esp-generate tooling.
---

# Advanced Embassy ESP32 Skill

This skill covers using Embassy executor and async futures with Espressif Systems microcontrollers (ESP32, ESP32-C3, ESP32-C6, ESP32-S2, ESP32-S3) via `esp-hal` and `esp-wifi`.

---

## 1. Supported Architecture Targets

Espressif chips fall into two CPU target categories:

| Chip Family | Core Architecture | Target Triple |
| :--- | :--- | :--- |
| **ESP32** | Dual-core Xtensa LX6 | `xtensa-esp32-none-elf` |
| **ESP32-S2 / S3** | Single/Dual-core Xtensa LX7 | `xtensa-esp32s2-none-elf` / `xtensa-esp32s3-none-elf` |
| **ESP32-C3 / C6** | Single-core 32-bit RISC-V | `riscv32imc-unknown-none-elf` / `riscv32imac-unknown-none-elf` |

---

## 2. Setting Up an `esp-hal` Project with Embassy

`esp-hal` provides native Embassy-compatible executors and timer drivers.

### Project Generation via `esp-generate`
To generate a new workspace:
```bash
cargo install esp-generate
esp-generate --chip esp32c3 my-esp-app
```

### `main.rs` with `esp-hal` Embassy Executor

```rust
#![no_std]
#![no_main]

use esp_backtrace as _;
use esp_hal::clock::ClockControl;
use esp_hal::peripherals::Peripherals;
use esp_hal::system::SystemControl;
use esp_hal::timer::timg::TimerGroup;
use esp_hal::entry;
use esp_hal_embassy::main;
use embassy_time::Timer;

#[main]
async fn main(spawner: embassy_executor::Spawner) {
    let peripherals = Peripherals::take();
    let system = SystemControl::new(peripherals.SYSTEM);
    let clocks = ClockControl::max(system.clock_control).freeze();

    // Initialize esp-hal timer driver for embassy-time
    let timg0 = TimerGroup::new(peripherals.TIMG0, &clocks);
    let init = esp_hal_embassy::init(&clocks, timg0.timer0);

    spawner.spawn(blink_task(peripherals.GPIO8.into())).unwrap();

    loop {
        Timer::after_secs(2).await;
        defmt::info!("Main heartbeat");
    }
}

#[embassy_executor::task]
async fn blink_task(mut pin: esp_hal::gpio::AnyPin) {
    let mut led = esp_hal::gpio::Output::new(pin, esp_hal::gpio::Level::Low);
    loop {
        led.toggle();
        Timer::after_millis(500).await;
    }
}
```

---

## 3. WiFi & BLE Stack Integration (`esp-wifi`)

`esp-wifi` provides async networking and Bluetooth LE controllers for ESP32 devices running on top of Embassy.

```rust
use esp_wifi::wifi::{WifiController, WifiDevice, WifiEvent, WifiModeConfiguration, WifiState};
use esp_wifi::EspWifiInitFor;

// Initialize WiFi radio hardware
let init = esp_wifi::initialize(
    EspWifiInitFor::Wifi,
    timer,
    rng,
    peripherals.RADIO_CLK,
    &clocks,
).unwrap();

let (wifi_interface, controller) = esp_wifi::wifi::new_with_mode(&init, peripherals.WIFI, WifiDevice::Sta).unwrap();
```

---

## 4. Flash Tooling & Runners (`espflash`)

Deploying to ESP32 boards uses `espflash`:

```toml
# In .cargo/config.toml
[target.'cfg(any(target_arch = "riscv32", target_arch = "xtensa"))']
runner = "espflash flash --monitor"
```
