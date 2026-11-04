# Routing Wazuh Agents from vmbr4 to the Manager (vmbr1)

This documents how agents on the isolated monitoring bridge `vmbr4` reach the Wazuh manager, which lives on the LAN/IT bridge `vmbr1`. `vmbr4` has no uplink of its own (`bridge-ports none`), so the Proxmox host (`192.168.2.1` on `vmbr4`) routes agent traffic between `vmbr4` and `vmbr1` to the manager at `192.168.0.67`.

> **How this build actually routes:** the host does **plain IP forwarding** — `ip_forward=1`, no NAT. (`iptables -t nat -S POSTROUTING` shows only the default `ACCEPT` policy.) That means the manager must have a return route back to the `192.168.2.0/24` subnet — see section 2, Option A. Section 2 Option B (masquerade) is the alternative if you'd rather not add a route on the manager.

## Addressing

| Host | Agent ID | vmbr4 IP |
|---|---|---|
| Active-Directory | 008 | 192.168.2.150 |
| Staff1 | 006 | 192.168.2.151 |
| Staff2 | 007 | 192.168.2.152 |
| OS-Ticket | 009 | 192.168.2.153 |
| Mailcow | 010 | 192.168.2.154 |
| Admin | 011 | 192.168.2.155 |
| JumpHost | 012 | 192.168.2.156 |
| Proxmox host (gateway) | — | 192.168.2.1 (on `vmbr4`), 192.168.0.137 (on `vmbr1`) |
| Wazuh manager | 000 | 192.168.0.67 (on `vmbr1`) |

Each VM's monitoring NIC is the one bridged to `vmbr4` in Proxmox — its index **varies per VM** (`net0`/`net1`/`net2`/…, e.g. `net2` on VM 101), so check Hardware rather than assuming a number. It has a static `/24` address and **no default gateway** — the only route out of `vmbr4` is a host route to the manager (section 4).

## Proxmox bridges

| Bridge | Role | Host config |
|---|---|---|
| `vmbr1` | IT network (LAN) | DHCP, uplink `enx7cc2c63ee714`; host is `192.168.0.137` |
| `vmbr2` | Admin network | internal only (`bridge-ports none`) |
| `vmbr3` | OT network | uplink `nic0` |
| `vmbr4` | Wazuh monitoring plane | static `192.168.2.1/24`, no uplink |
| `vmbr5` | External-link sim for Scenario 1 | uplink `enxc84d4422f4d8` |

## Ports

| Port | Proto | Purpose |
|---|---|---|
| 1514 | TCP | Agent to manager event channel (`wazuh-remoted`) |
| 1515 | TCP | Agent enrollment (`wazuh-authd`) |

---

## 1. Enable IP forwarding on the Proxmox host

Already set on this build (via the `vmbr4` stanza, section 3). To set it by hand:

```bash
# runtime
sysctl -w net.ipv4.ip_forward=1

# persistent
echo 'net.ipv4.ip_forward=1' > /etc/sysctl.d/99-wazuh-forward.conf
sysctl --system
```

---

## 2. Return path to the manager

Agents send to the host (`192.168.2.1`), which forwards to the manager on `vmbr1`. The only question is how replies get **back** to `192.168.2.0/24`. Pick one.

### Option A — static route on the manager (what this build uses)
With plain forwarding and no NAT, the manager sees the agent's real `192.168.2.x` source, so it needs to know how to return traffic. Add a route on the **Wazuh manager laptop** pointing the vmbr4 subnet at the Proxmox host's `vmbr1` address:

```bash
# on the manager (192.168.0.67)
sudo ip route add 192.168.2.0/24 via 192.168.0.137

# make it persistent (netplan example)
#   routes:
#     - to: 192.168.2.0/24
#       via: 192.168.0.137
```

Confirm it is there:
```bash
ip route get 192.168.2.153      # should go "via 192.168.0.137"
```

> If the vmbr4 agents are connecting but you have **not** added masquerade (Option B), this route — or an equivalent one on your IT-network router — is what is carrying the replies. Confirm it exists so the setup survives a manager reboot.

### Option B — masquerade on the host (alternative)
If you would rather not touch the manager's routing, SNAT the agent traffic on the Proxmox host so the manager sees it coming from `192.168.0.137` and needs no return route:

```bash
iptables -t nat -A POSTROUTING \
  -s 192.168.2.0/24 -d 192.168.0.67/32 \
  -o vmbr1 -p tcp -m multiport --dports 1514,1515 \
  -j MASQUERADE
```

Because the Proxmox firewall is enabled, give traffic off the firewall bridges its own conntrack zone, otherwise return traffic for NATed flows can be dropped:

```bash
iptables -t raw -I PREROUTING -i fwbr+ -j CT --zone 1
```

Use **either** Option A **or** Option B, not both.

---

## 3. Persist the host config in `/etc/network/interfaces`

Your live `vmbr4` stanza enables forwarding and marks where the Wazuh rules go:

```ini
auto vmbr4
iface vmbr4 inet static
        address 192.168.2.1/24
        bridge-ports none
        bridge-stp off
        bridge-fd 0
        post-up   echo 1 > /proc/sys/net/ipv4/ip_forward
        #Wazuh connections
```

If you chose **Option B (masquerade)**, add the NAT and conntrack rules under the `#Wazuh connections` marker so they survive reboots and `ifreload -a`:

