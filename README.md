# 🎵 **How to configure Sonos with VLANs & OpenWRT (2025)**

### Configuring OpenWRT VLANs for the new Sonos architecture (introduced late 2024)  

As of late 2024, all existing documentation on configuring VLANs for Sonos devices is now outdated due to many new changes to the Sonos network architecture.  

Sonos has shifted from a **LAN-first** model to a **cloud-first architecture**, which introduces updated networking requirements. This shift impacts how VLANs must be configured to support:  
- Device discovery and setup
- Media streaming
- Local library sharing
- Compatibility with the Sonos controller application and **AirPlay**

---

## 📋 **Key Requirements**  

A fully functional Sonos system working across VLANs with OpenWRT requires the following components:  

- **🔄 An mDNS System**: Avahi is installed by default in OpenWRT.
- **📡 IGMP Snooping**: To efficiently manage multicast traffic.  
- **🌐 IGMP Multicast Proxying**: To forward multicast traffic between VLANs.  
- **🛡️ Various Firewall Rules to**:  
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
   - Semi-trusted network for guests also using a Sonos application.  
   - Also considered an **"Upstream"** network for IGMP proxying.  
3. **IOT VLAN**:  
   - The untrusted VLAN for IOT devices.
   - Considered the **"Downstream"** network for IGMP proxying.

### **Assumptions**  
- VLANS for LAN, Guest & IOT have been created on a **bridge device**.
- LAN, Guest & IOT network interfaces have been configured with the their appropriate VLAN & static IP.  
- WAN access is available to all VLANs.
- Sonos devices are using **static IP addresses** (static dhcp reservations are recommended).
---

## 🚀 **Step-by-Step Configuration**  

### **Step 1: Install Required Packages** 
The `igmpproxy` package must be installed for proxying of IGMP

```bash
opkg update
opkg install igmpproxy
```

---

### **Step 2: Enable IGMP Snooping & STP**  
To enable IGMP snooping & Spanning Tree Protocol (STP), add the following lines to the  AN, Guest and IOT bridge devices found in `/etc/config/network`:  
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
To secure all traffic (and avoid spamming multicast everywhere), all 'input' traffic to the router & forwarding between zones must be implicitly denied. From this secure base all traffic can then be explicitly allowed as needed. To implement this change without breaking things, the below configuration achieves this whilst still allowing LUCI access, SSH, DNS, DHCP & ICMP to the router. This step forms a secure foundation to build additional rules around.  

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
Because all router interfaces & forwarding will be locked down by default, it is possible to keep firewall rules fairly simple as follows... (note the catch all rules for multicast).
Add the following to `/etc/config/firewall`:  
```plaintext
config rule
	option name 'Deny-ICMP-from-WAN'  #  Place at top of firewall rules
	option family 'ipv4'
	option src 'wan'
	option proto 'icmp'
	option target 'DROP'
	list icmp_type 'echo-request'

config rule
	option name 'Deny-Multicast-from-WAN'  #  Place at top of firewall rules
	option family 'ipv4'
	list proto 'udp'
	option src 'wan'
	option target 'DROP'
	list dest_ip '224.0.0.0/4'

config rule
	option name 'Deny-Multicast-WAN-to-Internal'  #  Place at top of firewall rules
	option family 'ipv4'
	option src 'wan'
	option dest '*'
	option proto 'udp'
	option target 'DROP'
	list dest_ip '224.0.0.0/4'

config rule
	option name 'Deny-Multicast-Internal-to-WAN'  #  Place at top of firewall rules
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
	list dest_ip 'your.sonos.ip.range/29'

config rule
	option name 'Allow-Sonos-from-Guest'
	option family 'ipv4'
	option src 'guest'
	option dest 'iot'
	option target 'ACCEPT'
	list proto 'all'
	list dest_ip 'your.sonos.ip.range/29'

config rule
	option name 'Allow-Sonos-TCP-to-LAN'
	option family 'ipv4'
	option src 'iot'
	option dest 'lan'
	option target 'ACCEPT'
	list proto 'tcp'
	option src_port '445 3445 1400 1433 3400 3401 3500 4070 4444'
	list src_ip '192.168.3.16/29'

config rule
	option name 'Allow-Sonos-TCP-to-GUEST'
	option family 'ipv4'
	option src 'iot'
	option dest 'guest'
	option target 'ACCEPT'
	list proto 'tcp'
	option src_port '445 3445 1400 1433 3400 3401 3500 4070 4444'
	list src_ip '192.168.3.16/29'

config rule
	option name 'Allow-Sonos-UDP-to-LAN'
	option family 'ipv4'
	list proto 'udp'
	option src 'iot'
	option dest 'lan'
	option target 'ACCEPT'
	option dest_port '319 320 1900 1901 2869 5353 6969 10280-10284 30000-65535'
	list src_ip '192.168.3.16/29'

config rule
	option name 'Allow-Sonos-UDP-to-GUEST'
	option family 'ipv4'
	list proto 'udp'
	option src 'iot'
	option dest 'lan'
	option target 'ACCEPT'
	option dest_port '319 320 1900 1901 2869 5353 6969 10280-10284 30000-65535'
	list src_ip '192.168.3.16/29'
```

---

### **Step 5: Configure IGMPproxy**  
Edit `/etc/config/igmpproxy` to configure the upstream & downstream networks:  
_Note: `list altnet` should be configured to secure igmpproxy from relaying outside of the desired networks._ 
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

The default igmpproxy launch script breaks Sonos by blocking SSDP/mDNS multicast from spamming to all interfaces (for security). To solve this, replace the entire `/etc/init.d/igmpproxy` script's contents with this patched version:  
👉 [Patched IGMPproxy Launch Script](https://raw.githubusercontent.com/itiligent/Sonos-OpenWRT-VLANs/refs/heads/main/example-config-files/etc/init.d/igmpproxy)

This is an issue that may be improved in future OpenWRT versions. See [here](https://github.com/openwrt/packages/pull/25156) for more background: 

---

### **Step 7: Configure Avahi for mDNS** (& Airplay)
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
For optional music library file sharing _**from the OpenWRT router**_, **Samba** & **WSDD2** should be installed.

See [this Youtube tutorial](https://www.youtube.com/watch?v=asN9aZ6Fg00) for sharing a usb drvie with OpenWRT  
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
Then add a guest (password-free) music file share at the bottom of `/etc/samba/smb/conf/template`.

```
[Music]
        path = /your/mount/path
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

📄 
This config was tested on 3rd Jan 2025 with the latest available Sonos application & firmware. The Sonos application, airplay, device discovery & even new device setup works fine from either LAN or Guest VLAN.
Please submit an issue or a suggestion if you find there's anything missed or no longer working as Sonos continues evolve their products.

### For hybrid or legacy Sonos S1 & S2 users:
For device discovery, S1 & S2 require SSDP (uses same UDP 5353 same as mDNS, but quite different) & brodcast UDP on 6969.
A potential (not fully tested) hybrid solution lies with the addition of this python script: 
https://github.com/alsmith/multicast-relay 

Run the script from rc.local with these switches
```
-homebrewNetifaces --ifNameStructLen=32 --noMDNS --interfaces br-lan br-iot br-guest
``` 
All above firewall settings should work ok with this script.

