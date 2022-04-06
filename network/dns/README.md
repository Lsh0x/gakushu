### Description 

DNS is used to resolve a domain name into an ip.
When going on your favorite web site, your enter an url on the browser.
Lets say `https://github.com`, in order to comunicate with github you need to communicate with their servers, to do so your need an ip to send the requests

DNS is here to transform domain name into ip. 

But this protocole have severe issue in term of privacy. The requests are in encrypted, which mean that someone spying can see the url of the sites you visit. 
For example even if the website is using `https` so the content is encapsuled in an encrypted payload in still can see the website you are visiting to.

In order to fix that there is multiple solutions like [dnscrypt](./dnscrypt.md) where the request are done in an encrypted form. Meaning that the website doesn't appear in plan text on the network

