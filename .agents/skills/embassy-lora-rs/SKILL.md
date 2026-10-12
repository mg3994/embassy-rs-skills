---
name: embassy-lora-rs
description: Comprehensive LoRaWAN and LoRa physical layer stack using lora-rs, Semtech SX126x/SX127x, STM32WL, OTAA/ABP join, and async packet transmission.
---

# Embassy LoRaWAN & LoRa PHY (`lora-rs`) Skill

This skill covers implementing long-range wireless communication using `lora-rs` (`lora-phy`, `lora-modulation`, `lorawan-device`, `lorawan-encoding`) with Embassy.

---

## 1. `lora-rs` Ecosystem Architecture

`lora-rs` is modularized into four zero-allocation crates:

| Crate | Responsibility |
| :--- | :--- |
| **`lora-phy`** | Async hardware radio drivers (SX1261/2, SX1272/6/7/8/9, STM32WL). |
| **`lora-modulation`** | Spreading factor (SF7-SF12), bandwidth, coding rate definitions. |
| **`lorawan-device`** | Async LoRaWAN 1.0.4/1.1 Class A/C state machine and MAC commands. |
| **`lorawan-encoding`** | Cryptographic payload encoding (AES-128 MIC verification). |

---

## 2. OTAA (Over-The-Air Activation) Network Join & Frame Send

```rust
use lora_phy::iv::Stm32wlBoard;
use lora_phy::lorawan_async::*;
use lora_phy::sx126x::Sx1262;
use lora_phy::LoRa;

#[embassy_executor::main]
async fn main(_spawner: Spawner) {
    let p = embassy_stm32::init(Default::default());

    // Initialize LoRa radio hardware driver
    let lora_radio = Sx1262::new(spi_dev, cs_pin, reset_pin, dio1_pin, busy_pin, board_config);
    let mut lora = LoRa::new(lora_radio, true, Delay).await.unwrap();

    // OTAA Credentials
    let dev_eui = [0x00, 0x11, 0x22, 0x33, 0x44, 0x55, 0x66, 0x77];
    let join_eui = [0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00];
    let app_key = [0x01, 0x02, 0x03, 0x04, 0x05, 0x06, 0x07, 0x08, 0x09, 0x0A, 0x0B, 0x0C, 0x0D, 0x0E, 0x0F, 0x10];

    // Join LoRaWAN Network
    let mut device = LoRaWAN::new(lora, Region::US915);
    device.join_otaa(dev_eui, join_eui, app_key).await.unwrap();
    defmt::info!("Joined LoRaWAN network successfully!");

    // Transmit Uplink Payload
    let payload = [0x12, 0x34, 0x56];
    device.send(&payload, 1, true).await.unwrap(); // Port 1, confirmed
}
```
