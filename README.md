# All networks are adversarial. 

## DigitalOcean DNS+VPN Tutorial 

A quick tutorial with example configuration files & instructions to manually setup a recursive DNS server with the ability to blocks ad-networks and known badness along with a strongSwan VPN setup. This creates a self-contained privacy & security enhancing service that you can use as a safe network exit for your phones, networks, etc. For this tutorial, I am using a FreeBSD 12 host, all directions will be for that system. Porting over to your Linux distro of choice should be trivial...just directory structure changes.


### Recursive caching DNS w/ Ad-blocking using unbound
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
	1. Modify `/usr/local/bin/void-zones-update.sh` with my version that includes 6 additional blacklist sources
	2. Add to crontab `30 07 * * * /usr/local/bin/void-zones-update.sh && service unbound reload > /dev/null 2>&1`
5. Configure new interface alias for dedicated DNS service on VPN network (change network to your own specifications)
	1. Edit /etc/rc.conf and add  `ifconfig_vtnet0_alias1="inet 10.99.99.99 netmask 255.255.255.0"`
	2. Take care with this new interface. Explore other services that may bind to 0.0.0.0 (e.g. SSH). Do not expose unnecessary services via this new network.

#### Configure Unbound
As configured, the service will run on 127.0.0.1 and 10.99.99.99. The 10.99.99.99 alias is for use on the to-be created VPN network.

