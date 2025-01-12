# 🎵 **How to configure Sonos with VLANs & OpenWRT (2025)**

### Configuring OpenWRT VLANs for the new Sonos architecture (introduced late 2024)  

As of late 2024, all existing documentation on configuring VLANs for Sonos devices is now outdated due to many new changes to the Sonos network architecture.  

Sonos has shifted from a **LAN-first** to a **cloud-first** architecture, which introduces updated networking requirements. This shift impacts how VLANs must be configured to support:  
- Device discovery & setup
- Media streaming
- Local library sharing
- Compatibility with the Sonos controller application & AirPlay

---

## 📋 **Key Requirements**  

A fully functional Sonos system working across VLANs with OpenWRT requires the following components:  

- **🔄 An mDNS System**: Avahi is typically installed by default in OpenWRT.
- **📡 IGMP Snooping**: To efficiently manage multicast traffic.  
- **🌐 IGMP Multicast Proxying**: To forward multicast traffic between VLANs.
- **📥 ICMP between Sonos devices, router & controller**: Facilitates various timing & background communications
- **🛡️ Various TCP & UDP firewall & proxy rules to**:  
  - _**Securely**_ direct multicast traffic   
  - Forward Sonos specific unicast traffic between VLANs 
---

## 🛠️ **Reference OpenWRT System**  
These [example config files](https://github.com/itiligent/Sonos-OpenWRT-VLANs/tree/main/example-config-files) form the reference system in this guide.

### **Example VLANs**  
1. **LAN VLAN**:  
   - The trusted network where the Sonos controller application will operate from.
   - This VLAN will be considered the **"Upstream"** network for IGMP proxying.  
2. **Guest VLAN** (optional):  
   - Trusted network for guests also using a Sonos application.
   - Also considered an **"Upstream"** network for IGMP proxying.  
3. **IOT VLAN**:  
   - The untrusted VLAN for IOT devices.
   - Considered the **"Downstream"** network for IGMP proxying.

### **Assumptions**  
- VLANS for LAN, Guest & IOT have been pre-created on a **DSA bridge device**. (OpenWRT 21.02 and above)
- LAN, Guest & IOT network interfaces have been configured with the their appropriate VLAN & static IP.  
- WAN access is available to all VLANs.
- Sonos devices are using **static IP addresses** (static dhcp reservations are recommended).
---

## 🚀 **Step-by-Step Configuration**  

### **Step 1: Install IGMPproxy** 
The `igmpproxy` package must be installed for proxying of multicast traffic between VLANs

For OpenwWRT 23.05.5 or lower:
```bash
opkg update
opkg install igmpproxy socat
```

For OpenWRT 24.10 and above:

```bash
apk update
apk add igmpproxy socat
```

**Warning with muticast proxy:** 

Sonos device discovery requires SSDP relay across router interfaces, **_however IGMPproxy (wisely) blocks the SSDP multicast address 239.255.255.250 by default because this address is also used by the broader uPnP suite._**  There are many security risks with unrestricted uPnP, especially if allowed to reach the WAN interface. This guide shows how to safely remove IGMPproxy's default multicast address restrictions whilst also **_limiting SSDP relay to just the LAN, GUEST & IOT VLANs_**.

It is strongly recommended to include all firewall rules provided in this guide as a baseline minimum. You should also **remove or disable any Universal Plug & Play packages present**. If legacy uPnP is a requirement, you may need to further restrict uPnP very carefully as required. [See OpenWRT's uPnP warning here.](https://openwrt.org/docs/guide-user/firewall/upnp/start)

---

### **Step 2: Enable IGMP Snooping & STP**  
To enable IGMP snooping & Spanning Tree Protocol (STP), add the following lines to the LAN, Guest & IOT bridge devices found in `/etc/config/network`:  
```plaintext
option igmp_snooping '1'
option stp '1'
```

Example bridge device configuration. **Do this for the LAN, Guest & IOT device**.  
```plaintext
config device
	option name 'br-lan'
	option type 'bridge'
	list ports 'eth0'
	option igmp_snooping '1' # add this
	option stp '1' # add this
```

---

### **Step 3: Deny All To & Through The Router Interface**  

_**Note: Don't commit changes or restart OpenWRT until all below step 3 changes are added or you will cut yourself off!!**_

To correctly restrict insecure uPnP multicast, all _**input**_ traffic to the router & _**forwarding**_ traffic between zones must first be denied by default. This change will break several built-in _**implied**_ firewall defaults, so to implement this change without breaking things, you must _**explicitly**_ re-enable LUCI http, SSH, DNS, DHCP & ICMP to the router. This important step supports the firewall best practice of _**implicitly deny all by default**_, and _**explicitly permit only where needed**_.  

Edit `/etc/config/firewall` to restrict traffic as follows:



Set the firewall default behavior to implicitly deny all to & through the router interface: 
```plaintext
config defaults
	option input 'REJECT'  # Prevents implied inputs to the router 
	option output 'ACCEPT' # Allow outgoing traffic from any zone
	option forward 'REJECT' # Reject forces explicit firewall rules to be used for forwarding
	option synflood_protect '1'
```

Next, configure each zone to implicitly deny all to & through the router interface:  
```plaintext
config zone
	option name 'lan'
	option family 'ipv4'
	option input 'REJECT'
	option output 'ACCEPT'
	option forward 'REJECT'
	list network 'lan'

config zone
	option name 'guest'
	option family 'ipv4'
	option input 'REJECT'
	option output 'ACCEPT'
	option forward 'REJECT'
	list network 'guest'

config zone
	option name 'iot'
	option family 'ipv4'
	option input 'REJECT'
	option output 'ACCEPT'
	option forward 'REJECT'
	list network 'iot'
```

Check for correct zone forwarding:  
```plaintext
config forwarding
	option src 'lan'
	option dest 'wan'

config forwarding
	option src 'guest'
	option dest 'wan'

config forwarding
	option src 'iot'
	option dest 'wan'
```

Now add _**explicit allow**_ rules for Luci, SSH, DNS, DHCP & ICMP to the router
```plaintext
config rule
	option name 'Allow-DNS-LAN'
	option family 'ipv4'
	list proto 'udp'
	option src 'lan'
	option dest_port '53'
	option target 'ACCEPT'

config rule
	option name 'Allow-DNS-GUEST'
	option family 'ipv4'
	list proto 'udp'
	option src 'guest'
	option dest_port '53'
	option target 'ACCEPT'

config rule
	option name 'Allow-DNS-IOT'
	option family 'ipv4'
	list proto 'udp'
	option src 'iot'
	option dest_port '53'
	option target 'ACCEPT'

config rule
	option name 'Allow-DHCP-WAN'
	option family 'ipv4'
	option src 'wan'
	option proto 'udp'
	option dest_port '68'
	option target 'ACCEPT'

config rule
	option name 'Allow-DHCP-LAN'
	option family 'ipv4'
	list proto 'udp'
	option src 'lan'
	option src_port '68'
	option dest_port '67'
	option target 'ACCEPT'

config rule
	option name 'Allow-DHCP-GUEST'
	option family 'ipv4'
	list proto 'udp'
	option src 'guest'
	option src_port '68'
	option dest_port '67'
	option target 'ACCEPT'

config rule
	option name 'Allow-DHCP-IOT'
	option family 'ipv4'
	list proto 'udp'
	option src 'iot'
	option src_port '68'
	option dest_port '67'
	option target 'ACCEPT'

config rule
	option name 'Allow-Luci-SSH-LAN'
	option family 'ipv4'
	list proto 'tcp'
	option src 'lan'
	option dest_port '22 80'
	option target 'ACCEPT'
config rule
	option name 'Allow-ICMP-Router'
	option family 'ipv4'
	list proto 'icmp'
	option src '*'
	option target 'ACCEPT'

config rule
	option name 'Allow-ICMP-Everywhere'
	option family 'ipv4'
	option target 'ACCEPT'
	list proto 'icmp'
	option src '*'
	option dest '*'
```

Only reboot OpenWRT after all above changes are saved.

---

### **Step 4: Explicitly Allow All Multicast (Internally) + Sonos Unicast**  
Add the following to `/etc/config/firewall`:  



With the addition of the below multicast restrictions from the internet to the router's WAN interface (restrictions to & from the WAN zone and also from internal zones), simpler catch-all rules to allow the entire RFC1112 multicast block (224.0.0.0/4) can be applied internally. 

This RFC1112 block strategy minimises complexity while enabling cross-VLAN multicast for any other devices that may also rely on protocols like mDNS, SSDP, or multicast for discovery. 

To allow new device types, simply add specific unicast firewall rules incrementally. Start by researching the traffic requirements for communication from the trusted VLAN to IOT, and vice versa (TCP dump and Wireshark are very useful for this). Then update the firewall rules accordingly.

```plaintext
config rule
	option name 'Deny-ICMP-from-WAN'  #  Place at top of firewall rules
	option family 'ipv4'
	option src 'wan'
	option proto 'icmp'
	option target 'DROP'
	list icmp_type 'echo-request'

config rule
	option name 'Deny-Multicast-from-WAN'  #  Deny multicast to the router itself from WAN.  Place at top of firewall rules
	option family 'ipv4'
	list proto 'udp'
	option src 'wan'
	option target 'DROP'
	list dest_ip '224.0.0.0/4'

config rule
	option name 'Deny-Multicast-WAN-to-Internal' # Deny multicast through the router. Place at top of firewall rules
	option family 'ipv4'
	option src 'wan'
	option dest '*'
	option proto 'udp'
	option target 'DROP'
	list dest_ip '224.0.0.0/4'

config rule
	option name 'Deny-Multicast-All-Zones-to-WAN' # Deny multicast to the WAN. #  Place at top of firewall rules
	option family 'ipv4'
	list proto 'udp'
	option target 'DROP'
	option src '*'
	option dest 'wan'
	list dest_ip '224.0.0.0/4'

config rule
	option name 'Allow-Multicast-LAN'
	option family 'ipv4'
	option target 'ACCEPT'
	option src 'lan'
	list proto 'udp'
	list dest_ip '224.0.0.0/4'

config rule
	option name 'Allow-Multicast-IOT'
	option family 'ipv4'
	option target 'ACCEPT'
	option src 'iot'
	list proto 'udp'
	list dest_ip '224.0.0.0/4'

config rule
	option name 'Allow-Multicast-GUEST'
	option family 'ipv4'
	option target 'ACCEPT'
	option src 'guest'
	list proto 'udp'
	list dest_ip '224.0.0.0/4'

config rule
	option name 'Allow-Sonos-from-LAN'
	option family 'ipv4'
	option src 'lan'
	option dest 'iot'
	option target 'ACCEPT'
	list proto 'all'
	list dest_ip 'sonos.static.ip.range/29'

config rule
	option name 'Allow-Sonos-from-Guest'
	option family 'ipv4'
	option src 'guest'
	option dest 'iot'
	option target 'ACCEPT'
	list proto 'all'
	list dest_ip 'sonos.static.ip.range/29'

config rule
	option name 'Allow-Sonos-TCP-to-LAN'
	option family 'ipv4'
	option src 'iot'
	option dest 'lan'
	option target 'ACCEPT'
	list proto 'tcp'
	option src_port '445 3445 1400 1433 3400 3401 3500 4070 4444'
	list src_ip 'sonos.static.ip.range/29'

config rule
	option name 'Allow-Sonos-TCP-to-GUEST'
	option family 'ipv4'
	option src 'iot'
	option dest 'guest'
	option target 'ACCEPT'
	list proto 'tcp'
	option src_port '445 3445 1400 1433 3400 3401 3500 4070 4444'
	list src_ip 'sonos.static.ip.range/29'

config rule
	option name 'Allow-Sonos-UDP-to-LAN'
	option family 'ipv4'
	list proto 'udp'
	option src 'iot'
	option dest 'lan'
	option target 'ACCEPT'
	option dest_port '319 320 1900 1901 2869 5353 6969 10280-10284 30000-65535'
	list src_ip 'sonos.static.ip.range/29'

config rule
	option name 'Allow-Sonos-UDP-to-GUEST'
	option family 'ipv4'
	list proto 'udp'
	option src 'iot'
	option dest 'lan'
	option target 'ACCEPT'
	option dest_port '319 320 1900 1901 2869 5353 6969 10280-10284 30000-65535'
	list src_ip 'sonos.static.ip.range/29'
```

---

### **Step 5: Configure IGMPproxy**  
Edit `/etc/config/igmpproxy` to configure the **upstream & downstream networks** that will be allowed to proxy multicast:  
_Note: `list altnet` should be used to restrict igmpproxy to just the desired internal networks. Don't use 0.0.0.0/0!_ 
```plaintext
config igmpproxy
	option quickleave 1

config phyint
	option network lan
	option zone lan
	option direction upstream
	list altnet 192.168.1.0/24 # Adjust to your LAN network address 

config phyint
	option network guest
	option zone guest
	option direction upstream
	list altnet 192.168.2.0/24 # Adjust to your Guest network address  
	
config phyint
	option network iot
	option zone iot
	option direction downstream
	list altnet 192.168.3.0/24 # a# Adjust to your IOT network address 
```

---

### **Step 6: Update The IGMPproxy Launch Script**

_**Warning: Step 5 above must be completed first.**_

For security, OpenWRT's default `/etc/init.d/igmpproxy` launch script creates hidden firewall rules that block UDP uPnP multicast traffic on 239.255.255.250, however Sonos multicast across VLANs requires this restriction to be removed. To lift this restriction, replace the `/etc/init.d/igmpproxy` script with the patched version linked below:

👉 [Patched IGMPproxy launch script](https://raw.githubusercontent.com/itiligent/Sonos-OpenWRT-VLANs/refs/heads/main/example-config-files/etc/init.d/igmpproxy)

_In future OpenWRT versions it may be possible to toggle this uPnP script restriction on/off in a config file, as is suggested [in this recent pull request](https://github.com/openwrt/packages/pull/25156)_

---

### **Step 7: Configure Avahi For mDNS & Apple Airplay Device Discovery**
Edit `/etc/avahi/avahi-daemon.conf` as follows:  
_Note: The `allow-interfaces` directive must be used to restrict mDNS access to just the required internal networks._

```ini
[server]
use-ipv4=yes
use-ipv6=yes # Or no as required
check-response-ttl=no
use-iff-running=no
allow-interfaces=br-lan.90,br-lan.100,br-lan.110  # Adapt to your specific VLAN interfaces here

[publish]
publish-addresses=yes
publish-hinfo=yes
publish-workstation=no
publish-domain=yes

[reflector]
enable-reflector=yes
reflect-ipv=no

[rlimits]
rlimit-core=0
rlimit-data=4194304
rlimit-fsize=0
rlimit-nofile=30
rlimit-stack=4194304
rlimit-nproc=3
```
---

### **Step 8: [Optional] Samba Music Library Share** 
Because a router is typically always on, music library file sharing _**from the OpenWRT router itself**_ offeres a very simple & low-power approach to keeping your music collection always online. To achieve this, additionally install the Samba & WSDD2 packages and see [this Youtube tutorial](https://www.youtube.com/watch?v=asN9aZ6Fg00) for sharing a usb drive with Samba & OpenWRT. 

```
opkg update
opkg install luci-app-samba4 samba4-server wsdd2

or

apk update
apk add luci-app-samba4 samba4-server wsdd2
```
Edit the [Global] section of `/etc/samba/smb/conf/template` as follows:
```plaintext
disable netbios = yes
min protocol = SMB2
smb ports = 445
mdns name = mdns
```
Now add a guest (password-free) music file share at the bottom of `/etc/samba/smb/conf/template`.

```
[Music]
        path = /mnt/disk/path
        create mask = 0666
        directory mask = 0777
        read only = yes
        guest ok = yes
        vfs objects = io_uring
        hosts allow = 192.168.1.0/24, 192.168.3.0/24  # Adjust to your LAN & IOT ip network addresses
        hosts deny = 0.0.0.0/0 # deny everything else
```

Lastly, add the following firewall rules: 
```
config rule
	option name 'Allow-Router-SMB-LAN'
	option family 'ipv4'
	option dest_port '445 5355 3702'
	option target 'ACCEPT'
	option src 'lan'

config rule
	option name 'Allow-Router-SMB-IOT'
	option family 'ipv4'
	option dest_port '445 5355 3702'
	option target 'ACCEPT'
	option src 'iot'
```

If using a virtual instance of OpenwWRT on x86, a very robust approach for adding a persistent file storage to OpenWRT is to create a separate EXT4 formatted vdisk and auto mount it via `/etc/fstab`. For auto mount & vdisk file share persistence across firmware resets or firmware upgrades, create a new firmware image with the updated `/etc/fstab` file baked in as a customised default via this useful script: https://github.com/itiligent/Easy-OpenWRT-Builder.     


---


**Sonos Desktop Application:** 
- Apple OS:	The above guide should work fine thanks to Bonjour/Avahi being used for discovery, which is allowed by the above (not tested).
- Windows:  	Because the Windows version rather crudely relies on UDP 1900 broadcasts to 255.255.255.255 for Sonos discovery, therefore these broadcasts must relayed by the following steps

Adjust and copy the following to `/etc/config/socat` 

```
config socat 'sonos_bcast_forward'
    option enable '1'
    option SocatOptions '-d -d udp4-recvfrom:1900,broadcast,fork udp4-sendto:192.168.3.255:1900' # Adjust ip address to the broadcast address of the IOT VLAN
    option user 'nobody'
```

Restart Socat with `/etc/init.d/socat restart` or from Luci "Startup" page.

## 🛠️ Legacy Sonos S1 & S2 App Users:
- The above guide should work fine with either Android or Apple IOS because S1 & S2 both utilise ICMP, UDP 6969 and UDP 5353 which are all allowed by the above firewall rules (not tested).  


## Future Sonos Changes?

The above config was tested on both Android and Apple devices on 12th Jan 2025 with the current latest available Sonos S80 application & firmware. All functions at that time were 100% working (Sonos S80 app, Airplay, new device discovery, new device setup, volume control, speaker grouping and Samba music library access all whilst connected to either the LAN VLAN or Guest VLAN). 

Please submit an issue if you notice something is not working, or share any helpful suggestions.  


