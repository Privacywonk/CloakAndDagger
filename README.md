# All networks are adversarial...

...just depends on your perspective.

## DigitalOcean DNS+VPN Tutorial 

A quick tutorial with example configuration files & instructions to manually setup a recursive DNS server with the ability to blocks ad-networks and known badness along with a strongSwan VPN setup. This creates a self-contained privacy & security enhancing service that you can use as a safe network exit for your phones, networks, etc. For this tutorial, I am using a FreeBSD 12 host, all directions will be for that system. Porting over to your Linux distro of choice should be trivial....


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
	1. Modify `/usr/local/bin/void-zones-update.sh` with [my version](https://github.com/Privacywonk/void-zones-tools/blob/master/void-zones-update.sh) that includes 6 additional blacklist sources
	2. Add to crontab `30 07 * * * /usr/local/bin/void-zones-update.sh && service unbound reload > /dev/null 2>&1`
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

### 2. strongswan VPN 

#### Pre-work
1. Verify kernel has NAT Tunneling (natt) and the crypto device (user-mode access to hardware-accelerated cryptography) enabled `/sbin/sysctl -a |egrep "crypto|ipsec"` look for `device crypto` and `kern.feature.ipsec` and `kern.feature.ipsec_natt = 1`
	1. if these are set to 0 or the device is missing, see [strongSwan documentation](https://wiki.strongswan.org/projects/strongswan/wiki/FreeBSD) for instructions to fix 
2. Install strongSwan `pkg install strongswan`
3. Add `strongswan_enable="YES"` to `/etc/rc.conf`

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

`ipsec pki --pub --in private/ipsec-server-key.pem | pki --issue --outform pem --digest sha256 --lifetime 3650 --cacert cacerts/ipsec-ca-cert.pem --cakey private/ipsec-ca-key.pem --flag serverAuth --flag ikeIntermediate --dn "C=US, O=TEST, CN=VPN_SERVER_IP" --san "VPN_SERVER_IP" > certs/ipsec-server-cert.pem`

##### 5. Client Keys & Certs 

Replace "client" with your desired name for the client certificate. This could me "mobile" or "laptop". Pay closee attention to the differences between the linux and windows cert handling.  Windows needs an extra clientAuth flag and *must* have a `--san dns:client` tag that is the same as the CN.

For linux/android/iOS certificates:

```
ipsec pki --gen --outform pem > private/client.key.pem
ipsec pki --pub --in private/client.key.pem | ipsec pki --issue --cacert cacerts/ipsec-ca-cert.pem --cakey cacerts/ipsec-ca-key.pem --dn "C=US, O=TEST, CN=client" --outform pem > certs/client.cert.pem
```

For Windows Certificates:


```
ipsec pki --gen --outform pem > private/client.key.pem
ipsec pki --pub --in private/client.key.pem | ipsec pki --issue --cacert cacerts/ipsec-ca-cert.pem --cakey cacerts/ipsec-ca-key.pem --dn "C=US, O=TEST, CN=client" --flag clientAuth --san dns:client --outform pem > certs/client.cert.pem
```

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

   ipsec pki --pub --in private/ipsec-server-key.pem | pki --issue --outform pem --digest sha256 --lifetime 3650 --cacert cacerts/ipsec-ca-cert.pem --cakey private/ipsec-ca-key.pem --flag serverAuth --flag ikeIntermediate --dn "C=${country}, O=${organization}, CN=${vpn_ip}" --san "${vpn_ip}" --san dns:"${vpn_ip}" > certs/ipsec-server-cert.pem
   
   #To prevent any whoopsies later, I suggest editing this file to comment out each directive line related to the CA after it's initial run.
```

##### 7. Script for client certs and keys

```
    #!/bin/bash
    country="US"
    organization="TEST"
    cd /usr/local/etc/ipsec.d

    if [ "$1" == "" ]; then

       echo -n "Enter a client name: "
           read client
    fi

    echo "Creating "${client}" private key..."
    ipsec pki --gen --outform pem > private/"${client}".key.pem
    
    echo "Creating "${client}" certificate..."
    ipsec pki --pub --in private/"${client}".key.pem | ipsec pki --issue --cacert cacerts/ipsec-ca-cert.pem --cakey private/ipsec-ca-key.pem --dn C="${country}", O="${organization}", CN="${client}" --outform pem > certs/"${client}".cert.pem
    ipsec pki --print -i certs/"${client}".cert.pem
       
    echo "Creating "${client}" Windows certificate..."
    ipsec pki --pub --in private/"${client}".key.pem | ipsec pki --issue --cacert cacerts/ipsec-ca-cert.pem --cakey private/ipsec-ca-key.pem --dn "C=${country}, O=${organization}, CN=${client}" --flag clientAuth --san dns:"${client}" --outform pem > certs/"${client}"-windows.cert.pem

    ipsec pki --print -i certs/"${client}"-windows.cert.pem

   echo "Packing for Windows Export (creating .p12 bundle)...certs/"${client}"-windows.cert.p12"
   openssl pkcs12 -export -inkey private/"${client}".key.pem -in certs/"${client}".cert.pem -name \""${client}"\" -certfile cacerts/ipsec-ca-cert.pem -out certs/"${client}"-windows.cert.p12 -passout pass:

```


##### 8. Export Options for client certs, bundles, etc.
###### PCKS 12 Export

`openssl pkcs12 -export -inkey private/client.key.pem -in certs/client.cert.pem -name "client" -certfile cacerts/ipsec-ca-cert.pem -out certs/client.cert.p12`

###### Package for Mobile (android). Install ipsec-ca-cert.pem separately

`openssl pkcs12 -export -inkey private/client.key.pem -in certs/client.cert.pem -name "client" -caname "VPN_SERVER_IP"  -out certs/client.cert.p12`

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
 
4. `service ipsec start`
5. Setup a test client (I suggest a linux client or android client using the strongSwan app). Follow the logs at `tail -f /var/log/charon.log` to see a connection working through. The logs will be pretty clear about connections not succeeding. They are *very* verbose...have fun googling errors. 
6. A tip for Windows 10 clients: There are limits to the ciphers you can use by default but they can be expanded either by regedit or via power shell
	1. Create the new VPN Connection `Settings > Network & Internet > VPN > Add a VPN connection.` Note the name you chose for the connection.
	2. Open PowerShell as an admin and run (make sure to update the ConnectionName) `Set-VpnConnectionIPsecConfiguration -ConnectionName "..." -AuthenticationTransformConstants SHA256128 -CipherTransformConstants AES256 -EncryptionMethod AES256 -IntegrityCheckMethod SHA256 -DHGroup Group14 -PfsGroup PFS2048`. References: [Trail Of Bits](https://github.com/trailofbits/algo/issues/9) and [strongSwan Windows Documentation](https://wiki.strongswan.org/projects/strongswan/wiki/WindowsClients)
7. A tip for iOS clients: Check out [Apple Configurator Two](https://apps.apple.com/us/app/apple-configurator-2/id1037126344?mt=12) to help build configurations and ship them to your iOS devices.

### 3. Firewall 

 IPFW firewall setup for SSH, DNS, IPSEC+NAT. 

#### Pre-work

1. Modify `/etc/rc.conf` to include:

```
firewall_enable="YES"
firewall_script="/usr/local/etc/ipfw.rules"
firewall_logif="YES"
gateway_enable="YES"
natd_enable="YES"
natd_interface="vtnet0"
natd_flags="-dynamic -m"
```
*Note* - comment out or delete `#firewall_type="open"` as it will conflict with the `firewall_script` directive.

2. Create `/usr/local/etc/ipfw.rules` and add the content below. Modify variables to your environment.

```
IPF="ipfw -q add"
WAN="[WANY interface here, e.g. vtnet0]"
WAN_IP="[YOUR IP HERE]"
strongSwanNetwork="10.99.99.0/24"
ipfw -q -f flush


ipfw -q -f flush
/sbin/ipfw -q table all flush

#Establish NAT 1 config
/sbin/ipfw -q nat 1 config if $WAN unreg_only reset

#Loop back interfaces
$IPF 10 allow all from any to any via lo0
$IPF 20 deny all from any to 127.0.0.0/8
$IPF 30 deny all from 127.0.0.0/8 to any
$IPF 40 deny tcp from any to any frag

# Catch spoofing from outside.
$IPF 45 deny ip from any to any not antispoof via $WAN

#NAT Inbound
$IPF 100 nat 1 ip4 from any to any in via $WAN
$IPF 105 check-state

#allow ping
$IPF 120 allow icmp from any to me

#Allow DNS
$IPF 130 allow tcp from me to any 53 out xmit $WAN setup keep-state
$IPF 140 allow udp from me to any 53 out xmit $WAN keep-state

#Allow strongSwan network
$IPF 187 allow ip from any to $strongSwanNetwork

# NAT Outbound traffic allow
$IPF 200 skipto 10000 tcp from $strongSwanNetwork to any out xmit $WAN
$IPF 210 skipto 10000 udp from $strongSwanNetwork to any out xmit $WAN
$IPF 215 allow tcp from me to any out via $WAN setup keep-state

# Incoming Rules - SSH (22), other services as needed
$IPF 500 allow tcp from any to me 22 in recv $WAN setup keep-state

#Incoming Rules - StrongSwan
$IPF 1010 allow udp from any to me 500 in recv $WAN keep-state
$IPF 1011 allow udp from any to me 4500 in recv $WAN keep-state
$IPF 1012 allow esp from any to any
$IPF 1013 allow ah from any to any
$IPF 1014 allow ipencap from any to any
$IPF 1030 allow udp from any to me in recv $WAN frag

# NAT rule for outgoing packets.
$IPF 10000 nat 1 tag 10000 ip4 from any to any out via $WAN
$IPF 10010 allow tcp from any to any out via vtnet0 tagged 10000 setup keep-state
$IPF 10020 allow tcp from any to any out via vtnet0 tagged 10000 setup keep-state
$IPF 10030 allow icmp from any to any out via vtnet0 tagged 10000

# deny everything else
$IPF 65534 deny log all from any to any


```

4. Load the in kernel NAT module: `kldload ipfw_nat` if not loaded.


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
