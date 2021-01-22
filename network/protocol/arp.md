## Adress Resolution Protocol 

It's use to link mac adress into an ip within a network.

### Header

| Hardware type | Protocol type | length hardware adress | Length protocol adress | Op code |
|---------------|---------------|------------------------|------------------------|---------|
| 2 | 2 | 1 | 1 | 2 |

* Harware type is the physical link used to transfere packet.
Typically ethernet for example

* Protocol type is the protocol used, ex 0x0800 represent ipv4

### Example with ethernet and ipv4

| Hardware type | Protocol Type | hardware/protocol length | op code | source mac adress | source ip | destination mac adress | destinations ip | 
|---------------|---------------|--------------------------|---------|-------------------|-----------|------------------------|-----------------|
| 2 | 2 | 2 | 2 |6 | 4 | 6 | 4 |

Source:
* file://usr/include/if_arp.h
* https://en.wikipedia.org/wiki/Address_Resolution_Protocol
