---
name: embassy-futures-utilities
description: Future composition utilities (join, select, yield_now) from embassy-futures and cancellation safety in embedded async Rust.
---

# Embassy Futures & Utilities Skill

This skill explains how to compose async operations using `embassy-futures` (`join`, `select`, `yield_now`) and manage cancellation safety in resource-constrained embedded tasks.

---

## 1. Concurrency Primitives (`embassy-futures`)

`embassy-futures` provides zero-allocation `no_std` combinators for driving multiple futures concurrently.

| Utility | Behavior |
| :--- | :--- |
| **`join(fut1, fut2)`** | Runs both futures concurrently until **both** complete. Returns `(T1, T2)`. |
| **`select(fut1, fut2)`** | Runs both futures concurrently until **either** completes. Drops the uncompleted future and returns `Either::First(T1)` or `Either::Second(T2)`. |
| **`yield_now()`** | Cooperatively yields execution back to the executor, placing the current task at the back of the run queue. |

---

## 2. Examples

### Running Tasks Concurrently with `join`
```rust
use embassy_futures::join::join;

async fn run_usb_device_and_serial(usb: UsbDevice, serial: SerialDriver) {
    let usb_fut = usb.run();
    let serial_fut = serial.process_packets();

    // Runs both futures until both return
    join(usb_fut, serial_fut).await;
}
```

### Implementing Timeouts with `select`
```rust
use embassy_futures::select::{select, Either};
use embassy_time::{Duration, Timer};

async fn read_with_timeout(sensor: &mut Sensor) -> Result<Data, TimeoutError> {
    let read_fut = sensor.read_sample();
    let timeout_fut = Timer::after(Duration::from_secs(2));

    match select(read_fut, timeout_fut).await {
        Either::First(sample) => Ok(sample),
        Either::Second(_) => Err(TimeoutError),
    }
}
```

---

## 3. Cancellation Safety in Embedded Async Code

When `select` completes, the losing future is immediately dropped.

- **Cancellation Safe Operations:** Simple status flag reads, atomic operations, and timers.
- **Non-Cancellation Safe Operations:** Multi-step DMA transfers or hardware state machine routines where dropping the future mid-transfer leaves hardware in an inconsistent state.
