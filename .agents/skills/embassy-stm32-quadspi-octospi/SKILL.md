---
name: embassy-stm32-quadspi-octospi
description: External QSPI, OCTOSPI, and FMC memory controller interfacing, indirect vs memory-mapped Execute-In-Place (XIP) mode in embassy-stm32.
---

# Embassy STM32 QSPI / OCTOSPI External Memory Skill

This skill explains how to interface with high-speed Quad-SPI (QSPI), Octo-SPI (OCTOSPI), and Flexible Memory Controller (FMC) external NOR/NAND flash chips.

---

## 1. Operating Modes

1. **Indirect Read/Write Mode:** Standard command/address/data transfer blocks for erasing or writing flash pages.
2. **Memory-Mapped (XIP) Mode:** Maps external flash directly into CPU address space (`0x90000000`), allowing code execution directly from external NOR flash.

---

## 2. QSPI Configuration Example

```rust
use embassy_stm32::qspi::{Qspi, Config};

let mut config = Config::default();
config.memory_size = embassy_stm32::qspi::enums::MemorySize::Size16MB;

let mut qspi = Qspi::new_bank1(
    p.QUADSPI,
    p.PB2,  // CLK
    p.PB6,  // BK1_NCS
    p.PF6,  // IO0
    p.PF7,  // IO1
    p.PF8,  // IO2
    p.PF9,  // IO3
    p.DMA1_CH7,
    config,
);

// Read chip JEDEC ID
let mut id = [0u8; 3];
qspi.blocking_read_id(&mut id).unwrap();
defmt::info!("External Flash JEDEC ID: {=[u8]:hex}", id);
```
