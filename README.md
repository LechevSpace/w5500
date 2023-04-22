# W5500 Driver

This crate is a driver for the WIZnet W5500 chip.  The W5500 chip is a hardwired TCP/IP embedded Ethernet controller
that enables embedded systems using SPI (Serial Peripheral Interface) to access the LAN. It is one of the
more popular ethernet modules on Arduino platforms.


[![Build Status](https://github.com/kellerkindt/w5500/workflows/Rust/badge.svg)](https://github.com/kellerkindt/w5500/actions?query=workflow%3ARust)
[![License](https://img.shields.io/badge/license-MIT%2FApache--2.0-blue.svg)](https://github.com/kellerkindt/w5500)
[![Crates.io](https://img.shields.io/crates/v/w5500.svg)](https://crates.io/crates/w5500)
[![Documentation](https://docs.rs/w5500/badge.svg)](https://docs.rs/w5500)
[![PRs Welcome](https://img.shields.io/badge/PRs-welcome-brightgreen.svg)](https://github.com/kellerkindt/w5500/issues/new)


## Embedded-HAL

Embedded-HAL is a standard set of traits meant to permit communication between MCU implementations and hardware drivers
like this one.  Any microcontroller that implements the
[`spi::FullDuplex<u8>`](https://docs.rs/embedded-hal/0.2.3/embedded_hal/spi/trait.FullDuplex.html) interface can use
this driver.

# Example Usage

Below is a basic example of sending UDP packets to a remote host.  An important thing to confirm is the configuration
of the SPI implementation.  It must be set up to work as the W5500 chip requires.  That configuration is as follows:

* Data Order: Most significant bit first
* Clock Polarity: Idle low
* Clock Phase: Sample leading edge
* Clock speed: 33MHz maximum

```rust,no_run
use embedded_nal::{IpAddr, Ipv4Addr, SocketAddr};
#
# struct Mock;
# 
# impl embedded_hal::blocking::spi::Transfer<u8> for Mock {
#     type Error = ();
# 
#     fn transfer<'w>(&mut self, words: &'w mut [u8]) -> Result<&'w [u8], Self::Error> {
#         words[0] = 0x04;
#         Ok(words)
#     }
# }
# 
# impl embedded_hal::spi::FullDuplex<u8> for Mock {
#     type Error = ();
# 
#     fn read(&mut self) -> nb::Result<u8, Self::Error> {
#         Ok(0)
#     }
# 
#     fn send(&mut self, word: u8) -> nb::Result<(), Self::Error> {
#         Ok(())
#     }
# }
# 
# impl embedded_hal::blocking::spi::Write<u8> for Mock {
#     type Error = ();
# 
#     fn write(&mut self, words: &[u8]) -> Result<(), Self::Error> {
#         Ok(())
#     }
# }
# 
# impl embedded_hal::digital::v2::OutputPin for Mock {
#     type Error = ();
# 
#     fn set_low(&mut self) -> Result<(), Self::Error> {
#         Ok(())
#     }
# 
#     fn set_high(&mut self) -> Result<(), Self::Error> {
#         Ok(())
#     }
# }

{
    use embedded_nal::UdpClientStack;

    let mut spi = Mock;
    let mut cs = Mock;

    let mut device = w5500::UninitializedDevice::new(w5500::bus::FourWire::new(spi, cs))
            .initialize_manual(
                    w5500::MacAddress::new(0, 1, 2, 3, 4, 5),
                    Ipv4Addr::new(192, 168, 86, 79),
                    w5500::Mode::default()
            ).unwrap();

    // Allocate a UDP socket to send data with
    let mut socket = device.socket().unwrap();

    // Connect the socket to the IP address and port we want to send to.
    device.connect(&mut socket,
        SocketAddr::new(IpAddr::V4(Ipv4Addr::new(192, 168, 86, 38)), 8000),
    ).unwrap();

    // Send the data
    nb::block!(device.send(&mut socket, &[104, 101, 108, 108, 111, 10]));

    // Optionally close the socket and deactivate the device
    device.close(socket);
    let (spi_bus, inactive_device) = device.deactivate();
}

// Alternatively, you can use the W5500 where a SPI bus is shared between drivers:
{
    use embedded_nal::TcpClientStack;

    let mut spi = Mock;
    let mut cs = Mock;

    let mut device: Option<w5500::InactiveDevice<w5500::Manual>> = None; // maybe: previously initialized device
    let mut socket: Option<w5500::tcp::TcpSocket> = None; // maybe: previously opened socket

    if let (Some(socket), Some(device)) = (socket.as_mut(), device.as_mut()) {
        let mut buffer = [0u8; 1024];
        let mut active = device.activate_ref(w5500::bus::FourWireRef::new(&mut spi, &mut cs));

        // Read from the device.
        active.receive(socket, &mut buffer).unwrap();
    }
}
```
## Todo

In no particular order, things to do to improve this driver.

* Add support for TCP server implementations
* Add support for DHCP