```ini
        #Wazuh connections
        post-up   iptables -t nat -A POSTROUTING -s 192.168.2.0/24 -d 192.168.0.67/32 -o vmbr1 -p tcp -m multiport --dports 1514,1515 -j MASQUERADE
        post-down iptables -t nat -D POSTROUTING -s 192.168.2.0/24 -d 192.168.0.67/32 -o vmbr1 -p tcp -m multiport --dports 1514,1515 -j MASQUERADE
        post-up   iptables -t raw -I PREROUTING -i fwbr+ -j CT --zone 1
        post-down iptables -t raw -D PREROUTING -i fwbr+ -j CT --zone 1
```

Apply without rebooting: `ifreload -a`, then confirm with section 6. For **Option A** the stanza stays as-is (no masquerade) and the route lives on the manager instead.

> **Host Wi-Fi note:** the `wlp2s0` / `wpa_supplicant` interface on the host is left as `manual` (not brought up). With it active, `pve-firewall` and these rules did not reliably survive a reboot or `ifreload -a`. The host reaches the LAN solely via `vmbr1` (`192.168.0.137`).

---

## 4. Guest-side route to the manager

Since the `vmbr4` NIC has no gateway, each VM needs a single host route sending manager traffic through the Proxmox host's `vmbr4` address (`192.168.2.1`). This also keeps agent traffic on `vmbr4` even on VMs that also have a NIC on another bridge.

**Ubuntu (OS-Ticket, Mailcow, Admin, JumpHost)** — netplan. This is OS-Ticket's live config; the interface name (`ens19` here, matched by MAC) and the `.15X` address differ per VM:

```yaml
network:
  version: 2
  ethernets:
    ens19:
      dhcp4: false
      addresses:
        - 192.168.2.153/24
      routes:
        - to: 192.168.0.67/32
          via: 192.168.2.1
      match:
        macaddress: bc:24:11:3b:71:73
      set-name: ens19
```

```bash
sudo netplan apply
```

**Windows (AD, Staff1, Staff2)** — elevated prompt:

```powershell
route -p add 192.168.0.67 mask 255.255.255.255 192.168.2.1
```

The agent points at the manager directly in `ossec.conf` on each VM:

```xml
<client>
  <server>
    <address>192.168.0.67</address>
    <port>1514</port>
    <protocol>tcp</protocol>
  </server>
</client>
```

---

## 5. Proxmox firewall design

The monitoring NIC is locked down to the manager only, via a cluster-level security group bound to each VM's `vmbr4` NIC (the `netX` index varies per VM — `net2` on VM 101).

| Level | Setting | Effect |
|---|---|---|
| Datacenter | `enable: 1`, `policy_in: ACCEPT` | Firewall on; inbound default accept for the CTF paths |
| Per-VM | `policy_out: DROP` | Outbound default drop on the VM |
| Group `wazuh-only` on the `vmbr4` NIC | see below | Only agent↔manager traffic allowed on the monitoring NIC |

### Cluster config (`/etc/pve/firewall/cluster.fw`)
An IPSET holds the manager's IP so the group reads cleanly:

```ini
[OPTIONS]

policy_in: ACCEPT
enable: 1

[IPSET wazuh-laptop] # The external laptop containing wazuh

192.168.0.67 # The Wazuh laptop's IP

[group wazuh-only]

OUT ACCEPT -dest +dc/wazuh-laptop -p udp -dport 1514:1515 -log nolog
IN ACCEPT -source +dc/wazuh-laptop -p tcp -log nolog
OUT ACCEPT -dest +dc/wazuh-laptop -p tcp -dport 1514:1515 -log nolog
```

### Per-VM config (e.g. `/etc/pve/firewall/101.fw`)
Each monitored VM applies the group to its `vmbr4` NIC (`net2` here on VM 101 — use that VM's actual index) and drops everything else outbound:

```ini
[OPTIONS]

enable: 1
log_level_in: info
policy_out: DROP
log_level_out: info

[RULES]

GROUP wazuh-only -i net2
```

Anything else leaving that NIC hits the `policy_out: DROP` and is logged to `pve-firewall.log`, which the `proxmox-host` agent ships to Wazuh (rules `100100`–`100102`).

---

## 6. Verify the path

On the Proxmox host:

```bash
sysctl net.ipv4.ip_forward                            # expect 1
tcpdump -ni vmbr4 'tcp port 1514 or tcp port 1515'    # agent -> host
tcpdump -ni vmbr1 host 192.168.0.67 and 'tcp port 1514'
#   Option A: source stays 192.168.2.x    Option B: source is 192.168.0.137
# Option B only:
iptables -t nat -L POSTROUTING -n -v                  # MASQUERADE counters rise
```

On the manager:

```bash
ip route get 192.168.2.153          # Option A: via 192.168.0.137
sudo /var/ossec/bin/agent_control -l # vmbr4 agents should read Active
```

---

## Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| Agents send but never connect; no NAT in use | Manager has no return route to `192.168.2.0/24` | Add the Option A route on the manager (section 2), or switch to Option B masquerade |
| Works until the manager reboots | Option A route was runtime-only (`ip route add`) | Make it persistent in the manager's netplan |
| Intermittent connectivity on dual-homed Windows VMs | Windows used the other NIC's gateway (lower metric) as default route, plus a stale persistent route | Remove the gateway from the secondary adapter and delete the stale route (`route print`, `route delete`) |
| Re-enrollment fails with duplicate agent name | Manager rejects duplicate names; a stale enrollment exists | `/var/ossec/bin/manage_agents -r <id>` then re-enroll |
| NAT counters rise but agent never connects (Option B) | Return traffic dropped by firewall conntrack | Ensure the `raw` `CT --zone 1` rule is present (section 2, Option B) |
| Agent traffic blocked on `net2` | `wazuh-only` group not applied, or on the wrong NIC index | Confirm `GROUP wazuh-only -i net2` matches the VM's actual vmbr4 NIC |
