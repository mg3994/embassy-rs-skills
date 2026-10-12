---
name: embassy-design-patterns
description: Best-practice software architecture design patterns for large-scale Embassy applications, state machines, actor model, and driver isolation.
---

# Embassy Software Design Patterns Skill

This skill details architectural design patterns for building maintainable, scalable, and crash-resilient embedded applications in Embassy.

---

## 1. Actor Pattern with Channels

Decouple hardware peripheral controllers from application state using dedicated actor tasks that process message commands sent over `embassy_sync::channel::Channel`.

```text
[ Sensor Task ] ───(Command::SensorData)───┐
                                          ├───► [ Actor Task ] ───► [ Flash Storage ]
[ Network Task ] ───(Command::SyncTime)────┘
```

---

## 2. Event-Driven State Machines

Model complex peripheral control flows (such as cellular modem initialization or USB handshake states) as explicit Rust `enum` state machines driven inside an async loop.

```rust
enum ModemState {
    PowerOn,
    AtCheck,
    NetworkRegister,
    Connected,
    Failed,
}

async fn run_modem_state_machine(modem: &mut Modem) {
    let mut state = ModemState::PowerOn;
    loop {
        state = match state {
            ModemState::PowerOn => {
                modem.toggle_power_pin().await;
                ModemState::AtCheck
            }
            ModemState::AtCheck => {
                if modem.send_at_ping().await.is_ok() {
                    ModemState::NetworkRegister
                } else {
                    ModemState::Failed
                }
            }
            ModemState::NetworkRegister => {
                modem.wait_registration().await;
                ModemState::Connected
            }
            ModemState::Connected => {
                // Maintain telemetry loop...
                ModemState::Connected
            }
            ModemState::Failed => {
                Timer::after_secs(5).await;
                ModemState::PowerOn
            }
        };
    }
}
```
