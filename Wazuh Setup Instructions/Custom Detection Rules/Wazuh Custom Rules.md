# Wazuh Custom Rules (`local_rules.xml`)

## Goal
Document every custom detection rule built for the CyberForge lab — where it lives, what data source it depends on, and how to add it to a fresh Wazuh manager. These rules turn raw agent telemetry (Sysmon, the Proxmox firewall log, xrdp, sshd) into the alerts that drive the per-VM and CTF dashboards. Install the manager first ([Wazuh Manager Install.md](../Wazuh%20Manager%20Setup/Wazuh%20Manager%20Install.md)) and have agents reporting ([Windows_Staff-VM_Install.md](../Wazuh%20Agent%20Setup/Windows_Staff-VM_Install.md), [Ubuntu osTicket-VM Install.md](../Wazuh%20Agent%20Setup/Ubuntu%20osTicket-VM%20Install.md)) before loading these.

| Fact | Value |
|---|---|
| Rules file | `/var/ossec/etc/rules/local_rules.xml` |
| Decoders file | `/var/ossec/etc/decoders/local_decoder.xml` |
| Custom ID range | `100000`–`100299` (plus one override of built-in `86003`) |
| Validate command | `/var/ossec/bin/wazuh-logtest` |
| Reload command | `systemctl restart wazuh-manager` |

The full, verbatim rules and decoders are in section 5 — copy them straight onto a new manager. Sections 1–3 explain what each group detects and what it needs to fire.

### Key lab indicators the rules key off
These are the lab-specific values baked into the rules. Change them here if your addressing differs.

| Indicator | Value | Used by |
|---|---|---|
| C2 handler ports | `4444`, `4445` | 100210, 100230 |
| vmbr2 jumphost subnet | `192.168.100.0/24` | 100211, 100232, 100241 |
| External Kali range | `192.168.200.0/24` | 100212, 100231 |
| Targeted lab users | `administrator`, `jsmith` | 100213 |

---

## 1.0 Prerequisites: the data sources the rules match on

A rule only fires if the event reaches the manager **and** is decoded into the fields the rule references. Set these up first, or the rules stay silent. Note the split: **Windows** pivot rules chain off Wazuh's *built-in* Sysmon groups (`sysmon_event1` / `sysmon_event3`), while **Linux** pivot rules use the *custom* decoders in section 5.1.

### 1.1 Sysmon on Windows (Staff1, Staff2, AD)
Windows rules use `<if_group>sysmon_event1</if_group>` (process creation) and `sysmon_event3` (network connect), which Wazuh produces automatically from the Sysmon event channel. Install Sysmon, then forward its channel from the agent.

```powershell
.\Sysmon64.exe -accepteula -i sysmonconfig.xml
```
Add to the agent's `ossec.conf` (`C:\Program Files (x86)\ossec-agent\ossec.conf`) and restart the agent:
```xml
<localfile>
  <location>Microsoft-Windows-Sysmon/Operational</location>
  <log_format>eventchannel</log_format>
</localfile>
```

### 1.2 Sysmon for Linux (Admin, JumpHost, OS-Ticket, Mailcow)
Linux rules match `sysmon.*` fields produced by the **custom** decoders `sysmon-net` / `sysmon-proc` (section 5.1), which parse the raw Sysmon XML that **sysmon-for-linux** emits (program name `sysmon`, EventID 3 = network, EventID 1 = process). The agent picks this up from the journal, so a `journald` localfile is enough:
```xml
<localfile>
  <log_format>journald</log_format>
  <location>journald</location>
</localfile>
```

### 1.3 xrdp on the JumpHost
Rules `100240`/`100241` read xrdp session grants via the `xrdp-grant` decoder (section 5.1), which matches the `xrdp-sesman` "created session (access granted)" line. Ship the sesman log from the jumphost agent:
```xml
<localfile>
  <location>/var/log/xrdp-sesman.log</location>
  <log_format>syslog</log_format>
</localfile>
```

