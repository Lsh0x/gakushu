## Ethernet Layer

This is the base of a pcap packet, the first bytes of an pcap packet represent the ethernet.
A physical Ethernet packet will look like this: 

```
| Preambule | Destination MAC Address |  Source MAC Address   |  Type/Length |  User Data  | Frame Check Sequence  |
|     8     |            6            |          6            |       2      |  46-1500    |          4            |
```

Source: 
* https://wiki.wireshark.org/Ethernet
