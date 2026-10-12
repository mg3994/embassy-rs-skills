---
name: embassy-trouble-ble
description: Pure-Rust portable Bluetooth Low Energy (BLE) host stack (trouble), bt-hci driver interface, GATT server/client, and L2CAP channels.
---

# Embassy TrouBLE BLE Host Stack Skill

This skill details using `trouble`—a portable, pure-Rust Bluetooth Low Energy (BLE 4.x/5.x) Host stack designed specifically for Embassy and `no_std` environments.

---

## 1. `trouble` Architecture & HCI Controller Abstraction

Unlike vendor-locked binary blobs, `trouble` runs on top of any Bluetooth controller implementing the `bt-hci` interface (nRF52/54, RP2040 CYW43, ESP32, USB/UART dongles).

```text
[ Application Task / GATT Services ]
               │
               ▼
   trouble (Pure-Rust BLE Host)
               │
               ▼
   bt-hci (HCI Transport Interface)
               │
               ▼
[ Hardware BLE Controller / Radio ]
```

---

## 2. Implementing a GATT Server with `trouble`

```rust
use trouble_host::prelude::*;

#[gatt_server]
pub struct BatteryServer {
    pub battery_service: BatteryService,
}

#[gatt_service(uuid = "180f")]
pub struct BatteryService {
    #[characteristic(uuid = "2a19", read, notify)]
    pub level: u8,
}

#[embassy_executor::main]
async fn main(spawner: Spawner) {
    let controller = initialize_hci_controller().await;

    let mut resources = trouble_host::new_resources::<1, 2>();
    let stack = trouble_host::new(controller, &mut resources);

    let mut server = BatteryServer::new();

    // Start BLE Advertisements
    let mut adv = stack.advertiser();
    adv.set_data(b"\x02\x01\x06\x03\x03\x0f\x18"); // Battery Service UUID
    adv.start().await.unwrap();

    loop {
        let conn = adv.accept().await.unwrap();
        defmt::info!("BLE Client Connected!");

        // Handle GATT requests or notify battery level changes
        server.battery_service.level.set(&conn, &85).await.unwrap();
    }
}
```
