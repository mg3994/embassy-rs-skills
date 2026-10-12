---
name: embassy-hal-peripherals
description: Comprehensive hardware abstraction layer (HAL) usage, peripheral drivers, DMA channels, interrupts, and ISR binding across supported chip families.
---

# Embassy HAL & Peripherals Skill

This skill details how to interact with hardware peripherals across Embassy's supported microcontroller families (STM32, nRF, RP2040/RP2350, ESP32, MCX-A, MSPM0, iMXRT).

---

## 1. Supported HAL Ecosystem Overview

| Family | Crate | PAC Basis | Key Features |
| :--- | :--- | :--- | :--- |
| **STM32** | `embassy-stm32` | `stm32-metapac` | Support across F0–F7, G0/G4, H5/H7, L0–L5, U5, WB, WL. Flexible DMA/BDMA allocation. |
| **nRF** | `embassy-nrf` | `nrf-rs` PACs | nRF52, nRF5340, nRF54, nRF91 series. Low-power RTC timekeeping, PPI, UARTE, SPIM, SAADC. |
| **Raspberry Pi** | `embassy-rp` | `rp-pac` | RP2040 & RP235x dual-core support, PIO engine, USB, DMA, Hardware PWM, ADC, SPI, I2C. |
| **Espressif** | `esp-hal` | Espressif PACs | ESP32, ESP32-C3/C6, ESP32-S2/S3. Integrated WiFi/BLE stack support. |
| **NXP MCX** | `embassy-mcxa` | `nxp-pac` | MCX-A series (ADC, LPUart, SPI, I2C, I3C, TRNG, CTIMER, DMA). |
| **TI MSPM0** | `embassy-mspm0` | TI PACs | Ultra-low power Arm Cortex-M0+ peripherals. |
| **NXP i.MX RT** | `embassy-imxrt` | `mimxrt` PACs | High-performance Cortex-M7 peripherals (GPIO, DMA, UART, RNG, CRC). |

---

## 2. GPIO & EXTI Interrupts

### GPIO Output & Input
GPIO pins can be initialized using HAL-provided `Output` and `Input` types:

```rust
use embassy_stm32::gpio::{Input, Level, Output, Pull, Speed};

let p = embassy_stm32::init(Default::default());

// Output pin initialization
let mut led = Output::new(p.PB14, Level::Low, Speed::Low);
led.set_high();
led.toggle();

// Input pin initialization
let button = Input::new(p.PC13, Pull::Up);
if button.is_low() {
    defmt::info!("Button pressed!");
}
```

### Async EXTI Pin Interrupts
To wait asynchronously for edge transitions without busy-looping:

```rust
use embassy_stm32::exti::ExtiInput;
use embassy_stm32::gpio::Pull;

let mut button = ExtiInput::new(p.PC13, p.EXTI13, Pull::Up, Irqs);

loop {
    button.wait_for_falling_edge().await;
    defmt::info!("Button pressed!");
    button.wait_for_rising_edge().await;
    defmt::info!("Button released!");
}
```

---

## 3. Peripheral Binding & Interrupt Handlers

Embassy uses the `bind_interrupts!` macro to bind hardware interrupts safely to driver handlers:

```rust
use embassy_stm32::bind_interrupts;
use embassy_stm32::usart::{self, Uart};
use embassy_stm32::peripherals;

bind_interrupts!(struct Irqs {
    USART1 => usart::InterruptHandler<peripherals::USART1>;
});

#[embassy_executor::main]
async fn main(_spawner: Spawner) {
    let p = embassy_stm32::init(Default::default());

    let mut config = usart::Config::default();
    config.baudrate = 115200;

    let mut usart = Uart::new(
        p.USART1,
        p.PA10, // RX
        p.PA9,  // TX
        Irqs,
        p.DMA1_CH1, // TX DMA
        p.DMA1_CH2, // RX DMA
        config,
    ).unwrap();

    usart.write(b"Hello Embassy!\r\n").await.unwrap();
}
```

### Manual ISRs Alongside Embassy
If microsecond latency is needed, manual interrupt handlers can co-exist:

```rust
use embassy_stm32::interrupt;

#[interrupt]
fn USART1() {
    // Custom ultra-low latency ISR logic
}
```

---

## 4. Serial Communication (UART, SPI, I2C)

### Async SPI Transfer
```rust
use embassy_stm32::spi::{Config, Spi};
use embassy_stm32::time::Hertz;

let mut spi_config = Config::default();
spi_config.frequency = Hertz(10_000_000);

let mut spi = Spi::new(
    p.SPI1,
    p.PA5, // SCK
    p.PA7, // MOSI
    p.PA6, // MISO
    p.DMA1_CH3,
    p.DMA1_CH4,
    spi_config,
);

let tx_buf = [0xA5, 0x01, 0x02];
let mut rx_buf = [0u8; 3];
spi.transfer(&mut rx_buf, &tx_buf).await.unwrap();
```

### Async I2C Master
```rust
use embassy_stm32::i2c::{Config, I2c};
use embassy_stm32::time::Hertz;

let mut i2c = I2c::new(
    p.I2C1,
    p.PB6, // SCL
    p.PB7, // SDA
    Irqs,
    p.DMA1_CH5,
    p.DMA1_CH6,
    Hertz(100_000),
    Default::default(),
);

let mut read_buf = [0u8; 2];
i2c.write_read(0x68, &[0x3B], &mut read_buf).await.unwrap();
```

---

## 5. Direct Memory Access (DMA) & Special RAM Regions

DMA offloads memory transfers from the CPU.

### STM32 BDMA / Dedicated Memory Regions (e.g. STM32H7)
On chips like STM32H7, BDMA or certain peripherals can only access specific RAM areas (D2/D3 SRAM).

```rust
// Place DMA buffers in accessible RAM section
#[unsafe(link_section = ".sram3")]
static mut DMA_BUF: [u8; 1024] = [0u8; 1024];
```

If memory is incorrectly mapped for DMA, transfers will trigger hardware errors such as `DMA: error on BDMA@... channel`.

---

## 6. Hardware Compatibility & Peripheral Splitting

### Assigning Peripherals
When passing peripheral instances to sub-modules, use generic pin traits (`AnyPin`) or generic peripheral traits to keep driver functions reusable across hardware revisions.
