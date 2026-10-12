---
name: embassy-stm32-metapac-internals
description: Deep dive into stm32-data-sources, stm32-data matching engine (chips.rs), register block versioning, and stm32-metapac code generation.
---

# Embassy STM32 Metapac Internals Skill

This skill explains how Embassy manages device peripheral definitions across hundreds of STM32 chips using `stm32-data-sources`, `stm32-data`, and `stm32-metapac`.

---

## 1. The Multi-Crate Code Generation Pipeline

```text
[ ST Microelectronics STM32CubeDB XML Data ]
                    │
                    ▼
          stm32-data-sources
                    │
                    ▼
      stm32-data (chips.rs regex matcher)
                    │
                    ▼
          stm32-metapac-gen
                    │
                    ▼
        stm32-metapac (Generated crate)
                    │
                    ▼
             embassy-stm32
```

---

## 2. IP Version Matching in `stm32-data/chips.rs`

`stm32-data` parses raw XML hardware definitions and assigns specific register block implementations based on IP version strings:

```rust
// In stm32-data/chips.rs regex rules:
(".*:I2C:i2c2_v1_1", ("i2c", "v2", "I2C")),
(r".*TIM\d.*:gptimer.*", ("timer", "v1", "TIM_GP16")),
```

When compiling for `STM32F051K4`:
- `I2C1` IP version `"i2c2_v1_1"` matches driver `i2c_v2.yaml`.
- Output JSON generates memory offsets, interrupt vectors, and register block definitions consumed by `embassy-stm32`.

---

## 3. Conditional Module Compilation in `embassy-stm32`

In `embassy-stm32/src/lib.rs`:
```rust
#[cfg(i2c)]
pub mod i2c;
```

In `embassy-stm32/src/i2c/mod.rs`:
```rust
#[cfg_attr(i2c_v2, path = "v2.rs")]
#[cfg_attr(i2c_v3, path = "v3.rs")]
pub struct I2c<'d, M: Mode> { ... }
```

Because `i2c` and `i2c_v2` flags are auto-generated for `STM32F051K4`, `embassy-stm32` compiles `v2.rs` seamlessly!
