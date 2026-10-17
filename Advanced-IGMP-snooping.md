# VLAN-aware IGMP Snooping on OpenWRT

This guide explains how to establish a single IGMP querier on a primary OpenWRT router and configure per-VLAN IGMP snooping. It also covers the optional configuration of multiple downstream OpenWRT dumb access points connected by VLAN trunks for consistent IGMP multicast in a multi access point network.

## Example topology

| VLAN | Purpose | VLAN ID |
| --- | --- | ---: |
| LAN | Trusted clients | `100` |
| Guest | Guest devices | `200` |
| IoT | Smart-home and discovery devices | `300` |

```text
                   Primary OpenWrt router
                  (Layer 3 + IGMP querier)
                         br-lan
                    VLANs 100/200/300
                       /       \
                    eth1       eth2
                     |           |
                 VLAN trunk   VLAN trunk
                     |           |
                   AP 1        AP 2
              (snooping only; no querier)
                     |           |
                Wi-Fi clients on each AP
```

For each VLAN, the **primary router is the designated IGMP querier** and all downstream APs must perform snooping only.

## Prerequisites

- OpenWrt 25.12.x or another release with the required Linux bridge per-VLAN multicast features.
- A Linux bridge. The example `br-lan` is already configured with **VLAN filtering** and VLANs `100`, `200` and `300`.
- One OpenWRT primary router, or optionally one or more downstream OpenWRT access points connected using VLAN trunks.
- Root/SSH access to each participating router or AP

**Install the required utilities on every participating OpenWRT device:**

```sh
apk update
apk add ip-tiny ip-bridge nano
```

**Confirm that VLAN filtering is enabled:**

```sh
cat /sys/class/net/br-lan/bridge/vlan_filtering
bridge vlan show
```

The first command must return `1`. The second should list your VLANs on the relevant bridge ports. This guide assumes VLAN membership and tagging are already configured correctly; it does not create VLANs or trunk ports.

## 1. Configure bridge-level IGMP snooping

On **every** participating router and AP, edit the `config device` section for `br-lan` in `/etc/config/network` to include:

```text
config device
        option name 'br-lan'
        option type 'bridge'
        option igmp_snooping '1'
        option multicast_querier '0'
        # Keep your existing port and VLAN configuration.
```

## 2. Install the per-VLAN multicast script

