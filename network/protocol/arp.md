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


### Op code

| Code name | Value |
|-----------|-------|
| REQUEST | 0x1 |
|  REPLY | 0x2 |
| RREQUEST | 0x3 |
| RREPLY | 0x4 |
| InREQUEST | 0x8 |
| InREPLY | 0x09 |
| NAK | 0xa |


```
/* ARP protocol opcodes. */
#define	ARPOP_REQUEST	1		/* ARP request.  */
#define	ARPOP_REPLY	2		/* ARP reply.  */
#define	ARPOP_RREQUEST	3		/* RARP request.  */
#define	ARPOP_RREPLY	4		/* RARP reply.  */
#define	ARPOP_InREQUEST	8		/* InARP request.  */
#define	ARPOP_InREPLY	9		/* InARP reply.  */
#define	ARPOP_NAK	10		/* (ATM)ARP NAK.  */
```

Source:
* file://usr/include/if_arp.h
* https://en.wikipedia.org/wiki/Address_Resolution_Protocol
