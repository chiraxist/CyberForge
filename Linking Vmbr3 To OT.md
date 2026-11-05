# Linking Vmbr3 to the OT Network

This documentation explains how the `vmbr3` Linux bridge in Proxmox was connected to the physical OT network and how firewall rules were adjusted to enable communication.

## 1. Creating the Linux Bridge in Proxmox

In the Proxmox web interface, select the node and navigate to **System → Network → Create → Linux Bridge**.

<img width="1101" height="388" alt="Creating a Linux bridge in Proxmox" src="https://github.com/user-attachments/assets/c18b09d1-45ce-4c19-9223-ac48f6f05315" />

Name the bridge `vmbr3` and set its IPv4/CIDR to `192.168.0.1/24`.

<img width="1038" height="529" alt="Configuring vmbr3 with the address 192.168.0.1/24" src="https://github.com/user-attachments/assets/3426a0c1-4851-47ca-a211-36a36d61e562" />

## 2. Physically Connecting Proxmox to the OT Network

The OT devices were connected to an Ethernet switch. An Ethernet cable then connected the switch to the Proxmox host's physical network port.

<img width="600" alt="Physical Ethernet connections for the OT network" src="https://github.com/user-attachments/assets/727e0979-95f8-443d-b16c-1564e3894b9f" />

<img width="600" alt="Ethernet connection between the switch and Proxmox host" src="https://github.com/user-attachments/assets/fe11bb10-1072-4489-b859-7c388955f01f" />

## 3. Adding the Physical Interface to the Linux Bridge

In Proxmox, navigate to **System → Network**, select `vmbr3` and click **Edit**.

Under **Bridge ports**, enter `nic0`, which is the physical interface connected to the OT switch. Save and apply the configuration.

<img width="1042" height="541" alt="Adding nic0 to the bridge ports for vmbr3" src="https://github.com/user-attachments/assets/48e36c07-c6fe-4db3-9a96-579db40a4a89" />

## 4. Connecting the Ubuntu Jumphost

The jumphost's network adapter was attached to `vmbr3` in its Proxmox hardware settings. Inside Ubuntu, this adapter appeared as `ens19` and used `192.168.0.34/24`.

| Component | Configuration |
| --- | --- |
| Linux bridge | `vmbr3` |
| Proxmox bridge address | `192.168.0.1/24` |
| Physical bridge port | `nic0` |
| Jumphost interface | `ens19` |
| Jumphost address | `192.168.0.34/24` |
| Ubuntu test laptop | `192.168.0.35/24` |
| Physical connection | Proxmox → Ethernet switch → OT devices |

## 5. Investigating the Connectivity Issue

After the bridge was connected, the jumphost could discover devices through ARP but could not ping them.

### Checking the Jumphost

The following commands were run on the **Ubuntu jumphost**:

```bash
ip -br addr
ip route
ping -c 4 -W 2 192.168.0.35
ip neigh show 192.168.0.35
sudo ufw status verbose
```

The results showed that:

- `ens19` was up with the expected IP address.
- The target's MAC address appeared in the neighbour table.
- UFW was inactive on the jumphost.
- Ping returned 100% packet loss.

### Checking the Bridge

The following commands were run in the **Proxmox node shell**:

```bash
bridge link show master vmbr3
bridge fdb show br vmbr3
qm config 106
```

These checks confirmed that the jumphost adapter and `nic0` belonged to `vmbr3`. The laptop's MAC address was learned through `nic0`.

### Tracing the Traffic

Packet captures were performed on the bridge and physical interface:

```bash
tcpdump -ni vmbr3 'icmp and host 192.168.0.35'
```

```bash
tcpdump -eni nic0 'icmp and host 192.168.0.35'
```

Each capture was left running while the ping was repeated from the jumphost:

```bash
ping -c 4 -W 2 192.168.0.35
```

ICMP requests appeared on `vmbr3` but did not reach `nic0`, narrowing the issue to forwarding on the Proxmox host.

## 6. Identifying the Firewall Rule

The Proxmox forwarding configuration was inspected:

```bash
iptables -nvL FORWARD --line-numbers
sysctl net.bridge.bridge-nf-call-iptables
```

Bridge netfilter was enabled:

```text
net.bridge.bridge-nf-call-iptables = 1
```

An existing rule dropped forwarded IPv4 traffic entering `vmbr3`:

```text
-A FORWARD -i vmbr3 -j DROP
```

Because bridged IPv4 traffic was being processed by iptables, this rule blocked communication between the jumphost and physical devices. ARP communication continued to work.

Temporary ICMP exceptions between the jumphost and test laptop restored ping replies, confirming the cause.

## 7. Allowing Communication Within Vmbr3

The lab required communication with multiple devices and services. The following command was therefore used in the **Proxmox node shell**:

```bash
iptables -I FORWARD 1 -i vmbr3 -o vmbr3 -j ACCEPT
```

This inserts an ACCEPT rule before the existing DROP rules and allows all IPv4 protocols and addresses for traffic entering and leaving `vmbr3`.

The rule covers communication within the bridge. Devices must still have suitable IP settings, and their own firewalls must permit the required services.

## 8. Making the Rule Persistent

Back up the saved firewall configuration and open it:

```bash
cp -a /etc/iptables/rules.v4 /etc/iptables/rules.v4.before-vmbr3-fix
nano /etc/iptables/rules.v4
```

Inside the `*filter` section, add the following line before the first existing `-A FORWARD` rule:

```text
-A FORWARD -i vmbr3 -o vmbr3 -j ACCEPT
```

Save the file, then validate its syntax:

```bash
iptables-restore --test < /etc/iptables/rules.v4
```

Check that the service responsible for restoring the rules at boot is enabled:

```bash
systemctl is-enabled netfilter-persistent
```

The syntax check should finish without errors, and the service should report `enabled`.

> The `-A FORWARD` line above belongs inside the configuration file. It is not a standalone terminal command. The syntax check validates the file without applying it.

## 9. Verifying Connectivity

From the **Ubuntu jumphost**, test communication with the connected devices:

```bash
ping -c 4 -W 2 192.168.0.35
ping -c 4 -W 2 192.168.0.5
```

On **Proxmox**, inspect the active rule order and packet counters:

```bash
iptables -nvL FORWARD --line-numbers
```

The broad ACCEPT rule should appear before the DROP rules, with both its input and output interfaces set to `vmbr3`.

Successful ping replies from `192.168.0.35` confirmed connectivity with the Ubuntu test laptop. Access to additional OT devices and their required services should be verified separately.
