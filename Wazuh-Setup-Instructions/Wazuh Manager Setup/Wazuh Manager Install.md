# Wazuh Manager (All-in-One) Install

## Goal
Stand up the Wazuh manager, indexer and dashboard as a single all-in-one install on the external Ubuntu laptop. This is the server every agent reports to, and it must be running before you install any agent (see `Windows_Staff_Install.md` and `Ubuntu_OS-Ticket_Install.md`).

| Fact | Value |
|---|---|
| Host name | `acdc-blue01` |
| OS | Ubuntu 24.04 LTS |
| Wazuh version | 4.14.7 |
| Manager IP (on vmbr1 / LAN) | `192.168.0.67` |
| Dashboard | `https://192.168.0.67` |
| Agent event port | `1514/tcp` |
| Agent enrollment port | `1515/tcp` |

## 1.0 Install

### 1.1 Run the all-in-one installer
On the laptop, open a terminal and run the official quick-install script. This deploys the manager, indexer and dashboard together:

```bash
curl -sO https://packages.wazuh.com/4.14/wazuh-install.sh
sudo bash ./wazuh-install.sh -a -i
```

`-a` installs all components on one host; `-i` ignores the hardware-check warning on a laptop.

### 1.2 Record the admin password
When the installer finishes it prints the dashboard URL and the generated `admin` credentials. Note them. You can reprint them later from the install tarball:

```bash
sudo tar -O -xvf wazuh-install-files.tar wazuh-install-files/wazuh-passwords.txt
```

### 1.3 Confirm the services are up
```bash
sudo systemctl status wazuh-manager wazuh-indexer wazuh-dashboard
```
All three should read `active (running)`.

## 2.0 First login

### 2.1 Open the dashboard
Browse to `https://192.168.0.67` from a device that can reach the laptop over vmbr1, and log in as `admin` with the password from step 1.2. Accept the self-signed certificate warning.

### 2.2 Confirm the enrollment service is listening
Agents enroll on 1515 and stream events on 1514. Confirm both are open on the manager:

```bash
sudo ss -lntp | grep -E '1514|1515'
```

You should see `wazuh-remoted` on 1514 and `wazuh-authd` on 1515. If 1515 is missing, enable auto-enrollment in `/var/ossec/etc/ossec.conf` under `<auth>` and restart the manager.

## 3.0 Next steps
- Install agents: `Windows_Staff_Install.md`, `Ubuntu_OS-Ticket_Install.md`.
- If agents sit on the isolated monitoring bridge, set up routing first: `Wazuh_Agent_NAT_vmbr4_to_vmbr1.md`.
- Load the project's detection rules: `Wazuh_Custom_Rules.md`.
