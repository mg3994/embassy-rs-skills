---
name: embassy-rp-advanced
description: Raspberry Pi RP2040 and RP2350 (Arm Cortex-M33 & Hazard3 RISC-V) peripherals, PIO assembly program execution, dual-core spawner, and Pico W WiFi integration.
---

# Advanced Embassy RP2040 / RP2350 Skill

This skill covers the Raspberry Pi RP2040 and RP2350 (dual Arm Cortex-M33 or dual Hazard3 RISC-V cores), Programmable I/O (PIO) assembly integration, dual-core task spawning (`multicore`), and CYW43 WiFi on the Pico W.

---

## 1. RP2040 vs RP2350 Architecture Differences

| Feature | RP2040 | RP2350 |
| :--- | :--- | :--- |
| **CPU Core Architecture** | Dual Arm Cortex-M0+ @ 133 MHz | Dual Arm Cortex-M33 OR Dual Hazard3 RISC-V @ 150 MHz |
| **Target Triple (Arm)** | `thumbv6m-none-eabi` | `thumbv8m.main-none-eabihf` |
| **Target Triple (RISC-V)** | N/A | `riscv32imac-unknown-none-elf` |
| **SRAM** | 264 KiB | 520 KiB |
| **PIO Blocks** | 2 blocks (8 state machines) | 3 blocks (12 state machines) |
| **Security & Boot** | Basic bootrom | Secure Boot (Arm TrustZone-M), SHA-256 accelerator |

---

## 2. Programmable I/O (PIO) State Machines

RP2040/RP2350 PIO state machines allow building custom hardware protocols (WS2812 LED strings, VGA video drivers, custom parallel buses) in software without CPU overhead.

### Writing & Running a Custom PIO Assembly Program
Using the `pio-proc` macro or `pio` crate:

```rust
use embassy_rp::pio::{Pio, Config};
use pio_proc::pio_file;

// Load compiled PIO assembly file or inline assembly macro
let pio_program = pio_file!("src/ws2812.pio");

#[embassy_executor::main]
async fn main(_spawner: Spawner) {
    let p = embassy_rp::init(Default::default());

    // Initialize PIO block 0
    let mut pio = Pio::new(p.PIO0, Irqs);

    // Common program instance
    let program = pio.common.load_program(&pio_program.program);

    // Configure State Machine 0
    let mut cfg = Config::default();
    cfg.set_out_pins(&[&p.PIN_16]);
    cfg.use_program(&program, &[]);

    let mut sm = pio.sm0;
    sm.set_config(&cfg);
    sm.set_enable(true);

    // Push data to State Machine FIFO
    sm.tx().push(0x00FF0000); // Send RGB pixel value
}
```

---

## 3. Dual-Core Task Execution (`multicore`)

RP2040 and RP2350 allow running two independent Embassy executor instances on Core 0 and Core 1 simultaneously.

```rust
use embassy_executor::Executor;
use embassy_rp::multicore::{spawn_core1, Stack};
use static_cell::StaticCell;

static CORE1_STACK: Stack<4096> = Stack::new();
static EXECUTOR_CORE1: StaticCell<Executor> = StaticCell::new();

#[embassy_executor::main]
async fn main(spawner: Spawner) {
    let p = embassy_rp::init(Default::default());

    // Spawn Core 1 executor loop
    spawn_core1(p.CORE1, &CORE1_STACK, move || {
        let executor = EXECUTOR_CORE1.init(Executor::new());
        executor.run(|spawner| {
            spawner.spawn(core1_background_task()).unwrap();
        });
    });

    // Core 0 loop continues...
    loop {
        Timer::after_secs(1).await;
    }
}

#[embassy_executor::task]
async fn core1_background_task() {
    loop {
        defmt::info!("Hello from Core 1!");
        Timer::after_secs(2).await;
    }
}
```

---

## 4. Raspberry Pi Pico W WiFi (`cyw43`)

On the Raspberry Pi Pico W, the onboard LED and WiFi chip are connected over SPI to an Infineon CYW43439 chip.

### Initializing CYW43 & Blinking the Pico W LED
```rust
use cyw43_pio::PioSpi;
use embassy_rp::bind_interrupts;
use embassy_rp::gpio::{Level, Output};
use embassy_rp::peripherals::{DMA_CH0, PIO0};
use embassy_rp::pio::Pio;

bind_interrupts!(struct Irqs {
    PIO0_IRQ_0 => embassy_rp::pio::InterruptHandler<PIO0>;
});

#[embassy_executor::task]
async fn cyw43_runner(mut runner: cyw43::Runner<'static, Output<'static>, PioSpi<'static, PIO0, 0, DMA_CH0>>) -> ! {
    runner.run().await
}

#[embassy_executor::main]
async fn main(spawner: Spawner) {
    let p = embassy_rp::init(Default::default());

    let fw = include_bytes!("../cyw43-fw/43439A0.bin");
    let clm = include_bytes!("../cyw43-fw/43439A0_clm.bin");

    let pwr = Output::new(p.PIN_23, Level::Low);
    let cs = Output::new(p.PIN_25, Level::High);
    let mut pio = Pio::new(p.PIO0, Irqs);
    let spi = PioSpi::new(&mut pio.common, pio.sm0, pio.irq0, cs, p.PIN_24, p.PIN_29, p.DMA_CH0);

    let state = static_cell::make_static!(cyw43::State::new());
    let (mut control, runner) = cyw43::new(state, pwr, spi, fw).await;
    spawner.spawn(cyw43_runner(runner)).unwrap();

    control.init(clm).await;

    // Toggle onboard Pico W LED via CYW43 GPIO 0
    loop {
        control.gpio_set(0, true).await;
        Timer::after_millis(500).await;
        control.gpio_set(0, false).await;
        Timer::after_millis(500).await;
    }
}
```

---

## 5. Deployment Without Debug Probe (`picotool`)

For flashing RP2040/RP2350 boards over USB BOOTSEL mode:

```toml
# In .cargo/config.toml
[target.'cfg(all(target_arch = "arm", target_os = "none"))']
runner = "picotool load --update --verify --execute -t elf"
```
