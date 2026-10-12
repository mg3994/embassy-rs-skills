---
name: embassy-core-async
description: Master the core async Rust concepts, Embassy executor, task spawner, embassy-time, and timekeeping mechanisms.
---

# Embassy Core Async & Executor Skill

This skill provides comprehensive guidance on using Embassy's async core, executor model, task lifecycle, timekeeping, and power-efficient scheduling in embedded Rust applications.

---

## 1. Core Concepts & Async Model

Embassy transforms `async/await` in Rust into zero-allocation, lightweight futures ideal for bare-metal embedded targets (`no_std`).

### Key Principles
- **No Dynamic Allocation Required:** Tasks and futures are allocated statically at compile time or at initialization.
- **Cooperative Multitasking:** Tasks yield execution back to the executor using `.await` when waiting on I/O, timers, or signals.
- **Interrupt & WFE Integration:** When all tasks are blocked waiting for events or timers, the CPU enters sleep mode (WFE/WFI) automatically.
- **Fair & Efficient Polling:** Waking a task polls only that specific task, not all tasks. Task scheduling is fair—a constantly waking task cannot starve other pending tasks.

---

## 2. The Embassy Executor

The `embassy-executor` manages task lifecycle, queuing, and execution.

### Macro-Based Main Entry
The `#[embassy_executor::main]` attribute macro handles executor creation, main task registration, and running the event loop automatically.

```rust
#![no_std]
#![no_main]

use embassy_executor::Spawner;
use embassy_time::Timer;
use defmt_rtt as _;
use panic_probe as _;

#[embassy_executor::task]
async fn blink_task() {
    loop {
        defmt::info!("Blink tick");
        Timer::after_secs(1).await;
    }
}

#[embassy_executor::main]
async fn main(spawner: Spawner) {
    defmt::info!("Initializing application...");

    // Spawn background task
    spawner.spawn(blink_task()).unwrap();

    loop {
        Timer::after_secs(5).await;
        defmt::info!("Main loop heart-beat");
    }
}
```

### Manual Executor Creation
If custom execution logic or multiple priority levels are needed, you can instantiate `Executor` manually:

```rust
use embassy_executor::Executor;
use static_cell::StaticCell;

static EXECUTOR: StaticCell<Executor> = StaticCell::new();

#[cortex_m_rt::entry]
fn main() -> ! {
    // Custom setup...
    let executor = EXECUTOR.init(Executor::new());
    executor.run(|spawner| {
        spawner.spawn(blink_task()).unwrap();
    });
}
```

### Multi-Priority & Interrupt Executors
To run tasks at different hardware interrupt priorities (preempting lower-priority thread-mode tasks):

```rust
use embassy_executor::InterruptExecutor;
use embassy_stm32::interrupt;
use embassy_stm32::interrupt::Priority;

static EXECUTOR_HIGH: InterruptExecutor = InterruptExecutor::new();

#[interrupt]
fn USART1() {
    unsafe { EXECUTOR_HIGH.on_interrupt() };
}

#[embassy_executor::main]
async fn main(spawner: Spawner) {
    // Set interrupt priority
    interrupt::USART1.set_priority(Priority::P6);
    interrupt::USART1.enable();

    let high_spawner = EXECUTOR_HIGH.start(interrupt::USART1);
    high_spawner.spawn(high_priority_task()).unwrap();
}
```

---

## 3. Tasks & Spawner

### Task Definitions & Task Pools
Tasks are declared using `#[embassy_executor::task]`. By default, only 1 instance of a task can be active at a time. To allow multiple instances, set `pool_size`:

```rust
#[embassy_executor::task(pool_size = 4)]
async fn worker_task(id: u8) {
    defmt::info!("Worker {} active", id);
    Timer::after_millis(500).await;
}

#[embassy_executor::main]
async fn main(spawner: Spawner) {
    for id in 0..4 {
        spawner.spawn(worker_task(id)).unwrap();
    }
}
```

### Passing Parameters
Task parameters must have `'static` lifetimes or be owned value types (`u32`, `&'static Mutex`, etc.).

---

## 4. Timekeeping (`embassy-time`)

`embassy-time` provides unified async time primitives (`Timer`, `Ticker`, `Instant`, `Duration`) and sync generic delay implementations (`Delay`).

### `Timer` (Async Delays)
- `Timer::after(Duration)`: Delays for a duration relative to execution.
- `Timer::at(Instant)`: Delays until a target system timestamp.

```rust
use embassy_time::{Duration, Instant, Timer};

async fn example_timing() {
    let start = Instant::now();

    Timer::after(Duration::from_millis(250)).await;

    let target = start + Duration::from_secs(2);
    Timer::at(target).await;
}
```

### `Ticker` (Periodic Execution Without Drift)
Unlike `Timer` in a loop, `Ticker` accounts for execution overhead and avoids cumulative drift:

```rust
use embassy_time::{Duration, Ticker};

async fn periodic_task() {
    let mut ticker = Ticker::every(Duration::from_hz(100)); // 100 Hz = 10ms
    loop {
        // Do work...
        ticker.next().await; // Waits for the exact next tick boundary
    }
}
```

### Embedded HAL Delay (`Delay`)
For third-party drivers expecting `embedded_hal::delay::DelayNs`:

```rust
use embassy_time::Delay;
use embedded_hal_async::delay::DelayNs;

async fn initialize_sensor<D: DelayNs>(delay: &mut D) {
    delay.delay_ms(10).await;
}

// Called as: initialize_sensor(&mut Delay).await;
```

---

## 5. Power Management & Sleep Modes

Embassy automatically puts the CPU to sleep when no tasks are ready to run:
- **Thread-Mode Executor:** Uses `WFE` (Wait For Event) or `SEV` (Send Event).
- **Interrupt Executors:** Woken by hardware interrupts.

### Preventing Sleep (High-Throughput Workaround)
If putting the CPU to sleep introduces unwanted current/clock switching spikes:

```rust
#[embassy_executor::task]
async fn keep_awake() {
    loop {
        embassy_futures::yield_now().await;
    }
}
```

---

## 6. Common Gotchas & Troubleshooting

1. **`missing main macro` error:**
   Ensure features in `Cargo.toml` for `embassy-executor` include:
   ```toml
   features = ["arch-cortex-m", "executor-thread"] # for Cortex-M
   ```
2. **Undefined symbol `_embassy_time_now` or `_embassy_time_schedule_wake`:**
   You forgot to enable a time driver in your HAL crate (e.g., `features = ["time-driver-any"]` or `"time-driver-rtc1"` in `embassy-stm32` / `embassy-nrf`).
3. **Task Blocking / Starvation:**
   Do not run busy synchronous loops without `.await` calls inside async tasks. Synchronous loops block the entire thread-mode executor.
