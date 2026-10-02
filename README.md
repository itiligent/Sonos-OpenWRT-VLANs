## 🎵 **Sonos & OpenWRT with multiple VLANs (2027)**

### 📋 **Sonos between OpenWRT VLANs needs the following items:** 

- 🔄 Avahi (mDNS)
- 🌐 IGMPProxy
- 🛡️ A quite specific firewall, mDNS & multicast proxy configuration to:
     - Facilitate Sonos multicast discovery between VLANs
     - Facilitate bi-directional unicast traffic between Speakers & Controller App located in separate VLANs 

### 🛠️ Sonos Core Networking Needs

| Function | Address / port | Multi-VLAN Approach |
|---|---|---|
| Sonos SSDP multicast | `239.255.255.250` / UDP | Forwarding via IGMPproxy |
| mDNS / Bonjour / AirPlay  | `224.0.0.251:5353` | Forwarding via Avahi |
| Sonos controller app traffic | various unicast | Normal firewall rules |
**Optional**
| Samba Hosted Music Library | TCP 445 | Normal firewall rules |

---


## 🛠️ Reference OpenWRT Setup 

### **Example Networks**  
1. **LAN (VLAN 100)**:  
   - The main trusted network for users with a Sonos controller application
      - Assumes 192.168.1.0/24
      -  IGMProxy **"Upstream"** network
2. **Guest (VLAN 200)** (optional):  
   - Another trusted network for guests with a Sonos controller application
      -  Assumes 192.168.2.0/24
     - An IGMProxy **"Upstream"** network
3. **IOT (VLAN 300)**:  
   - The untrusted VLAN for your Sonos speakers (and other IOT devices)
      - Assumes 192.168.1.0/24
      - An IGMProxy **"Downstream"** network

[This example network config file](https://github.com/itiligent/Sonos-OpenWRT-VLANs/tree/main/example-config-files) mirrors the above VLAN structure. Adapt this to your own system.

### 🔧 Assumptions
- You are using a recent OpenWRT build (v21.x & above)
- WAN access is available to all VLANs
- All Sonos speakers are configured within a contiguous block of **static IP addresses**. (A block of static dhcp reservations is recommended - eg: `192.168.3.16/28`)
---




## 🚀 **Step-by-Step OWRT Configuration**  

### **Step 1: Install IGMPproxy & Avahi** 
The below packages are needed:

```
apk update
apk add igmpproxy avahi-daemon

optional packages for music share:
apk add samba4-server luci-app-samba4
```

---

### **Step 2: Setup Firewall Rules**  

Adapt [this example firewall configuration file](example-config-files/etc/config/firewall) as needed.

The example firewall configuration:

- Defaults internal input and forwarding to `REJECT`
- Permits only explicit DHCP, DNS, management and discovery traffic to and through the router
- Permits LAN and Guest Sonos controllers to initiate unicast traffic to a set Sonos IP reservation block
- Permits the limited Sonos-initiated return callback traffic needed by the Sonos controller application
- Permits ICMP between all three internal example subnets
- Permits SMB TCP/445 to the router from LAN, Guest and IoT (for Samba music library access)
- Allows mDNS only to the local Avahi daemon
- Bocks router-originated multicast toward WAN

---

### **Step 3: Configure IGMPproxy**  
Edit [`/etc/config/igmpproxy`](https://github.com/itiligent/Sonos-OpenWRT-VLANs/blob/beta/example-config-files/etc/config/igmpproxy) to configure the **upstream & downstream networks** that will be allowed to proxy multicast traffic:  


> [!NOTE]
> Note: `list altnet` must be used to restrict igmpproxy to just the desired internal networks. Don't use 0.0.0.0/0!

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
	list altnet 192.168.3.0/24 # Adjust to your IOT network address
	list altnet 169.254.0.0/16 # This stops unnecessary log chatter
```


---

### **Step 4 Update The IGMPproxy Launch Script**

For security, OpenWRT's default `/etc/init.d/igmpproxy` launch script creates a hidden firewall rule that blocks all UDP uPnP multicast traffic on 239.255.255.250, however for Sonos discovery across VLANs we need to remove this restriction. [This replacement `/etc/init.d/igmpproxy` launch script](https://raw.githubusercontent.com/itiligent/Sonos-OpenWRT-VLANs/refs/heads/main/example-config-files/etc/init.d/igmpproxy) allows multicast UDP forwarding on 239.255.255.250 as well as selctively preventing Windows WS-Discovery and Avahi muticast from clashing. The exact changes from the default installed file are shown below: 

```
# Allow select multicast
        json_add_object ""
        json_add_string type rule
        json_add_string src "$upstream"
        json_add_string dest "$zone"
        json_add_string family ipv4
        json_add_string proto udp
        json_add_string dest_ip "239.255.255.250/32"
        json_add_string target ACCEPT
        json_close_object

}

igmp_add_firewall_network() {
        config_get direction $1 direction
        config_get zone $1 zone

        [ -n "$zone" ] || return

        # Do not let igmpproxy process link-local discovery memberships.
        # 224.0.0.251 = mDNS
        # 224.0.0.252 = LLMNR
        # These are handled locally / by Avahi and must not be proxied.
        json_add_object ""
        json_add_string type rule
        json_add_string src "$zone"
        json_add_string family ipv4
        json_add_string proto igmp
        json_add_string dest_ip "224.0.0.251/32"
        json_add_string target DROP
        json_close_object

        json_add_object ""
        json_add_string type rule
        json_add_string src "$zone"
        json_add_string family ipv4
        json_add_string proto igmp
        json_add_string dest_ip "224.0.0.252/32"
        json_add_string target DROP
        json_close_object

        # Allow remaining IGMP control traffic
        json_add_object ""
        json_add_string type rule
        json_add_string src "$zone"
        json_add_string family ipv4
        json_add_string proto igmp
        json_add_string target ACCEPT
        json_close_object

        [ "$direction" = "upstream" ] && {
                upstream="$zone"
                config_foreach igmp_add_firewall_routing phyint
        }
}

```
---

### **Step 5: Configure Avahi For mDNS Discovery & Apple Airplay**
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


### **Step 6: [Optional] Samba Music Library Share** 
Because a router is typically powered on 24/7, hosting your music library **directly from the OpenWrt router** provides a simple, low-power way to keep your collection continuously available on the network.

To set this up, install `samba4-server luci-app-samba4 wsdd2` packages, then follow [this YouTube tutorial](https://www.youtube.com/watch?v=asN9aZ6Fg00) for instructions on sharing a USB drive through Samba on OpenWrt.

The attached [example smb.conf.template file](https://github.com/itiligent/Sonos-OpenWRT-VLANs/blob/beta/example-config-files/etc/samba/smb.conf.template) includes settings for a **read-only guest music share**. 

The below firewall rules are required to allow both Sonos devices and application access to the OpenWRT smb music share**.

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

### **Step 7: [Optional] Additional Persistent Disk Storage**
For OpenWRT on x86, the most reliable way to add persistent music storage is to create a separate EXT4-formatted vdisk and auto-mount it via `/etc/fstab`. To ensure persistence across firmware resets or upgrades, you can bake your modified `/etc/fstab` into a custom firmware image. This permanently sets the extra EXT4 partition in place and prevents it from being lost upon firmware resets or upgrades. See here for more on adding additional partitions to OpenWRT: [https://github.com/itiligent/Easy-OpenWRT-Builder](https://github.com/itiligent/Easy-OpenWRT-Builder?tab=readme-ov-file#-persistent-filesystem-expansion-without-resizing-partitions)    

---
