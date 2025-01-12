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
- **📥 ICMP between Sonos devices, router interfaces & controller**: Facilitates various timing & background communications
- **🛡️ Various TCP & UDP firewall & proxy rules to**:  
  - Securely direct multicast traffic   
  - Forward Sonos unicast traffic between VLANs 
---

## 🛠️ **Reference OpenWRT System**  
These [example config files](https://github.com/itiligent/Sonos-OpenWRT-VLANs/tree/main/example-config-files) form the basis of the below reference system.

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

```bash
opkg update
opkg install igmpproxy
```
**Warning with muticast proxy:** 

Sonos device discovery requires SSDP relay across router interfaces, **_however IGMPproxy disables SSDP relay by default because it shares the same multicast address (239.255.255.250) with the broader uPnP suite._** As such, be advised that without properly restricting multicast, there is potential for all other uPnP traffic to punch holes everywhere (especially if uPnP can reach the WAN interface!). This guide shows how to remove the default IGMPproxy restriction and **_safely allow SSDP relay_** between just the LAN, GUEST & IOT VLANs.

It is strongly recommended to include all firewall rules below as a baseline minimum. You should also **remove/disable any Universal Plug & Play packages present**. If legacy uPnP is a requirement, you may need to further restrict uPnP very carefully as required. [See OpenWRT's uPnP warning here.](https://openwrt.org/docs/guide-user/firewall/upnp/start)

---

### **Step 2: Enable IGMP Snooping & STP**  
To enable IGMP snooping & Spanning Tree Protocol (STP), add the following lines to the LAN, Guest & IOT bridge devices found in `/etc/config/network`:  
```plaintext
option igmp_snooping '1'
option stp '1'
```

Example bridge device configuration. **Do this for each LAN, Guest & IOT device**.  
```plaintext
config device
	option name 'br-lan'
	option type 'bridge'
	list ports 'eth0'
	option igmp_snooping '1'
	option stp '1'
```

---

### **Step 3: Configure Firewall Defaults**  
To secure all traffic (and to avoid spamming multicast everywhere), all _**input**_ traffic to the router & _**forwarding**_ between zones must first be denied. To implement this change without breaking things, the below configuration achieves this whilst still allowing LUCI access, SSH, DNS, DHCP & ICMP to the router. This step forms the secure foundation to build further explicit firewall rules around.  

Edit `/etc/config/firewall` to restrict traffic input to the router as follows:  

Set firewall default behaviour: 
```plaintext
config defaults
	option input 'REJECT'  # Prevents implied inputs to the router 
	option output 'ACCEPT' # Allow outgoing traffic from any zone
	option forward 'REJECT' # Reject forces explicit firewall rules to be used for forwarding
	option synflood_protect '1'
```

Next configure the specific zone rules for each VLAN:  
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

Now check for correct zone forwarding:  
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

Lastly, add the explicit rules for LUCI access, SSH, DNS, DHCP & ICMP
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
---

### **Step 4: Add Sonos-Specific Firewall Rules**  
Add the following to `/etc/config/firewall`:  

Because all router input, forwarding and multicast to/from/through the WAN interface is to be blocked, simpler catch-all rules to allow the entire multicast (224.0.0.0/4) block internally can be used. This way, all device discovery that relies on mDNS, SSDP or multicast in any form can be easily supported with just the incremental addition of unicast firewall rules relevant to each device. (For non-Sonos devces, research the traffic requirements from trusted -> IOT, and in reverse from IOT -> trusted, then update the firewall as needed.)


```plaintext
config rule
	option name 'Deny-ICMP-from-WAN'  #  Place at top of firewall rules
	option family 'ipv4'
	option src 'wan'
	option proto 'icmp'
	option target 'DROP'
	list icmp_type 'echo-request'

config rule
	option name 'Deny-Multicast-from-WAN'  #  Deny multicast to the router itself.  Place at top of firewall rules
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
Edit `/etc/config/igmpproxy` to configure the upstream & downstream networks:  
_Note: `list altnet` should be used to restrict igmpproxy from relaying outside of the desired networks. (Don't use 0.0.0.0/0 !!)_ 
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
	list altnet 192.168.3.0/24 # a# Adjust to your IoT network address 
```

---

### **Step 6: Patch the IGMPproxy Launch Script**

For security, the default igmpproxy launch script blocks UDP uPnP traffic to 239.255.255.250. However, Sonos SSDP device discovery also uses this same address.  _**Warning: Removing this default restriction poses a significant security risk if uPnP package(s) are present or firewall rules any less restrictive than those provided are not already in place.**_

To lift this restriction, replace the `/etc/init.d/igmpproxy` script with the patched version linked below:

👉 [Patched IGMPproxy launch script](https://raw.githubusercontent.com/itiligent/Sonos-OpenWRT-VLANs/refs/heads/main/example-config-files/etc/init.d/igmpproxy)

_In future OpenWRT versions it may be possible to toggle this uPnP script restriction on/off in a config file, as is suggested [in this recent pull request](https://github.com/openwrt/packages/pull/25156)_

---

### **Step 7: Configure Avahi for mDNS & Apple Airplay discovery**
Edit `/etc/avahi/avahi-daemon.conf` as follows:  
_Note: The `allow-interfaces` directive must be used to limit which interfaces will be allowed to proxy mDNS multicast._

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

### **Step 8: [Optional] Samba music library share** 
For optional SMB music library file sharing _**from the OpenWRT router itself**_, Samba & WSDD2 should be installed.

See [this Youtube tutorial](https://www.youtube.com/watch?v=asN9aZ6Fg00) for sharing a usb drvie with OpenWRT. 

For those using a virtual instance of OpenwWRT on x86, you can optionally create a separate ext4 formatted vdisk and mount it via fstab. Build and install a new firmware image with the new fstab included to ensure music storage will be persistent across upgrades or even SquashFS firmware resets.    
```
opkg install luci-app-samba4 samba4-server wsdd2
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



## 🛠️ For hybrid, legacy S1 & S2 or desktop app users:
**S1 & S2** 
- Should work fine, but not tested. Older S1 & S2 Sonos applications utilise ICMP, broadcast on UDP 6969 and SSDP on UDP 5353 and the above firewall rules support this. 

**Desktop App:** 
- This app is slowly being deprecated by Sonos, and relies on udp broadcast/255.255.255.255 for discovery therefore the above solution will not work. Some form of broadcast relay is needed, perhaps installing socat will work. (See below, not tested). 
```
config socat 'sonos_bcast_forward'
    option enable '1'
    option SocatOptions '-d -d udp4-recvfrom:1900,broadcast,fork udp4-sendto:192.168.3.255:1900'
    option user 'nobody'
```

## Tracking of Changes
The Sonos ecosystem is evolving and may change agains in future. The above config was last tested on 12th Jan 2025 with the latest available Sonos S80 application & firmware 100% working with all features. (Sonos controller IOS/Android application, Airplay, device discovery & new device setup working perfectly from either LAN or Guest VLANs.) 


Please submit an issue or a suggestion if you find there's anything missed or not working.





