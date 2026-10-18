# Sonos Across VLANs on OpenWrt

**A practical configuration guide for Sonos discovery and control across segregated OpenWrt VLANs.**

This guide uses **SMCRoute** for SSDP multicast routing, **Avahi** for mDNS reflection, and targeted **firewall rules** for the unicast traffic required between Sonos speakers and controller applications on different VLANs. Optional sections cover IGMP snooping, a Samba music library, Windows network discovery, and persistent storage.

> [!NOTE]
> The configurations below are examples. Adjust VLAN IDs, IP ranges, interface names, firewall zones, and package installation commands to match your OpenWrt environment.

## Contents

- [Network requirements](#network-requirements)
- [Reference topology and prerequisites](#reference-topology-and-prerequisites)
- [Step-by-step configuration](#step-by-step-configuration)
  - [1. Install SMCRoute and Avahi](#1-install-smcroute-and-avahi)
  - [2. Configure the firewall](#2-configure-the-firewall)
  - [3. Configure SMCRoute](#3-configure-smcroute)
  - [4. Configure Avahi for mDNS and AirPlay](#4-configure-avahi-for-mdns-and-airplay)
  - [5. Choose an IGMP snooping approach](#5-choose-an-igmp-snooping-approach)
  - [6. Optional: Host a Samba music library](#6-optional-host-a-samba-music-library)
  - [7. Preserve configuration during upgrades](#7-preserve-configuration-during-upgrades)
  - [8. Optional: Add persistent disk storage](#8-optional-add-persistent-disk-storage)

---

## Network requirements

To operate Sonos speakers and controller applications across VLANs, the network must support:

1. **SSDP multicast forwarding** between the relevant VLANs using SMCRoute.
2. **mDNS/Bonjour reflection** between the relevant VLANs using Avahi.
3. **Bidirectional unicast communication** between Sonos speakers and controllers, permitted through the firewall.

### Core protocols and services

| Function | Address, protocol, or port | Implementation |
| --- | --- | --- |
| Sonos SSDP discovery | `239.255.255.250` (UDP) | SMCRoute multicast forwarding |
| mDNS, Bonjour, and AirPlay | `224.0.0.251:5353` (UDP) | Avahi mDNS reflection |
| Sonos unicast traffic and callbacks | TCP/UDP | Firewall allow rules |

### Optional services

| Function | Protocol or port | Implementation |
| --- | --- | --- |
| Samba-hosted music library | TCP `445` | Firewall allow rules |
| Windows discovery for Samba shares | TCP/UDP `3702`, `5355` | WSDD2 and firewall allow rules |
| Per-VLAN IGMP snooping | IGMP | [Advanced IGMP Snooping Guide](Advanced-IGMP-snooping.md) |

## Reference topology and prerequisites

The repository includes [example configuration files](example-config-files) providing a baseline OpenWrt setup for a multi-VLAN Sonos deployment. Adapt them to your network rather than applying them unchanged.

### Example VLANs

| Network | VLAN | Subnet | Purpose |
| --- | ---: | --- | --- |
| LAN | `100` | `192.168.1.0/24` | Trusted network with Sonos controller applications |
| Guest (optional) | `200` | `192.168.2.0/24` | Guest network with Sonos controller applications |
| IoT | `300` | `192.168.3.0/24` | Network containing Sonos speakers and other IoT devices |

### Assumptions

- OpenWrt v21.x or later is installed.
- WAN access is available to all three example VLANs.
- Sonos speakers have addresses assigned within a **contiguous static IP range**. Static DHCP reservations are recommended; for example, use the `192.168.3.16/28` address block.
- Root access over SSH is available to the OpenWrt router.

> [!TIP]
> The configuration snippets use interfaces `br-lan.100`, `br-lan.200`, and `br-lan.300`. Replace these names and the example addresses with the values used by your network.

---

## Step-by-step configuration

### 1. Install SMCRoute and Avahi

Install the packages required for multicast routing and mDNS reflection:

```sh
apk update
apk add smcroute avahi-daemon nano
```

If you also intend to host a music library on the router, install these optional packages:

```sh
apk add samba4-server luci-app-samba4 wsdd2
```

### 2. Configure the firewall

Use the [example firewall configuration](example-config-files/etc/config/firewall) as a starting point, adapting the rules to your VLANs, firewall zones, and Sonos IP range.

The example configuration:

- Sets internal `input` and `forward` policies to `REJECT`, providing an **implicit-deny, explicit-allow** model.
- Permits the DHCP, DNS, SSH, and LuCI traffic required for basic router operation.
- Restricts LAN and Guest Sonos controller access to a specified range of IoT Sonos IP addresses.
- Permits the required bidirectional Sonos callback traffic.
- Allows mDNS traffic from the relevant VLANs to Avahi.
- Allows SSDP and multicast forwarding between the relevant VLANs through SMCRoute.
- Includes optional access rules for the router-hosted Samba music library and Windows service discovery.

### 3. Configure SMCRoute

Edit [`/etc/smcroute.conf`](example-config-files/etc/smcroute.conf) to establish SSDP multicast routing between the VLANs.

```conf
# SSDP multicast routing for Sonos / DLNA / UPnP
#
# LAN   : br-lan.100 / 192.168.1.0/24
# GUEST : br-lan.200 / 192.168.2.0/24
# IOT   : br-lan.300 / 192.168.3.0/24

phyint br-lan.100 enable
phyint br-lan.200 enable
phyint br-lan.300 enable

# Ensure SSDP reaches the router on each source VLAN.
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

### 4. Configure Avahi for mDNS and AirPlay

Edit [`/etc/avahi/avahi-daemon.conf`](https://github.com/itiligent/Sonos-OpenWRT-VLANs/blob/beta/example-config-files/etc/avahi/avahi-daemon.conf) to reflect mDNS between the required VLAN interfaces:

```ini
[server]
use-ipv4=yes
use-ipv6=yes # Or no as required
check-response-ttl=no
use-iff-running=no
allow-interfaces=br-lan.100,br-lan.200,br-lan.300  # Adapt to your VLAN interfaces

[publish]
publish-addresses=yes
publish-hinfo=yes
publish-workstation=no
publish-domain=yes

[reflector]
enable-reflector=yes # One reflector per network; use no on downstream dumb APs
reflect-ipv=no

[rlimits]
rlimit-core=0
rlimit-data=4194304
rlimit-fsize=0
rlimit-nofile=30
rlimit-stack=4194304
rlimit-nproc=3
```

> [!IMPORTANT]
> Use `allow-interfaces` to restrict Avahi to the VLANs that require mDNS discovery. Enable **only one Avahi reflector** per network; disable reflection on additional downstream dumb access points.

### 5. Choose an IGMP snooping approach

IGMP snooping is optional. Choose the approach appropriate for your topology and multicast traffic.

#### **Typical home networks: Leave snooping unconfigured**

For a typical home deployment, IGMP snooping is not required. Leave the OpenWrt settings in their default, unconfigured state; SMCRoute handles the multicast routing needed for this setup.

> [!IMPORTANT]
> Some OpenWrt builds may leave Wi-Fi multicast-to-unicast conversion enabled despite the default configuration. This can contribute to Sonos lag, discovery problems, or dropouts.
>
> Check the active settings:
>
> ```sh
> grep -H . /sys/class/net/*/brport/multicast_to_unicast
> ```
>
> - `0` — multicast-to-unicast conversion is disabled.
> - `1` — multicast-to-unicast conversion is enabled.
>
> Unexpected `1` values may require corrective configuration. See [Disable Wi-Fi multicast-to-unicast conversion](https://github.com/itiligent/Sonos-OpenWRT-VLANs/blob/main/Advanced-IGMP-snooping.md#step-3-disable-wi-fi-multicast-to-unicast-conversion) in the advanced guide.

#### **Larger or advanced networks: Consider per-VLAN snooping**

For networks with significant multicast traffic and several VLANs trunked to downstream OpenWrt access points, IGMP snooping can reduce unnecessary multicast delivery by restricting traffic to ports with interested receivers.

> [!IMPORTANT]
> When multiple OpenWrt access points are interconnected over VLAN trunks, **configure IGMP snooping individually for each VLAN** to maintain correct multicast forwarding.
>
> For this topology, **do not rely on LuCI or UCI alone** to configure IGMP snooping. Those interfaces do not expose all required bridge and per-VLAN settings.
>
> Refer to the [Advanced IGMP Snooping Guide](Advanced-IGMP-snooping.md) for the detailed configuration procedure.

### 6. Optional: Host a Samba music library

A router that operates continuously can also provide a low-power, always-available Samba music library for Sonos.

Install the [optional Samba packages](#1-install-smcroute-and-avahi), then follow this [tutorial on sharing a USB drive through Samba on OpenWrt](https://www.youtube.com/watch?v=asN9aZ6Fg00).

The example [`smb.conf.template`](https://github.com/itiligent/Sonos-OpenWRT-VLANs/blob/beta/example-config-files/etc/samba/smb.conf.template) includes a **read-only guest music share**.

#### Allow access to the Samba share

To allow Sonos speakers and controller applications to reach the music share hosted on the router, add these rules to `/etc/config/firewall` as applicable:

```conf
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

#### Enable Windows discovery across VLANs (WSDD2)

By default, WSDD2 advertises Samba shares for Windows Network discovery only on the default LAN interface.

To advertise shares on additional VLANs:

1. Replace `/etc/init.d/wsdd2` with the [updated WSDD2 startup script](example-config-files/init.d/wsdd2).
2. Near the top of the script, configure the OpenWrt network interfaces that should receive WSDD2 advertisements:

   ```sh
   WSD_NETWORKS="lan guest iot"
   ```

3. Permit TCP/UDP ports **3702** and **5355** to the router from each listed network, using the following firewall rules as applicable:

```conf
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

### 7. Preserve configuration during upgrades

Update `/etc/sysupgrade.conf` to include the custom multicast hotplug scripts and WSDD2 startup script:

```text
# For optional Windows Discovery
/etc/init.d/wsdd2

# For optioal per-VLAN IGMP Snooping settings 
/etc/hotplug.d/iface/*multicast*
/etc/hotplug.d/net/*multicast*
```

### 8. Optional: Add persistent disk storage

For **OpenWrt on x86**, a separate EXT4-formatted virtual disk mounted through `/etc/fstab` can provide persistent storage for a music library.

To preserve the storage configuration across firmware resets or upgrades, include the modified `/etc/fstab` in a custom firmware image. This retains the extra EXT4 partition's mount configuration across firmware changes.

For additional details, see [Persistent filesystem expansion without resizing partitions](https://github.com/itiligent/Easy--Builder?tab=readme-ov-file#-persistent-filesystem-expansion-without-resizing-partitions) in the **Easy--Builder** project.

---

## Related resources

- [Example OpenWrt configuration files](example-config-files)
- [Advanced IGMP Snooping Guide](Advanced-IGMP-snooping.md)
- [SMCRoute example configuration](example-config-files/etc/smcroute.conf)
- [Avahi example configuration](https://github.com/itiligent/Sonos-OpenWRT-VLANs/blob/beta/example-config-files/etc/avahi/avahi-daemon.conf)
- [WSDD2 startup script](example-config-files/init.d/wsdd2)
