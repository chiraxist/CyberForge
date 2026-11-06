# VM, network and access inventory

Addresses below are lab values from the technical report. Verify them against the restored guest and Proxmox bridge configuration. Service and telemetry addresses serve different purposes.

| VM ID | VM and installed stack | Service IP and bridge | Wazuh agent IP |
| --- | --- | --- | --- |
| 100 | AD Server · Windows Server 2012 family, AD DS and DNS | `192.168.0.200` · vmbr1 | `192.168.2.150` |
| 101 | Staff1 · Windows 10 Pro, domain client | `192.168.0.x` (DHCP) · vmbr1, with a Scenario 1 connection on vmbr5 | `192.168.2.151` |
| 102 | Staff2 · Windows 10 Pro, domain client | `192.168.0.x` (DHCP) · vmbr1 | `192.168.2.152` |
| 103 | Help desk · Ubuntu, osTicket | `192.168.0.11` · vmbr1 | `192.168.2.153` |
| 104 | Mail · Ubuntu, Mailcow | `192.168.0.10` · vmbr1 | `192.168.2.154` |
| 105 | Admin · Ubuntu, domain integration and remote desktop | `192.168.0.102` in the validation run · vmbr1. `192.168.100.2` · vmbr2 | `192.168.2.155` |
| 106 | JumpHost · Ubuntu, OT access host | `192.168.100.3` · vmbr2. `192.168.0.37` · vmbr3 | `192.168.2.156` |
| 108 | Proxmox Backup Server | `192.168.0.207:8007` · vmbr1 in the VM inventory | — |

The report gives different Ubuntu versions for Admin and JumpHost across its VM and telemetry tables. Check `/etc/os-release` on each restored guest. The report lists the Proxmox host as the eighth monitored system. vmbr2 uses the administration range `192.168.100.x`. vmbr3 connects via `nic0` to the physical testbed. vmbr4 carries Wazuh telemetry. vmbr5 represents the isolated external network for Scenario 1. Treat the overlapping `192.168.0.x` addresses on vmbr1 and vmbr3 as belonging to separate Layer 2 segments.

| Login surface | Software / connection | Credential source |
| --- | --- | --- |
| Proxmox | Web UI on TCP 8006 | Facilitator-issued Proxmox account, realm and approved vault entry |
| PBS | Web UI on TCP 8007 | Backup operator account and PBS realm from the facilitator |
| AD and Windows staff guests | VM console or approved remote access | Assigned lab domain account for your role |
| Admin, JumpHost, osTicket and Mailcow guests | VM console or approved remote access | Guest account in the lab vault. Application logins are issued separately |
| Blue endpoint `acdc-blue01` | Physical laptop, Ubuntu 24.04.5 LTS | Blue-team OS account issued for the session |
| Wazuh dashboard | On `acdc-blue01`, Wazuh 4.14.7 manager, indexer and dashboard | Facilitator-provided dashboard URL and Wazuh analyst account. Dashboard and OS logins are separate |
| Red endpoint | Kali Linux on the approved exercise segment | Red-team OS account and network access details issued for the session |

Use the approved lab vault or contact the exercise facilitator for actual usernames, passwords and endpoint URLs. Keep recovered credentials and live flag values in restricted exercise records.
