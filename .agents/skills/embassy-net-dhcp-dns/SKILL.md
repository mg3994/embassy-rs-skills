---
name: embassy-net-dhcp-dns
description: Advanced embassy-net network stack configuration, DHCP client/server, static IP fallback, and async DNS domain name resolution.
---

# Embassy Network DHCP & DNS Skill

This skill details DHCP IP assignment, static IP fallback, and async DNS domain resolution using `embassy-net`.

---

## 1. Async DNS Domain Resolution (`dns_query`)

To resolve domain names (e.g. `api.example.com`) to IPv4/IPv6 addresses asynchronously:

```rust
use embassy_net::dns::DnsQueryType;
use embassy_net::Stack;

pub async fn resolve_server_ip(stack: &Stack<impl embassy_net::driver::Driver>) -> Option<embassy_net::IpAddress> {
    // Perform DNS query
    let addrs = stack.dns_query("api.example.com", DnsQueryType::A).await.ok()?;
    addrs.first().cloned()
}
```

---

## 2. Static IP Fallback Configuration

When DHCP fails to lease an IP within a timeout interval, fall back to a static IP address:

```rust
use embassy_net::{Config, Ipv4Address, Ipv4Cidr, StaticConfigV4};
use embassy_time::{Duration, Timer};

async fn configure_network_with_fallback(stack: &Stack<impl embassy_net::driver::Driver>) {
    // Wait for DHCP lease up to 5 seconds
    let dhcp_timeout = Timer::after(Duration::from_secs(5));
    let wait_dhcp = stack.wait_config_up();

    if embassy_futures::select::select(wait_dhcp, dhcp_timeout).await.is_err() {
        defmt::warn!("DHCP timeout! Applying static IP fallback...");

        let static_config = StaticConfigV4 {
            address: Ipv4Cidr::new(Ipv4Address::new(192, 168, 1, 100), 24),
            gateway: Some(Ipv4Address::new(192, 168, 1, 1)),
            dns_servers: heapless::Vec::new(),
        };

        stack.set_config_v4(Config::ipv4_static(static_config));
    }
}
```
