# 🎵 **Configure Sonos with VLANs & OpenWRT (2026)**

After late 2024 much Sonos networking documentation became outdated as Sonos shifted from a LAN-first to a CLOUD-first model (aka ensh*tification), requiring new VLAN considerations for:

- Device discovery & setup
- Sonos controller apps (all platforms & versions)
- Streaming & local music libraries
- Apple AirPlay

---

## 📋 **Key Setup Requirements**  

For reliable Sonos across OpenWRT VLANs, you’ll need to establish the following: 

- 🔄 mDNS
- 📡 IGMP Snooping
- 🌐 IGMP Multicast Proxy
- 📥 ICMP between Sonos speakers, router & app controllers
- 📥 Broadcast forwarding (only for legacy desktop controller app)
- 🛡️ Firewall & IGMPproxy multicast rules to:
     - Restrict multicast to only specific internal VLANs
     - Forward Sonos unicast traffic between VLANs securely
---

### 🛠️ Reference OpenWRT Setup 

### **Example VLANs**  
1. **LAN VLAN**:  
   - The trusted network with the Sonos controller application
      - This VLAN is the **"Upstream"** network 
2. **Guest VLAN** (optional):  
   - Another trusted network for guests to use a Sonos application
      -  This VLAN is also an **"Upstream"** network
3. **IOT VLAN**:  
   - The untrusted VLAN for your Sonos speakers and other IOT devices
      - This VLAN is the **"Downstream"** network

This link provides [companion OpenWRT config files](https://github.com/itiligent/Sonos-OpenWRT-VLANs/tree/main/example-config-files) that mirror the above VLAN structure and can be adapted to your own system.

---

### 🔧 Assumptions
- You are using a recent OpenWRT build (v21.x & above)
- Your VLANS for LAN, GUEST & IOT are pre-created and ready to start configuring (see supplied config files)
- WAN access is available to all VLANs
- Your Sonos speakers are setup using **static IP addresses** (static dhcp reservations are recommended)
---

### ⚠️ Security Warning With Sonos & Multicast Proxy

**Sonos multicast traffic must not be allowed to reach the WAN interface.**
 
   **Why?**

**Sonos uses multicast on the same address used by Universal Plug-n-Play (uPnP) 239.255.255.250. To prevent Sonos multicast proxy from inadvertently allowing uPnP to reach the WAN and punch holes all over your network, this guide shows how to restrict uPnP to only the example LAN, GUEST & IOT VLANs. You may need to adapt this to your own VLAN configuration.**

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

### **Step 2: Enable IGMP Snooping & Spanning Tree Protocol**  
To enable IGMP snooping & STP, add the following lines to the LAN, Guest & IOT bridge devices in `/etc/config/network`:  
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

To securely restrict insecure uPnP multicast, all router input and inter-zone forwarding traffic should be denied by default. To to this we must override hidden firewall defaults by requiring explicit firewall rules for all traffic such as LUCI HTTP, SSH, DNS, DHCP, and ICMP. This approach enforces a least priviledage approach via implicit deny and explicit permit.

Edit `/etc/config/firewall` as follows:

Set the firewall default behavior to implicitly deny everything to and through the router interface: 
```plaintext
config defaults
	option input 'REJECT'  # Prevents implied inputs to the router 
	option output 'ACCEPT' # Allow outgoing traffic from any zone
	option forward 'REJECT' # Reject forces explicit firewall rules to be used for forwarding
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

With WAN multicast already restricted (step 3), a single catch-all rule for 224.0.0.0/4 can now be applied internally. This RFC1112 block simplifies firewall management and enables cross-VLAN multicast for device discovery.


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
    list altnet 169.254.0.0/16  # this line stops IGMPproxy log noise from link local  
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

## Step 9: Older Sonos Desktop App & Legacy S1/S2 App Support
### Windows Desktop Application:
Older versions of the Windows Desktop controller used UDP 1900 broadcasts to 255.255.255.255 for discovery. From a Desktop in LAN or GUEST VLAN, these broadcasts can be relayed to the IOT VLAN via Socat. Test before making this change as newer application versions may work with just igmpproxy and the above rules.

Adjust the below to your IOT broadcast IP address and copy the following to `/etc/config/socat` 

```
config socat 'sonos_bcast_forward'
    option enable '1'
    option SocatOptions '-d -d udp4-recvfrom:1900,broadcast udp4-sendto:192.168.3.255:1900,broadcast'  # Adjust to the broadcast address of your IOT VLAN
    option user 'nobody'
```

Restart Socat with `/etc/init.d/socat restart` or from Luci "Startup" page.

### Apple OS Desktop Application 
Socat won't be needed as Apple relies on Bonjour/Avahi for discovery.

### Legacy Sonos S1 & S2 Apps: 
Android & Apple apps should work fine without Socat.  

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
For OpenWRT on x86, the most reliable way to add persistent music storage is to create a separate EXT4-formatted vdisk and auto-mount it via /etc/fstab. To ensure persistence across firmware resets or upgrades, bake your modified /etc/fstab into a custom firmware image. This prevents the extra EXT4 partition from being lost upon firmware resets or upgrades. See here for more on adding additional partitions to OpenWRT: [https://github.com/itiligent/Easy-OpenWRT-Builder](https://github.com/itiligent/Easy-OpenWRT-Builder?tab=readme-ov-file#-persistent-filesystem-expansion-without-resizing-partitions)    

---

## Future Sonos Changes?

The above config continues to work as at January 2026 with the current versions of Sonos application & OpenWRT firmware. All functions are 100% working (Sonos app, Airplay, new device discovery, new device setup, volume control, speaker grouping and Samba music library access all whilst connected to either the LAN VLAN or Guest VLAN). 

Please submit an issue if you notice something is not working, or share any helpful suggestions.  


