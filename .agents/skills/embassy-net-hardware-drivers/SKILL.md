---
name: embassy-net-hardware-drivers
description: Hardware network driver integration for embassy-net including W5500/W5100S Wiznet SPI Ethernet, ENC28J60, and ESP-HOSTED co-processors.
---

# Embassy Network Hardware Drivers Skill

This skill details how to integrate external hardware network controllers (SPI Ethernet chips, WiFi co-processors) into the `embassy-net` stack.

---

## 1. Wiznet W5500 / W5100S SPI Ethernet Controller

The Wiznet W5500 connects over SPI and an EXTI interrupt pin to provide Ethernet MAC and PHY functionality.

### Initializing W5500 with `embassy-net-wiznet`
```rust
use embassy_net_wiznet::chip::W5500;
use embassy_net_wiznet::Device;

// Create SPI device with Chip Select
let spi_dev = SpiDevice::new(spi_bus, cs_pin);

// Initialize W5500 driver
let mac_addr = [0x02, 0x00, 0x00, 0x00, 0x00, 0x01];
let state = static_cell::make_static!(embassy_net_wiznet::State::<8, 8>::new());

let (device, runner) = embassy_net_wiznet::new(
    mac_addr,
    state,
    spi_dev,
    int_pin, // EXTI interrupt pin
    reset_pin,
    W5500,
).await.unwrap();

// Spawn runner task in background
spawner.spawn(wiznet_runner_task(runner)).unwrap();

// Pass `device` into `embassy_net::Stack::new(...)`
```

---

## 2. ESP-HOSTED WiFi Co-Processor

On MCUs like STM32 or nRF, Espressif ESP32 chips running ESP-HOSTED firmware can act as slave WiFi/Bluetooth co-processors over SPI or SDIO.
