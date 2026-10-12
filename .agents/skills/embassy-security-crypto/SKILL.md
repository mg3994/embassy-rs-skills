---
name: embassy-security-crypto
description: Hardware cryptographic engines (AES, SHA-256, TRNG), Arm TrustZone-M, MPU memory protection, secure boot signing keys, and flash encryption.
---

# Embassy Security & Cryptography Skill

This skill covers hardware-accelerated cryptographic acceleration (AES, SHA-256, TRNG), Memory Protection Unit (MPU) setup, Arm TrustZone-M security domains, and signed firmware bootloaders.

---

## 1. Hardware True Random Number Generation (TRNG)

Hardware TRNG peripherals generate cryptographic entropy for seed generation, network TLS keys, and nonce values.

```rust
use embassy_stm32::rng::Rng;

let mut rng = Rng::new(p.RNG, Irqs);

let mut random_bytes = [0u8; 16];
rng.async_fill_bytes(&mut random_bytes).await.unwrap();
defmt::info!("Random key entropy: {=[u8]:hex}", random_bytes);
```

---

## 2. Hardware Cryptographic Accelerators (AES & HASH/SHA-256)

Hardware AES and SHA acceleration offloads CPU processing during packet encryption.

```rust
// On supported chips (e.g., STM32 / ESP32 / RP2350):
use embassy_stm32::cryp::*;

let mut cryp = Cryp::new(p.CRYP);
let key = [0u8; 16]; // AES-128 key
let mut cipher_text = [0u8; 64];
let plain_text = [0x42u8; 64];

cryp.encrypt_cbc(&key, &iv, &plain_text, &mut cipher_text).await;
```

---

## 3. Memory Protection Unit (MPU) & Stack Overflow Protection

The ARM Cortex-M Memory Protection Unit (MPU) traps unauthorized null-pointer accesses and stack overflow memory corruption.

### Protecting Stack Boundaries (`flip-link`)
In `.cargo/config.toml`, use `flip-link` as the target runner wrapper:

```toml
[target.'cfg(all(target_arch = "arm", target_os = "none"))']
runner = "probe-rs run --chip STM32F407VGTx"
# flip-link places the zero-cost stack overflow guard region below the stack
```

---

## 4. Secure Firmware Signatures (ed25519)

As detailed in `embassy-usb-bootloader`, store public verification keys in code and sign firmware image binaries using `signify` or `ed25519-dalek`:

```rust
static PUBLIC_KEY: &[u8] = include_bytes!("key.pub");

// Verify image before triggering A/B partition swap
updater.verify_and_mark_updated(PUBLIC_KEY, signature, firmware_len).unwrap();
```
