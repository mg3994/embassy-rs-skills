---
name: embassy-resource-sharing
description: Resource sharing across tasks, concurrency models, embassy-sync synchronization primitives (Mutex, Channel, Signal, Watch), shared SPI/I2C buses, and task composition.
---

# Embassy Resource Sharing & Concurrency Skill

This skill explains how to share peripherals, state, and buffers safely across tasks using Embassy's `embassy-sync` crate, as well as choosing between task-based vs future-based concurrency (`join`/`select`).

---

## 1. Concurrency Models: Tasks vs Futures

When building Embassy applications, concurrent operations can be handled in two ways:

1. **Multiple Tasks (`#[embassy_executor::task]`):**
   - Each task has a static memory allocation for its state machine.
   - Faster wake latency: the executor wakes *only* the woken task.
   - Ideal for long-running decoupled loops (e.g. sensor polling, networking).

2. **Multiple Futures in One Task (`embassy_futures::join` / `select`):**
   - Shares a single task stack allocation.
   - Easy borrowing across futures within the same main loop.
   - Ideal for tightly coupled operations (e.g. running USB device driver alongside CDC state loop).

```rust
use embassy_futures::join::join;

#[embassy_executor::main]
async fn main(spawner: Spawner) {
    // Run two futures concurrently inside a single task without extra task allocations
    join(usb_fut, echo_fut).await;
}
```

---

## 2. Synchronization Primitives (`embassy-sync`)

`embassy-sync` provides zero-alloc, `no_std` synchronization primitives configured with a RawMutex implementation (e.g., `ThreadModeRawMutex` for single-core thread-mode execution, or `CriticalSectionRawMutex` for cross-interrupt access).

### RawMutex Types
- **`ThreadModeRawMutex`:** Zero cost, fast, usable only in thread mode (non-ISR).
- **`CriticalSectionRawMutex`:** Disables interrupts briefly when locking. Safe for sharing data between thread-mode tasks and hardware ISRs or across multiple cores.

---

## 3. Peripheral & State Sharing Patterns

### Pattern 1: Async Mutex (`Mutex`)
Use `Mutex` when multiple tasks need exclusive, sequential access to a peripheral or state object.

```rust
use embassy_sync::blocking_mutex::raw::ThreadModeRawMutex;
use embassy_sync::mutex::Mutex;
use embassy_stm32::gpio::{AnyPin, Level, Output};

type LedMutex = Mutex<ThreadModeRawMutex, Option<Output<'static>>>;
static LED: LedMutex = Mutex::new(None);

#[embassy_executor::main]
async fn main(spawner: Spawner) {
    let p = embassy_stm32::init(Default::default());
    let led_pin = Output::new(p.PB14, Level::Low, Default::default());

    // Initialize global static reference
    {
        *(LED.lock().await) = Some(led_pin);
    }

    spawner.spawn(toggle_task()).unwrap();
}

#[embassy_executor::task]
async fn toggle_task() {
    loop {
        {
            let mut guard = LED.lock().await;
            if let Some(led) = guard.as_mut() {
                led.toggle();
            }
        } // Mutex guard dropped here, releasing peripheral
        Timer::after_millis(500).await;
    }
}
```

### Pattern 2: Multi-Producer Channel (`Channel`)
Channels decouple command dispatchers from hardware drivers.

```rust
use embassy_sync::blocking_mutex::raw::ThreadModeRawMutex;
use embassy_sync::channel::{Channel, Sender};

enum Command {
    SetSpeed(u32),
    Stop,
}

// Fixed-capacity queue of 16 messages
static CMD_CHANNEL: Channel<ThreadModeRawMutex, Command, 16> = Channel::new();

#[embassy_executor::main]
async fn main(spawner: Spawner) {
    spawner.spawn(motor_driver_task()).unwrap();

    let sender = CMD_CHANNEL.sender();
    sender.send(Command::SetSpeed(1000)).await;
}

#[embassy_executor::task]
async fn motor_driver_task() {
    loop {
        let cmd = CMD_CHANNEL.receive().await;
        match cmd {
            Command::SetSpeed(rpm) => { /* Update PWM... */ },
            Command::Stop => { /* Disable motor... */ },
        }
    }
}
```

### Pattern 3: One-Shot Signal (`Signal`)
`Signal` allows single-value notification between an interrupt/task producer and a consumer.

```rust
use embassy_sync::blocking_mutex::raw::CriticalSectionRawMutex;
use embassy_sync::signal::Signal;

static IRQ_SIGNAL: Signal<CriticalSectionRawMutex, u32> = Signal::new();

// In hardware ISR or high priority task:
fn on_hardware_event(data: u32) {
    IRQ_SIGNAL.signal(data);
}

// In consumer task:
async fn wait_for_event() {
    let value = IRQ_SIGNAL.wait().await;
    defmt::info!("Received signal value: {}", value);
}
```

### Pattern 4: State Broadcast (`Watch` / `PubSubChannel`)
- **`Watch`:** Holds the latest state value; readers are notified whenever value changes.
- **`PubSubChannel`:** Multi-subscriber pub-sub queue where all subscribers receive every published message.

---

## 4. Sharing SPI and I2C Buses

Sharing a hardware I2C or SPI bus across multiple sensors connected to different CS pins is handled via Embassy's shared bus utilities or `embedded-hal-bus`.

### Shared SPI Device
```rust
use embassy_embedded_hal::shared_bus::asynch::spi::SpiDevice;
use embassy_sync::blocking_mutex::raw::NoopRawMutex;
use embassy_sync::mutex::Mutex;

static SPI_BUS: StaticCell<Mutex<NoopRawMutex, Spi<'static, SPI1>>> = StaticCell::new();

// Wrap underlying SPI bus in Mutex
let spi_bus = SPI_BUS.init(Mutex::new(spi_driver));

// Create individual SpiDevice handles with dedicated Chip Select (CS) pins
let sensor_1_device = SpiDevice::new(spi_bus, cs_pin_1);
let sensor_2_device = SpiDevice::new(spi_bus, cs_pin_2);
```

---

## 5. Memory & Buffer Best Practices

### Pass Buffers by Reference (`&mut [u8]`)
Avoid copying large byte arrays across functions or stackframes. Passing arrays by value duplicates stack memory usage and risks stack overflow:

```rust
// BAD: Allocates 1024 bytes twice on stack
fn process_buf(mut buf: [u8; 1024]) -> [u8; 1024] { ... }

// GOOD: In-place modification via mutable slice reference
fn process_buf(buf: &mut [u8]) {
    for byte in buf.iter_mut() {
        *byte = 0;
    }
}
```