### 1.4 Proxmox firewall log
Rules `100100`–`100102` decode the host firewall log with the `pvefw` decoder (section 5.1). The `proxmox-host` agent ships it (see [Wazuh Agent NAT vmbr4 to vmbr1.md](../NAT%20Rules/Wazuh%20Agent%20NAT%20vmbr4%20to%20vmbr1.md), section 5):
```xml
<localfile>
  <location>/var/log/pve-firewall.log</location>
  <log_format>syslog</log_format>
</localfile>
```

---

## 2.0 Rule catalogue

Grouped by purpose. Every pivot rule carries a MITRE ATT&CK `<mitre>` tag (see section 3.0). The exact XML for all of these is in section 5.2.

### 2.1 Overrides and noise suppression

| ID | Level | Description | Groups |
|---|---|---|---|
| 86003 | 0 | Docker error messages silenced (Mailcow noise). Overrides built-in `86000`. | `docker`, `docker-error` |
| 100001 | 5 | sshd authentication failure from `1.1.1.1` (example; PCI DSS 10.2.4/10.2.5) | `local`, `syslog`, `sshd`, `authentication_failed` |

### 2.2 Proxmox host firewall (100100–100102)

| ID | Level | Description |
|---|---|---|
| 100100 | 3 | Proxmox firewall event (`pvefw` decoder) |
| 100101 | 5 | Firewall `DROP`/`REJECT` on VM `$(vmid)`: `$(srcip)` → `$(dstip):$(dstport)` |
| 100102 | 10 | 8+ drops from the same `$(srcip)` in 60 s — possible scan |

### 2.3 Pivot CTF detections (`pivot_ctf`)

These detect the double-pivot in [pivot-runbook.md](../../pivot-runbook.md): **external attacker → STAFF1 → ADMIN → RDP jumphost on vmbr2**, run with Metasploit/msfvenom, `autoroute` and `proxychains`.

#### Base / marker rules (level 0 — no alert, chain source only)
| ID | Decoder / source | Platform |
|---|---|---|
| 100200 | `sysmon-net` (network connect) | Linux |
| 100201 | `sysmon-proc` (process create) | Linux |
| 100219 | marker off 100210/211/212 | Linux |
| 100233 | marker off 100230/231/232 | Windows |
| 100268 | internal-connection marker off 100200 | Linux |

#### C2, pivot and lateral movement
| ID | Level | Description | Platform |
|---|---|---|---|
| 100210 | 12 | C2 callback to handler port 4444/4445 | Linux |
| 100211 | 12 | Pivot into vmbr2 subnet (`192.168.100.`) | Linux |
| 100212 | 10 | Outbound to external Kali range (`192.168.200.`) | Linux |
| 100213 | 8 | Shell/tool as `administrator`/`jsmith` (desktop noise excluded) | Linux |
| 100230 | 12 | C2 callback to 4444/4445 | Windows |
| 100231 | 10 | Outbound to external Kali range | Windows |
| 100232 | 12 | Pivot into vmbr2 subnet | Windows |
| 100240 | 5 | xrdp session granted | JumpHost |
| 100241 | 12 | RDP pivot landing from a vmbr2 source | JumpHost |

#### Correlation (multi-stage)
| ID | Level | Description | Platform |
|---|---|---|---|
| 100220 | 14 | 3+ pivot-stage events (100219) within 10 min | Linux |
| 100234 | 14 | 3+ pivot-stage events (100233) within 10 min | Windows |

#### Initial access
| ID | Level | Description | Platform |
|---|---|---|---|
| 100250 | 10 | Payload run from Downloads/Temp/Outlook dir | Windows |
| 100251 | 12 | Office/browser spawned a script interpreter | Windows |
| 100264 | 10 | Payload run from a download/temp dir | Linux |

