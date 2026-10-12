---
name: embassy-project-setup
description: Project creation, directory structure, configuration files, probe-rs, build optimization, binary size tuning, and troubleshooting for Embassy projects.
---

# Embassy Project Setup & Infrastructure Skill

This skill provides step-by-step instructions on setting up new Embassy projects from scratch, configuring workspace dependencies, linker scripts, flash memory layouts, runner tools (`probe-rs`), build optimizations, and resolving common build and link errors.

---

## 1. Project Directory Structure

An Embassy standalone project requires specific files in its structure:

```text
my-embassy-app/
├── .cargo/
│   └── config.toml        # Target triple, runner (probe-rs), env variables
├── src/
│   └── main.rs            # Entry point with #[embassy_executor::main]
├── build.rs               # Linker script configuration (memory.x, defmt)
├── Cargo.toml             # Dependencies, features, profile overrides
├── memory.x               # Target microcontroller flash and RAM memory layout
└── rust-toolchain.toml    # Fixed Rust channel and compiler target architecture
```

---

## 2. Configuration Files

### `.cargo/config.toml`
Configures `probe-rs` runner, target architecture, and log levels:

```toml
[target.'cfg(all(target_arch = "arm", target_os = "none"))']
runner = "probe-rs run --chip STM32G474RETx"

[build]
target = "thumbv7em-none-eabi"

[env]
DEFMT_LOG = "trace"
```

*Note for RP2040 / RP2350 without probe:*
```toml
runner = "picotool load --update --verify --execute -t elf"
```

### `rust-toolchain.toml`
Pins the Rust toolchain version and target components:

```toml
[toolchain]
channel = "1.85"
components = ["rust-src", "rustfmt", "llvm-tools"]
targets = ["thumbv7em-none-eabi"]
```

### `build.rs`
Links defmt and places `memory.x` in the build path:

```rust
use std::env;
use std::path::PathBuf;

fn main() {
    let out = &PathBuf::from(env::var_os("OUT_DIR").unwrap());
    println!("cargo:rustc-link-search={}", out.display());
    println!("cargo:rerun-if-changed=memory.x");
}
```

### `memory.x`
Defines physical flash and RAM addresses for your target board:

```linker
MEMORY
{
  FLASH (rx) : ORIGIN = 0x08000000, LENGTH = 512K
  RAM (xrw)  : ORIGIN = 0x20000000, LENGTH = 128K
}

_stack_start = ORIGIN(RAM) + LENGTH(RAM);
```

---

## 3. `Cargo.toml` Dependencies & Features

### Published Crates (crates.io) Setup
```toml
[package]
name = "my-embassy-app"
version = "0.1.0"
edition = "2021"

[dependencies]
embassy-stm32 = { version = "0.1.0", features = ["defmt", "time-driver-any", "stm32g474re", "memory-x", "exti"] }
embassy-executor = { version = "0.6.3", features = ["arch-cortex-m", "executor-thread", "defmt"] }
embassy-time = { version = "0.3.2", features = ["defmt", "defmt-timestamp-uptime", "tick-hz-32_768"] }

defmt = "0.3.5"
defmt-rtt = "0.4.0"
cortex-m = { version = "0.7.7", features = ["critical-section-single-core"] }
cortex-m-rt = "0.7.3"
panic-probe = "0.3.1"

[profile.release]
debug = 2
opt-level = "s"
lto = "fat"
codegen-units = 1
incremental = false
```

### Patching Dependencies to Embassy Git Main
If using unreleased features or git revisions across crates:

```toml
[patch.crates-io]
embassy-executor = { git = "https://github.com/embassy-rs/embassy", rev = "HEAD" }
embassy-time = { git = "https://github.com/embassy-rs/embassy", rev = "HEAD" }
embassy-stm32 = { git = "https://github.com/embassy-rs/embassy", rev = "HEAD" }
```

---

## 4. Binary Size Optimization & Profiling

### Profile Flags
Reduce binary footprint dramatically by optimizing in `Cargo.toml`:

```toml
[profile.release]
opt-level = "z"        # Optimize aggressively for size
lto = true             # Enable Link-Time Optimization
codegen-units = 1      # Maximize LTO optimization scope
panic = "abort"
```

### Stripping Formatting with `build-std`
Remove `std::fmt` overhead on panics by adding this to `.cargo/config.toml`:

```toml
[unstable]
build-std = ["core"]
build-std-features = ["panic_immediate_abort"]
```

### Measuring Static Memory & Stack Usage
- **Static usage:** `cargo size --bin my-app --release` (inspect `.data` + `.bss`).
- **Stack usage analysis:** `cargo call-stack --bin my-app`

---

## 5. Generator Tools

Quickly scaffold new projects using existing CLI templates:
- **`cargo-embassy`:** CLI for STM32 and nRF.
- **`cargo generate --git https://github.com/lulf/embassy-template`:** Scaffold for STM32, nRF, and RP.
- **`esp-generate`:** Project generator for ESP32 with `esp-hal`.

---

## 6. Troubleshooting & Common Errors

| Error Signature | Root Cause | Solution |
| :--- | :--- | :--- |
| `undefined symbol: _embassy_time_now` | Missing time driver in HAL dependency | Add `time-driver-any` or target-specific timer feature to HAL in `Cargo.toml`. |
| `undefined symbol: __pender` | Executor feature or architecture flag missing | Add `arch-cortex-m` / `executor-thread` to `embassy-executor` dependency features. |
| `Only one package in dependency graph may specify same links value` | Mismatched git and crates.io dependencies | Patch all `embassy-*` dependencies to the exact same git commit via `[patch.crates-io]`. |
| `<invalid location: defmt frame-index>` | Missing debug symbol information in release build | Ensure `debug = 2` or `debug = true` is in `[profile.release]`. |
| `Error: can't find crate for std` | Accidental `cargo add probe-rs` as a crate dependency | Remove `probe-rs` from `Cargo.toml`; install it as system binary via `cargo install probe-rs-tools`. |
