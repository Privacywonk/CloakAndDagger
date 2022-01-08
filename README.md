# All networks are adversarial...

...just depends on your perspective.

## DigitalOcean DNS+VPN Tutorial 

A quick tutorial with example configuration files & instructions to manually setup a recursive DNS server with the ability to blocks ad-networks and known badness along with a  VPN setup. This creates a self-contained privacy & security enhancing service that you can use as a safe network exit for your phones, networks, etc. For this tutorial, I am using a FreeBSD 12 host, all directions will be for that system. Porting over to your Linux distro of choice should be trivial....


### 1. Recursive caching DNS w/ Ad-blocking using unbound
The service uses DNSSEC and 0x20-encoded random bits to foil spoofing attempts and creates a blackhole for known advertising networks, malware sites, and more. There will be a lot unexplained in this post. I highly encourage you to read up on each of the configuration options selected and understand how they work. Unbound's [documentation is fantastic](https://nlnetlabs.nl/documentation/unbound/). I have this in an IPv4 configuration only but you may want to embrace IPv6. Should make for simple config tweaks. 

#### Pre-work
1. Install unbound and bind-tools on your cloud host.  `pkg install unbound bind-tools`
2. Download latest root.hints file. `wget https://www.internic.net/domain/named.root -O /usr/local/etc/unbound/root.hints`
	1. Add this to crontab (root) to update every six months: 
	`0 0 1 */6 * wget https://www.internic.net/domain/named.root -O /usr/local/etc/unbound/root.hints && service unbound reload 2>&1`
3. Setup `/usr/local/etc/unbound/root.key` for initial trust -- See [How to Enable DNSSEC](https://nlnetlabs.nl/documentation/unbound/howto-anchor/) for authoritative documentation
	1. Essentially, you need to create `/usr/local/etc/unbound/root.key` with the latest data displayed on the page above...Here are the 2010-2011 and the 2017 trust anchors for the root zone. This is the syntax that you can use to provide an initial value for the root.key file. Unbound will update moving forward:
	
	```
    . IN DS 19036 8 2 49AAC11D7B6F6446702E54A1607371607A1A41855200FD2CE1CDDE32F24E8FB5
	. IN DS 20326 8 2 E06D44B80B8F1D39A95C0B0D7C65D08458E880409BBC683457104237C7F8EC8D
    ```
    
4. Install void-zones-tools package for ad blocking `pkg install void-zones-tools`
	1. Modify `/usr/local/bin/void-zones-update.sh` with [my version](https://github.com/Privacywonk/void-zones-tools/blob/master/void-zones-update.sh) that includes additional blocklist sources and an domain-based allowlist function.
	2. Add to crontab `30 07 * * * /usr/local/bin/void-zones-update.sh && service unbound restart > /dev/null 2>&1`
	3. Execute `/usr/local/bin/void-zones-update.sh` to download initial files
5. Configure new interface alias for dedicated DNS service on VPN network (change network to your own specifications)
	1. Create new virtual IP address for DNS server: `ifconfig vtnet0 10.99.99.99 netmask 255.255.255.0 alias`
	2. Edit `/etc/rc.conf` and add  `ifconfig_vtnet0_alias1="inet 10.99.99.99 netmask 255.255.255.0"` to make this address permanent (surivive reboots)
	3. Take care with this new interface. Explore other services that may bind to 0.0.0.0 (e.g. SSH). Do not expose unnecessary services via this new network.
6. Edit `/etc/rc.conf` and add `unbound_enable="YES"`

#### Configure Unbound
As configured, the service will run on 127.0.0.1 and 10.99.99.99. The 10.99.99.99 alias is for use on the to-be created VPN network.

1. Download the [unbound.conf](https://github.com/Privacywonk/CloakAndDagger/blob/master/unbound.conf) file
2. Edit variables (e.g. allowed networks, etc.)
3. `service unbound start`

#### Update resolv.conf
1. Add `127.0.0.1` and `10.99.99.99` to `/etc/resolv.conf`
2. Comment out whatever DNS servers DigitalOcean had pre-loaded

#### Test DNS
1. `tail -f /usr/local/etc/unbound/unbound.log` to verify successful startup or correct configuration issues
2. Test DNS & DNSSEC:
	1. `dig @127.0.0.1 dnssec-failed.org` should return `SERVFAIL` and no IP address
	2. `dig @127.0.0.1 internetsociety.org` should return `NOERROR` and an IP address

### 2. Configure VPN
Two options are presented below. Wireguard is prefered for simplicity and performance. StrongSwan is presented for knowledge around IPSEC, certificates, etc., but it is much more complicated to configure and get running.

 - [Wireguard Instructions](https://github.com/Privacywonk/CloakAndDagger/tree/master/wireguard)
 - [StrongSwan Instructions](https://github.com/Privacywonk/CloakAndDagger/tree/master/strongswan)


### 3. Test it all

1. Login to the VPN again and attempt to browser the web, SSH to remote hosts, ping remote hosts, etc. All should be working.
2. Reboot the server to ensure all settings remain in tact upon server restart...troubleshoot what goes bump.

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
