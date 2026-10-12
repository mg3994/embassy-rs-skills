---
name: embassy-baremetal-to-async
description: Comparative guide illustrating progressive embedded Rust code evolution across 4 distinct abstraction layers (PAC, HAL, Interrupt-Driven, and Async Embassy).
---

# From Bare Metal to Async Rust Skill

This skill illustrates how a simple embedded application ("push button, toggle LED") evolves across the 4 primary levels of abstraction in embedded Rust development.

---

## Level 1: Peripheral Access Crate (PAC) Level

PAC provides direct, low-level register block types without memory protection or clock management abstractions.

```rust
#![no_std]
#![no_main]

use stm32_metapac as pac;

#[cortex_m_rt::entry]
fn main() -> ! {
    let rcc = pac::RCC;
    rcc.ahb2enr().modify(|w| {
        w.set_gpioben(true);
        w.set_gpiocen(true);
    });

    let gpioc = pac::GPIOC;
    gpioc.pupdr().modify(|w| w.set_pupdr(13, pac::gpio::vals::Pupdr::PullUp));
    gpioc.moder().modify(|w| w.set_moder(13, pac::gpio::vals::Moder::Input));

    let gpiob = pac::GPIOB;
    gpiob.moder().modify(|w| w.set_moder(14, pac::gpio::vals::Moder::Output));

    loop {
        if gpioc.idr().read().idr(13) == pac::gpio::vals::Idr::Low {
            gpiob.bsrr().write(|w| w.set_bs(14, true));
        } else {
            gpiob.bsrr().write(|w| w.set_br(14, true));
        }
    }
}
```
*Drawback:* High verbosity, manual register setting, busy-loop polling drains maximum CPU battery power.

---

## Level 2: Blocking Hardware Abstraction Layer (HAL) Level

The HAL hides register addresses and automatically handles clock gating.

```rust
#![no_std]
#![no_main]

use embassy_stm32::gpio::{Input, Level, Output, Pull, Speed};

#[cortex_m_rt::entry]
fn main() -> ! {
    let p = embassy_stm32::init(Default::default());
    let mut led = Output::new(p.PB14, Level::High, Speed::VeryHigh);
    let button = Input::new(p.PC13, Pull::Up);

    loop {
        if button.is_low() {
            led.set_high();
        } else {
            led.set_low();
        }
    }
}
```
*Drawback:* Code is concise, but still busy-loops on CPU without sleep mode.

---

## Level 3: Manual Interrupt-Driven Level

Interrupts allow the CPU to sleep via `wfe()`, but require global static variables, mutexes, and `cortex_m::interrupt::free`.

```rust
static BUTTON: Mutex<RefCell<Option<Input<'static>>>> = Mutex::new(RefCell::new(None));
static LED: Mutex<RefCell<Option<Output<'static>>>> = Mutex::new(RefCell::new(None));

#[entry]
fn main() -> ! {
    // Setup NVIC interrupt, store pins in static Mutex...
    loop {
        cortex_m::asm::wfe(); // CPU sleeps
    }
}

#[interrupt]
fn EXTI15_10() {
    cortex_m::interrupt::free(|cs| {
        // Unlocks mutex, checks button state, toggles LED
    });
}
```
*Drawback:* High complexity, boilerplate state management, mutex lock contention.

---

## Level 4: Async Embassy Level

Embassy combines HAL ease-of-use with automatic low-power interrupt handling and zero boilerplate.

```rust
#![no_std]
#![no_main]

use embassy_executor::Spawner;
use embassy_stm32::exti::ExtiInput;
use embassy_stm32::gpio::{Level, Output, Pull, Speed};
use embassy_stm32::bind_interrupts;

bind_interrupts!(struct Irqs {
    EXTI15_10 => embassy_stm32::exti::InterruptHandler<embassy_stm32::peripherals::EXTI13>;
});

#[embassy_executor::main]
async fn main(_spawner: Spawner) {
    let p = embassy_stm32::init(Default::default());
    let mut led = Output::new(p.PB14, Level::Low, Speed::VeryHigh);
    let mut button = ExtiInput::new(p.PC13, p.EXTI13, Pull::Up, Irqs);

    loop {
        button.wait_for_any_edge().await; // Task yields & CPU sleeps automatically via WFE
        if button.is_low() {
            led.set_high();
        } else {
            led.set_low();
        }
    }
}
```
*Advantage:* Maximum readable code, zero-cost abstractions, automatic low-power CPU sleep!
