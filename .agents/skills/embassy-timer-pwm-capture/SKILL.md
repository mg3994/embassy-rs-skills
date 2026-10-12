---
name: embassy-timer-pwm-capture
description: Advanced hardware timer modes, complementary PWM with dead-time insertion, Input Capture signal analysis, Quadrature Encoder Interface (QEI), and timer cascading.
---

# Embassy Advanced Timers, PWM & Capture Skill

This skill explains advanced hardware timer modes: complementary PWM generation with dead-time insertion, Input Capture for frequency measurement, Quadrature Encoder Interface (QEI), and timer synchronization.

---

## 1. Advanced PWM with Dead-Time Insertion (H-Bridge Driving)

Driving power MOSFETs in H-bridge inverter or motor drive circuits requires **complementary output channels** (CH1 and CH1N) with **dead-time** to prevent shoot-through short circuits.

```rust
use embassy_stm32::timer::complementary_pwm::{ComplementaryPwm, ComplementaryPwmPin};
use embassy_stm32::timer::simple_pwm::PwmPin;
use embassy_stm32::time::Hertz;

let ch1 = PwmPin::new_ch1(p.PA8, embassy_stm32::gpio::OutputType::PushPull);
let ch1n = ComplementaryPwmPin::new_ch1(p.PB13, embassy_stm32::gpio::OutputType::PushPull);

let mut pwm = ComplementaryPwm::new(
    p.TIM1,
    Some(ch1),
    Some(ch1n),
    None, None, None, None, None, None,
    Hertz(20_000), // 20 kHz PWM frequency
    Default::default(),
);

// Set Dead-Time insertion (e.g. 1.5 microseconds)
pwm.set_dead_time(150);
pwm.enable(embassy_stm32::timer::Channel::Channel1);
pwm.set_duty(embassy_stm32::timer::Channel::Channel1, pwm.get_max_duty() / 2);
```

---

## 2. Input Capture (Measuring Frequency & Pulse Width)

Input Capture measures external signal periods or pulse high times in hardware without CPU polling latency.

```rust
use embassy_stm32::timer::input_capture::InputCapture;

let mut ic = InputCapture::new(p.TIM2, Some(ch1), None, None, None, Irqs, Hertz(1_000_000), Default::default());

loop {
    // Wait asynchronously for next input pulse edge
    let period_ticks = ic.wait_for_rising_edge().await;
    let frequency_hz = 1_000_000 / period_ticks;
    defmt::info!("Measured Signal Frequency: {} Hz", frequency_hz);
}
```

---

## 3. Quadrature Encoder Interface (QEI Motor Tracking)

Hardware timers decode A/B quadrature signals from rotary encoders to track motor shaft position and direction:

```rust
use embassy_stm32::timer::qei::Qei;

let mut qei = Qei::new(p.TIM3, p.PA6, p.PA7);
qei.set_count(0);

loop {
    let position = qei.count();
    let direction = qei.direction();
    defmt::info!("Encoder Position: {}, Direction: {:?}", position, direction);
    Timer::after_millis(10).await;
}
```
