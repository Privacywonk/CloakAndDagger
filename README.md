# All networks are adversarial...

...just depends on your perspective.

## DigitalOcean DNS+VPN Tutorial 

A quick tutorial with example configuration files & instructions to manually setup a recursive DNS server with the ability to blocks ad-networks and other known badness along with a  VPN setup. This creates a self-contained privacy & security enhancing service that you can use as a safe network exit for your phones, networks, etc. For this tutorial, I am using a FreeBSD 12 host on DigitalOcean, all directions will be for that system. Porting over to your Linux distro of choice should be trivial....


### 2. Chose your adventure
Two options are presented below. Wireguard is prefered for simplicity and performance. StrongSwan is presented for knowledge around IPSEC, certificates, etc., but it is much more complicated to configure and get running.

 - [Wireguard Instructions](https://github.com/Privacywonk/CloakAndDagger/tree/master/wireguard)
 - [StrongSwan Instructions](https://github.com/Privacywonk/CloakAndDagger/tree/master/strongswan)



## Authors

* **me** - *Initial work* - [PrivacyWonk](https://github.com/PrivacyWonk)

## Contributors/Credits/References

* **Rob Seastrom** - *DNS Ninja* - [Rob Seastrom](https://github.com/res3066) - guidance and a gut check on all things DNS.
* **oogali** - *Network Ninja* - [oogali](https://github.com/oogali) - guidance, gut check, and troubleshooting on firewall and ipsec
* [Pi-hole](https://docs.pi-hole.net/guides/unbound/) as All-Around DNS Solution -- initial inspiration to replicate
* [Calomel](https://calomel.org/unbound_dns.html) Unbound Write up -- write up that got me thinking...
* [strongSwan](https://wiki.strongswan.org/projects/strongswan/wiki) -- great documentation
* [IPFW Firewall Examples](http://www.freebsdonline.com/content/view/725/531/)
* [FreeBSD Handbook (PDF)](http://ftp.freebsd.org/pub/FreeBSD/doc/handbook/book.pdf)
