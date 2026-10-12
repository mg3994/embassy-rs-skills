---
name: embassy-display-ui
description: Embedded graphics, display drivers (SSD1306, ST7789), UI task layering, DMA framebuffers, and keyboard firmware patterns (RMK).
---

# Embassy Displays & UI Skill

This skill covers driven displays (OLED, TFT LCD, e-Paper) using `embedded-graphics`, async DMA framebuffers, layered UI task architecture, and embedded keyboard firmware patterns (RMK).

---

## 1. Display Integration (`embedded-graphics`)

`embedded-graphics` is the standard `no_std` drawing library for Rust. Combined with Embassy's async SPI/I2C drivers, it enables low-power graphical UIs.

### Drawing to an OLED Display (SSD1306 over Async I2C)
```rust
use ssd1306::{prelude::*, I2CDisplayInterface, Ssd1306};
use embedded_graphics::{
    mono_font::{ascii::FONT_6X10, MonoTextStyle},
    pixelcolor::BinaryColor,
    prelude::*,
    text::Text,
};

// Initialize async I2C driver
let i2c = I2c::new(p.I2C1, p.PB6, p.PB7, Irqs, p.DMA1_CH1, p.DMA1_CH2, Hertz(400_000), Default::default());

let interface = I2CDisplayInterface::new(i2c);
let mut display = Ssd1306::new(interface, DisplaySize128x64, DisplayRotation::Rotation0)
    .into_buffered_graphics_mode();

display.init().unwrap();

let style = MonoTextStyle::new(&FONT_6X10, BinaryColor::On);
Text::new("Embassy OS", Point::new(10, 20), style).draw(&mut display).unwrap();
display.flush().unwrap();
```

---

## 2. Layered UI Architecture Pattern

To decouple sensor sampling, blinkers, multiplexers, and UI rendering, separate them into layered Embassy tasks connected via channels or channels/mutexes.

```text
[ Hardware Sensors / Inputs ]
           │ (Send state data over Channel)
           ▼
[ State Manager / Blinker Task ]
           │ (Notify UI update over Channel/Watch)
           ▼
[ UI Render Task ] ───(Async DMA / SPI)───► [ Physical Display ]
```

### UI Render Task Implementation
```rust
use embassy_sync::channel::Channel;
use embassy_sync::blocking_mutex::raw::ThreadModeRawMutex;

enum UiMessage {
    UpdateTemperature(f32),
    SetStatus(&'static str),
}

static UI_CHANNEL: Channel<ThreadModeRawMutex, UiMessage, 8> = Channel::new();

#[embassy_executor::task]
async fn ui_renderer_task(mut display: DisplayType) {
    loop {
        let msg = UI_CHANNEL.receive().await;
        match msg {
            UiMessage::UpdateTemperature(temp) => {
                // Redraw temperature gauge...
                display.flush().await;
            }
            UiMessage::SetStatus(status) => {
                // Render status bar...
                display.flush().await;
            }
        }
    }
}
```

---

## 3. Keyboard Firmware Framework (RMK)

RMK is a feature-rich, open-source embedded keyboard firmware framework built entirely on top of Embassy.

### Features
- Target Microcontrollers: STM32, RP2040, nRF52, ESP32.
- Layer support, wireless BLE support (via `trouble` / `nrf-softdevice`).
- Real-time keymap editing via Vial API.
