---
name: embassy-bootloader-dfu-advanced
description: Advanced embassy-boot state partition sizing, trial boots, automatic rollbacks, external SPI flash DFU partitions, and watchdog integration.
---

# Advanced Embassy Bootloader & DFU Skill

This skill provides advanced formulas, flash layout algorithms, trial boot verification, and automatic rollback handling in `embassy-boot`.

---

## 1. Precise Partition Sizing Formulas

To guarantee power-fail safe operation during A/B page swapping, `embassy-boot` partition sizes must strictly obey these formulas:

### 1. DFU Partition Size Requirement
```text
DFU Partition Size >= Active Partition Size + Active Page Size
```

### 2. State Partition Size Requirement
```text
State Partition Size = (2 * Write_Size) + (4 * Write_Size * Active_Size / Active_Page_Size)
```

---

## 2. Trial Boots & Automatic Rollback Logic

1. **Bootloader Swap:** When `mark_updated()` is called, the bootloader swaps Active and DFU partitions on reboot.
2. **Trial Boot Period:** The application boots into trial state.
3. **Confirmation:** If the application successfully initializes and calls `mark_booted()`, the firmware update is permanently confirmed.
4. **Automatic Rollback:** If the application crashes or panics before calling `mark_booted()`, the bootloader automatically restores the previous working firmware partition on the next reset!

```rust
use embassy_boot_stm32::FirmwareUpdater;

#[embassy_executor::main]
async fn main(_spawner: Spawner) {
    let mut updater = FirmwareUpdater::new();

    // Perform self-tests...
    if self_test_passed() {
        // Permanently confirm update and prevent rollback
        updater.mark_booted().unwrap();
        defmt::info!("Firmware boot confirmed!");
    } else {
        // Trigger reboot to roll back to previous image
        cortex_m::peripheral::SCB::sys_reset();
    }
}
```
