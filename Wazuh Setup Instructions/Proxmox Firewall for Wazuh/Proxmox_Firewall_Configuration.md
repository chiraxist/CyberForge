# Proxmox Firewall Configuration

## Goal
Document how the Proxmox firewall is configured for the CyberForge lab: the datacenter-level policy and security group that lock the Wazuh monitoring plane (`vmbr4`) down to the manager only, and the per-VM policy that enforces it. The same firewall produces the `pve-firewall.log` that the `proxmox-host` Wazuh agent ships, feeding detection rules `100100`–`100102`.

| Fact | Value |
|---|---|
| Datacenter config | `/etc/pve/firewall/cluster.fw` |
| Per-VM config | `/etc/pve/firewall/<VMID>.fw` (e.g. `101.fw`) |
| Host log | `/var/log/pve-firewall.log` |
| Manager (allowed peer) | `192.168.0.67` (IPSET `wazuh-laptop`) |
| Monitoring NIC on each VM | the VM's NIC bridged to `vmbr4` — index **varies per VM** (`net0`/`net1`/`net2`/…) |
| Allowed ports | `1514`/`1515` TCP + UDP (agent ↔ manager) |

> **Design intent:** the CTF red-team scenarios need open movement across the lab networks, so the **datacenter inbound policy stays ACCEPT**. Security is applied narrowly instead — each VM drops outbound by default and only the `vmbr4` monitoring NIC is allowed to reach the Wazuh manager, via a reusable security group. This keeps the monitoring plane isolated (an agent can talk to the manager and nothing else on `vmbr4`) without interfering with the attack paths.

---

## 1.0 How the Proxmox firewall is layered

The PVE firewall is enabled and evaluated at three levels, all of which must be on for VM rules to take effect:

| Level | File | Role here |
|---|---|---|
| Datacenter (cluster) | `cluster.fw` | Master enable, default policies, shared IPSETs and security groups |
| Node (host) | `host.fw` | Host firewall; generates `/var/log/pve-firewall.log` |
| VM | `<VMID>.fw` | Per-VM policy and the rules/groups bound to each vNIC |

Additionally, the **firewall checkbox must be enabled on the VM's `vmbr4` vNIC** (Hardware → Network Device → Firewall), or none of the VM-level rules apply to that interface.

> **Which NIC is the `vmbr4` one?** The Proxmox NIC index (`net0`, `net1`, `net2`, …) is **not the same on every VM** — it depends on the order the NICs were added. Open **Hardware** and find the Network Device whose **Bridge** is `vmbr4`; its name (e.g. `net2` on VM 101, but `net0`/`net1`/`net3` on others) is the index you use everywhere below. Do not assume `net2`.

---

## 2.0 Datacenter configuration (`cluster.fw`)

This enables the cluster firewall, keeps inbound open for the CTF, defines an IPSET for the Wazuh manager, and declares the reusable `wazuh-only` security group.

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

What each part does:
- **`enable: 1`** turns the cluster firewall on. **`policy_in: ACCEPT`** leaves the default inbound policy open so the red-team scenarios can move between segments.
- **`[IPSET wazuh-laptop]`** names the manager's IP (`192.168.0.67`) once. Rules reference it as `+dc/wazuh-laptop`, so if the manager ever moves you change it in one place.
- **`[group wazuh-only]`** is a security group (a named, reusable rule set) that permits only agent ↔ manager traffic:
  - `OUT` to the manager on `1514:1515` over **UDP** and **TCP** (events + enrollment).
  - `IN` from the manager over TCP (manager-initiated / return traffic).
  - `-log nolog` keeps these expected flows out of the firewall log so only anomalies are recorded.

---

## 3.0 Per-VM configuration (`<VMID>.fw`)

Every monitored VM uses the same pattern. Example from VM `101`:

```ini
[OPTIONS]

enable: 1
log_level_in: info
policy_out: DROP
log_level_out: info

[RULES]

GROUP wazuh-only -i net2
```

> **`net2` is specific to VM 101.** On this VM the `vmbr4` NIC happens to be `net2`. On other VMs it will be a different index — set `-i netX` to whatever that VM's `vmbr4` NIC actually is (see section 1.0). Binding the group to the wrong interface is the most common reason an agent goes silent.

