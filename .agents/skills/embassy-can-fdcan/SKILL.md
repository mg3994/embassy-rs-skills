---
name: embassy-can-fdcan
description: Controller Area Network (CAN 2.0B & FDCAN), bit timing calculations, acceptance hardware filtering, FIFO queues, and bus error recovery in Embassy.
---

# Embassy CAN & FDCAN Bus Skill

This skill details implementing CAN 2.0B and CAN-FD (Flexible Data-Rate) bus communications across STM32, ESP32, and external CAN controllers.

---

## 1. CAN 2.0B vs FDCAN Comparison

| Feature | Classic CAN 2.0B | FDCAN |
| :--- | :--- | :--- |
| **Max Payload Size** | 8 bytes per frame | Up to 64 bytes per frame |
| **Max Bitrate** | 1 Mbps | 1 Mbps nominal / 5–8 Mbps data phase |
| **CRC Field** | 15-bit CRC | 17-bit / 21-bit CRC |
| **Identifier Types** | 11-bit Standard / 29-bit Extended | 11-bit Standard / 29-bit Extended |

---

## 2. Setting Up FDCAN with Acceptance Filters in Embassy

```rust
use embassy_stm32::can::{Can, Frame, StandardId, ExtendedId, filter::*};
use embassy_stm32::bind_interrupts;

bind_interrupts!(struct Irqs {
    FDCAN1_IT0 => embassy_stm32::can::IT0InterruptHandler<embassy_stm32::peripherals::FDCAN1>;
    FDCAN1_IT1 => embassy_stm32::can::IT1InterruptHandler<embassy_stm32::peripherals::FDCAN1>;
});

#[embassy_executor::main]
async fn main(_spawner: Spawner) {
    let p = embassy_stm32::init(Default::default());

    let mut can = Can::new(p.FDCAN1, p.PD0, p.PD1, Irqs);

    // Set Bitrate: 500 kbps Nominal, 2 Mbps Data
    can.set_bitrate(500_000);
    can.set_fd_bitrate(2_000_000);

    // Hardware Acceptance Filter: Accept only IDs 0x100 to 0x1FF
    can.set_filter(0, StandardFilter::range(0x100, 0x1FF));

    // Transmit FDCAN Frame (up to 64 bytes)
    let payload = [0xA5u8; 32];
    let frame = Frame::new_fd(StandardId::new(0x120).unwrap(), &payload).unwrap();
    can.write(&frame).await.unwrap();

    // Async Read Frame from RX FIFO
    let rx_frame = can.read().await.unwrap();
    defmt::info!("Received CAN ID: {:?}", rx_frame.id());
}
```
