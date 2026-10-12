---
name: embassy-industrial-protocols
description: Industrial protocol drivers including RS485 DE hardware pin control, UART idle line detection, Modbus RTU/TCP, and CANopen in Embassy.
---

# Embassy Industrial Protocols Skill

This skill covers industrial automation protocols (RS485 half-duplex direction control, UART idle line framing, Modbus RTU, Modbus TCP) implemented asynchronously in Embassy.

---

## 1. RS485 Half-Duplex Direction Control (DE Pin)

RS485 differential communication requires toggling a Driver Enable (DE / RE) pin HIGH during transmission and LOW during reception.

```rust
use embassy_stm32::gpio::{Output, Level, Speed};
use embassy_stm32::usart::Uart;

pub struct Rs485Device<'d, U> {
    uart: U,
    de_pin: Output<'d>,
}

impl<'d, U> Rs485Device<'d, U>
where
    U: embedded_io_async::Write + embedded_io_async::Read,
{
    pub fn new(uart: U, de_pin: Output<'d>) -> Self {
        Self { uart, de_pin }
    }

    pub async fn transmit(&mut self, data: &[u8]) {
        // Assert Driver Enable HIGH
        self.de_pin.set_high();

        // Write packet data asynchronously
        self.uart.write_all(data).await.unwrap();

        // De-assert Driver Enable LOW
        self.de_pin.set_low();
    }
}
```

---

## 2. UART Idle Line Detection (`RingBuffered`)

In protocols like Modbus RTU, variable-length packet frames are separated by silent idle line intervals (>= 3.5 character times).

```rust
use embassy_stm32::usart::RingBufferedUartRx;

let mut rx = usart.into_ring_buffered(rx_dma_buf);

let mut frame_buf = [0u8; 256];
loop {
    // Wait until UART hardware detects an idle line condition
    let bytes_read = rx.read_idle(&mut frame_buf).await.unwrap();
    defmt::info!("Received Modbus Frame: {=[u8]:hex}", &frame_buf[..bytes_read]);
}
```

---

## 3. Async Modbus RTU Slave / Master

Modbus RTU uses CRC-16 (Modbus polynomial `0xA001`) error verification.

```rust
pub fn calculate_crc16(data: &[u8]) -> u16 {
    let mut crc = 0xFFFF;
    for &byte in data {
        crc ^= byte as u16;
        for _ in 0..8 {
            if (crc & 0x0001) != 0 {
                crc = (crc >> 1) ^ 0xA001;
            } else {
                crc >>= 1;
            }
        }
    }
    crc
}
```