On each router/AP:
```sh
nano /etc/hotplug.d/iface/99-vlan-multicast
```
Then paste the script [`99-vlan-multicast`](https://github.com/itiligent/Sonos-OpenWRT-VLANs/blob/main/example-config-files/etc/hotplug.d/iface/99-vlan-multicast) then save.

```sh
chmod +x /etc/hotplug.d/iface/99-vlan-multicast
mkdir -p /etc/hotplug.d/net
ln -s ../iface/99-vlan-multicast /etc/hotplug.d/net/99-vlan-multicast
```

Now edit the settings at the top of `/etc/hotplug.d/iface/99-vlan-multicast`:

```sh
BRIDGE="br-lan"
VLAN_IDS="100 200 300"
DISABLED_VLAN_IDS=""
TRIGGER_INTERFACE="lan"
ROLE="primary"
TRUNKS=""      # Optional space-separated list of trunk ports
```

- `BRIDGE`: Linux bridge device containing the VLANs.
- `VLAN_IDS`: Space-separated VLAN IDs that require IGMP snooping.
- `DISABLED_VLAN_IDS`: Optional space-separated VLAN IDs on which to explicitly **disable both IGMP snooping and the IGMP querier**. Leave empty if you do not need this behaviour. These VLANs must already exist and must not overlap with `VLAN_IDS`.
- `TRIGGER_INTERFACE`: OpenWrt **logical** interface that generates `ifup` (commonly `lan`). This is not necessarily the bridge device name.
- `ROLE`: `primary` enables the IGMP querier on each configured VLAN; `ap` disables it.
- `TRUNKS`: Space-separated physical bridge-port interfaces connecting to downstream APs. For example, `"eth1` or `"eth1 eth4"`. Leave empty when there are no trunks to configure.

### Choose the settings for each OpenWRT device

**A. Primary router without downstream APs**

```sh
ROLE="primary"
TRUNKS=""
```

**B. Primary router with multiple downstream APs**

```sh
ROLE="primary"
TRUNKS="eth1"    # The VLAN trunk port to each downstream AP
```

**C. Downstream dumb AP**

```sh
ROLE="ap"
TRUNKS="eth5"              # Uplink/trunk port connected toward the primary router
```

If an AP also has downstream APs connected to its other ports, you can list those as extra trunks too, provided they belong to the same bridge and carry the same selected VLANs in `VLAN_IDS`. 

Replace the example interface names with the actual **bridge-port** names (not VLAN subinterfaces or OpenWrt logical network names). Check with `bridge link` and `bridge vlan show`. Every configured trunk must already be a member of `BRIDGE` and carry **all** VLANs in `VLAN_IDS`. 

### Optionally disable snooping on selected VLANs

If some VLANs should **not** participate in IGMP snooping, set `DISABLED_VLAN_IDS` to their IDs. A VLAN must not appear in both `VLAN_IDS` and `DISABLED_VLAN_IDS` lists; the script will log an error and stop if there is an overlap. For example:

```sh
BRIDGE="br-lan"
VLAN_IDS="100 200 300"
DISABLED_VLAN_IDS="20 400"
TRIGGER_INTERFACE="lan"
ROLE="primary"
TRUNKS="eth1 eth2"
```

The above settings will configures snooping and the querier on VLANs `100`, `200`, and `300`, and then explicitly disables IGMP snooping on VLANs `20` and `400`.

he `99-vlan-multicast` script executes after `ifup` to enable VLAN-aware IGMP snooping and apply per-VLAN multicast settings to the VLANs specified in `VLAN_IDS`. For each specificed VLAN, the script designates all listed trunk ports as permanent multicast-router ports (`mcast_router 2`), ensuring multicast traffic is forwarded appropriately across VLAN trunks. 


## 3. Disable Wi-Fi multicast-to-unicast conversion (All Access Points)

On each OpenWrt device hosting the Wi-Fi access point interfaces, explicitly set every wifi inteface with `option multicast_to_unicast '0'` in `/etc/config/wireless`:

```text
for s in $(uci show wireless | sed -n "s/^\(wireless\.[^=]*\)=wifi-iface$/\1/p"); do
    uci set "$s.multicast_to_unicast=0"
done

uci commit wireless
wifi reload
```

All wifi-iface entries in `/etc/config/wireles` should look like:

```text
config wifi-iface
        option device 'radio0'
        option mode 'ap'
        option network 'iot'
        option ssid 'IOT'
        # existing configuration
        option multicast_to_unicast '0'
```

> [!WARNING]
> Some OpenWrt builds may leave the runtime multicast-to-unicast setting enabled (`1`) even when UCI explicitly specifies `option multicast_to_unicast '0'` ([OpenWrt issue #23399](https://github.com/openwrt/openwrt/issues/23399)).
>
> As a "belt and braces" workaround, the following hotplug script checks the runtime setting at `/sys/class/net/<interface>/brport/multicast_to_unicast` and writes `0` where necessary to disable multicast-to-unicast conversion.

Create the hotplug script:

```sh
nano /etc/hotplug.d/net/99-multicast-to-unicast-fix
```

Paste the contents of [`99-multicast-to-unicast-fix`](https://github.com/itiligent/Sonos-OpenWRT-VLANs/blob/main/example-config-files/etc/hotplug.d/net/99-multicast-to-unicast-fix) and save the file.

Make the script executable:

```sh
chmod +x /etc/hotplug.d/net/99-multicast-to-unicast-fix
```

The script responds to network-device `add`, `move` and `bind` events, briefly waits for Wi-Fi bridge ports to initialise, and disables multicast-to-unicast conversion where required.

**Note:** The workaround applies to **all detected Wi-Fi bridge ports**, not just those associated with VLANs `100`, `200` and `300`. Changes affect runtime settings only and are reapplied when matching hotplug events occur.


## 4. Update the firewall to allow IGMP to reach the router (All Access Points)
```
config rule
        option name 'Allow-IGMP-LAN-to-Router'
        option family 'ipv4'
        option src 'lan'
        option proto 'igmp'
        option target 'ACCEPT'

config rule
        option name 'Allow-IGMP-GUEST-to-Router'
        option family 'ipv4'
        option src 'guest'
        option proto 'igmp'
        option target 'ACCEPT'

config rule
        option name 'Allow-IGMP-IOT-to-Router'
        option family 'ipv4'
        option src 'iot'
        option proto 'igmp'
        option target 'ACCEPT'
```

## 4. Reboot and verify

Reboot each participating router/AP after completing the configuration:

```sh
reboot
```

### 4.1 Verify the bridge

```sh
cat /sys/class/net/br-lan/bridge/multicast_snooping
cat /sys/class/net/br-lan/bridge/multicast_querier
```

Expected on **both** primary router and downstream AP:

```text
1
0
```

The bridge-global querier stays disabled; per-VLAN query settings are checked next.

### 4.2 Verify VLAN settings

```sh
bridge vlan global show dev br-lan
```

For VLANs `100`, `200` and `300`, look for:

| Setting | Primary router | Downstream AP |
| --- | --- | --- |
| `mcast_snooping` | `1` | `1` |
| `mcast_querier` | `1` | `0` |
| `mcast_igmp_version` | `2` | `2` |

If `DISABLED_VLAN_IDS` is configured, confirm those VLANs report `mcast_snooping 0` and `mcast_querier 0`.

Also confirm `vlan_filtering` is still `1`:

```sh
cat /sys/class/net/br-lan/bridge/vlan_filtering
```

### 4.3 Verify every configured trunk

Replace these interface names with the device's `TRUNKS` values. On each device with one or more configured VLAN trunks, check **every** listed bridge port with:

```sh
for port in eth1 eth4; do  # VLAN
    echo "=== $port ==="
    bridge -d vlan show dev "$port"
done
```
For each VLAN in `VLAN_IDS`, **every** configured trunk should show `mcast_router 2`. Any disabled VLANs from D`ISABLED_VLAN_IDS` should show `mcast_router 1`.


### 4.4 Verify Wi-Fi runtime settings

```sh
for p in /sys/class/net/*; do
    [ -e "$p/phy80211" ] || continue
    f="$p/brport/multicast_to_unicast"
    [ -r "$f" ] && printf '%s = %s\n' "${p##*/}" "$(cat "$f")"
done
```

All enabled Wi-Fi bridge interfaces should report `0`. For exaxmple:
```
phy0-ap0 = 0
phy1-ap0 = 0
```

### 4.5 Check logs and multicast group membership

```sh
logread -e mcast-vlan
logread -e mcast-fix
bridge mdb show dev br-lan
```

Look for a successful `vlan-mcast` log entry on each device and `mcast-fix` entries if any runtime Wi-Fi values needed correction. MDB membership entries appear when multicast clients join groups; an empty table does not necessarily mean configuration failed.


## Expected configuration

| Feature | Primary router | Downstream AP |
| --- | --- | --- |
| Bridge IGMP snooping | On | On |
| VLAN-aware multicast snooping | On | On |
| VLAN 100 / 200 / 300 snooping | On | On |
| VLAN 100 / 200 / 300 IGMP querier | On | Off |
| Optional `DISABLED_VLAN_IDS` snooping / querier | Off / Off | Off / Off (if configured) |
| IGMP version | 2 | 2 |
| All configured inter-device trunks `mcast_router` | `2` for each enabled VLAN | `2` for each enabled VLAN |
| Wi-Fi multicast-to-unicast (relevant AP interfaces) | Off | Off |

The trunk values are **not applicable** on a standalone primary router without configured trunks, and Wi-Fi settings are not applicable to devices with no AP interfaces.



