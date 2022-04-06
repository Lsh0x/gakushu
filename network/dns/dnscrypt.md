### Overview
A flexible DNS proxy, with support for modern encrypted DNS protocols such as 
- [DNSCrypt v2](https://dnscrypt.info/protocol),
- [DNS-over-HTTPS](https://www.rfc-editor.org/rfc/rfc8484.txt)
- [Anonymized DNSCrypt](https://github.com/DNSCrypt/dnscrypt-protocol/blob/master/ANONYMIZED-DNSCRYPT.txt)
- [ODoH (Oblivious DoH)](https://github.com/DNSCrypt/dnscrypt-resolvers/blob/master/v3/odoh-servers.md).

### Installation 

Installation can be done using this [instruction](https://github.com/dnscrypt/dnscrypt-proxy/wiki/installation)

### Configuration file 

```sh
curl https://raw.githubusercontent.com/jedisct1/encrypted-dns-server/master/example-encrypted-dns.toml > dnscrypt-proxy.toml
```

### Adding your own encrypted-dns server

You might want run your own [[encrypted-dns]] server. 
In order to make dnscrypt-proxy able to communicate with it you need to create a new source.


The basic format looks like this.

```sh
  [sources.'public-resolvers']  
    urls = ['https://raw.githubusercontent.com/DNSCrypt/dnscrypt-resolvers/master/v3/public-resolvers.md', 'https://download.dnscrypt.info/resolvers-list/v3/public-resolvers.md', 'https://ipv6.download.dnscrypt.info/resolvers-list/v3/public-resolvers.md', 'https://download.dnscrypt.net/resolvers-list/v3/public-resolvers.md']
    cache_file = 'public-resolvers.md'
    minisign_key = 'RWQf6LRCGA9i53mlYecO4IzT51TGPpvWucNSCh1CBM0QTaLn73Y7GFO3'
    refresh_delay = 72
    prefix = ''
```

The different urls will be fetch and the content saved into the cache_file.
The format looks like that : 

```sh
## acsacsar-ams-ipv4

Public non-censoring, non-logging, DNSSEC-capable, DNSCrypt-enabled DNS resolver hosted on Scaleway by @acsacsar (twitter)

sdns://AQcAAAAAAAAADTUxLjE1OC4xNjYuOTcgAyfzz5J-mV9G-yOB4Hwcdk7yX12EQs5Iva7kV3oGtlEgMi5kbnNjcnlwdC1jZXJ0LmFjc2Fjc2FyLWFtcy5jb20
```

In case where your source file is local you can avoid the urls params and use cache_file property directly

```sh

  [sources.'my-sources']
    cache_file = '/path/to/my-sources.md'
    minisign_key = 'RWQf6LRCGA9i53mlYecO4IzT51TGPpvWucNSCh1CBM0QTaLn73Y7GFO3'
    refresh_delay = 72
    prefix = ''
```

The minisign_key can be obtain by using  [minisign](https://github.com/jedisct1/minisign) and sign il the `/path/to/my-sources.md` with the old format

```sh
# Initialise key pair
minisign -G

# Sign the file
minisign -Slm /path/to/my-sources.md
```
