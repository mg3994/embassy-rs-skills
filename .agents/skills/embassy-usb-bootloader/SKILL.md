---
name: embassy-usb-bootloader
description: USB device stack (embassy-usb), CDC ACM, HID, VBUS sensing, and embassy-boot power-fail safe bootloader & DFU updates.
---

# Embassy USB & Bootloader Skill

This skill covers implementing USB device classes (`embassy-usb`) and power-fail safe firmware updates (`embassy-boot`).

---

## 1. USB Stack (`embassy-usb`)

`embassy-usb` implements a zero-copy device-side USB stack with class drivers for Serial (CDC ACM), Human Interface Devices (HID), and custom endpoints.

### Key Components
1. **USB Driver (`Driver`):** Hardware USB peripheral driver (e.g., `embassy-stm32::usb`, `embassy-rp::usb`, `embassy-nrf::usb`).
2. **`Builder`:** Assembles USB descriptors (VID, PID, strings, configuration descriptors, endpoint allocations).
3. **`UsbDevice`:** Runs `usb.run().await` in background.

---

## 2. CDC ACM (Virtual Serial Port) Implementation

```rust
use embassy_executor::Spawner;
use embassy_futures::join::join;
use embassy_usb::class::cdc_acm::{CdcAcmClass, State};
use embassy_usb::{Builder, Config};
use static_cell::StaticCell;

static CONFIG_DESCRIPTOR: StaticCell<[u8; 256]> = StaticCell::new();
static BOS_DESCRIPTOR: StaticCell<[u8; 256]> = StaticCell::new();
static CONTROL_BUF: StaticCell<[u8; 128]> = StaticCell::new();
static STATE: StaticCell<State> = StaticCell::new();

#[embassy_executor::main]
async fn main(_spawner: Spawner) {
    let p = embassy_stm32::init(Default::default());

    // Create hardware USB driver
    let driver = embassy_stm32::usb::Driver::new_fs(p.USB, Irqs, p.PA12, p.PA11);

    // Create USB Builder
    let mut config = Config::new(0xc0de, 0xcafe);
    config.manufacturer = Some("Embassy");
    config.product = Some("USB Serial");
    config.serial_number = Some("12345678");

    // VBUS Sensing
    // If powered only by USB bus and VBUS pin is not wired, set vbus_detection = false
    config.vbus_detection = false;

    let mut builder = Builder::new(
        driver,
        config,
        CONFIG_DESCRIPTOR.init([0; 256]),
        BOS_DESCRIPTOR.init([0; 256]),
        &mut [],
        CONTROL_BUF.init([0; 128]),
    );

    // Create CDC ACM class
    let state = STATE.init(State::new());
    let mut class = CdcAcmClass::new(&mut builder, state, 64);

    // Build USB device
    let mut usb = builder.build();

    // Run USB bus stack and CDC echo loop concurrently
    let usb_fut = usb.run();
    let echo_fut = async {
        class.wait_connection().await;
        defmt::info!("USB Connected");
        let mut buf = [0u8; 64];
        loop {
            let n = class.read_packet(&mut buf).await.unwrap();
            class.write_packet(&buf[..n]).await.unwrap();
        }
    };

    join(usb_fut, echo_fut).await;
}
```

### VBUS Sensing Troubleshooting
- **Issue:** USB device unrecognized or doesn't connect.
- **Cause:** `vbus_detection` is `true` in code, but hardware lacks VBUS monitoring pin wiring (e.g. PA9/PB13 on STM32).
- **Fix:** Set `config.vbus_detection = false` for self-powered/bus-powered boards without dedicated VBUS sensing hardware.

---

## 3. Power-Fail Safe Bootloader (`embassy-boot`)

`embassy-boot` is a lightweight, platform-agnostic bootloader supporting fail-safe firmware updates using an A/B partition swap scheme.

### Flash Memory Layout
Flash memory is divided into 4 mandatory partitions:

```text
+-----------------------+
| BOOTLOADER (8-24 KiB) | -> Bootloader binary execution address
+-----------------------+
| ACTIVE Partition      | -> Main Application binary location
+-----------------------+
| DFU Partition         | -> New firmware download buffer
| (Size = Active + 1 pg)|    (Size must be >= Active + 1 erase page)
+-----------------------+
| BOOTLOADER STATE      | -> Partition swap state & progress markers
+-----------------------+
```

---

## 4. Firmware Updating (`FirmwareUpdater`)

In the main application, download new firmware chunk-by-chunk into the DFU partition and mark it ready for swap on reboot.

```rust
use embassy_boot_stm32::FirmwareUpdater;
use embedded_storage::nor_flash::NorFlash;

async fn apply_firmware_update<F: NorFlash>(
    dfu_flash: &mut F,
    updater: &mut FirmwareUpdater,
    new_binary_chunks: &[&[u8]],
) {
    let mut offset = 0;

    for chunk in new_binary_chunks {
        updater.write_firmware(offset, chunk).unwrap();
        offset += chunk.len();
    }

    // Mark DFU partition as valid for swap on next boot/reset
    updater.mark_updated().unwrap();

    defmt::info!("Firmware marked updated! Resetting MCU...");
    cortex_m::peripheral::SCB::sys_reset();
}
```

### Cryptographic Signature Verification (ed25519)
To prevent unauthorized or corrupted updates, enable signature verification (`ed25519-salty` feature):

```rust
static PUBLIC_KEY: &[u8] = include_bytes!("key.pub");

// Verify signature and mark updated
updater.verify_and_mark_updated(PUBLIC_KEY, signature, firmware_len).unwrap();
```

---

## 5. Bootloader Loop Troubleshooting

| Symptom | Cause | Solution |
| :--- | :--- | :--- |
| **Bootloader reboots in continuous loop** | Partition alignment / size assertion failure in `memory.x`. | Ensure `PAGE_SIZE % WRITE_SIZE == 0` and DFU size is at least 1 page larger than Active partition. |
| **Reset during app startup** | Watchdog timer reset in bootloader. | Ensure application feeds or resets watchdog immediately after boot. |