#### Attacker tooling and scanning
| ID | Level | Description | Platform |
|---|---|---|---|
| 100260 | 10 | Known tool (nmap/nc/socat/hydra/…) executed | Linux |
| 100261 | 10 | Reverse-shell / msfvenom / meterpreter pattern | Linux |
| 100262 | 10 | Known tool (nmap/ncat/certutil/…) executed | Windows |
| 100263 | 10 | Encoded / download PowerShell command | Windows |
| 100270 | 12 | 15+ internal connections from one source in 60 s | Linux |
| 100271 | 12 | 15+ connections from one Windows host in 60 s (off built-in 61605) | Windows |

#### Data hunting and recon
| ID | Level | Description | Platform |
|---|---|---|---|
| 100280 | 8 | Filesystem search for pass/secret/flag/cred | Linux |
| 100281 | 10 | Access to `/etc/shadow`, `.ssh/id_*`, creds | Linux |
| 100282 | 6 | Host recon (`whoami`/`id`/`ip a`/…) | Linux |
| 100283 | 6 | Windows recon (`whoami`/`net user`/`systeminfo`/…) | Windows |
| 100284 | 10 | Windows sensitive-data search (`findstr … pass`) | Windows |

---

## 3.0 MITRE ATT&CK coverage

| Technique | Rules |
|---|---|
| T1071 — Application Layer / C2 | 100210, 100212, 100230, 100231 |
| T1090 — Proxy / pivot | 100211, 100220, 100232, 100234, 100241 |
| T1059 — Command & scripting (`.001` PowerShell) | 100213, 100251, 100261, 100263 |
| T1046 — Network service discovery | 100260, 100262, 100270, 100271 |
| T1021.001 — Remote services: RDP | 100240, 100241 |
| T1566.001 — Spearphishing attachment | 100250, 100251, 100264 |
| T1204.002 — User execution: malicious file | 100250, 100264 |
| T1083 — File & directory discovery | 100280, 100284 |
| T1552.001 — Credentials in files | 100281 |
| T1033 / T1016 / T1087 — Owner, network & account discovery | 100282, 100283 |

---

## 4.0 ID allocation scheme

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

## 5.0 The actual files

Copy these verbatim onto a fresh manager. Decoders must be in place before the rules that use their fields.

### 5.1 `/var/ossec/etc/decoders/local_decoder.xml`

```xml
<decoder name="local_decoder_example">
    <program_name>local_decoder_example</program_name>
</decoder>

<decoder name="pvefw">
  <prematch>^\d+ \d+ \S+ \d+/\w+/\d+:</prematch>
</decoder>
<decoder name="pvefw-fields">
  <parent>pvefw</parent>
  <regex type="pcre2">^(\d+) \d+ (\S+) \S+ \S+ (?:policy )?(\w+): .*?SRC=(\S+) DST=(\S+) .*?PROTO=(\w+)(?: SPT=(\d+) DPT=(\d+))?</regex>
  <order>vmid,chain,fwaction,srcip,dstip,protocol,srcport,dstport</order>
</decoder>

<decoder name="sysmon-net">
  <program_name>sysmon</program_name>
  <prematch>EventID>3</prematch>
</decoder>
<decoder name="sysmon-net-fields">
  <parent>sysmon-net</parent>
  <regex type="pcre2">Name="Image"\x3e([^\x3c]+).*?Name="User"\x3e([^\x3c]+).*?Name="SourceIp"\x3e([^\x3c]+).*?Name="SourcePort"\x3e([^\x3c]+).*?Name="DestinationIp"\x3e([^\x3c]+).*?Name="DestinationPort"\x3e([^\x3c]+)</regex>
  <order>sysmon.image,sysmon.user,sysmon.srcip,sysmon.srcport,sysmon.dstip,sysmon.dstport</order>
</decoder>

<decoder name="sysmon-proc">
  <program_name>sysmon</program_name>
  <prematch>EventID>1</prematch>
</decoder>
<decoder name="sysmon-proc-fields">
  <parent>sysmon-proc</parent>
  <regex type="pcre2">Name="Image"\x3e([^\x3c]+).*?Name="CommandLine"\x3e([^\x3c]+).*?Name="User"\x3e([^\x3c]+)</regex>
  <order>sysmon.image,sysmon.commandline,sysmon.user</order>
</decoder>

<decoder name="xrdp-session">
  <program_name>xrdp-sesman</program_name>
</decoder>
<decoder name="xrdp-session-grant">
  <parent>xrdp-session</parent>
  <regex type="pcre2">created session \(access granted\): username (\S+), ip (?:::ffff:)?([\d.]+)</regex>
  <order>xrdp.user,xrdp.srcip</order>
</decoder>
<decoder name="xrdp-grant">
  <prematch>created session \(access granted\)</prematch>
</decoder>
<decoder name="xrdp-grant-fields">
  <parent>xrdp-grant</parent>
  <regex type="pcre2">username (\S+), ip (?:::ffff:)?([\d.]+)</regex>
  <order>xrdp.user,xrdp.srcip</order>
</decoder>
```

