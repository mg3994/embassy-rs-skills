---
name: embassy-ptp-time-sync
description: Sub-microsecond time synchronization using IEEE 1588 Precision Time Protocol (embassy-ptp-driver) and Ethernet hardware timestamping.
---

# Embassy IEEE 1588 PTP Time Synchronization Skill

This skill details sub-microsecond network clock synchronization using `embassy-ptp-driver` and hardware timestamping peripherals.

---

## 1. IEEE 1588 PTP Overview

Precision Time Protocol (PTP) allows distributed networked devices to synchronize their system clocks with nanosecond precision over Ethernet.

---

## 2. Hardware Clock Interface (`embassy-ptp-driver`)

`embassy-ptp-driver` defines a network-stack-independent interface for reading and adjusting hardware PTP timers.

```rust
use embassy_ptp_driver::{PtpClock, Timestamp};

pub async fn synchronize_clock<C: PtpClock>(clock: &mut C, offset_nanos: i64) {
    // Adjust hardware clock frequency or add step offset
    clock.adjust_add_end_nanos(offset_nanos).unwrap();

    let now: Timestamp = clock.now().unwrap();
    defmt::info!("Synchronized PTP Hardware Time: {}.{:09} s", now.seconds, now.nanos);
}
```
