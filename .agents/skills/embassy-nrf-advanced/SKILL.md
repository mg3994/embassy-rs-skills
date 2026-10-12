---
name: embassy-nrf-advanced
description: Advanced Nordic Semiconductor nRF52/53/54/91 series, PPI/DPPI programmable peripheral interconnect, NVMC internal flash, nRF5340 IPC dual-core, and nRF SoftDevice BLE integration.
---

# Advanced Embassy nRF Skill

This skill covers Nordic Semiconductor microcontrollers (nRF52, nRF5340, nRF54, nRF91 series), hardware event routing via PPI/DPPI, internal non-volatile memory controller (NVMC), dual-core IPC, and SoftDevice BLE integration.

---

## 1. Programmable Peripheral Interconnect (PPI / DPPI)

PPI (nRF52) and DPPI (nRF53/nRF54) allow direct hardware-to-hardware triggering between peripheral events and tasks without CPU intervention or interrupt latency.

### Connecting Hardware Event to Task via PPI (e.g., Timer to GPIOTE)
```rust
use embassy_nrf::ppi::Ppi;
use embassy_nrf::gpiote::{OutputChannel, Level, OutputChannelPolarity};
use embassy_nrf::timer::Timer;

let p = embassy_nrf::init(Default::default());

// Initialize hardware timer
let mut timer = Timer::new_counter(p.TIMER1);
timer.set_frequency(embassy_nrf::timer::Frequency::F1MHz);
timer.cc(0).write(1000); // 1 ms match boundary

// Initialize GPIOTE output channel
let led_channel = OutputChannel::new(p.GPIOTE, p.P0_13, Level::Low, OutputChannelPolarity::Toggle);

// Connect Timer CC[0] event directly to GPIOTE toggle task using PPI channel 0
let mut ppi = Ppi::new_one_to_one(p.PPI_CH0, timer.cc(0).event_compare(), led_channel.task_out());
ppi.enable();
```

---

## 2. Non-Volatile Memory Controller (NVMC) & Internal Flash

`embassy-nrf` provides async page erase and word write operations on internal flash memory.

```rust
use embassy_nrf::nvmc::Nvmc;
use embedded_storage::nor_flash::{NorFlash, ReadNorFlash};

let mut flash = Nvmc::new(p.NVMC);

let target_address = 0x000F_0000; // Address within app data flash page

// Erase 4096-byte page
flash.erase(target_address, target_address + 4096).unwrap();

// Write byte buffer
let data = [0xDE, 0xAD, 0xBE, 0xEF];
flash.write(target_address, &data).unwrap();

// Read back data
let mut read_buf = [0u8; 4];
flash.read(target_address, &mut read_buf).unwrap();
defmt::info!("Flash read back: {:#X}", read_buf);
```

---

## 3. nRF5340 Dual-Core IPC & Multi-Core Architecture

The nRF5340 contains an Application Core (Cortex-M33) and a Network Core (Cortex-M33). Inter-Processor Communication (IPC) transmits signals and shared RAM buffers between the two cores.

### Inter-Processor Communication (IPC) Setup
```rust
use embassy_nrf::ipc::Ipc;

let ipc = Ipc::new(p.IPC);
// Send IPC event to network core running BLE stack
ipc.send(0);
```

---

## 4. Low-Power SAADC (Analog-to-Digital Converter)

`embassy-nrf` supports oversampled, low-power ADC sampling with internal reference voltages.

```rust
use embassy_nrf::saadc::{Config, Saadc, ChannelConfig};

let config = Config::default();
let channel_config = ChannelConfig::single_ended(p.P0_03);

let mut saadc = Saadc::new(p.SAADC, Irqs, config, [channel_config]);

let mut sample_buf = [0i16; 1];
saadc.sample(&mut sample_buf).await.unwrap();
defmt::info!("Analog voltage sample: {} mV", sample_buf[0]);
```

---

## 5. Bluetooth LE via `nrf-softdevice`

On nRF52 devices, Nordic's official SoftDevice binary (e.g., S140) can be wrapped by `nrf-softdevice`.

### `memory.x` Configuration for SoftDevice
SoftDevice occupies lower flash and RAM. Adjust `memory.x` origins accordingly:

```linker
MEMORY
{
  /* Reserve Flash 0x00000000 - 0x00027000 for SoftDevice S140 */
  FLASH : ORIGIN = 0x00027000, LENGTH = 868K
  RAM   : ORIGIN = 0x20020000, LENGTH = 128K
}
```

### SoftDevice Initialization & BLE Task
```rust
use nrf_softdevice::{raw, Softdevice};

let config = nrf_softdevice::Config {
    clock: Some(raw::nrf_clock_lf_cfg_t {
        source: raw::NRF_CLOCK_LF_SRC_XTAL as u8,
        rc_ctiv: 0,
        rc_temp_ctiv: 0,
        accuracy: raw::NRF_CLOCK_LF_ACCURACY_20_PPM as u8,
    }),
    ..Default::default()
};

let sd = Softdevice::enable(&config);
spawner.spawn(softdevice_task(sd)).unwrap();

#[embassy_executor::task]
async fn softdevice_task(sd: &'static Softdevice) -> ! {
    sd.run().await
}
```
