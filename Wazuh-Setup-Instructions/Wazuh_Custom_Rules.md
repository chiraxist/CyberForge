# Wazuh Custom Rules (`local_rules.xml`)

## Goal
Document every custom detection rule built for the CyberForge lab, where it lives, what data source it depends on, and how to add it to a fresh Wazuh manager. These rules turn raw agent telemetry (Sysmon, the Proxmox firewall log, xrdp, sshd) into the alerts that drive the per-VM and CTF dashboards. Install the manager first (`Wazuh_Manager_Install.md`) and have agents reporting (`Windows_Staff_Install.md`, `Ubuntu_OS-Ticket_Install.md`) before loading these.

| Fact | Value |
|---|---|
| Rules file | `/var/ossec/etc/rules/local_rules.xml` |
| Custom decoders file | `/var/ossec/etc/decoders/local_decoder.xml` |
| Custom ID range | `100000`–`100299` (plus one override of built-in `86003`) |
| Validate command | `/var/ossec/bin/wazuh-logtest` |
| Reload command | `systemctl restart wazuh-manager` |

> **Where these numbers come from:** the rule IDs, levels, descriptions and groups in this document are the catalogue currently loaded on the manager (`acdc-blue01`, Wazuh 4.14.7). The XML blocks below show the **structure** of each rule group so a new build can recreate them — match them against your live `local_rules.xml` before relying on exact field names and regex, since decoders and log formats vary per build.

---

## 1.0 Prerequisites: the data sources the rules match on

A rule only fires if the event reaches the manager **and** is decoded into the fields the rule references. Set these up first, or the rules stay silent.

### 1.1 Sysmon on Windows (Staff1, Staff2, AD)
The Windows pivot rules match `win.eventdata.*` and `win.system.*` fields from Sysmon's event channel. Install Sysmon with a config, then tell the agent to forward the channel.

Install Sysmon (elevated PowerShell on each Windows VM):
```powershell
.\Sysmon64.exe -accepteula -i sysmonconfig.xml
```

Add the event channel to the agent's `ossec.conf` (`C:\Program Files (x86)\ossec-agent\ossec.conf`), then restart the agent:
```xml
<localfile>
  <location>Microsoft-Windows-Sysmon/Operational</location>
  <log_format>eventchannel</log_format>
</localfile>
```

### 1.2 Sysmon for Linux (Admin, JumpHost, OS-Ticket, Mailcow)
The Linux pivot rules match `sysmon.*` fields produced by **sysmon-for-linux**. Install it, apply a config, and ship its journal to the agent:
```bash
# install sysmon-for-linux from the Microsoft repo, then:
sudo sysmon -accepteula -i /etc/sysmon/config.xml
```
```xml
<localfile>
  <location>/var/log/syslog</location>
  <log_format>syslog</log_format>
</localfile>
```

### 1.3 xrdp on the JumpHost
Rules `100240`/`100241` read xrdp session logs on the Ubuntu jumphost. Ship the xrdp log:
```xml
<localfile>
  <location>/var/log/xrdp-sesman.log</location>
  <log_format>syslog</log_format>
</localfile>
```

### 1.4 Proxmox firewall log
Rules `100100`–`100102` read the host firewall log. The `proxmox-host` agent ships it (see `Wazuh_Agent_NAT_vmbr4_to_vmbr1.md`, section 5):
```xml
<localfile>
  <location>/var/log/pve-firewall.log</location>
  <log_format>syslog</log_format>
</localfile>
```

---

## 2.0 Custom decoders

The Sysmon and Proxmox integrations use Wazuh's shipped decoders, but xrdp session parsing is custom. Decoders live in `/var/ossec/etc/decoders/local_decoder.xml` and must exist **before** the rules that reference their fields.

> 📸 **Screenshot:** the `local_decoder.xml` open in an editor showing the xrdp decoder block. Place it here.

Representative xrdp decoder:
```xml
<decoder name="xrdp-session">
  <program_name>xrdp-sesman</program_name>
</decoder>

<decoder name="xrdp-session-granted">
  <parent>xrdp-session</parent>
  <regex>session reconnection/granted to (\S+) from (\d+.\d+.\d+.\d+)</regex>
  <order>xrdp.user, xrdp.srcip</order>
</decoder>
```

Test a decoder against a sample log line:
```bash
echo '<paste a real xrdp-sesman log line>' | /var/ossec/bin/wazuh-logtest
```

---

## 3.0 Rule catalogue

All custom rules sit in `/var/ossec/etc/rules/local_rules.xml`. They are grouped by purpose and allocated to the ID ranges in section 4.0.

