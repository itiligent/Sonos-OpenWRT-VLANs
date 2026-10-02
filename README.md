## 🎵 **Sonos with VLANs and  (2027)**

### 📋 **Sonos communication across  VLANs requirements:** 

- 🔄 Avahi (mDNS)
- 🌐 SMCRoute (multicast router)
- 🛡️ A specific firewall, mDNS & multicast forwarding configuration to facilitate:
     - Sonos multicast discovery between VLANs
     - Bi-directional unicast traffic between Speakers & Controller App located in separate VLANs 

### 🛠️ Core Networking Approach

| Function | Address / Protocol / port | Multi-VLAN Approach |
|---|---|---|
| Sonos SSDP multicast | `239.255.255.250` / UDP | Forwarding via SMCRoute |
| mDNS / Bonjour / AirPlay  | `224.0.0.251:5353` | Forwarding via Avahi |
| Sonos controller app traffic | Various unicast | Normal firewall rules |
**Optional**
| Samba hosted music library share | TCP 445 | Normal firewall rules |

---


## 🛠️ Reference OpenWRT Setup 

[This set of config files](https://github.com/itiligent/Sonos-OpenWRT-VLANs/tree/main/example-config-files) provides an entire OpenWRT baseline configuration for multiple VLANs with Sonos. Adapt this to suit.

### **Example Networks**  
1. LAN (VLAN 100) 192.168.1.0/24:  
   - The main trusted network for users with a Sonos controller application       
2. Guest (VLAN 200) 192.168.2.0/24 (optional):  
   - Another trusted network for guests with a Sonos controller application
3. IOT (VLAN 300) 192.168.3.0/24:  
   - The untrusted VLAN for your Sonos speakers (and other IOT devices)

### 🔧 Assumptions
- You are using a recent OpenWRT build (v21.x & above)
- WAN access is available to all 3 example VLANs
- All Sonos speakers IP addressing is configured within a contiguous block of **static IP addresses**. (A block of static dhcp reservations is recommended - eg: `192.168.3.16/28`)
-  Root/SSH access is available to OpenWRT

---

## 🚀 **Step-by-Step OWRT Configuration**  

### **Step 1: Install Smcroute & Avahi** 
The below packages are needed:

```
apk update
apk add smcroute avahi-daemon nano

optional packages for music share:
apk add samba4-server luci-app-samba4 wsdd2
```

---

### **Step 2: Setup Firewall Rules**  

Adapt [this example baseline firewall configuration](example-config-files/etc/config/firewall) as needed.

The provided example firewall configuration:

- Sets internal `input` and `forward` policies to `REJECT` for an implicit deny / explicit allow model.
- Allows only required DHCP, DNS, SSH and LuCI traffic for minimum OWRT operation.
- Allows LAN and Guest Sonos controllers to access only a defined Sonos IoT IP range.
- Allows only required intra-VLAN Sonos bidirectional callback traffic.
- Allows mDNS for all VLANs (via Avahi).
- Allows SSDP & multicast forwarding between all VLANS (via SMCRoute).
- Allows SMB and WSD Discovery to the router's (optional) Samba music share.
---

### **Step 3: Configure SMCRoute**  
Edit [`/etc/smcroute.conf`](https://github.com/itiligent/Sonos-OpenWRT-VLANs/blob/main/example-config-files/etc/smcroute.conf) to configure the required multicast forwarding route tables.

```plaintext
# SSDP multicast routing for Sonos / DLNA / UPnP
#
# LAN   : br-lan.100  / 192.168.1.0/24
# GUEST : br-lan.200/ 192.168.2.0/24
# IOT   : br-lan.300 / 192.168.3.0/24

phyint br-lan.100 enable
phyint br-lan.200 enable
phyint br-lan.300 enable

# Ensure SSDP reaches the router on each source VLAN.
# Particularly useful when IGMP snooping is enabled.
mgroup from br-lan.100 group 239.255.255.250
mgroup from br-lan.200 group 239.255.255.250
mgroup from br-lan.300 group 239.255.255.250

# LAN -> IoT
mroute from br-lan.100 group 239.255.255.250 to br-lan.300

# Guest -> IoT
mroute from br-lan.200 group 239.255.255.250 to br-lan.300

# IoT -> LAN and Guest
mroute from br-lan.300 group 239.255.255.250 to br-lan.100 br-lan.200
```

---

### **Step 4: Configure Avahi For mDNS Discovery & Apple Airplay**
Edit [`/etc/avahi/avahi-daemon.conf`](https://github.com/itiligent/Sonos-OpenWRT-VLANs/blob/beta/example-config-files/etc/avahi/avahi-daemon.conf) as follows:  
Note: The `allow-interfaces` directive must be used to restrict mDNS access to just the required internal networks.

```ini
[server]
use-ipv4=yes
use-ipv6=yes # Or no as required
check-response-ttl=no
use-iff-running=no
allow-interfaces=br-lan.100,br-lan.200,br-lan.300  # Adapt to your specific VLAN interfaces here

[publish]
publish-addresses=yes
publish-hinfo=yes
publish-workstation=no
publish-domain=yes

[reflector]
enable-reflector=yes # Only one reflector per network, =no for all additional downstream dumb APs
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

### **Step 5: Decide Your IGMP Snooping Approach (enable/disable)**

> [!NOTE]
> OpenWRT disables IGMP snooping by default.
> 
> _**For typical home networks**_, forcing IGMP Snooping to disabled will help prevent device discovery and connectivity issues with Sonos and other multicast-dependent devices. 
> 
> For _**larger advanced networks with significant multicast streaming traffic and multiple VLANs trunked to downstream OpenWRT access points**_, enabling IGMP snooping can improve network efficiency by forwarding multicast traffic only to ports with devices subscribed to relevant multicast groups.
>
### **For Typical Home Networks:**
Some OpenWRT builds have a bug that allows IGMP even in the default (disabled) state. It is recommended to forcefully disable IGMP snooping on the example network's `br-lan` bridge device. Edit `/etc/config/network` and ensure the following options are set as follows:

```text
config device
        option name 'br-lan'
        option type 'bridge'
        # ... existing configuration ...
        option igmp_snooping '0'
        option multicast_querier '0'
```

### **For Larger or Advanced Networks:**

> [!IMPORTANT]
> When using multiple OpenWRT access points interconnected via VLAN trunks, **IGMP snooping must be configured individually for each VLAN** to ensure correct multicast forwarding.
>
> For detailed configuration instructions, see the included [Advanced IGMP Snooping Guide](https://github.com/itiligent/Sonos-OpenWRT-VLANs/blob/main/Advanced-IGMP-snooping.md).
>
> **Do not configure IGMP snooping through LuCI or UCI for this topology.** These interfaces do not expose all the bridge and per-VLAN settings necessary for IGMP snooping to function correctly across VLAN trunks.
---

### **Step 6: [Optional] Samba Music Library Share** 
Because a router is typically powered on 24/7, hosting your music library **directly from the OpenWRT router** provides a simple, low-power way to keep your music collection continuously available on the network.

To set this up, install `samba4-server luci-app-samba4 wsdd2` packages, then follow [this YouTube tutorial](https://www.youtube.com/watch?v=asN9aZ6Fg00) for instructions on sharing a USB drive through Samba on OpenWRT

The attached [example smb.conf.template](https://github.com/itiligent/Sonos-OpenWRT-VLANs/blob/beta/example-config-files/etc/samba/smb.conf.template) file includes settings for a **read-only guest music share**. 

The below firewall rules are required to allow both Sonos devices and controller access to the OpenWRT smb music share.

```
config rule
        option name 'Allow-SMB-LAN-to-Router'
        option family 'ipv4'
        option src 'lan'
        option proto 'tcp'
        option dest_port '445'
        option target 'ACCEPT'

config rule
        option name 'Allow-SMB-GUEST-to-Router'
        option family 'ipv4'
        option src 'guest'
        option proto 'tcp'
        option dest_port '445'
        option target 'ACCEPT'

config rule
        option name 'Allow-SMB-IOT-to-Router'
        option family 'ipv4'
        option src 'iot'
        option proto 'tcp'
        option dest_port '445'
        option target 'ACCEPT'
```

> [!NOTE]
> By default, OpenWRT/WSDD2 advertises Samba shares for Windows Network discovery only on the default LAN interface.
>
> To make Samba shares discoverable via WSDD2 on additional VLAN interfaces, replace the contents of `/etc/init.d/wsdd2` with [this updated WSDD2 startup script](https://github.com/itiligent/Sonos-OpenWRT-VLANs/blob/main/example-config-files/init.d/wsdd2)
>
> Near the top of the script, edit the following line to specify the OpenWRT network interfaces on which WSDD2 should advertise Samba discovery:
>
> `WSD_NETWORKS="lan guest iot"`
>
> You must also permit TCP/UDP ports **3702** and **5355** to each router interface listed in the `WSD_NETWORKS` section of the startup script, as shown below:
```
config rule
	option name 'Allow-WSD-LLMNR-LAN'
	option family 'ipv4'
	option src 'lan'
	list proto 'udp'
	list proto 'tcp'
	option dest_port '3702 5355'
	option target 'ACCEPT'

config rule
	option name 'Allow-WSD-LLMNR-GUEST'
	option family 'ipv4'
	option src 'guest'
	list proto 'udp'
	list proto 'tcp'
	option dest_port '3702 5355'
	option target 'ACCEPT'

config rule
	option name 'Allow-WSD-LLMNR-IOT'
	option family 'ipv4'
	option src 'iot'
	list proto 'udp'
	list proto 'tcp'
	option dest_port '3702 5355'
	option target 'ACCEPT'
```


### **Step 7: [Optional] Additional Persistent Disk Storage**
For  on x86, the most reliable way to add persistent music storage is to create a separate EXT4-formatted vdisk and auto-mount it via `/etc/fstab`. To ensure persistence across firmware resets or upgrades, you can bake your modified `/etc/fstab` into a custom firmware image. This permanently sets the extra EXT4 partition in place and prevents it from being lost upon firmware resets or upgrades. See here for more on adding additional partitions to : [https://github.com/itiligent/Easy--Builder](https://github.com/itiligent/Easy--Builder?tab=readme-ov-file#-persistent-filesystem-expansion-without-resizing-partitions)    

---
