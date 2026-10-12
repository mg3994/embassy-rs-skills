---
name: embassy-power-optimization
description: Ultra-low-power strategies, WFE/WFI idle sleeping, RTC/LPTIM periodic wakeups, EXTI wakeups, dynamic clock gating, and power consumption measurement.
---

# Embassy Power Optimization Skill

This skill details techniques for minimizing microamp / nanoamp battery consumption in Embassy applications using hardware sleep modes, low-power timers, EXTI pin wakeups, and clock gating.

---

## 1. Automatic Low-Power WFE Idle Sleeping

By default, when all tasks in the Embassy executor are `.await`ing timers or peripheral I/O, the thread-mode executor calls ARM `WFE` (Wait For Event) or RISC-V `WFI` (Wait For Interrupt), powering down CPU clock trees while preserving SRAM and register contents.

### Thread Executor Sleep Loop
```rust
// Internally executed when no tasks are ready:
loop {
    cortex_m::asm::wfe();
}
```

---

## 2. Low-Power Time Drivers (RTC & LPTIM)

Standard high-speed timers (e.g., TIM2 @ 80 MHz) draw milliamp currents when running. To operate in microamp sleep modes:

1. Enable 32.768 kHz Low-Speed External (LSE) crystal or Low-Speed Internal (LSI) oscillator.
2. Select RTC / LPTIM as the Embassy time driver feature in `Cargo.toml`.

```toml
# In Cargo.toml for embassy-stm32 or embassy-nrf
features = ["time-driver-rtc1"] # Use RTC instead of high-speed timers
```

---

## 3. Asynchronous EXTI Low-Power Pin Wakeup

Keep the CPU in sleep mode until an external event (e.g. motion sensor interrupt or button press) triggers a hardware pin edge:

```rust
use embassy_stm32::exti::ExtiInput;
use embassy_stm32::gpio::Pull;

let mut motion_sensor = ExtiInput::new(p.PA0, p.EXTI0, Pull::Down, Irqs);

loop {
    // CPU sleeps via WFE until PA0 transitions LOW -> HIGH
    motion_sensor.wait_for_rising_edge().await;

    defmt::info!("Motion detected! Sampling sensors...");
    perform_sensor_poll().await;
}
```

---

## 4. Clock Gating & Disabling Unused Peripherals

Peripherals consume dynamic current when their clock gates are enabled.

- **Disable Unneeded Clocks:** Disable GPIO banks, UARTs, or SPI modules when not in active use.
- **Dynamic Clock Scaling:** Lower SYSCLK frequency during low-throughput phases (e.g. scale down from 170 MHz to 16 MHz while waiting for network packets).

---

## 5. Measuring Idle & Peak Power Consumption

### Static Memory Usage vs Sleep Power
Large SRAM arrays consume static leakage current. In extreme battery-operated devices, shut down unneeded SRAM banks in Stop/Standby modes.

### Measuring Idle Ratio
Calculate active CPU vs idle sleep time using hardware timers:

1. Start timer on task wake.
2. Accumulate active execution ticks.
3. Compare active ticks against system uptime (`Instant::now()`).
