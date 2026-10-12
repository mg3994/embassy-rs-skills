---
name: embassy-usb-dfu
description: Implementing USB Device Firmware Upgrade (DFU) class using embassy-usb-dfu, handling detach requests, and triggering embassy-boot firmware updates.
---

# Embassy USB DFU (Device Firmware Upgrade) Skill

This skill explains how to expose a USB DFU interface on devices using `embassy-usb-dfu` and integrate USB firmware flashing with `embassy-boot`.

---

## 1. USB DFU Protocol Overview

USB DFU defines a standard class specification allowing host utilities (such as `dfu-util`) to download new application binaries over USB directly into microcontroller flash.

- **DFU Mode / Runtime:** Device advertises DFU runtime capability during normal operation and resets into DFU bootloader mode when a DETACH request is received from the host.

---

## 2. Implementing USB DFU Class

```rust
use embassy_usb_dfu::usb_dfu;
use embassy_boot_stm32::FirmwareUpdater;

// Create USB DFU runtime class handler
let mut updater = FirmwareUpdater::new();

// USB DFU interface initialization
let dfu_class = usb_dfu::usb_dfu(&mut builder, &mut dfu_state, |request| {
    match request {
        usb_dfu::Request::Detach => {
            defmt::info!("USB DFU Detach requested by host. Resetting into bootloader...");
            cortex_m::peripheral::SCB::sys_reset();
        }
        usb_dfu::Request::WriteBlock { offset, data } => {
            // Write payload chunk to DFU flash partition
            updater.write_firmware(offset, data).unwrap();
        }
        usb_dfu::Request::Manifest => {
            // Mark firmware as updated and trigger reboot
            updater.mark_updated().unwrap();
            cortex_m::peripheral::SCB::sys_reset();
        }
    }
});
```