What each part does:
- **`enable: 1`** turns the firewall on for this VM.
- **`policy_out: DROP`** makes outbound **default-deny** for the VM. Combined with the VM firewall's default inbound DROP, the VM is closed by default in both directions.
- **`GROUP wazuh-only -i netX`** applies the datacenter security group, scoped to the VM's `vmbr4` interface (`net2` on VM 101; **use the correct index for each VM**). This is the only thing that re-opens traffic — and only to the Wazuh manager.
- **`log_level_in/out: info`** logs dropped/accepted decisions at info level to `pve-firewall.log`.

Net effect: on the `vmbr4` NIC, the VM can reach `192.168.0.67:1514-1515` and nothing else; anything else it tries to send out is dropped and logged.

Apply this to each monitored VM (`100.fw`–`107.fw`), changing both the VMID in the filename **and** the `-i netX` index to match that VM's `vmbr4` NIC.

---

## 4.0 Applying the configuration

### Via the GUI
1. **Datacenter → Firewall → Options** → set *Firewall: Yes*.
2. **Datacenter → Firewall → IPSet** → create `wazuh-laptop`, add `192.168.0.67`.
3. **Datacenter → Firewall → Security Group** → create `wazuh-only`, add the three rules from section 2.
4. On each VM: **Firewall → Options** → *Firewall: Yes*, *Output Policy: DROP*.
5. On each VM: **Firewall → Add → Security Group** → select `wazuh-only`, interface = that VM's `vmbr4` NIC (`netX`).
6. On each VM: **Hardware → Network Device** (the one bridged to `vmbr4`) **→ Edit** → tick *Firewall*.

### Via the CLI
The `.fw` files can be edited directly; changes are picked up by `pve-firewall` without a reboot:

```bash
nano /etc/pve/firewall/cluster.fw     # datacenter: IPSET + group
nano /etc/pve/firewall/101.fw         # per-VM policy + group binding
pve-firewall compile                  # check the ruleset compiles
pve-firewall restart
```

---

## 5.0 How this feeds Wazuh

The host firewall writes every logged decision to `/var/log/pve-firewall.log`. The `proxmox-host` Wazuh agent ships that file (see the agent's `localfile` block), where the custom `pvefw` decoder parses it and these rules fire:

| Rule | Level | Fires on |
|---|---|---|
| 100100 | 3 | any decoded Proxmox firewall event |
| 100101 | 5 | a `DROP`/`REJECT` — `$(srcip)` → `$(dstip):$(dstport)` |
| 100102 | 10 | 8+ drops from the same source in 60 s (possible scan) |

Full decoder and rule XML: [Wazuh Custom Rules.md](../Wazuh%20Setup%20Instructions/Custom%20Detection%20Rules/Wazuh%20Custom%20Rules.md). For how the `vmbr4` agents reach the manager in the first place, see [Wazuh Agent NAT vmbr4 to vmbr1.md](../Wazuh%20Setup%20Instructions/NAT%20Rules/Wazuh%20Agent%20NAT%20vmbr4%20to%20vmbr1.md).

Because the `wazuh-only` rules use `-log nolog`, normal agent traffic is silent; what reaches `pve-firewall.log` (and therefore Wazuh) is the *blocked* traffic — exactly the anomalies worth alerting on.

---

## 6.0 Verify

```bash
# the compiled, live ruleset for a VM
pve-firewall compile

# watch drops as they happen
tail -f /var/log/pve-firewall.log

# from a monitored VM: the manager is reachable on the Wazuh ports...
nc -vz 192.168.0.67 1514
# ...but other outbound destinations on vmbr4 are dropped
nc -vz 192.168.0.67 22        # should fail / time out
```

---

## Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| VM rules have no effect | Firewall not enabled on the `vmbr4` vNIC | Tick *Firewall* on the VM's network device that is bridged to `vmbr4` |
| Agent can't reach the manager after enabling the firewall | `policy_out: DROP` with the group bound to the wrong interface | Check Hardware for which `netX` is on `vmbr4` on *that* VM, and make the rule read `GROUP wazuh-only -i netX` to match |
| No Proxmox firewall events in Wazuh | Host firewall off, or `pve-firewall.log` not shipped | Enable the node firewall; confirm the `proxmox-host` agent has the `pve-firewall.log` localfile |
| Log flooded with normal agent traffic | `-log nolog` missing on the group rules | Re-add `-log nolog` to the `wazuh-only` rules (section 2) |
| Rules vanish after reboot | `host.fw`/`cluster.fw` edited while firewall service was confused by host networking | See the Wi-Fi note in the NAT doc — keep host networking on `vmbr1` so `pve-firewall` survives reboot |
