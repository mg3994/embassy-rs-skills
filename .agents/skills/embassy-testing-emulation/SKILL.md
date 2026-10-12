---
name: embassy-testing-emulation
description: Hardware-free testing, std-mode host execution, Renode emulator integration, defmt/RTT debugging, mock driver traits, and stack analysis.
---

# Embassy Testing & Emulation Skill

This skill covers hardware-free testing strategies, running Embassy tasks on host PCs (`std`), hardware emulation via Renode, `defmt` logging, and static memory analysis.

---

## 1. Running Embassy Applications on Host PC (`std` Target)

Embassy supports running on desktop OS targets (Linux, macOS, Windows) using `std` features. This allows unit testing and logic verification without flashing physical microcontrollers.

### Host `Cargo.toml` Setup
```toml
[dependencies]
embassy-executor = { version = "0.6.3", features = ["arch-std", "executor-thread"] }
embassy-time = { version = "0.3.2", features = ["std"] }
```

### Host `main.rs`
```rust
use embassy_executor::Spawner;
use embassy_time::Timer;

#[embassy_executor::task]
async fn host_background_task() {
    loop {
        println!("Tick on host PC!");
        Timer::after_secs(1).await;
    }
}

#[embassy_executor::main]
async fn main(spawner: Spawner) {
    println!("Starting host Embassy executor...");
    spawner.spawn(host_background_task()).unwrap();

    Timer::after_secs(5).await;
    println!("Execution complete.");
}
```

---

## 2. Hardware Emulation with Renode

Renode is an open-source instruction-set emulator capable of simulating Cortex-M/RISC-V MCUs, GPIOs, Timers, and UARTs without physical hardware.

### Sample Renode Script (`emulation.resc`)
```renode
# Load machine platform description
mach create "stm32f4"
machine LoadPlatformDescription @platforms/boards/stm32f4_discovery-kit.repl

# Load compiled Rust ELF binary
sysbus LoadELF @target/thumbv7em-none-eabi/release/my-embassy-app

# Connect virtual UART to terminal window
showAnalyzer sysbus.usart1

# Start execution
start
```

---

## 3. High-Efficiency Logging (`defmt` + RTT)

`defmt` (deferred formatting) transmits compact integer indexes over Real-Time Transfer (RTT) to the host computer, where the string index table is decoded.

### Log Level Configuration
Set log verbosity dynamically via environment variables in `.cargo/config.toml`:

```toml
[env]
DEFMT_LOG = "trace" # Options: trace, debug, info, warn, error, off
```

### Formatting Output
```rust
defmt::trace!("Low-level register write: {:#04X}", 0x40);
defmt::debug!("Peripheral state: {}", state);
defmt::info!("Application booted successfully!");
defmt::warn!("Buffer utilization > 80%");
defmt::error!("Hardware timeout waiting for ACK!");
```

---

## 4. Stack & Static Memory Analysis

### Static Memory Usage (`cargo size`)
Check `.bss` (uninitialized static variables) and `.data` (initialized statics):

```bash
cargo size --bin my-app --release -- -A
```

### Worst-Case Stack Analysis (`cargo-call-stack`)
Statically calculate the maximum stack frame size required by your tasks:

```bash
cargo install cargo-call-stack
cargo call-stack --bin my-app > stack_analysis.txt
```
