---
name: embassy-audio-dsp
description: Digital audio streams (I2S, SAI), DMA circular/double-buffering, PWM audio synthesis, and no_std DSP signal processing in Embassy.
---

# Embassy Digital Audio & DSP Skill

This skill covers digital audio streaming (I2S, SAI), circular DMA double-buffering, PWM audio tone generation, and `no_std` DSP processing.

---

## 1. Digital Audio Interfaces (I2S / SAI)

Inter-IC Sound (I2S) and Serial Audio Interface (SAI) stream multi-channel PCM audio data to DACs or receive audio from digital MEMS microphones.

### I2S Master Transmitter Configuration
```rust
use embassy_stm32::spi::Spi; // On STM32, I2S peripherals share hardware blocks with SPI

// Configure I2S audio sample rate (e.g., 44.1 kHz, 16-bit stereo)
// Streams audio PCM frames out via DMA...
```

---

## 2. Double-Buffering / Circular DMA Audio Streaming

To prevent audio glitches (underruns), DMA double-buffering triggers an interrupt when the first half of the audio buffer completes, allowing the CPU to synthesize or read the next PCM frame into half-buffer A while DMA plays half-buffer B.

```text
[ DMA Active Half-Buffer B ] ───► I2S Transceiver (Hardware Playing)
[ CPU Modifying Half-Buffer A ] ◄─── (Task filling next audio samples)
```

```rust
// Half-transfer complete callback -> Fill Buffer 0
// Transfer complete callback      -> Fill Buffer 1
```

---

## 3. PWM Audio Synthesis

For low-cost audio tones or buzzer output without a dedicated DAC chip:

```rust
use embassy_stm32::timer::simple_pwm::{PwmPin, SimplePwm};

let pin = PwmPin::new_ch1(p.PA8, embassy_stm32::gpio::OutputType::PushPull);
let mut pwm = SimplePwm::new(p.TIM1, Some(pin), None, None, None, Hertz(440), Default::default());

pwm.enable(embassy_stm32::timer::Channel::Channel1);
pwm.set_duty(embassy_stm32::timer::Channel::Channel1, pwm.get_max_duty() / 2); // 50% duty cycle 440 Hz tone
```