### 5.2 `/var/ossec/etc/rules/local_rules.xml`

```xml
<!-- Local rules -->
<!-- Modify it at your will. -->
<!-- Copyright (C) 2015, Wazuh Inc. -->

<!-- Example -->
<group name="local,syslog,sshd,">
  <rule id="100001" level="5">
    <if_sid>5716</if_sid>
    <srcip>1.1.1.1</srcip>
    <description>sshd: authentication failed from IP 1.1.1.1.</description>
    <group>authentication_failed,pci_dss_10.2.4,pci_dss_10.2.5,</group>
  </rule>
</group>

<group name="proxmox,firewall,">
  <rule id="100100" level="3">
    <decoded_as>pvefw</decoded_as>
    <description>Proxmox firewall event</description>
  </rule>
  <rule id="100101" level="5">
    <if_sid>100100</if_sid>
    <field name="fwaction">DROP|REJECT</field>
    <description>Proxmox firewall blocked VM $(vmid): $(srcip) to $(dstip):$(dstport)</description>
  </rule>
  <rule id="100102" level="10" frequency="8" timeframe="60">
    <if_matched_sid>100101</if_matched_sid>
    <same_field>srcip</same_field>
    <description>Repeated Proxmox firewall drops from $(srcip) (possible scan)</description>
  </rule>
</group>

<group name="pivot_ctf,sysmon,">
  <rule id="100200" level="0">
    <decoded_as>sysmon-net</decoded_as>
    <description>Sysmon network connection on $(hostname)</description>
  </rule>
  <rule id="100201" level="0">
    <decoded_as>sysmon-proc</decoded_as>
    <description>Sysmon process creation on $(hostname)</description>
  </rule>
  <rule id="100210" level="12">
    <if_sid>100200</if_sid>
    <field name="sysmon.dstport">^4444$|^4445$</field>
    <description>C2 callback: $(sysmon.image) on $(hostname) connected to $(sysmon.dstip):$(sysmon.dstport)</description>
    <mitre><id>T1071</id></mitre>
  </rule>
  <rule id="100211" level="12">
    <if_sid>100200</if_sid>
    <field name="sysmon.dstip">^192\.168\.100\.</field>
    <description>Pivot into vmbr2 jumphost subnet: $(hostname) ($(sysmon.srcip)) connected to $(sysmon.dstip):$(sysmon.dstport)</description>
    <mitre><id>T1090</id></mitre>
  </rule>
  <rule id="100212" level="10">
    <if_sid>100200</if_sid>
    <field name="sysmon.dstip">^192\.168\.200\.</field>
    <description>Outbound connection to external Kali range: $(hostname) to $(sysmon.dstip):$(sysmon.dstport)</description>
    <mitre><id>T1071</id></mitre>
  </rule>
  <rule id="100213" level="8">
    <if_sid>100201</if_sid>
    <field name="sysmon.image">bash|/sh$|dash|/nc$|ncat|socat|python</field>
    <field name="sysmon.user">administrator|jsmith</field>
    <field name="sysmon.commandline" negate="yes" type="pcre2">(?i)(xdg-|lesspipe|/usr/bin/which\b|Xwayland|ibus-|dbus-|gvfs|gnome-|update-notifier)</field>
    <description>Shell/tool process as targeted user on $(hostname): $(sysmon.commandline)</description>
    <mitre><id>T1059</id></mitre>
  </rule>
  <rule id="100219" level="0">
    <if_sid>100210,100211,100212</if_sid>
    <description>Pivot stage marker</description>
  </rule>
  <rule id="100220" level="14" frequency="3" timeframe="600">
    <if_matched_sid>100219</if_matched_sid>
    <description>Multi-stage pivot: multiple C2/Kali/vmbr2 events within 10 minutes</description>
    <mitre><id>T1090</id></mitre>
  </rule>
</group>

<group name="pivot_ctf,sysmon,windows,">
  <rule id="100230" level="12">
    <if_group>sysmon_event3</if_group>
    <field name="win.eventdata.destinationPort">^4444$|^4445$</field>
    <description>C2 callback (Windows): $(win.eventdata.image) connected to $(win.eventdata.destinationIp):$(win.eventdata.destinationPort)</description>
    <mitre><id>T1071</id></mitre>
  </rule>
  <rule id="100231" level="10">
    <if_group>sysmon_event3</if_group>
    <field name="win.eventdata.destinationIp">^192\.168\.200\.</field>
    <description>Outbound to external Kali range (Windows): $(win.eventdata.image) to $(win.eventdata.destinationIp):$(win.eventdata.destinationPort)</description>
    <mitre><id>T1071</id></mitre>
  </rule>
  <rule id="100232" level="12">
    <if_group>sysmon_event3</if_group>
    <field name="win.eventdata.destinationIp">^192\.168\.100\.</field>
    <description>Pivot into vmbr2 subnet (Windows): $(win.eventdata.image) to $(win.eventdata.destinationIp)</description>
    <mitre><id>T1090</id></mitre>
  </rule>
  <rule id="100233" level="0">
    <if_sid>100230,100231,100232</if_sid>
    <description>Windows pivot stage marker</description>
  </rule>
  <rule id="100234" level="14" frequency="3" timeframe="600">
    <if_matched_sid>100233</if_matched_sid>
    <description>Multi-stage pivot (Windows): multiple C2/Kali/vmbr2 events within 10 minutes</description>
    <mitre><id>T1090</id></mitre>
  </rule>
</group>

<group name="pivot_ctf,xrdp,">
  <rule id="100240" level="5">
    <decoded_as>xrdp-grant</decoded_as>
    <description>xrdp session granted for user $(xrdp.user) from $(xrdp.srcip)</description>
    <mitre><id>T1021.001</id></mitre>
  </rule>
  <rule id="100241" level="12">
    <if_sid>100240</if_sid>
    <field name="xrdp.srcip">^192\.168\.100\.</field>
    <description>RDP pivot landing: $(xrdp.user) logged into jumphost from vmbr2 source $(xrdp.srcip)</description>
    <mitre><id>T1021.001</id><id>T1090</id></mitre>
  </rule>
</group>

<group name="pivot_ctf,sysmon,windows,initial_access,">
  <rule id="100250" level="10">
    <if_group>sysmon_event1</if_group>
    <field name="win.eventdata.image" type="pcre2">(?i)\\(Downloads|Temp|AppData\\\\Local\\\\Temp|Outlook)\\\\[^\\\\]+\.(exe|scr|bat|cmd|ps1|hta|js|vbs)$</field>
    <description>Possible phishing payload execution on $(win.system.computer): $(win.eventdata.image)</description>
    <mitre><id>T1566.001</id><id>T1204.002</id></mitre>
  </rule>
  <rule id="100251" level="12">
    <if_group>sysmon_event1</if_group>
    <field name="win.eventdata.parentImage" type="pcre2">(?i)(OUTLOOK|winword|excel|powerpnt|chrome|firefox|msedge)\.exe$</field>
    <field name="win.eventdata.image" type="pcre2">(?i)\\(cmd|powershell|wscript|cscript|mshta|rundll32|regsvr32)\.exe$</field>
    <description>Office/browser spawned a script interpreter on $(win.system.computer) - likely phishing execution: $(win.eventdata.image)</description>
    <mitre><id>T1566.001</id><id>T1059</id></mitre>
  </rule>
</group>

<group name="pivot_ctf,attacker_tooling,">
  <rule id="100260" level="10">
    <if_sid>100201</if_sid>
    <field name="sysmon.image" type="pcre2">(?i)/(nmap|masscan|nc|ncat|socat|hydra|nikto|sqlmap)$</field>
    <description>Attacker tool executed on $(hostname): $(sysmon.commandline)</description>
    <mitre><id>T1046</id></mitre>
  </rule>
  <rule id="100261" level="10">
    <if_sid>100201</if_sid>
    <field name="sysmon.commandline" type="pcre2">(?i)(python[23]?\s+-c|base64\s+-d|/dev/tcp/|bash\s+-i|msfvenom|meterpreter)</field>
    <description>Suspicious command/reverse-shell pattern on $(hostname): $(sysmon.commandline)</description>
    <mitre><id>T1059</id></mitre>
  </rule>
  <rule id="100262" level="10">
    <if_group>sysmon_event1</if_group>
    <field name="win.eventdata.image" type="pcre2">(?i)\\(nmap|ncat|nc|powercat|certutil|bitsadmin)\.exe$</field>
    <description>Attacker tool executed on $(win.system.computer): $(win.eventdata.image)</description>
    <mitre><id>T1046</id></mitre>
  </rule>
  <rule id="100263" level="10">
    <if_group>sysmon_event1</if_group>
    <field name="win.eventdata.commandLine" type="pcre2">(?i)(-enc\s|-encodedcommand|downloadstring|frombase64string|iex\s|invoke-expression|certutil.*-urlcache)</field>
    <description>Suspicious encoded/download command on $(win.system.computer): $(win.eventdata.commandLine)</description>
    <mitre><id>T1059.001</id></mitre>
  </rule>
  <rule id="100268" level="0">
    <if_sid>100200</if_sid>
    <field name="data.dstip">^192\.168\.</field>
    <description>Internal network connection marker</description>
  </rule>
  <rule id="100270" level="12" frequency="15" timeframe="60">
    <if_matched_sid>100268</if_matched_sid>
    <same_field>data.srcip</same_field>
    <description>Possible internal port scan from $(data.srcip) - many internal connections in 60s</description>
    <mitre><id>T1046</id></mitre>
  </rule>
  <rule id="100271" level="12" frequency="15" timeframe="60">
    <if_matched_sid>61605</if_matched_sid>
    <same_field>win.eventdata.srcip</same_field>
    <description>Possible port scan from Windows host - many connections in 60s</description>
    <mitre><id>T1046</id></mitre>
  </rule>
</group>

<group name="pivot_ctf,data_hunting,">
  <rule id="100280" level="8">
    <if_sid>100201</if_sid>
    <field name="sysmon.commandline" type="pcre2">(?i)(grep\s+-r.*(pass|secret|flag|cred|key)|find\b.*-name\b.*(pass|flag|secret|cred|key|shadow|id_rsa)|locate\b.*(pass|flag|secret|cred))</field>
    <description>Filesystem search for sensitive data on $(hostname): $(sysmon.commandline)</description>
    <mitre><id>T1083</id></mitre>
  </rule>
  <rule id="100281" level="10">
    <if_sid>100201</if_sid>
    <field name="sysmon.commandline" type="pcre2">(?i)(/etc/shadow|/etc/passwd|\.ssh/id_|\.bash_history|/\.aws/|/\.mozilla/)</field>
    <description>Access to sensitive system/credential file on $(hostname): $(sysmon.commandline)</description>
    <mitre><id>T1552.001</id></mitre>
  </rule>
  <rule id="100282" level="6">
    <if_sid>100201</if_sid>
    <field name="sysmon.commandline" type="pcre2">(?i)^(whoami|id|hostname|uname|ifconfig|ip\s+a|arp|netstat|ss\s)</field>
    <description>Host recon/enumeration command on $(hostname): $(sysmon.commandline)</description>
    <mitre><id>T1033</id><id>T1016</id></mitre>
  </rule>
  <rule id="100283" level="6">
    <if_group>sysmon_event1</if_group>
    <field name="win.eventdata.commandLine" type="pcre2">(?i)(whoami|net\s+user|net\s+group|net\s+localgroup|ipconfig|arp\s+-a|systeminfo|net\s+view)</field>
    <description>Windows recon/enumeration command on $(win.system.computer): $(win.eventdata.commandLine)</description>
    <mitre><id>T1033</id><id>T1087</id></mitre>
  </rule>
  <rule id="100284" level="10">
    <if_group>sysmon_event1</if_group>
    <field name="win.eventdata.commandLine" type="pcre2">(?i)(findstr.*(pass|flag)|dir\s+/s.*(pass|flag)|reg\s+query.*password|where\s+/r)</field>
    <description>Windows sensitive-data search on $(win.system.computer): $(win.eventdata.commandLine)</description>
    <mitre><id>T1083</id></mitre>
  </rule>
</group>

<group name="pivot_ctf,initial_access,">
  <rule id="100264" level="10">
    <if_sid>100201</if_sid>
    <field name="sysmon.image" type="pcre2">^/(home/[^/]+/Downloads|tmp|var/tmp|dev/shm)/</field>
    <description>Payload executed from download/temp directory on $(hostname): $(sysmon.commandline)</description>
    <mitre><id>T1204.002</id><id>T1566.001</id></mitre>
  </rule>
</group>

<group name="docker,">
  <rule id="86003" level="0" overwrite="yes">
    <if_sid>86000</if_sid>
    <field name="docker.level">error</field>
    <description>Docker: Error message (silenced - mailcow noise)</description>
    <group>docker-error,</group>
  </rule>
</group>
```

