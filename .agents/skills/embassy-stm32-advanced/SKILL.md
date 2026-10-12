---
name: embassy-stm32-advanced
description: Advanced STM32 HAL configurations, RCC clock trees, PWR voltage scaling/LDO/SMPS, BDMA/DMA2D memory alignment, FDCAN, ADC/DAC, RTC, and STM32H5/H7 power troubleshooting.
---

# Advanced Embassy STM32 Skill

This skill provides in-depth technical guidance for advanced features, clock tree configurations, power modes, DMA memory constraints, and specialized hardware peripherals on STMicroelectronics STM32 microcontrollers.

---

## 1. RCC Clock Tree Configuration & Speed Optimization

To run STM32 microcontrollers at maximum frequency, configure the Reset and Clock Control (RCC) settings during HAL initialization.

### Maximum Performance Setup (e.g., STM32H7 / STM32G4)
```rust
use embassy_stm32::rcc::*;
use embassy_stm32::time::Hertz;
use embassy_stm32::Config;

let mut config = Config::default();

// Configure RCC for maximum clock frequency with HSE crystal
config.rcc.hse = Some(Hse {
    freq: Hertz(8_000_000), // 8 MHz external crystal
    mode: HseMode::Oscillator,
});

config.rcc.pll1 = Some(Pll {
    source: PllSource::HSE,
    prediv: PllPreDiv::DIV2,
    mul: PllMul::MUL170,
    divp: Some(PllDiv::DIV2),
    divq: Some(PllDiv::DIV2),
    divr: Some(PllDiv::DIV2),
});

config.rcc.sys = Sysclk::PLL1_R;
config.rcc.ahb_pre = AHBPrescaler::DIV1;
config.rcc.apb1_pre = APBPrescaler::DIV1;
config.rcc.apb2_pre = APBPrescaler::DIV1;

let p = embassy_stm32::init(config);
```

### Enabling Flash Memory Caches & ART Accelerator
Ensure flash prefetch, instruction cache (ICACHE), and data cache (DCACHE) are enabled for maximum instruction execution throughput:

```rust
// On Cortex-M7 (STM32H7) enable L1 I-Cache and D-Cache:
unsafe {
    cortex_m::peripheral::SCB::enable_icache();
    cortex_m::peripheral::SCB::enable_dcache(&mut cortex_m::peripheral::CPUID::ptr());
}
```

---

## 2. PWR Power Supply Management & STM32H5/H7 Troubleshooting

STM32 chips with internal SMPS (Switch Mode Power Supply) and LDO regulators (e.g. STM32H7, STM32H5) require matching hardware board wiring in `config.pwr`.

### Configuring Internal Regulators
```rust
use embassy_stm32::pwr::VoltageScale;

let mut config = Config::default();
// Match hardware layout: Direct LDO, Bypass, or SMPS
config.pwr = embassy_stm32::pwr::Config {
    voltage_scale: VoltageScale::Scale0, // High performance mode
};
```

### Hardware PWR Troubleshooting
- **Symptom:** Chip boots once and hangs, crashes on clock speed increase, or fails to connect over probe-rs.
- **Cause:** Mismatched PWR configuration (e.g., specifying SMPS mode when board uses LDO or external supply).
- **Fix:** Perform a full board power cycle (discharge capacitors) and verify board schematic before setting `config.pwr`.

---

## 3. BDMA & Special RAM Memory Regions (D2/D3 SRAM)

On multi-bus architecture chips like STM32H7, BDMA (Basic DMA) and certain peripherals can only access specific RAM regions (SRAM3/SRAM4 in D2/D3 domain).

### Linker Section Placement
Place DMA buffers intended for BDMA in the accessible section in `memory.x` and mark with `#[unsafe(link_section = "...")]`:

```rust
#[unsafe(link_section = ".sram3")]
static mut BDMA_BUFFER: [u8; 512] = [0u8; 512];
```

---

## 4. FDCAN (Controller Area Network)

Embassy provides async drivers for CAN / FDCAN peripherals:

```rust
use embassy_stm32::can::{Can, Frame, StandardId};
use embassy_stm32::bind_interrupts;
use embassy_stm32::peripherals;

bind_interrupts!(struct Irqs {
    FDCAN1_IT0 => embassy_stm32::can::IT0InterruptHandler<peripherals::FDCAN1>;
    FDCAN1_IT1 => embassy_stm32::can::IT1InterruptHandler<peripherals::FDCAN1>;
});

#[embassy_executor::main]
async fn main(_spawner: Spawner) {
    let p = embassy_stm32::init(Default::default());

    let mut can = Can::new(p.FDCAN1, p.PD0, p.PD1, Irqs);
    can.set_bitrate(500_000);

    let frame = Frame::new_data(StandardId::new(0x123).unwrap(), &[0x01, 0x02, 0x03]).unwrap();
    can.write(&frame).await.unwrap();

    let rx_frame = can.read().await.unwrap();
    defmt::info!("Received CAN ID: {:?}", rx_frame.id());
}
```

---

## 5. Analog Peripherals (ADC, DAC)

### Async ADC Readings
```rust
use embassy_stm32::adc::{Adc, SampleTime};

let mut adc = Adc::new(p.ADC1);
adc.set_sample_time(SampleTime::Cycles247_5);

let mut pin = p.PA0;
let sample = adc.read(&mut pin).await;
defmt::info!("ADC Raw Sample: {}", sample);
```

---

## 6. Real-Time Clock (RTC) & Backup Domain

```rust
use embassy_stm32::rtc::{Rtc, DateTime, DayOfWeek};

let mut rtc = Rtc::new(p.RTC, Default::default());
rtc.set_datetime(DateTime::new(2026, 10, 12, DayOfWeek::Monday, 12, 0, 0).unwrap()).unwrap();

let now = rtc.now().unwrap();
defmt::info!("Current Time: {:02}:{:02}:{:02}", now.hour(), now.minute(), now.second());
```
