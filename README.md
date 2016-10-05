# unifi-inform-protocol

Information about the reverse engineered inform protocol used in Ubiquiti's UniFi access points
Keeping this really simple, everything is in this file (README.md) so you can read it easilly on github.

# Discovery
When an unadopted UAP is connected to the network, it will start announcing itself on the layer 2 network. It will do this in 2 ways:

 * As a broadcast packet (`255.255.255.255`)
 * As a multicast packet to `233.89.188.1`

The same announcement packet is sent to both these addresses. This packet is an UDP packet sent to port 10001

This packet is in the TLV format (type, length, value)

The following packet types are available:

 * `0x01`: Hardware address
 * `0x02`: IP Info
 * `0x03`: Firmware version
 * `0x06`: Username
 * `0x07`: Salt
 * `0x08`: Random Challenge
 * `0x09`: Challenge
 * `0x0A`: Uptime
 * `0x0B`: Hostname (Always `UBNT` for an unadopted AP)
 * `0x0C`: Platform
 * `0x0D`: ESSID
 * `0x0E`: WMode ??
 * `0x0F`: Webui ??
 * `0x14`: Model

Note that not all these options may be available at all times

### Full packet format
Note: all data is in network byte format (Big endian).

 * 2 bytes: Packet length
 * Repeated `n` times:
   * 1 byte: Value type (see above)
   * 2 bytes: Value length `l`
   * `l` bytes: Value

# Adoption process
When the unifi controller is on the same layer 2 network as the UAP, the controller will have discovered the UAP after this announcement packet is received and will list it in the devices list as unadopted
If the UAP is not on the same layer 2 network as the controller, and you want to use layer 3 adoption, you have to ssh into the UAP (default username `ubnt`, default password `ubnt`), and issue the command `set-inform http://ip-of-controller:port/inform`. When this is done, the UAP will try to connect to the controller to be adopted. If the UAP was not known yet to the controller, it will be listed in the devices list after this packet is sent, but it is not adopted yet.

If the UAP is on the same L2 network as the controller and you click `Adopt` in the controller, the controller will connect to the UAP over ssh (using the default `ubnt`:`ubnt` credentials) and issue the following command:
`/usr/bin/syswrapper.sh set-adopt http://ip-of-controller:port/inform <random generated 16 bit hex string (the encryption key)>`

Now the UAP knows the encryption key and will use it to connect to the controller

# Inform protocol spec
An inform packet is an http POST request to the `/inform` url. It is a binary packet. The format is as follows:
 * 4 bytes: Magic header. Always `TNBU` (`UBNT` reversed)
 * 4 bytes: Packet version (Currently always 0)
 * 6 bytes: AP mac address
 * 2 bytes: Flags
   * 0x01: Encrypted
   * 0x02: Compressed
 * 16 bytes: Initialization Vector (IV) for encryption
 * 4 bytes: Data Version
 * 4 bytes: Payload length `l`
 * `l` bytes: Payload

The encryption of the payload is aes-128-cbc, without padding.
The encryption key is the key sent to the UAP while adopting it (see Adoption Process section).
If the UAP is already adopted, you can find the encryption key in the `cfg/mgmt` file in the default ssh folder on the UAP. See the `mgmt.authkey` line for the encryption key.

When decrypted, you see some json data. What all these values mean should be pretty clear.

# Configuring an UAP
Coming soon