1. Download the [unbound.conf](https://github.com/Privacywonk/CloakAndDagger/blob/master/unbound.conf) file
2. Edit variables (e.g. allowed networks, etc.)
3. `service unbound enable`
4. `service unbound start`
5. `tail -f /usr/local/etc/unbound/unbound.log` to verify successful startup or correct configuration issues
6. Test DNS & DNSSEC:
	1. `dig @127.0.0.1 dnssec-failed.org. soa` should return `SERVFAIL` and no IP address
	2. `dig @127.0.0.1 internetsociety.org. soa` should return `NOERROR` and an IP address


#### Update resolv.conf
1. Add 127.0.0.1 and 10.99.99.99 to /etc/resolv.conf


### strongswan VPN 

#### Pre-work
1. Verify kernel has NAT Tunneling (natt) and the crypto device (user-mode access to hardware-accelerated cryptography) enabled`/sbin/sysctl -a |egrep crypto|ipsec` look for `device crypto` and `kern.feature.ipsec` and `kern.feature.ipsec_natt = 1`
	1. if these are set to 0 or the device is missing, see [strongSwan documentation](https://wiki.strongswan.org/projects/strongswan/wiki/FreeBSD) for instructions to fix 
2. Install strongSwan `pkg install strongswan`

#### Create Certificate Infrastructure

Execute all of these commands from `/usr/local/etc/ipsec.d/`. Note, the Subject Alternative Name (SAN) must contain the IP address or DNS of the VPN server. I suggest keeping it simple and sticking the IP address. It is required for interoperability with android, IOS, Windows, Linux, etc. strongSwan has a wealth of documentation on PKI particulars for each situation and I highly suggest reading through it to fully understand what is being done below. 

##### 1. A Private Key

`ipsec pki --gen --type rsa --size 4096 --outform pem > private/ipsec-ca-key.pem`

##### 2. CA self-sign certificate

Update the Country (C), Organization (O), and Common Name (CN) to your own environment variables. 

`ipsec pki --self --flag serverAuth --in private/ipsec-ca-key.pem --type rsa --digest sha256 --dn "C=US, O=TEST, CN=TEST VPN CA" --ca --lifetime 3650 --outform pem > cacerts/ipsec-ca-cert.pem`

##### 3. VPN Server Key 

`ipsec pki --gen --type rsa --size 4096 --outform pem > private/ipsec-server-key.pem`

##### 4. Server Cert

`pki --pub --in private/ipsec-server-key.pem | pki --issue --outform pem --digest sha256 --lifetime 3650 --cacert cacerts/ipsec-ca-cert.pem --cakey private/ipsec-ca-key.pem --flag serverAuth --flag ikeIntermediate --dn "C=US, O=TEST, CN=VPN_SERVER_IP" --san "VPN_SERVER_IP" > certs/ipsec-server-cert.pem`

##### 5. Client Keys & Certs 

Pro tip - this section and the follow export sections into notepad and search/replace "client" with your client name for copy/pasta cert generation.


`ipsec pki --gen --outform pem > private/client.key.pem
`ipsec pki --pub --in private/client.key.pem | ipsec pki --issue --cacert cacerts/ipsec-ca-cert.pem --cakey cacerts/ipsec-ca-key.pem --dn "C=US, O=TEST, CN=client" --outform pem > certs/client.cert.pem`

##### 6. Script for CA & Server Keys

```
   #!/bin/bash
   # Change your values:
   cd /usr/local/etc/ipsec.d/
   vpn_ip="1.2.3.4"
   country="US"
   organization="TEST"


   ### do not change anything below this line
   ipsec pki --gen --type rsa --size 4096 --outform pem > private/ipsec-ca-key.pem

   ipsec pki --self --flag serverAuth --in private/ipsec-ca-key.pem --type rsa --digest sha256 --dn "C=${country}, O="${organization}", CN=${organization} CA" --ca --lifetime 3650 --outform pem > cacerts/ipsec-ca-cert.pem

   ipsec pki --gen --type rsa --size 4096 --outform pem > private/ipsec-server-key.pem   

   pki --pub --in private/ipsec-server-key.pem | pki --issue --outform pem --digest sha256 --lifetime 3650 --cacert cacerts/ipsec-ca-cert.pem --cakey private/ipsec-ca-key.pem --flag serverAuth --flag ikeIntermediate --dn "C=${country}, O=${organization}, CN=${vpn_ip}" --san "${vpn_ip}" > certs/ipsec-server-cert.pem
```

##### 7. Script for client certs and keys

```
   #!/bin/bash
   vpn_ip="1.2.3.4"
   country="US"
   organization="TEST"

   if [ "$1" == "" ]; then

       echo -n "Enter a client name:"
	   read client
   fi

   ipsec pki --gen --outform pem > private/client.key.pem
   ipsec pki --pub --in private/client.key.pem | ipsec pki --issue --cacert cacerts/ipsec-ca-cert.pem --cakey cacerts/ipsec-ca-key.pem --dn "C=${country}, O="${organization}", CN=${client}" --outform pem > certs/client.cert.pem

```


##### 8. Export Options for client certs, bundles, etc.
###### PCKS 12 Export

`openssl pkcs12 -export -inkey private/client.pem -in certs/client.cert.pem -name "client" -certfile cacerts/ipsec-ca-cert.pem -out cacerts/client.cert.p12`

###### Package for Mobile (android). Install ipsec-ca-cert.pem separately

`openssl pkcs12 -export -inkey private/client.key.pem -in certs/client.cert.pem -name "client" -caname "VPN_SERVER_IP"  -out cacerts/client.cert.p12`

###### Windows Cert File needs

`openssl x509 -outform der -in cacerts/ipsec-ca-cert.pem -out cacerts/ipsec-ca-cert.der`

### Configure strongSwan

Download the [ipsec.conf](https://github.com/Privacywonk/CloakAndDagger/blob/master/ipsec.conf) file from this repo and place it in ```/usr/local/etc/```. strongSwan uses terminology referring to the right and left of a connection. The right is the *server* side and the left side is the *client*

1. Update the ```rightca``` line to match the variables you changed in the Certificate Authority creation. Specifically the C, O, and CN need to match perfectly.
2. Update the ```leftid``` line to match the variables you set for C, O, and CN of the Server Certificate. Using the example above, CN should be the Server IP Address.
3. Update ```/usr/local/etc/ipsec.secrets``` to include the key files for your server and clients: 

```
: RSA /usr/local/etc/ipsec.d/private/ipsec-server-key.pem
: RSA /usr/local/etc/ipsec.d/private/laptop.key.pem
: RSA /usr/local/etc/ipsec.d/private/phone.key.pem
```
 
4. `service ipsec enable`
5. `service ipsec start`
6. Setup a test client (I suggest a linux client or android client using the strongSwan app). Follow the logs at `tail -f /var/log/charon.log` to see a connection working through. The logs will be pretty clear about connections not succeeding. They are *very* verbose...have fun googling errors. 
7. A tip for Windows 10 clients: There are limits to the ciphers you can use by default but they can be expanded either by regedit or via power shell with `Set-VpnConnectionIPsecConfiguration -ConnectionName "..." -AuthenticationTransformConstants SHA256128 -CipherTransformConstants AES256 -EncryptionMethod AES256 -IntegrityCheckMethod SHA256 -DHGroup Group14 -PfsGroup PFS2048`. Make sure to update the ConnectionName to what you want it called. References: [Trail Of Bits] (https://github.com/trailofbits/algo/issues/9) and [strongSwan Windows Documentation](https://wiki.strongswan.org/projects/strongswan/wiki/WindowsClients)
8. Check out [Apple Configurator Two](https://apps.apple.com/us/app/apple-configurator-2/id1037126344?mt=12) to help build configurations and ship them to your iOS devices.

## Todo 
1. Add firewall configuration 
2. Add suricata configuration for IDS

## Authors

* **me** - *Initial work* - [PrivacyWonk](https://github.com/PrivacyWonk)

## Contributors/Credits/References

* **Rob Seastrom** - *DNS Ninja* - [Rob Seastrom](https://github.com/res3066) - guidance and a gut check on all things DNS.
* **oogali** - *Network Ninja* - [oogali](https://github.com/oogali) - guidance and gut check on ipsec
* [Pi-hole] (https://docs.pi-hole.net/guides/unbound/) as All-Around DNS Solution -- initial inspiration to replicate
* [Calomel] Unbound Write up (https://calomel.org/unbound_dns.html) -- write up that got me thinking...
* [strongSwan] (https://wiki.strongswan.org/projects/strongswan/wiki) -- great documentation
