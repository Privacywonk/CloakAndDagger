# Cloak and Dagger, wireguard version

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
	2. Add to crontab `30 07 * * * /usr/local/bin/void-zones-update.sh && service unbound restart > /dev/null 2>&1`
	3. Execute `/usr/local/bin/void-zones-update.sh` to download initial files
5. Configure new interface alias for dedicated DNS service on VPN network (change network to your own specifications)
	1. Create new virtual IP address for DNS server: `ifconfig vtnet0 10.99.99.99 netmask 255.255.255.0 alias`
	2. Edit `/etc/rc.conf` and add  `ifconfig_vtnet0_alias1="inet 10.99.99.99 netmask 255.255.255.0"` to make this address permanent (surivive reboots)
	3. Take care with this new interface. Explore other services that may bind to 0.0.0.0 (e.g. SSH). Do not expose unnecessary services via this new network.
6. Edit `/etc/rc.conf` and add `unbound_enable="YES"`

#### Configure Unbound
As configured, the service will run on 127.0.0.1 and 10.99.99.99. The 10.99.99.99 alias is for use on the to-be created VPN network.

1. Download the [unbound.conf](https://github.com/Privacywonk/CloakAndDagger/blob/master/wireguard/unbound.conf) file
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

### 2. Wireguard VPN 

#### Pre-work
1. Install wireguard `pkg install wireguard`
2. Ensure `/etc/rc.conf` is configured for firewall, NAT, etc:
```
gateway_enable="YES"
ipv6_gateway_enable="YES"
firewall_enable="YES"
firewall_nat_enable="YES"
firewall_script="/usr/local/etc/ipfw.rules"
firewall_logging="YES"
```
3. Edit `/etc/sysctl.conf` for forwarding & in-kernel NAT
```
# Wireguard specifics
net.inet.tcp.tso=0 #disable TCP segmentation offloading, see https://docs.freebsd.org/en/books/handbook/firewalls/#firewalls-ipfw
net.inet.ip.forwarding=1 #enabling forwarding
net.inet.ip.redirect=0 #no icmp redirects v4
net.inet6.ip6.redirect=0 #no icmp redirects v6
```

#### Configure Wireguard
Below is a walk through of how to get the wireguard server and two clients functioning with pre-shared keys. Adding additional clients is very easy. All of this is derrived from [https://www.wireguard.com/quickstart/](https://www.wireguard.com/quickstart/). I will provide some commentary below on the configuration to draw your attention to a few things. But I highly, highly, suggest reading the wireguard docs top to bottom.

##### 1. Generate Server key material
    cd /usr/local/etc/wireguard && umask 077 
    # Generate Server Private Key
    wg genkey > ServerPrivkey
    # Derive the server's public key from it  
    cat ServerPrivkey | wg pubkey > Serverpubkey

##### 2. Generate Client1 key material
    #Generate Client Private Key
    wg genkey > Client1Privkey
    # Derive the client's public key from it  
    cat Client1Privkey | wg pubkey > Client1pubkey
    # Optionally, create a pre-shared key (PSK) for a client 
    wg genpsk > Client1PSK

##### 3. Generate Client2 key material
    #Generate Client Private Key
    wg genkey > Client2Privkey
    # Derive the client's public key from it  
    cat Client2Privkey | wg pubkey > Client2pubkey
    # Optionally, create a pre-shared key (PSK) for a client 
    wg genpsk > Client2PSK
    
##### 4. Edit server wg0.conf
Edit the  `/usr/local/etc/wireguard/wg0.conf` file and add the following content. 

```
[Interface]
ListenPort = 443
PrivateKey = <contents ServerPrivKey>

[Peer]
PublicKey = <contents of Client1pubkey>
PresharedKey = <contents of Client1PSK>
AllowedIPs = 10.99.99.1/32

[Peer]
PublicKey = <contents of Client2pubkey>
PresharedKey = <contents of Client2PSK>
AllowedIPs = 10.99.99.2/32
``` 
Brief server config context...
 - **ListenPort** is the UDP port that your wireguard server will listen on. I am using 443 in this example as I do not use this host for anything and, generally, most networks allow 443 outbound. If you are running a HTTPS web server on your host, this may not work for you, Adjust here and be sure to adjust in the IPFW rules
 - **AllowedIPs** acts as an ACL for [cryptokey-routing](https://www.wireguard.com/#cryptokey-routing). In the server configuration, each peer (a client) will be able to send packets to the network interface with a source IP matching the corresponding list of allowed IPs. For example, when a packet is received by the server from Client1 above, after being decrypted and authenticated, if its source IP is 10.99.99.1, then it's allowed onto the interface; otherwise it's dropped. In this example there is one only allowed IP. However, lets say this peer was another router and behind it was a network of hosts on the 192.168.10.0/24 range. We would add that there so that our side could send to that side and vice versa: `AllowedIPs = 10.99.99.2/32, 192.168.10.0/24`
 - **PresharedKey** - Optional but used here education. PSKs are used to provide post-quantum resistance (sounds cool. Math is beyond me). Critically important, generate a PSK per client. Do not reuse PSKs.

##### 4. Edit client1.conf
```
[Interface]
Address = 10.99.99.1/32
PrivateKey = <contents of Client1Privkey>
DNS = 10.99.99.99

[Peer]
PublicKey = <contents of server pubkey>
PresharedKey = <contents of Client1PSK>
Endpoint = YourServerIP:443
AllowedIPs = 0.0.0.0/0, ::/0
```
Brief client config context:
 - **Address** - in the client config, sets and claims the address to use on the wireguard network. Note the /32 again.
 - **DNS** - sets the IP address the client should use for making DNS queries over the WG network. Note the absence of the /32 here. It is purposeful. 
 - **Endpoint** - the IP or hostname of your wireguard server and the port it is running on
 - **AllowedIPs** - in the client config is used to chose what traffic goes over the wireguard network. It could used to connect the client to the servers subnet or, as in our example, forward *all* traffic through wireguard. 

##### 5. Edit client2.conf
    [Interface]
    Address = 10.99.99.1/32
    PrivateKey = <contents of Client2Privkey>
    DNS = 10.99.99.99
    
    [Peer]
    PublicKey = <contents of server pubkey>
    PresharedKey = <contents of Client2PSK>
    Endpoint = YourServerIP:443
    AllowedIPs = 0.0.0.0/0, ::/0

### 3. Firewall 

IPFW firewall setup for SSH, DNS, IPSEC+NAT. Be sure to check out https://www.freebsd.org/doc/handbook/firewalls-ipfw.html for more information about IPFW, In Kernel NATing, and performance tuning options.

#### Pre-work

1. Check if ipfw_nat is loaded via kernel module: `kldstat |grep ipfw`. If not, `kldload ipfw_nat`

2. Create `/usr/local/etc/ipfw.rules` and add the content below. Modify variables to your environment.

```
#!/bin/sh

IPF="/sbin/ipfw -q add"
WAN="vtnet0"
WAN_IP="[YOUR SERVER IP]"
wgNetwork="10.99.99.0/24"
wg_port="443"

# Wireguard interface, matching the name in /etc/wireguard/*.conf
wg_iface="wg0"

/sbin/ipfw -q -f flush
/sbin/ipfw -q table all flush
/sbin/ipfw -q disable one_pass
/sbin/ipfw -q nat 1 config if $WAN same_ports unreg_only reset

#allow all for localhost/loop back
$IPF 10 allow ip from any to any via lo0
$IPF 15 allow ip from any to any via $wg_iface
$IPF 20 deny all from any to 127.0.0.0/8
$IPF 30 deny all from 127.0.0.0/8 to any
$IPF 40 deny tcp from any to any frag

# Catch spoofing from outside.
$IPF 45 deny ip from any to any not antispoof via $WAN

#nat config
$IPF 099 reass all from any to any in
$IPF 100 nat 1 ip from any to any in via $WAN

#check stateful rules. If marked as "keep-state" the packet has
#already passed through filters and is "OK" without futher rule matching
$IPF 105 check-state

#allow ping
$IPF 120 allow icmp from any to me
$IPF 121 allow icmp from me to any

#Allow DNS
$IPF 130 allow tcp from me to any 53 out xmit $WAN setup keep-state
$IPF 140 allow udp from me to any 53 out xmit $WAN keep-state

#Allow wireguard network
$IPF 187 allow ip from any to $wgNetwork

# NAT Outbound traffic allow - allows all traffic. To restrict, comment 200+210 and authorize specific outbound ports
$IPF 200 skipto 10000 tcp from $wgNetwork to any out xmit $WAN
$IPF 210 skipto 10000 udp from $wgNetwork to any out xmit $WAN
$IPF 215 allow tcp from me to any out via $WAN setup keep-state

# Incoming Rules - SSH (22)
$IPF 500 allow tcp from any to me 22 in recv $WAN setup keep-state

#Incoming Rules - WireGuard
$IPF 1010 allow udp from any to me $wg_port in recv $WAN keep-state

# deny everything else, and log it
$IPF 5000 deny log all from any to any

# NAT rule for outgoing packets.
$IPF 10000 nat 1 ip from any to any out via $WAN
$IPF 10010 allow ip from any to any
```

3. Start the firewall for the first time. **Warning** this will bounce you from your current SSH session. Suggest doing so from the console `/etc/rc.d/ipfw start`. 


### 4. Configure wireguard client and test 
Quick test on Android:
1. Install [wireguard](https://play.google.com/store/apps/details?id=com.wireguard.android&hl=en_US&gl=US) via Google Play store
2. Install qrencode to get terminal printed qrcodes, `pkg install libqrencode`
4. Print our a QR code in your terminal: `qrencode -t ansiutf8 < /usr/local/etc/wireguard/client1.conf`
5. Open the wireguard app, click the + in the lower right, select "scan from QR code", scan the QR code and connect to the VPN
6. Troubleshoot as necessary
7. Reboot the server to ensure all settings remain in tact upon server restart...troubleshoot what goes bump.

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
