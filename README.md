# 🎵 **How to configure Sonos with VLANs & OpenWRT (2025)**

### Configuring OpenWRT VLANs for the new Sonos architecture

As of late 2024, much online documentation on configuring VLANs for Sonos is somewhat outdated due to several recent changes to the Sonos network architecture.  

Also, Sonos has shifted from a **LAN-first** to a **CLOUD-first** architecture, which introduces some new networking requirements and security concerns. This change of product focus impacts how VLANs should be configured to support:  

- Device discovery & setup
- All Sonos controller application types & versions
- Music streaming
- Local music library sharing
- Apple AirPlay connectivity

---

## 📋 **Key Requirements**  

A fully functional Sonos system operating over VLANs with OpenWRT requires the following networking components:  

- 🔄 **An mDNS System**: Avahi is typically installed by default in OpenWRT
- 📡 **IGMP Snooping**: To efficiently manage multicast traffic
- 🌐 **IGMP Multicast Proxying**: To forward multicast traffic between VLANs
- 📥 **ICMP between Sonos devices, router & controller**: Facilitates various timing & background communications
- 📥 **A broadcast forwarding mechanism**: To support the legacy Sonos desktop application
- 🛡️ **Various TCP & UDP firewall & proxy rules to**:  
  - _**Securely**_ direct Sonos multicast traffic   
  - Forward Sonos-specific unicast traffic between VLANs 

---

