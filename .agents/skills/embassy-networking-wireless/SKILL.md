---
name: embassy-networking-wireless
description: Network stack management with embassy-net, Ethernet, WiFi, Bluetooth LE (trouble & nrf-softdevice), LoRaWAN (lora-rs), and PTP.
---

# Embassy Networking & Wireless Skill

This skill covers embedded networking with `embassy-net`, Ethernet/WiFi connectivity, Bluetooth Low Energy (BLE), and LoRa wireless protocols in Embassy.

---

## 1. Network Stack (`embassy-net`)

`embassy-net` provides zero-copy async networking implementing Ethernet, IPv4, IPv6, TCP, UDP, ICMP, and DHCP client capability.

### Architecture
1. **Network Driver:** Hardware Ethernet (MAC/PHY) or WiFi chip driver (e.g. `cyw43` for Pico W, `esp-radio` for ESP32, `enc28j60`, `w5500`).
2. **Network Stack (`Stack`):** Manages sockets, buffers, packet queues, and ARP/IP routing.
3. **Background Runner Task:** Drives `stack.run().await` loop in an Embassy task.

---

## 2. Setting Up an Async TCP / UDP Network Stack

```rust
use embassy_executor::Spawner;
use embassy_net::tcp::TcpSocket;
use embassy_net::{Config, Stack, StackResources};
use static_cell::StaticCell;

static STACK: StaticCell<Stack<EthernetDriver>> = StaticCell::new();
static RESOURCES: StaticCell<StackResources<3>> = StaticCell::new();

#[embassy_executor::task]
async fn net_task(stack: &'static Stack<EthernetDriver>) -> ! {
    stack.run().await
}

#[embassy_executor::main]
async fn main(spawner: Spawner) {
    let eth_driver = initialize_ethernet_driver();

    let config = Config::dhcpv4(Default::default());
    let seed = 0x1234_5678_9ABC_DEF0; // Hardware RNG seed

    let resources = RESOURCES.init(StackResources::new());
    let stack = STACK.init(Stack::new(eth_driver, config, resources, seed));

    // Spawn background stack runner
    spawner.spawn(net_task(stack)).unwrap();

    // Wait until DHCP assigns an IP address
    stack.wait_config_up().await;
    defmt::info!("Network UP! IP: {:?}", stack.config_v4());

    // Allocate TCP socket
    let mut rx_buffer = [0u8; 1024];
    let mut tx_buffer = [0u8; 1024];
    let mut socket = TcpSocket::new(stack, &mut rx_buffer, &mut tx_buffer);

    // Accept incoming connection on port 80
    socket.accept(80).await.unwrap();
    socket.write_all(b"HTTP/1.1 200 OK\r\nContent-Length: 13\r\n\r\nHello Embassy").await.unwrap();
    socket.flush().await.unwrap();
}
```

---

## 3. Bluetooth Low Energy (BLE)

Embassy supports BLE through two primary crates:
1. **`trouble` (Portable BLE Host Stack):**
   - Pure Rust BLE 4.x / 5.x Host stack.
   - Works on any MCU implementing `bt-hci` (nRF52, nRF54, RP2040/RP2350, ESP32, USB/Serial HCI controllers).
2. **`nrf-softdevice`:**
   - Wraps Nordic's official SoftDevice binary firmware for nRF52 MCUs.

### Example: BLE Advertising with `trouble`
```rust
use trouble_host::prelude::*;

// Define GATT services and characteristics
#[gatt_server]
struct Server {
    battery_service: BatteryService,
}

// Drive BLE HCI controller in background task...
```

---

## 4. LoRa & LoRaWAN (`lora-rs`)

`lora-rs` provides async LoRa and LoRaWAN networking integrated with Embassy:
- Supports Semtech SX126x, SX127x, and STM32WL wireless MCUs.
- Crates: `lora-phy`, `lora-modulation`, `lorawan-device`, `lorawan-encoding`.

```rust
use lora_phy::lorawan_async::*;
use lora_phy::sx126x::Sx1262;

// Configure LoRa radio pins and spi...
// Perform OTTA join & send uplink payload...
```

---

## 5. Precision Time Protocol (`embassy-ptp-driver`)

For high-precision IEEE 1588 time synchronization:
- `embassy-ptp-driver` provides hardware clock reading and rate adjustment interfaces independent of the network stack.
