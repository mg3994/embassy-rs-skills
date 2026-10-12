---
name: embassy-async-storage-flash
description: Flash storage drivers, wear leveling, sequential-storage key-value stores, and embedded-storage-async in Embassy.
---

# Embassy Async Storage & Flash Memory Skill

This skill details non-volatile flash storage management, key-value configuration storage, and wear-leveling algorithms in Embassy applications.

---

## 1. Non-Volatile Flash Memory Traits (`embedded-storage-async`)

NorFlash chips require **erasing a page to 0xFF** before writing new bits (which flip 1s to 0s).

```rust
use embedded_storage_async::nor_flash::NorFlash;

pub async fn save_configuration<F: NorFlash>(flash: &mut F, address: u32, config_bytes: &[u8]) {
    // 1. Erase flash page boundary
    flash.erase(address, address + 4096).await.unwrap();

    // 2. Write new payload
    flash.write(address, config_bytes).await.unwrap();
}
```

---

## 2. Key-Value Store & Wear Leveling (`sequential-storage`)

Repeatedly erasing the same flash page reduces flash lifespan (typically 10,000 to 100,000 erase cycles). `sequential-storage` provides zero-allocation key-value storage with automatic wear leveling across flash pages.

```rust
use sequential_storage::map::{fetch_item, store_item};

let mut flash = get_flash_driver();
let mut buf = [0u8; 128];

// Store key-value configuration item
store_item(&mut flash, flash_range, &mut buf, &mut cache, 0x01, b"my_config_value").await.unwrap();

// Retrieve value on boot
if let Some(value) = fetch_item(&mut flash, flash_range, &mut buf, &mut cache, 0x01).await.unwrap() {
    defmt::info!("Restored config: {=[u8]:str}", value);
}
```