### 🛠️ **Reference OpenWRT System**  
These [example config files](https://github.com/itiligent/Sonos-OpenWRT-VLANs/tree/main/example-config-files) form the example reference system described in this guide.

### **Example VLANs**  
1. **LAN VLAN**:  
   - This is the trusted network where the Sonos controller application will operate from
      - This VLAN is the **"Upstream"** network for IGMP proxying
2. **Guest VLAN** (optional):  
   - Trusted network for guests also using a Sonos application
      - This VLAN is also an **"Upstream"** network for IGMP proxying
3. **IOT VLAN**:  
   - The untrusted VLAN for IOT devices
      - This VLAN is the **"Downstream"** network for IGMP proxying

---

### **Assumptions**  
- OpenWRT 21.02 or above (the newer DSA architecture)
- VLANS for LAN, Guest & IOT have been pre-created similar to the supplied example comfig files.
    - LAN, Guest & IOT network interfaces have all been configured with the their appropriate VLAN assigment & static IP.  
- WAN access is available to all VLANs.
- Sonos devices are using **static IP addresses** (static dhcp reservations are recommended).
---

### **Muticast Proxy Security Warning** 

**Unrestricted multicast of uPnP traffic poses significant security risks, especially if allowed to reach the WAN interface**.

Be advised that Sonos device discovery and various other Sonos communications requires a range of multicast to be relayed across router interfaces. There are security considerations with this due to Sonos multicast traffic sharing the same address block used by the broader uPnP suite (239.255.255.250). As such, extra steps must be taken to carefully limit multicast. Wherever possible, legacy Universal Plug & Play OpenWRT packages should be removed or disabled. 

This guide explains how to limit all Sonos multicast traffic to just the LAN, GUEST, and IOT VLANs interfaces only. As such it is strongly recommended to apply all firewall rules in this guide as a minimum baseline. 

---

## 🚀 **Step-by-Step Configuration**  

### **Step 1: Install IGMPproxy & Socat** 
The `igmpproxy` & `socat` packages are needed to proxy specific traffic between VLANs

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

---

### **Step 2: Enable IGMP Snooping & STP**  
To enable IGMP snooping & Spanning Tree Protocol (STP), add the following lines to the LAN, Guest & IOT bridge devices in `/etc/config/network`:  
```plaintext
option igmp_snooping '1'
option stp '1'
```

Here is an example bridge device configuration. **Duplicate this step for the LAN, Guest & IOT devices** in `/etc/config/network`:  
```plaintext
config device
	option name 'br-lan'
	option type 'bridge'
	list ports 'eth0'
	option igmp_snooping '1' # add this line to existing bridge config
	option stp '1' # add this line to existing bridge config
```
---

### **Step 3: Deny All Traffic To And Through The Router Interface**  

**Warning: Don't commit any changes or restart OpenWRT until all step 3 changes are completed or you may cut yourself off!!**

To correctly restrict insecure uPnP multicast, all _**input traffic**_ to the router & all _**forwarding traffic**_ between zones must first be denied by default. This change overrides several hidden built-in default firewall rules, therefore these must be manually re-created to _**explicitly**_ enable LUCI http, SSH, DNS, DHCP & ICMP to the router. This very important step underpins best practice of _**implicitly deny everything by default**_, and _**explicitly permit only what traffic is needed**_.  

Edit `/etc/config/firewall` as follows:

Set the firewall default behavior to implicitly deny everything to and through the router interface: 
```plaintext
config defaults
	option input 'REJECT'  # Prevents implied inputs to the router 
	option output 'ACCEPT' # Allow outgoing traffic from any zone
	option forward 'REJECT' # Reject forces explicit firewall rules to be used for forwarding
	option synflood_protect '1'
```

Next, configure each firewall zone to implicitly deny everything to and through the router interface: 
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

Now block ICMP and multicast to and from WAN, and in the same step _**explicitly**_ re-enable LAN access to Luci, SSH, DNS, DHCP & ICMP to the router. Copy after the above forwarding section in the EXACT ORDER shown below:
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

### **Step 4: Explicitly Allow All Multicast Only between LAN, GUEST & IOT VLANs**  

Because step 3 added multicast restrictions to and from the WAN interface, simpler catch-all rules for the full multicast block (224.0.0.0/4) can now be used internally. 
This RFC1112 block strategy keeps firewall rules managable, and also supports cross-VLAN multicast for any other devices that use mDNS or similar for network discovery. 

Add the following to `/etc/config/firewall` directly below the rules from step 3:  

```plaintext
config rule
	option name 'Allow-Multicast-LAN'
	option family 'ipv4'
	option target 'ACCEPT'
	option src 'lan'
	list proto 'tcp'
	list proto 'udp'
	list dest_ip '224.0.0.0/4'

config rule
	option name 'Allow-Multicast-IOT'
	option family 'ipv4'
	option target 'ACCEPT'
	option src 'iot'
	list proto 'tcp'
	list proto 'udp'
	list dest_ip '224.0.0.0/4'

config rule
	option name 'Allow-Multicast-GUEST'
	option family 'ipv4'
	option target 'ACCEPT'
	option src 'guest'
	list proto 'tcp'
	list proto 'udp'
	list dest_ip '224.0.0.0/4'
```

---

### **Step 5: Explicitly Allow Sonos Unicast Traffic**  

Add the following to `/etc/config/firewall` directly below the rules from step 4. To keep firewall rules simple, it is optimal to assign all Sonos devices a consecutive static IP range that falls inside a single CIDR bit boundary. Adjust the `list_dest_ip` and `list_src_ip` lines as needed: 

```plaintext
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
	option name 'Allow-Sonos-TCP-to-LAN-Desktop-App'
	list proto 'tcp'
	option src 'iot'
	option dest 'lan'
	option dest_port '3400'
	option target 'ACCEPT'
	option family 'ipv4'
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
	option name 'Allow-Sonos-TCP-to-GUEST-Desktop-App'
	list proto 'tcp'
	option src 'iot'
	option dest 'guest'
	option dest_port '3400'
	option target 'ACCEPT'
	option family 'ipv4'
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
	option dest 'guest'
	option target 'ACCEPT'
	option dest_port '319 320 1900 1901 2869 5353 6969 10280-10284 30000-65535'
	list src_ip 'sonos.static.ip.range/29'
```

---

### **Step 6: Configure IGMPproxy**  
Edit `/etc/config/igmpproxy` to configure the **upstream & downstream networks** that will be allowed to proxy multicast traffic:  
_Note: `list altnet` must be used to restrict igmpproxy to just the desired internal networks. Don't use 0.0.0.0/0!_ 
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

### **Step 7 Update The IGMPproxy Launch Script**

For security, OpenWRT's default `/etc/init.d/igmpproxy` launch script creates a hidden firewall rule that blocks UDP uPnP multicast traffic on 239.255.255.250, however Sonos multicast needs this restriction removed. To lift this restriction, replace the `/etc/init.d/igmpproxy` script with the patched version linked below:

👉 [Patched IGMPproxy launch script](https://raw.githubusercontent.com/itiligent/Sonos-OpenWRT-VLANs/refs/heads/main/example-config-files/etc/init.d/igmpproxy)

---

### **Step 8: Configure Avahi For mDNS & Apple Airplay Device Discovery**
Edit `/etc/avahi/avahi-daemon.conf` as follows:  
_Note: The `allow-interfaces` directive must be used to restrict mDNS access to just the required internal networks._

```ini
[server]
use-ipv4=yes
use-ipv6=yes # Or no as required
check-response-ttl=no
use-iff-running=no
allow-interfaces=br-lan.100,br-lan.200,br-lan.300  # Adapt to your specific VLAN interfaces here

[publish]
publish-addresses=no
publish-hinfo=no
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

## Step 9: Older Sonos Desktop App & Legacy S1/S2 App Support
### Windows Desktop Application:
Older versions of the Windows Desktop controller application relied on UDP 1900 broadcasts to 255.255.255.255 in the controller's network for discovery. From a Desktop Application located in either GUEST or LAN VLAN, these broadcasts can be relayed through to the IOT VLAN broadcast address via Socat as follows. _(Befofre making these changes, test to see if your Desktop application works from LAN to IOT without Socat - later versions seem to work ok just with igmpproxy and rules above)_ :

Adjust and copy the following to `/etc/config/socat` 

```
config socat 'sonos_bcast_forward'
    option enable '1'
    option SocatOptions '-d -d udp4-recvfrom:1900,broadcast udp4-sendto:172.17.11.255:1900,broadcast'  # Adjust to the broadcast address of your IOT VLAN
    option user 'nobody'
```

Restart Socat with `/etc/init.d/socat restart` or from Luci "Startup" page.

### Apple OS Desktop Application 
Should work fine without Socat thanks to Bonjour/Avahi being used for discovery.
### Legacy Sonos S1 & S2 Apps: 
Either Android or Apple IOS should work fine with without Socat.  

---

### **Step 10: [Optional] Samba Music Library Share** 
Because a router is typically always on, music library file sharing _**from the OpenWRT router itself**_ offeres a low-power approach to hosting your music collection 24x7. To achieve this, install the Samba & WSDD2 packages and see [this Youtube tutorial](https://www.youtube.com/watch?v=asN9aZ6Fg00) for how to share a usb drive via Samba & OpenWRT. 

```
opkg update
opkg install luci-app-samba4 samba4-server wsdd2

or for OpenWRT 24.10.x and above

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

Lastly, add the following Samba firewall rules: 

Add the following to `/etc/config/firewall` below the rules from step 5 as shown: 

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

### **Step 11: [Optional] Additional Persistent Disk Storage**
If using a virtualised instance of OpenwWRT on x86, a robust approach for adding persistent music disk storage to OpenWRT is to create a separate EXT4 formatted vdisk that is auto mounted via `/etc/fstab`. For auto mount & vdisk storage persistence across firmware resets or even firmware upgrades, it is possible to create a custom firmware image with your modified `/etc/fstab` file baked in as the default. This approach ensures the extra EXT4 storage partition is not lost of destroyed upon firmware resets or upgrades. See here for further instructions on resizing and adding additional partitions to OpenWRT: [https://github.com/itiligent/Easy-OpenWRT-Builder](https://github.com/itiligent/Easy-OpenWRT-Builder?tab=readme-ov-file#-persistent-filesystem-expansion-without-resizing-partitions)    

---

## Future Sonos Changes?

The above config was tested on both Android and Apple devices on 8th March 2025 with the current latest available Sonos S80 application & firmware on OpenWRT 23.05.5. All functions at that time were 100% working (Sonos S80 app, Airplay, new device discovery, new device setup, volume control, speaker grouping and Samba music library access all whilst connected to either the LAN VLAN or Guest VLAN). 

Please submit an issue if you notice something is not working, or share any helpful suggestions.  


