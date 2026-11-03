# Routing Wazuh Agents from vmbr4 to the Manager (vmbr1)

This documents how agents on the isolated monitoring bridge `vmbr4` reach the Wazuh manager, which lives on the LAN bridge `vmbr1`. `vmbr4` has no uplink of its own, so the Proxmox host routes and NATs agent traffic from `vmbr4` out through `vmbr1` to the manager at `192.168.0.67`. The manager therefore never needs a return route back into the `vmbr4` subnet.

> **Addressing note:** agents enrolled from `vmbr4` report in the `192.168.2.0/24` range (`.150`–`.156`), and the manager sits at `192.168.0.67` on `vmbr1`. On this build the Proxmox host is `192.168.2.1/24` on `vmbr4`, and that address is the gateway the guest host-routes point at (step 4). Confirm it matches your `/etc/network/interfaces` before applying.

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
| Wazuh manager | 000 | 192.168.0.67 (on vmbr1) |

Each VM has a second NIC (`net1`) on `vmbr4` with a static `/24` address and **no default gateway** — the only route out of `vmbr4` is a host route to the manager (see step 4).

## Ports

| Port | Proto | Purpose |
|---|---|---|
| 1514 | TCP | Agent to manager event channel (`wazuh-remoted`) |
| 1515 | TCP | Agent enrollment (`wazuh-authd`) |

## 1. Enable IP forwarding on the Proxmox host

```bash
# runtime
sysctl -w net.ipv4.ip_forward=1

# persistent
echo 'net.ipv4.ip_forward=1' > /etc/sysctl.d/99-wazuh-forward.conf
sysctl --system
```

## 2. Add the NAT (masquerade) rule

Scope the masquerade tightly — only vmbr4 sources, only to the manager, only the Wazuh ports:

```bash
iptables -t nat -A POSTROUTING \
  -s 192.168.2.0/24 -d 192.168.0.67/32 \
  -o vmbr1 -p tcp -m multiport --dports 1514,1515 \
  -j MASQUERADE
```

Because the Proxmox firewall is enabled, give traffic coming off the firewall bridges its own conntrack zone, otherwise return traffic for NATed flows can be dropped:

```bash
iptables -t raw -I PREROUTING -i fwbr+ -j CT --zone 1
```

## 3. Persist the rules in `/etc/network/interfaces`

Attach the rules to the `vmbr4` stanza so they survive reboots and `ifreload -a`:

You already have the stanza with `ip_forward` enabled and a `#Wazuh connections` marker. Add the NAT and conntrack rules under that marker so the full stanza reads:

```ini
auto vmbr4
iface vmbr4 inet static
        address 192.168.2.1/24
        bridge-ports none
        bridge-stp off
        bridge-fd 0
        post-up   echo 1 > /proc/sys/net/ipv4/ip_forward
        #Wazuh connections
        post-up   iptables -t nat -A POSTROUTING -s 192.168.2.0/24 -d 192.168.0.67/32 -o vmbr1 -p tcp -m multiport --dports 1514,1515 -j MASQUERADE
        post-down iptables -t nat -D POSTROUTING -s 192.168.2.0/24 -d 192.168.0.67/32 -o vmbr1 -p tcp -m multiport --dports 1514,1515 -j MASQUERADE
        post-up   iptables -t raw -I PREROUTING -i fwbr+ -j CT --zone 1
        post-down iptables -t raw -D PREROUTING -i fwbr+ -j CT --zone 1
```

Apply without rebooting: `ifreload -a` (or `ifdown vmbr4 && ifup vmbr4`), then confirm with step 6.

> **Host Wi-Fi note:** the `wlp2s0` / `wpa_supplicant` stanza on the Proxmox host is commented out. With it active, `pve-firewall` and these rules did not reliably survive a reboot or `ifreload -a`. The host reaches the LAN solely via `vmbr1` (`192.168.0.137`).

## 4. Guest-side route to the manager

Since `net1` has no gateway, each VM needs a single host route sending manager traffic through the Proxmox host's `vmbr4` address. This also keeps agent traffic on `vmbr4` even on VMs that also have a NIC on `vmbr1`.

**Windows (AD, Staff1, Staff2)** — elevated prompt:

```powershell
route -p add 192.168.0.67 mask 255.255.255.255 192.168.2.1
```

**Ubuntu (OS-Ticket, Mailcow, Admin, JumpHost)** — netplan, on the `net1` interface:

```yaml
network:
  version: 2
  ethernets:
    <net1-iface>:
      addresses: [192.168.2.15X/24]
      routes:
        - to: 192.168.0.67/32
          via: 192.168.2.1
```

```bash
sudo netplan apply
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

## 5. Proxmox firewall design

| Level | Policy | Why |
|---|---|---|
| Datacenter | Input / Output / Forward = **ACCEPT** | The CTF scenario needs open movement between lab segments. |
| Per-VM | **DROP** in/out, enforced on the **vmbr4 NIC only** | Locks the monitoring plane down without interfering with the lab network paths. |
| Security group `wazuh-only` | Applied to each VM's `net1` | Permits only agent to manager traffic. |

Example `wazuh-only` security group (`/etc/pve/firewall/cluster.fw`):

```ini
[group wazuh-only]
OUT ACCEPT -dest 192.168.0.67 -p tcp -dport 1514:1515 -log nolog
```

Return traffic is allowed by connection tracking, so no matching `IN` rule is needed. Anything else on `vmbr4` hits the VM's DROP policy and is logged to `pve-firewall.log`, which the `proxmox-host` agent ships to Wazuh.

## 6. Verify the path

On the Proxmox host:

```bash
sysctl net.ipv4.ip_forward                            # expect 1
iptables -t nat -L POSTROUTING -n -v --line-numbers   # packet counters should rise
conntrack -L -d 192.168.0.67 2>/dev/null | grep -E '1514|1515'
tcpdump -ni vmbr4 'tcp port 1514 or tcp port 1515'
tcpdump -ni vmbr1 host 192.168.0.67 and 'tcp port 1514'   # source should be the host's vmbr1 IP
```

On the manager:

```bash
/var/ossec/bin/agent_control -l
tail -f /var/ossec/logs/ossec.log
```

## Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| Intermittent connectivity loss on dual-homed Windows VMs | Windows used the other NIC's gateway (lower metric) as the default route for all traffic, plus a stale persistent static route | Remove the gateway from the secondary adapter and delete the stale route (`route print`, `route delete`) |
| Re-enrollment fails with duplicate agent name | Manager rejects duplicate names; a stale/failed enrollment still exists | `/var/ossec/bin/manage_agents -r <id>` then re-enroll |
| `pve-firewall` or NAT rules missing after reboot / `ifreload -a` | `wlp2s0` Wi-Fi config on the host interfering | Comment out the `wlp2s0` stanza; keep host networking on `vmbr1` |
| NAT counters increase but the agent never connects | Return traffic dropped by firewall conntrack | Ensure the `raw` `CT --zone 1` rule is present (step 2) |
