# unifi-inform-protocol
This document contains information about the reverse engineered discovery and inform protocols used in Ubiquiti's UniFi access points, security gateways and switches.
All documentation is within this file (README.md) so you can read it easily on github.

## Discovery
When an unadopted UniFi AP, gateway, or switch (henceforth 'device') is connected to the network, it will start announcing itself on the layer 2 network. It will do this in 2 ways:

 * As a broadcast packet `255.255.255.255`
 * As a multicast packet to `233.89.188.1`

The same announcement packet is sent to both these addresses. This packet is a UDP packet sent to port 10001.

The packet always starts with `0x02 0x06` and is followed by a two-byte length field. The value of this length field does not include the header or the length field itself in the length calculation.

Following the header and length fields are the data fields. Each of these is in TLV format, described below.

The following data types are available:

 * `0x01`: Hardware address
 * `0x02`: IP Info
  * Per other docs, this may or may not have the IPv4 address preceded by the MAC.
  * On UniFi, I have only always ever seen the 6-byte MAC followed by the 4-byte IP.
 * `0x03`: Firmware version, long
  * This is the long version string, for example `BZ.qca956x.v3.4.14.3413.160119.2258`
 * `0x06`: Username
 * `0x07`: Salt
 * `0x08`: Random Challenge
 * `0x09`: Challenge
 * `0x0A`: Uptime in seconds
 * `0x0B`: Hostname
  * Always `UBNT` for an unadopted or factory reset AP
 * `0x0C`: Platform
  * Examples: `BZ2` for original UAP, `BZ2LR` for UAP-LR, `U7LR` for UAP-AC-LR, `US8` for US-8
 * `0x0D`: ESSID
 * `0x0E`: WMode ??
 * `0x0F`: Webui ??
 * `0x12`: Packet Increment
  * Length always four bytes, first packet starts at 1, increments for every packet sent
 * `0x13`: Hardware address (again)
 * `0x14`: Model
 * `0x15`: Platform (again)
 * `0x16`: Firmware version, short
  * This is the version seen in the controller and discovery tool, for example `3.7.17.5220`
 * `0x17`-`0x1a`: Unknown (always present together; `0x17` == 1, `0x18` == 0, `0x19` == 1, `0x20` == 0)
 * `0x1b`: Factory firmware version
  * Always in Major.Minor.Sub, for example `3.7.17`

Note that not all options are broadcast. In particular, UniFi APs on 3.7.x firmware are seen to broadcast these data fields, in this order:
`0x02 0x01 0x0a 0x0b 0x0c 0x03 0x16 0x15 0x17 0x18 0x19 0x1a 0x13 0x12 0x1b`

### Packet format
Note: all data is in network byte format (Big endian).

 * 2 bytes: `0x02 0x06`
 * 2 bytes: Payload length
 * Payload data of sequential TRV data fields, formatted as such:
   * 1 byte: Value type (see above)
   * 2 bytes: Value length `len`
   * `len` bytes: Value

## Adoption process
When the UniFi controller is on the same layer 2 network as the UniFi device, the controller will have discovered the device after this announcement packet is received and will list it in the device list as unadopted.

If the device is not on the same layer 2 network as the controller, and you want to use layer 3 adoption, you will need to use a discovery utility to set the inform URL, or SSH into the device (default username `ubnt`, default password `ubnt`), and issue the command `set-inform http://ip-of-controller:port/inform`. When this is done, the device will try to connect to the controller to be adopted. If the device was not known yet to the controller, it will be listed in the devices list after this packet is received.

If the device is on the same L2 network as the controller and you click `Adopt` in the controller, the controller will connect to the device over ssh (using the default `ubnt`:`ubnt` credentials) and issue the following command:
`/usr/bin/syswrapper.sh set-adopt http://ip-of-controller:port/inform <random generated 16 bit hex string (the encryption key)>`

Now the device knows the encryption key and will use it to connect to the controller.

TODO: document L3 adoption

## Inform protocol spec
An inform packet is an http POST request to the `/inform` url. It is a binary packet. The format is as follows:
 * 4 bytes: Magic header. Always `TNBU` (`UBNT` reversed)
 * 4 bytes: Packet version (Currently always 0)
 * 6 bytes: AP mac address
 * 2 bytes: Flags
   * `0x01`: Encrypted
   * `0x02`: Compressed
 * 16 bytes: Initialization Vector (IV) for encryption
 * 4 bytes: Data Version
 * 4 bytes: Payload length `len`
 * `len` bytes: Payload

The encryption of the payload is aes-128-cbc, without padding.
The encryption key is the key sent to the UAP while adopting it (see Adoption Process section).
If the UAP is already adopted, you can find the encryption key in the `cfg/mgmt` file in the default ssh folder on the UAP. See the `mgmt.authkey` line for the encryption key.

When decrypted, you see some json data. What all these values mean should be pretty clear.

# Configuring an UAP
Coming soon
