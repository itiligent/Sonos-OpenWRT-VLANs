# VLAN-Aware IGMP Snooping on OpenWrt

A practical guide to configuring **per-VLAN IGMP snooping** on OpenWrt, with a **single IGMP querier on the primary router** and optional downstream access points (APs) connected over VLAN trunks.

The configuration uses Linux bridge multicast settings and OpenWrt hotplug scripts to keep multicast behaviour consistent across participating devices.

> [!IMPORTANT]
> **Use one designated IGMP querier per VLAN.** In this topology, the primary router provides the per-VLAN querier, while downstream APs perform IGMP snooping only. The bridge-global querier remains disabled on every device.

## Contents

- [Network topology](#network-topology)
- [Prerequisites](#prerequisites)
- [Configuration](#configuration)
  - [Step 1: Enable bridge-level IGMP snooping](#step-1-enable-bridge-level-igmp-snooping)
  - [Step 2: Install the per-VLAN multicast hotplug script](#step-2-install-the-per-vlan-multicast-hotplug-script)
  - [Step 3: Disable Wi-Fi multicast-to-unicast conversion](#step-3-disable-wi-fi-multicast-to-unicast-conversion)
  - [Step 4: Allow IGMP through the firewall](#step-4-allow-igmp-through-the-firewall)
  - [Step 5: Reboot the devices](#step-5-reboot-the-devices)
- [Verify the configuration](#verify-the-configuration)
- [Expected configuration summary](#expected-configuration-summary)

---

## Network topology

The following example uses three VLANs:

| Network | Purpose | VLAN ID |
| :--- | :--- | ---: |
| **LAN** | Trusted clients | `100` |
| **Guest** | Guest devices | `200` |
| **IoT** | Smart-home and discovery devices | `300` |

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

| Device role | IGMP snooping | Per-VLAN IGMP querier | VLAN trunk handling |
| :--- | :---: | :---: | :--- |
| **Primary router** | Enabled | Enabled | Configure all downstream trunk ports |
| **Downstream AP** | Enabled | Disabled | Configure the uplink and any downstream trunk ports |

The primary router acts as the **designated IGMP querier for each participating VLAN**. Downstream APs use snooping to maintain multicast group membership without starting their own queriers.

---

## Prerequisites

Before starting, confirm the following:

- OpenWrt **25.12.x**, or another release with the required Linux bridge per-VLAN multicast features.
- An existing Linux bridge (in this guide, `br-lan`) with **VLAN filtering enabled**.
- VLANs `100`, `200` and `300` already configured on the relevant bridge ports.
- One primary OpenWrt router and, optionally, one or more downstream OpenWrt APs connected by VLAN trunks.
- Root/SSH access to every participating router or AP.

### Install the required utilities

On **every participating OpenWrt device**, run:

```sh
apk update
apk add ip-tiny ip-bridge nano
```

### Confirm VLAN filtering is present

```sh
cat /sys/class/net/br-lan/bridge/vlan_filtering
bridge vlan show
```

The first command must return **`1`**. The second command should show the configured VLANs on the relevant bridge ports.

> [!NOTE]
> This guide assumes that VLAN membership, tagging and trunk connectivity are all configured and working correctly.

---

## Configuration

### Step 1: Enable bridge-level IGMP snooping

**Applies to:** All participating OpenWrt routers and APs.

Edit the `br-lan` device section in `/etc/config/network` and ensure it includes:

```text
config device
        option name 'br-lan'
        option type 'bridge'
        option igmp_snooping '1'
        option multicast_querier '0'
        # Keep your existing port and VLAN configuration.
```

This enables bridge-level IGMP snooping while leaving the **bridge-global querier disabled**. The per-VLAN querier settings are applied by the hotplug script in the next step.

### Step 2: Install the per-VLAN multicast hotplug script

**Applies to:** All participating OpenWrt routers and APs.

#### 2.1 Create the script

```sh
nano /etc/hotplug.d/iface/99-vlan-multicast
```

Copy the contents of **[`99-vlan-multicast`](https://github.com/itiligent/Sonos-OpenWRT-VLANs/blob/main/example-config-files/etc/hotplug.d/iface/99-vlan-multicast)** into the new `99-vlan-multicast` file and save it.

Make the script executable, then create a symlink for additional hotplug event coverage:

```sh
chmod +x /etc/hotplug.d/iface/99-vlan-multicast
mkdir -p /etc/hotplug.d/net
ln -s ../iface/99-vlan-multicast /etc/hotplug.d/net/99-vlan-multicast
```

The symlink allows the same script to handle **network-device events** as well as **logical-interface events**, providing another opportunity to reapply multicast settings when bridge, VLAN or trunk devices change.

#### 2.2 Configure the script variables

Edit the settings at the top of `/etc/hotplug.d/iface/99-vlan-multicast` to setting that suit your network, below is an example only:

```sh
BRIDGE="br-lan"
VLAN_IDS="100 200 300"
DISABLED_VLAN_IDS=""
TRIGGER_INTERFACE="lan"
ROLE="primary"
TRUNKS="eth1 eth2"
```

| Variable | Description |
| :--- | :--- |
| `BRIDGE` | Linux bridge containing the VLANs, for example `br-lan`. |
| `VLAN_IDS` | Space-separated VLAN IDs on which IGMP snooping should be enabled. |
| `DISABLED_VLAN_IDS` | Optional VLAN IDs on which to explicitly disable **both** IGMP snooping and the IGMP querier. Leave empty if not needed. |
| `TRIGGER_INTERFACE` | OpenWrt **logical interface** that generates `ifup` events, commonly `lan`. This may differ from the bridge device name. |
| `ROLE` | `primary` enables the IGMP querier for the selected VLANs; `ap` disables the querier. |
| `TRUNKS` | Space-separated **bridge-port** names used as VLAN trunks between devices. Leave empty if there are no trunks. |

> [!IMPORTANT]
> Every VLAN in `DISABLED_VLAN_IDS` must already exist and **must not** also appear in `VLAN_IDS`. The script logs an error and stops if the lists overlap.

#### 2.3 Select the appropriate device role

**Option A — Primary router without downstream APs**

```sh
ROLE="primary"
TRUNKS=""
```

**Option B — Primary router with downstream APs**

```sh
ROLE="primary"
TRUNKS="eth1 eth2"  # Trunk ports to downstream APs in the example topology
```

**Option C — Downstream AP**

```sh
ROLE="ap"
TRUNKS="eth5"       # Uplink/trunk port to the primary router
```

If an AP also connects to additional downstream APs, include those trunk ports in `TRUNKS`, provided they belong to the same bridge and carry every VLAN specified in `VLAN_IDS`.

Check the actual interface names with:

```sh
bridge link
bridge vlan show
```

> [!WARNING]
> Use **actual bridge-port names** (such as `eth1` or `eth5`) for `TRUNKS`, not VLAN subinterfaces or OpenWrt logical network names. Each listed trunk must belong to `BRIDGE` and carry **all** VLANs listed in `VLAN_IDS`.


#### 2.5 How the script operates

After an `ifup` event, `99-vlan-multicast` applies the per-VLAN multicast settings to the VLANs in `VLAN_IDS`.

For each selected VLAN, it also marks every configured trunk as a **permanent multicast-router port** using `mcast_router 2`, ensuring the relevant multicast traffic can traverse the VLAN trunks.

### Step 3: Disable Wi-Fi multicast-to-unicast conversion

**Applies to:** Every participating device that hosts Wi-Fi interfaces.

> [!WARNING]
> Some OpenWrt builds may leave multicast-to-unicast enabled at runtime (`1`) even when the UCI configuration contains `option multicast_to_unicast '0'`. See [OpenWrt issue #23399](https://github.com/openwrt/openwrt/issues/23399).
>
> The additional hotplug script prevents the risk of incorrect behaviour by checking `/sys/class/net/<interface>/brport/multicast_to_unicast` and writing `0` where required.

#### 3.1 Disable multicast-to-unicast in the wireless configuration

Run the following commands on each device that hosts Wi-Fi interfaces:

```sh
for s in $(uci show wireless | sed -n "s/^\(wireless\.[^=]*\)=wifi-iface$/\1/p"); do
    uci set "$s.multicast_to_unicast=0"
done

uci commit wireless
wifi reload
```

All individual `wifi-iface` sections in `/etc/config/wireless` should then contain:

```text
config wifi-iface
        option device 'radio0'
        option mode 'ap'
        option network 'iot'
        option ssid 'IOT'
        # Keep your other existing wireless settings.
        option multicast_to_unicast '0'
```

#### 3.2 Install the multicast-to-unicast runtime fix

Create the hotplug script:

```sh
nano /etc/hotplug.d/net/99-multicast-to-unicast-fix
```

Copy the contents of **[`99-multicast-to-unicast-fix`](https://github.com/itiligent/Sonos-OpenWRT-VLANs/blob/main/example-config-files/etc/hotplug.d/net/99-multicast-to-unicast-fix)** into the file and save it.

Make it executable:

```sh
chmod +x /etc/hotplug.d/net/99-multicast-to-unicast-fix
```

The script responds to network-device `add`, `move` and `bind` events, waits briefly for Wi-Fi bridge ports to initialise, then disables multicast-to-unicast conversion where needed.

> [!NOTE]
> This workaround applies to **all detected Wi-Fi bridge ports**, not only those associated with VLANs `100`, `200` and `300`. It changes runtime settings, which are reapplied when matching hotplug events occur.

### Step 4: Allow IGMP to the primary router

**Applies to:** Participating OpenWrt devices with an active firewall, IGMP must be permitted to reach the router.

Add the relevant rules to `/etc/config/firewall`, adapting the zone names to your configuration:

```text
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

### Step 5: Reboot the devices

After completing the configuration, reboot each participating router and AP:

```sh
reboot
```

---

## Verify the configuration

Perform the following checks on the primary router and every participating downstream AP.

### 1. Check bridge-global multicast settings

```sh
cat /sys/class/net/br-lan/bridge/multicast_snooping
cat /sys/class/net/br-lan/bridge/multicast_querier
```

Both the primary router and downstream APs should return:

```text
1
0
```

This confirms that **bridge-level snooping is enabled** and the **bridge-global querier is disabled**. Check the per-VLAN querier settings separately below.

### 2. Check per-VLAN multicast settings

```sh
bridge vlan global show dev br-lan
```

For VLANs `100`, `200` and `300`, verify:

| Setting | Primary router | Downstream AP |
| :--- | :---: | :---: |
| `mcast_snooping` | `1` | `1` |
| `mcast_querier` | `1` | `0` |
| `mcast_igmp_version` | `2` | `2` |

If you have configured `DISABLED_VLAN_IDS`, those VLANs should report `mcast_snooping 0` and `mcast_querier 0`.

Confirm VLAN filtering remains enabled:

```sh
cat /sys/class/net/br-lan/bridge/vlan_filtering
```

**Expected result:** `1`.

### 3. Check every VLAN trunk

Replace the sample names below with **every bridge port** listed in that device's `TRUNKS` variable:

```sh
for port in eth1 eth2; do
    echo "=== $port ==="
    bridge -d vlan show dev "$port"
done
```

For each VLAN in `VLAN_IDS`, every configured trunk should report **`mcast_router 2`**. VLANs listed in `DISABLED_VLAN_IDS` should report **`mcast_router 1`**.

This check does not apply to a standalone primary router with `TRUNKS=""`.

### 4. Check Wi-Fi multicast-to-unicast runtime settings

```sh
for p in /sys/class/net/*; do
    [ -e "$p/phy80211" ] || continue
    f="$p/brport/multicast_to_unicast"
    [ -r "$f" ] && printf '%s = %s\n' "${p##*/}" "$(cat "$f")"
done
```

Every detected Wi-Fi bridge interface should report `0`. Example:

```text
phy0-ap0 = 0
phy1-ap0 = 0
```

This check is not applicable to devices without Wi-Fi AP interfaces.

### 5. Inspect logs and multicast group membership

```sh
logread -e mcast-vlan
logread -e mcast-fix
bridge mdb show dev br-lan
```

Check for successful per-VLAN multicast configuration messages and, where a runtime correction was required, entries from the multicast-to-unicast workaround.

The bridge multicast database (MDB) lists memberships when clients join multicast groups. **An empty MDB does not necessarily mean the configuration has failed.**

---

## Expected configuration summary

| Feature | Primary router | Downstream AP |
| :--- | :---: | :---: |
| Bridge-level IGMP snooping | Enabled | Enabled |
| Per-VLAN multicast snooping | Enabled | Enabled |
| Snooping on VLANs `100`, `200`, `300` | Enabled | Enabled |
| IGMP querier on VLANs `100`, `200`, `300` | Enabled | Disabled |
| Bridge-global IGMP querier | Disabled | Disabled |
| Optional `DISABLED_VLAN_IDS` snooping / querier | Disabled / Disabled | Disabled / Disabled (if configured) |
| IGMP version | `2` | `2` |
| `mcast_router` on configured trunks | `2` for each enabled VLAN | `2` for each enabled VLAN |
| Wi-Fi multicast-to-unicast conversion | Disabled (where applicable) | Disabled (where applicable) |

---

[Back to the main Sonos VLAN guide](README.md)

**Related configuration files:** [`99-vlan-multicast`](https://github.com/itiligent/Sonos-OpenWRT-VLANs/blob/main/example-config-files/etc/hotplug.d/iface/99-vlan-multicast) · [`99-multicast-to-unicast-fix`](https://github.com/itiligent/Sonos-OpenWRT-VLANs/blob/main/example-config-files/etc/hotplug.d/net/99-multicast-to-unicast-fix)
