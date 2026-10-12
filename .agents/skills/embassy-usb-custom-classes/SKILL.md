---
name: embassy-usb-custom-classes
description: Constructing custom USB device class drivers, handling control requests, bulk IN/OUT endpoints, and descriptor building using embassy-usb.
---

# Embassy Custom USB Class Drivers Skill

This skill explains how to build custom USB device classes (vendor-specific protocols, WinUSB, WebUSB) using `embassy-usb` endpoint abstractions and descriptor builders.

---

## 1. Custom USB Class Architecture

A custom USB class driver consists of:
1. **Endpoint Allocation:** Requesting Bulk/Interrupt IN and OUT endpoints from `embassy_usb::Builder`.
2. **Interface Descriptor:** Registering a custom USB interface descriptor (`Class = 0xFF` for vendor-specific).
3. **Control Request Handler:** Implementing `ControlHandler` to respond to setup packets (`GET_DESCRIPTOR`, `VENDOR_REQUEST`).
4. **Data Loop:** Asynchronously calling `endpoint.read_packet()` and `endpoint.write_packet()`.

---

## 2. Implementation Example

```rust
use embassy_usb::control::{ControlHandler, Request, RequestType, OutResponse, InResponse};
use embassy_usb::driver::{Driver, EndpointIn, EndpointOut};
use embassy_usb::Builder;

pub struct CustomUsbClass<'d, D: Driver<'d>> {
    read_ep: D::EndpointOut,
    write_ep: D::EndpointIn,
}

impl<'d, D: Driver<'d>> CustomUsbClass<'d, D> {
    pub fn new(builder: &mut Builder<'d, D>) -> Self {
        let mut function = builder.function(0xFF, 0x00, 0x00);
        let mut interface = function.interface();
        let mut alt = interface.alt_setting(0xFF, 0x00, 0x00, None);

        let read_ep = alt.endpoint_bulk_out(64);
        let write_ep = alt.endpoint_bulk_in(64);

        Self { read_ep, write_ep }
    }

    pub async fn read(&mut self, buf: &mut [u8]) -> Result<usize, D::Error> {
        self.read_ep.read_packet(buf).await
    }

    pub async fn write(&mut self, buf: &[u8]) -> Result<(), D::Error> {
        self.write_ep.write_packet(buf).await
    }
}
```
