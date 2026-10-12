---
name: embassy-time-driver-custom
description: Implementing custom hardware time drivers for embassy-time using the embassy-time-driver trait API.
---

# Custom Embassy Time Driver Skill

This skill explains how to implement custom hardware time drivers for unsupported targets or specialized timers using the `embassy-time-driver` trait interface.

---

## 1. The `embassy-time-driver` Trait Contract

When `embassy-time` is enabled, it requires a single static implementation of `embassy_time_driver::Driver`:

```rust
use embassy_time_driver::{Driver, AlarmHandle};

pub struct CustomTimeDriver {
    // Hardware timer registers or atomic tick counter
}

impl Driver for CustomTimeDriver {
    fn now(&self) -> u64 {
        // Return current hardware counter ticks (monotonic u64)
        get_hardware_ticks()
    }

    fn schedule_wake(&self, alarm: AlarmHandle, timestamp: u64) {
        // Program hardware timer compare interrupt for `timestamp`
        set_timer_compare(timestamp);
    }
}

// Register as global embassy-time driver
embassy_time_driver::time_driver_impl!(static DRIVER: CustomTimeDriver = CustomTimeDriver::new());
```
