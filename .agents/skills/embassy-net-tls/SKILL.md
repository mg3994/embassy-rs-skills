---
name: embassy-net-tls
description: Transport Layer Security (TLS 1.3) encrypted sockets using embedded-tls on top of embassy-net TCP sockets.
---

# Embassy Transport Layer Security (TLS) Skill

This skill explains how to establish encrypted, authenticated TLS 1.3 connections on embedded microcontrollers using `embedded-tls` on top of `embassy-net` TCP sockets.

---

## 1. Encrypted Sockets with `embedded-tls`

`embedded-tls` is a zero-allocation, `no_std` pure-Rust TLS 1.3 implementation designed for embedded devices.

---

## 2. HTTPS / TLS Client Connection Example

```rust
use embedded_tls::{TlsConfig, TlsContext, TlsConnection, Aes128GcmSha256};
use embassy_net::tcp::TcpSocket;
use embassy_net::Stack;

pub async fn connect_tls_server<'a>(
    stack: &'static Stack<impl embassy_net::driver::Driver>,
    rx_buf: &'a mut [u8],
    tx_buf: &'a mut [u8],
    record_buf: &'a mut [u8],
) {
    let mut tcp_socket = TcpSocket::new(stack, rx_buf, tx_buf);

    // 1. Connect TCP socket to remote host on port 443
    tcp_socket.connect((embassy_net::Ipv4Address::new(93, 184, 216, 34), 443)).await.unwrap();

    // 2. Configure TLS Context
    let config = TlsConfig::new().with_server_name("example.com");
    let mut tls_connection: TlsConnection<'_, TcpSocket, Aes128GcmSha256> =
        TlsConnection::new(tcp_socket, record_buf);

    // 3. Perform TLS 1.3 Handshake
    tls_connection.open(TlsContext::new(&config, &mut rand_provider)).await.unwrap();
    defmt::info!("TLS 1.3 Handshake Successful!");

    // 4. Send encrypted HTTPS GET request
    tls_connection.write_all(b"GET / HTTP/1.1\r\nHost: example.com\r\n\r\n").await.unwrap();
}
```
