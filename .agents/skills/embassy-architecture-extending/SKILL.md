---
name: embassy-architecture-extending
description: Internal Embassy architecture, metapac code generation, stm32-data, adding new microcontroller support, and writing HAL drivers.
---

# Embassy Architecture & Extending Skill

This skill explains Embassy's code generator architecture, metapac abstraction system (`stm32-data`), and step-by-step guidance on extending Embassy to support new microcontrollers, chip families, or custom hardware drivers.

---

## 1. Internal Architecture: The "Infinite Variant" Problem

Microcontroller families (such as STM32) contain hundreds of chip variants. Embassy avoids writing redundant peripheral code by recognizing that **peripheral hardware IP blocks** are identical across chip families.

### Metapac Data Flow Pipeline
```text
ST Microelectronics XML DBs (cubedb XML)
          │
          ▼
   stm32-data-sources (Raw vendor files)
          │
          ▼
      stm32-data (Rust generator parsing registers, IPs, pinouts)
          │
          ▼
    stm32-metapac (Auto-generated PAC crates per chip)
          │
          ▼
   embassy-stm32 HAL (HAL drivers conditionally included via cfg flags)
```

In `embassy-stm32/src/lib.rs`:
```rust
#[cfg(i2c)]
pub mod i2c;
```
In `embassy-stm32/src/i2c/mod.rs`:
```rust
#[cfg_attr(i2c_v2, path = "v2.rs")]
#[cfg_attr(i2c_v3, path = "v3.rs")]
pub struct I2c<'d, M: Mode> { ... }
```

---

## 2. Steps to Add Support for a New Microcontroller

To add support for a new MCU chip family or custom architecture to Embassy, follow this structured roadmap:

### Step 1: Probe & Debugger Support
Ensure the target chip is supported by `probe-rs`. If not, create or import a CMSIS-PACK definition.

### Step 2: Obtain or Generate SVD / PAC
1. Locate SVD or vendor peripheral definition files.
2. Run tools like `chiptool` or `svd2rust` to generate low-level register block types (Peripheral Access Crate).

### Step 3: Implement Basic Execution & Hello World
1. Write a minimal bare-metal application or fork `embassy-executor` for the new target architecture.
2. Verify hardware flashing and RTT logging via `defmt` + `probe-rs`.

### Step 4: Implement Hardware Timer & `embassy-time` Driver
1. Implement basic hardware timer counters and interrupts.
2. Implement the `embassy-time-driver` trait interface:

```rust
use embassy_time_driver::{Driver, AlarmHandle};

struct MyTimerDriver;

impl Driver for MyTimerDriver {
    fn now(&self) -> u64 {
        // Read hardware timer counter
    }

    fn schedule_wake(&self, alarm: AlarmHandle, timestamp: u64) {
        // Set hardware timer compare match interrupt
    }
}
```

### Step 5: Implement HAL Drivers & Traits
1. Implement GPIO, UART, SPI, and I2C peripherals.
2. Implement standard embedded traits (`embedded-hal`, `embedded-hal-async`, `embedded-io-async`).

---

## 3. Writing Custom Safe HAL Drivers

When creating custom async peripheral drivers:
1. Wrap raw unsafe PAC register reads/writes inside safe public APIs.
2. Register wakers before checking hardware status flags.
3. Pend wakers from hardware interrupt handlers.

```rust
use core::future::poll_fn;
use core::task::Poll;

pub struct CustomUart<'d> {
    _phantom: core::marker::PhantomData<&'d ()>,
}

impl<'d> CustomUart<'d> {
    pub async fn wait_for_data(&mut self) {
        poll_fn(|cx| {
            // Register current task waker
            WAKER_REGISTER.register(cx.waker());

            if HARDWARE_HAS_DATA() {
                Poll::Ready(())
            } else {
                Poll::Pending
            }
        }).await;
    }
}
```
