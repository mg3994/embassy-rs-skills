---
name: embassy-usb-msc
description: Implementing USB Mass Storage Class (MSC), SCSI command set, block device storage integration (SD cards, SPI flash), and FAT file systems.
---

# Embassy USB Mass Storage Class (MSC) Skill

This skill covers implementing USB Mass Storage Class (MSC) interfaces to present SD cards or SPI flash chips as USB flash drives to host operating systems.

---

## 1. USB MSC Architecture

USB MSC uses Bulk IN and Bulk OUT endpoints to process **SCSI Transparent Command Set** blocks (Block Read, Block Write, Capacity Inquiry) over **BBB (Bulk-Only Transport)**.

---

## 2. Block Storage Interface Integration

Connect a block device (e.g. `embedded-storage` flash or SD card SPI driver) to `embassy-usb`:

```rust
use embassy_usb::class::msc::MscClass;
use embedded_storage_async::nor_flash::NorFlash;

// Create USB MSC class wrapping underlying block storage
let mut msc = MscClass::new(&mut builder, block_storage, 512, "Vendor", "Embassy Drive", "1.0");

// Run MSC transport loop in background task
msc.run().await;
```
