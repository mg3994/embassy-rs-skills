---
name: embassy-faq-troubleshooting
description: Comprehensive troubleshooting guide and solutions for all frequently asked questions (FAQ) and common errors in Embassy.
---

# Embassy FAQ & Troubleshooting Skill

This skill provides immediate diagnostic and resolution steps for every common issue and FAQ documented in the Embassy Book.

---

## 1. Frequently Asked Questions & Resolution Matrix

### Q1: `Missing main macro` error
```text
could not find `main` in `embassy_executor`
```
- **Cause:** Missing architecture features in `embassy-executor`.
- **Fix:** In `Cargo.toml`, ensure `embassy-executor` has features:
  ```toml
  features = ["arch-cortex-m", "executor-thread"] # for Cortex-M
  ```

### Q2: Linker symbol missing: `_embassy_time_now` / `_embassy_time_schedule_wake`
- **Cause:** Missing time driver feature in HAL crate.
- **Fix:** In `Cargo.toml`, add `time-driver-any` or hardware timer feature:
  ```toml
  [dependencies.embassy-stm32]
  features = ["time-driver-any"]
  ```

### Q3: Linker symbol missing: `__pender`
- **Cause:** `embassy-executor` architecture feature not enabled AND HAL does not provide an executor.
- **Fix:** Enable `arch-cortex-m` in `embassy-executor`.

### Q4: `Only one package in the dependency graph may specify the same links value`
- **Cause:** Mixing `crates.io` published crates with `git` dependencies pulling different underlying crates.
- **Fix:** Patch ALL `embassy-*` crates to the exact same git commit in `Cargo.toml`:
  ```toml
  [patch.crates-io]
  embassy-executor = { git = "https://github.com/embassy-rs/embassy", rev = "HEAD" }
  embassy-time     = { git = "https://github.com/embassy-rs/embassy", rev = "HEAD" }
  embassy-stm32    = { git = "https://github.com/embassy-rs/embassy", rev = "HEAD" }
  ```

### Q5: `<invalid location: defmt frame-index: ...>`
- **Cause:** Release profile stripped debug symbol index required by `defmt`.
- **Fix:** In `Cargo.toml`:
  ```toml
  [profile.release]
  debug = 2
  ```

### Q6: `can't find crate for std` on thumb target
- **Cause:** Running `cargo add probe-rs` instead of installing probe-rs CLI tool.
- **Fix:** Remove `probe-rs` from `Cargo.toml`; install via `cargo install probe-rs-tools`.

### Q7: Code works in debug mode, but fails in release mode (or with LTO)
- **Checklist:**
  1. **Race conditions:** Faster execution misses hardware interrupts or DMA completion flags.
  2. **Missing volatile / memory barriers:** Check unsafe pointer reads or missing CPU/DMA fences.
  3. **Clock misconfiguration:** Clock frequencies set higher than hardware limits.
  4. **Async Waker registration:** Task not registering waker before checking status flag.

### Q8: Bootloader restarting in continuous loop
- **Checklist:**
  1. Validate `memory.x` partition assertions (`PAGE_SIZE % WRITE_SIZE == 0`).
  2. Ensure DFU partition size is >= Active partition size + 1 page.
  3. Feed watchdog timer immediately in application startup.

### Q9: Deploying to RP2040/RP2350 without debug probe
- **Fix:** Use `picotool` in `.cargo/config.toml`:
  ```toml
  runner = "picotool load --update --verify --execute -t elf"
  ```

### Q10: USB device not recognized
- **Fix:** If board lacks VBUS sensing pin wiring, set `config.vbus_detection = false`.
