---
name: embassy-embedded-hal-async
description: Writing portable, chip-agnostic driver crates for sensors, displays, and actuators using embedded-hal, embedded-hal-async, embedded-io-async, and embedded-storage.
---

# Embassy Embedded HAL & Generic Drivers Skill

This skill explains how to build and consume chip-agnostic driver crates using standard traits (`embedded-hal`, `embedded-hal-async`, `embedded-io-async`, `embedded-storage`).

---

## 1. Standard Embedded Trait Ecosystem

| Trait Crate | Purpose | Core Traits |
| :--- | :--- | :--- |
| **`embedded-hal`** | Synchronous hardware interfaces | `OutputPin`, `InputPin`, `SpiBus`, `I2c` |
| **`embedded-hal-async`** | Asynchronous hardware interfaces | `DelayNs`, `SpiBus`, `SpiDevice`, `I2c` |
| **`embedded-io-async`** | Asynchronous byte-stream I/O | `Read`, `Write`, `BufRead` |
| **`embedded-storage`** | Flash & non-volatile storage | `NorFlash`, `ReadNorFlash` |

---

## 2. Writing a Portable Sensor Driver Crate

By using `embedded-hal-async` traits instead of concrete HAL types (like `embassy_stm32::i2c::I2c`), driver crates work seamlessly across STM32, nRF, RP2040, ESP32, and host emulators.

```rust
use embedded_hal_async::delay::DelayNs;
use embedded_hal_async::i2c::I2c;

pub struct Bme280Driver<I2C> {
    i2c: I2C,
    address: u8,
}

impl<I2C, E> Bme280Driver<I2C>
where
    I2C: I2c<Error = E>,
{
    pub fn new(i2c: I2C, address: u8) -> Self {
        Self { i2c, address }
    }

    pub async fn read_temperature<D>(&mut self, delay: &mut D) -> Result<f32, E>
    where
        D: DelayNs,
    {
        // Reset/trigger measurement
        self.i2c.write(self.address, &[0xF4, 0x2E]).await?;

        // Wait 10ms asynchronously
        delay.delay_ms(10).await;

        // Read raw 16-bit register
        let mut buf = [0u8; 2];
        self.i2c.write_read(self.address, &[0xFA], &mut buf).await?;

        let raw = ((buf[0] as u16) << 8) | (buf[1] as u16);
        Ok(raw as f32 / 100.0)
    }
}
```

---

## 3. Instantiating Generic Drivers in Embassy

```rust
use embassy_time::Delay;

#[embassy_executor::main]
async fn main(_spawner: Spawner) {
    let p = embassy_stm32::init(Default::default());

    // Create Embassy async I2C peripheral
    let i2c = embassy_stm32::i2c::I2c::new(
        p.I2C1, p.PB6, p.PB7, Irqs, p.DMA1_CH1, p.DMA1_CH2, Hertz(400_000), Default::default(),
    );

    // Pass Embassy async I2C into generic driver
    let mut sensor = Bme280Driver::new(i2c, 0x76);

    // Pass Embassy Delay type into driver function
    let temp = sensor.read_temperature(&mut Delay).await.unwrap();
    defmt::info!("Temperature: {} C", temp);
}
```
