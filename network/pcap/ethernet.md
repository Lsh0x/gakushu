## Ethernet Layer

This is the base of a pcap packet, the first bytes of an pcap packet represent the ethernet.
A physical Ethernet packet will look like this: 

### Protocol version 1

| Preambule | Start frame delimite | Destination MAC Address | Source MAC Address |  Type/Length | User Data | Frame Check Sequence |
| --------- | -------------------- | ----------------------- | ------------------ | ------------ | --------- | -------------------- |
| 7 | 1 | 6 | 6 | 2 | 46-1500 | 4 |


### Protocol version 2

| Destination MAC Address | Source MAC Address |  Type/Length | User Data | Frame Check Sequence |
| ----------------------- | ------------------ | ------------ | --------- | -------------------- |
| 6 | 6 | 2 | 46-1500 | 4 |


Source: 
* https://wiki.wireshark.org/Ethernet
* https://en.wikipedia.org/wiki/Ethernet_frame
