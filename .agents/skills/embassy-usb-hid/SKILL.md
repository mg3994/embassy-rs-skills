---
name: embassy-usb-hid
description: Implementing USB Human Interface Device (HID) class, HID report descriptors, keyboards, mice, gamepads, and async report transmission with embassy-usb.
---

# Embassy USB HID (Human Interface Device) Skill

This skill explains how to build USB Human Interface Devices (keyboards, mice, gamepads, custom HID controllers) using `embassy-usb`.

---

## 1. HID Architecture & Report Descriptors

USB HID devices communicate using **Report Descriptors**, which tell the host OS how to decode byte arrays sent over Interrupt endpoints.

---

## 2. Implementing a USB Keyboard with `embassy-usb`

```rust
use embassy_usb::class::hid::{HidReaderWriter, State};
use usbd_hid::descriptor::{KeyboardReport, SerializedDescriptor};

static STATE: StaticCell<State> = StaticCell::new();

let config = embassy_usb::class::hid::Config {
    report_descriptor: KeyboardReport::desc(),
    request_handler: None,
    poll_ms: 10,
    max_packet_size: 64,
};

let state = STATE.init(State::new());
let hid = HidReaderWriter::<_, 1, 8>::new(&mut builder, state, config);
let (reader, mut writer) = hid.split();

// Send keystroke report asynchronously
let report = KeyboardReport {
    modifier: 0,
    reserved: 0,
    leds: 0,
    keycodes: [0x04, 0, 0, 0, 0, 0], // Keycode 0x04 = 'a'
};

writer.write_serialize(&report).await.unwrap();
```