### 3.1 Overrides and noise suppression

| ID | Level | Description | Groups |
|---|---|---|---|
| 86003 | 0 | Docker error messages silenced (Mailcow container noise). Overrides the built-in rule. | `docker`, `docker-error` |
| 100001 | 5 | sshd authentication failure (example rule; PCI DSS 10.2.4, 10.2.5) | `local`, `syslog`, `sshd`, `authentication_failed` |

Silencing a noisy built-in rule is done by re-declaring its ID at level 0 and the `overwrite="yes"` attribute:
```xml
<rule id="86003" level="0" overwrite="yes">
  <if_sid>86003</if_sid>
  <description>Docker: error message (silenced - mailcow noise)</description>
  <group>docker,docker-error,</group>
</rule>
```

### 3.2 Proxmox host firewall (100100–100102)

| ID | Level | Description |
|---|---|---|
| 100100 | 3 | Proxmox firewall event (base rule) |
| 100101 | 5 | Firewall blocked VM `$(vmid)`: `$(srcip)` → `$(dstip):$(dstport)` |
| 100102 | 10 | Repeated firewall drops from `$(srcip)` — possible scan |

Groups: `proxmox`, `firewall`. Structure — a base rule matches the log, a child adds severity, and a frequency rule escalates on repeats:
```xml
<group name="proxmox,firewall,">
  <rule id="100100" level="3">
    <decoded_as>pve-firewall</decoded_as>
    <description>Proxmox firewall event</description>
  </rule>

  <rule id="100101" level="5">
    <if_sid>100100</if_sid>
    <field name="action">DROP|REJECT</field>
    <description>Firewall blocked VM $(vmid): $(srcip) to $(dstip):$(dstport)</description>
  </rule>

  <rule id="100102" level="10" frequency="10" timeframe="60">
    <if_matched_sid>100101</if_matched_sid>
    <same_source_ip />
    <description>Repeated Proxmox firewall drops from $(srcip) (possible scan)</description>
  </rule>
</group>
```

### 3.3 Pivot CTF detections (`pivot_ctf`)

These detect the double-pivot scenario documented in `../pivot-runbook.md`: **external attacker → STAFF1 → ADMIN → RDP jumphost on vmbr2**, run with Metasploit/msfvenom payloads, `autoroute` and `proxychains`.

#### Base / marker rules (level 0)
Level-0 rules match raw events so higher-severity and correlation rules can chain off them without generating alerts of their own.

| ID | Description | Platform |
|---|---|---|
| 100200 | Sysmon network connection | Linux |
| 100201 | Sysmon process creation | Linux |
| 100219 | Pivot stage marker | Linux |
| 100233 | Windows pivot stage marker | Windows |
| 100268 | Internal network connection marker | Both |

#### C2, pivot and lateral movement

| ID | Level | Description | Platform |
|---|---|---|---|
| 100210 | 12 | C2 callback: process connected to external host/port | Linux |
| 100211 | 12 | Pivot into vmbr2 jumphost subnet | Linux |
| 100212 | 10 | Outbound connection to external Kali range | Linux |
| 100213 | 8 | Shell/tool process run as targeted user | Linux |
| 100230 | 12 | C2 callback | Windows |
| 100231 | 10 | Outbound to external Kali range | Windows |
| 100232 | 12 | Pivot into vmbr2 subnet | Windows |
| 100240 | 5 | xrdp session granted | JumpHost |
| 100241 | 12 | RDP pivot landing: login to jumphost from a vmbr2 source | JumpHost |

Representative pivot rule (Linux C2 callback and the vmbr2 pivot that chains off it):
```xml
<group name="pivot_ctf,sysmon,">
  <rule id="100200" level="0">
    <decoded_as>sysmon_event_3</decoded_as>
    <description>Sysmon network connection on $(hostname)</description>
  </rule>

  <rule id="100210" level="12">
    <if_sid>100200</if_sid>
    <field name="sysmon.dstip">^192\.168\.0\.2$</field>
    <description>C2 callback: $(sysmon.image) on $(hostname) connected to $(sysmon.dstip):$(sysmon.dstport)</description>
  </rule>

  <rule id="100211" level="12">
    <if_sid>100200</if_sid>
    <field name="sysmon.dstip">^192\.168\.100\.</field>
    <description>Pivot into vmbr2 jumphost subnet: $(hostname) ($(sysmon.srcip)) connected to $(sysmon.dstip):$(sysmon.dstport)</description>
  </rule>
</group>
```

#### Correlation (multi-stage)