---

## 6.0 Install the rules on a fresh manager

### 6.1 Add the decoders and rules
```bash
sudo nano /var/ossec/etc/decoders/local_decoder.xml
sudo nano /var/ossec/etc/rules/local_rules.xml
```

### 6.2 Validate before restarting
`wazuh-logtest` loads the full ruleset and reports XML or logic errors without taking the manager down. Paste a sample event to confirm the right rule fires:
```bash
sudo /var/ossec/bin/wazuh-logtest
```

> 📸 **Screenshot:** `wazuh-logtest` output showing a pivot event matching its rule ID (e.g. a C2 callback hitting `100210`). Place it here.

### 6.3 Apply
```bash
sudo systemctl restart wazuh-manager
```

### 6.4 Confirm in the dashboard
Trigger a known event (run a recon command on an agent, or replay a pivot step from [pivot-runbook.md](../../pivot-runbook.md)) and confirm the alert appears under the matching rule ID.

> 📸 **Screenshot:** the Wazuh dashboard **Threat Hunting / Events** view filtered on `rule.id` in the `1001xx` range, showing custom alerts firing. Place it here.

---

## 7.0 Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| Rule never fires though the event arrives | The event isn't decoded into the field the rule references | Run the raw log through `wazuh-logtest`; confirm the decoder populates `sysmon.*` / `xrdp.*` / `fwaction` before blaming the rule |
| `wazuh-logtest` reports an XML error on restart | Malformed `local_rules.xml` | Fix the reported line; the manager keeps the last good ruleset until the file parses |
| Windows pivot rules silent but events arrive | Sysmon event channel not forwarded, so no `sysmon_event1/3` group | Confirm the `eventchannel` localfile in the Windows agent's `ossec.conf` (section 1.1) |
| Linux `sysmon.*` fields empty | sysmon-for-linux not running, or journald not forwarded | Confirm `sysmon` is running and the agent has the `journald` localfile (section 1.2) |
| Flood of Docker errors from Mailcow | `86003` override missing | Re-add the level-0 override (end of section 5.2) |