| ID | Level | Description | Platform |
|---|---|---|---|
| 100220 | 14 | Multiple C2 / Kali / vmbr2 events within 10 minutes | Linux |
| 100234 | 14 | Multiple C2 / Kali / vmbr2 events within 10 minutes | Windows |

The level-14 correlation rules are the top of the chain — they fire when several pivot-stage events land inside the timeframe, which is the signal that the full attack chain (not just one stage) is in progress:
```xml
<rule id="100220" level="14" frequency="3" timeframe="600">
  <if_matched_group>pivot_ctf</if_matched_group>
  <same_source_ip />
  <description>Multi-stage pivot: multiple C2/Kali/vmbr2 events within 10 minutes</description>
</rule>
```

#### Initial access

| ID | Level | Description | Platform |
|---|---|---|---|
| 100250 | 10 | Possible phishing payload execution | Windows |
| 100251 | 12 | Office/browser spawned a script interpreter | Windows |
| 100264 | 10 | Payload executed from a download/temp directory | Linux |

#### Attacker tooling and scanning

| ID | Level | Description | Platform |
|---|---|---|---|
| 100260 | 10 | Known attacker tool executed | Linux |
| 100261 | 10 | Suspicious command / reverse-shell pattern | Linux |
| 100262 | 10 | Known attacker tool executed | Windows |
| 100263 | 10 | Suspicious encoded / download command | Windows |
| 100270 | 12 | Possible internal port scan (many internal connections in 60 s) | Linux |
| 100271 | 12 | Possible port scan from Windows host (many connections in 60 s) | Windows |

#### Data hunting and recon

| ID | Level | Description | Platform |
|---|---|---|---|
| 100280 | 8 | Filesystem search for sensitive data | Linux |
| 100281 | 10 | Access to sensitive system/credential file | Linux |
| 100282 | 6 | Host recon/enumeration command | Linux |
| 100283 | 6 | Recon/enumeration command | Windows |
| 100284 | 10 | Sensitive-data search | Windows |

---

## 4.0 ID allocation scheme

Keep new rules inside their band so the file stays readable and dashboards can filter by ID range.

| Range | Purpose |
|---|---|
| 100000–100099 | General / examples |
| 100100–100199 | Proxmox host firewall |
| 100200–100229 | Pivot CTF — Linux Sysmon |
| 100230–100239 | Pivot CTF — Windows Sysmon |
| 100240–100249 | Pivot CTF — xrdp / jumphost |
| 100250–100259 | Initial access |
| 100260–100279 | Attacker tooling and scanning |
| 100280–100299 | Data hunting and recon |

---

## 5.0 Install the rules on a fresh manager

### 5.1 Add the decoders and rules
Edit the two files on the manager:
```bash
sudo nano /var/ossec/etc/decoders/local_decoder.xml
sudo nano /var/ossec/etc/rules/local_rules.xml
```

### 5.2 Validate before restarting
`wazuh-logtest` loads the full ruleset and reports XML or logic errors without taking the manager down. Run it and paste a sample event to confirm the right rule fires:
```bash
sudo /var/ossec/bin/wazuh-logtest
```

> 📸 **Screenshot:** `wazuh-logtest` output showing a pivot event matching its rule ID (e.g. a C2 callback hitting `100210`). Place it here.

### 5.3 Apply
```bash
sudo systemctl restart wazuh-manager
```

### 5.4 Confirm in the dashboard
Trigger a known event (run a recon command on an agent, or replay a pivot step from `../pivot-runbook.md`) and confirm the alert appears under the matching rule ID.

> 📸 **Screenshot:** the Wazuh dashboard **Threat Hunting / Events** view filtered on `rule.id` in the `1001xx` range, showing custom alerts firing. Place it here.

---

## 6.0 Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| Rule never fires though the event arrives | The event isn't decoded into the field the rule references | Run the raw log through `wazuh-logtest`; confirm the decoder populates `sysmon.*` / `xrdp.*` before blaming the rule |
| `wazuh-logtest` reports an XML error on restart | Malformed `local_rules.xml` | Fix the reported line; the manager keeps the last good ruleset until the file parses |
| Alerts fire but at the wrong level | Child rule matched the base rule instead of the specific one | Check `<if_sid>` chaining and field regex order; more specific rules need higher IDs/levels |
| Sysmon fields empty on Linux agents | sysmon-for-linux not shipping, or wrong `localfile` | Confirm the service is running and the agent forwards the right log |
| Flood of low-value Docker alerts from Mailcow | `86003` override missing | Re-add the level-0 override in section 3.1 |
