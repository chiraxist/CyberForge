# Raspberry Pi WiFi-to-Ethernet Router for Proxmox

## Goal

A Raspberry Pi connects to WiFi and routes that connection out its onboard Ethernet port, which is plugged into Proxmox via a USB-to-Ethernet dongle. This gives the Proxmox host itself internet access, and also feeds a bridge (`vmbr4`) that VMs can attach to for their own internet access.

## Architecture

This is NAT/routing, **not** bridging — a WiFi client interface cannot be bridged to Ethernet (WiFi APs won't forward frames for MACs they haven't associated). The Pi acts as a router:

```
[WiFi/Internet] <--wlan0--> [Raspberry Pi] <--eth0--> [USB dongle] <--> [Proxmox vmbr4] <--> [VMs]
```

- **`wlan0`** on the Pi = WAN side (gets internet via WiFi)
- **`eth0`** on the Pi = LAN side, static IP `192.168.100.1/24`, runs a DHCP server (`dnsmasq`)
- The Pi does **NAT (masquerade)** from `eth0`'s subnet out through `wlan0`
- Proxmox's dongle interface is enslaved to bridge `vmbr4`, which gets a DHCP lease from the Pi
- VMs attached to `vmbr4` get their own DHCP leases from the Pi automatically — no extra config needed on them

---

## Part 1: Raspberry Pi Setup

Since we installed the headless version of Raspberry Pi OS, you will need to plug an HDMI cable into a monitor and attach a physical keyboard.

### 1.0 Login

When prompted, log in to the Pi with the username and password set during installation. For this example: `admin` : `admin`.

### 1.1 Connect to WiFi

> **Important:** these steps assume you're connecting to university WiFi via **eduroam**, which requires manual network configuration due to WPA2/WPA3-Enterprise security (username + password authentication).

Edit the `wpa_supplicant` config:

```bash
sudo nano /etc/wpa_supplicant/wpa_supplicant.conf
```

Remove all existing content and replace it with:

```conf
ctrl_interface=/var/run/wpa_supplicant GROUP=netdev
update_config=1
country=AU

network={
    ssid="eduroam"
    key_mgmt=WPA-EAP
    eap=PEAP
    identity="{INSERT VALID ECU EMAIL HERE}"
    password="{INSERT UNIVERSITY PASSWORD HERE}"
    phase2="auth=MSCHAPV2"
}
```

Save and exit: `Ctrl+O` → `Enter` → `Ctrl+X`

Reboot the Pi:

```bash
sudo reboot
```

### 1.2 Give `eth0` a static IP (becomes the gateway for Proxmox)

```bash
sudo nmcli connection add type ethernet ifname eth0 con-name eth0-lan \
  ipv4.method manual ipv4.addresses 192.168.100.1/24
sudo nmcli connection up eth0-lan
```

### 1.3 Enable IP forwarding

```bash
echo 'net.ipv4.ip_forward=1' | sudo tee /etc/sysctl.d/99-ip-forward.conf
sudo sysctl --system
```

Verify anytime with:

```bash
cat /proc/sys/net/ipv4/ip_forward   # should print 1
```

### 1.4 Install and configure `dnsmasq` (DHCP server for the `eth0` side)

```bash
sudo apt install dnsmasq
```

Press `y` + `Enter` to confirm the package install.

Create the config file:

```bash
sudo nano /etc/dnsmasq.d/eth0-lan.conf
```

Contents:

```conf
interface=eth0
dhcp-range=192.168.100.10,192.168.100.200,255.255.255.0,24h
dhcp-option=option:router,192.168.100.1
dhcp-option=option:dns-server,1.1.1.1,8.8.8.8
dhcp-authoritative
bind-interfaces
```

Save and exit: `Ctrl+O` → `Enter` → `Ctrl+X`

Enable the service:

```bash
sudo systemctl enable --now dnsmasq
```

### 1.5 NAT masquerade (nftables)

```bash
sudo nft add table ip nat
sudo nft add chain ip nat postrouting { type nat hook postrouting priority 100 \; }
sudo nft add rule ip nat postrouting oifname "wlan0" masquerade
```

Save it so it persists, and make sure it's actually loaded on boot:

```bash
sudo nft list ruleset | sudo tee /etc/nftables.conf
sudo systemctl enable nftables
sudo systemctl restart nftables
```

Confirm it's running correctly:

```bash
sudo systemctl start nftables
sudo systemctl status nftables --no-pager   # check for errors
cat /etc/nftables.conf                       # confirm the rule is actually saved here
```

> If the file is empty or missing the rule, redo step 1.5 and restart the service.

**After any reboot, always verify:**

```bash
sudo nft list ruleset
```

This should show the masquerade rule. If blank, NAT is not happening and nothing will route through the Pi, even though DHCP/routing may look fine.

### 1.6 Quick health check commands (Pi side)

```bash
ip addr show wlan0                   # confirm WiFi has an IP
ip addr show eth0                    # confirm 192.168.100.1/24
cat /proc/sys/net/ipv4/ip_forward    # should be 1
sudo nft list ruleset                # should show the masquerade rule
sudo systemctl status dnsmasq --no-pager
curl -v http://1.1.1.1 --max-time 5  # Pi's own internet access
```

---

## Part 2: Proxmox Setup

### 2.1 Identify the USB dongle interface

```bash
ip link show
```

USB Ethernet adapters typically show as `enx` + MAC address (e.g. `enxc84d4422f4d8`).

### 2.2 Configure `vmbr4` in `/etc/network/interfaces`

```bash
sudo nano /etc/network/interfaces
```

Modify the configuration to look like this:

```conf
auto lo
iface lo inet loopback

auto nic0
iface nic0 inet manual

iface wlp2s0 inet manual

auto enx7cc2c63ee714
iface enx7cc2c63ee714 inet manual

iface enxeeadb731d300 inet manual

auto enxc84d4422f4d8
iface enxc84d4422f4d8 inet manual

auto vmbr4
iface vmbr4 inet dhcp
        bridge-ports enxc84d4422f4d8
        bridge-stp off
        bridge-fd 0
        up ip route add default via 192.168.100.1 dev vmbr4

auto vmbr1
iface vmbr1 inet static
        address 192.168.0.137/24
        bridge-ports enx7cc2c63ee714
        bridge-stp off
        bridge-fd 0
# OT network

auto vmbr2
iface vmbr2 inet manual
        bridge-ports none
        bridge-stp off
        bridge-fd 0
# IT network

auto vmbr3
iface vmbr3 inet manual
        bridge-ports nic0
        bridge-stp off
        bridge-fd 0
# Admin network

source /etc/network/interfaces.d/*
```

Apply:

```bash
ifreload -a
```

### 2.3 Routing Proxmox's default route through the Pi

If Proxmox's other network (in this case `vmbr1` / `192.168.0.0/24`) has no real WAN uplink, the host needs its default route to go via the Pi instead.

Because `vmbr4` is `inet dhcp`, once it's the only interface offering a default route, `ifupdown2` will install the Pi's DHCP-supplied gateway (`192.168.100.1`) automatically — no static gateway line needed on the `vmbr4` block.

If you want both gateways to coexist (e.g. `vmbr1` kept as a low-priority fallback), use `metric` — **lower number = higher priority / preferred**:

```conf
auto vmbr4
iface vmbr4 inet dhcp
    bridge-ports enxc84d4422f4d8
    bridge-stp off
    bridge-fd 0
    metric 100

auto vmbr1
iface vmbr1 inet static
    address 192.168.0.137/24
    gateway 192.168.0.1
    metric 200
    bridge-ports enx7cc2c63ee714
    bridge-stp off
    bridge-fd 0
```

Apply and verify:

```bash
ifreload -a
ip route show
```

Expect to see `default via 192.168.100.1 dev vmbr4` as the preferred route.

### 2.4 One-off manual route (testing only, does not persist)

```bash
sudo ip route add default via 192.168.100.1 dev vmbr4
```

To persist a route this way instead of via `metric`, add an `up` hook in the interfaces file:

```conf
auto vmbr4
iface vmbr4 inet dhcp
    bridge-ports enxc84d4422f4d8
    bridge-stp off
    bridge-fd 0
    up ip route add default via 192.168.100.1 dev vmbr4
```

---

## Part 3: Reboot Persistence Checklist

Things that are known to not always survive a reboot cleanly on this kind of setup — verify each explicitly after rebooting either device, don't assume.

**On the Pi:**

- [ ] `wlan0` reconnects to WiFi automatically → `ip addr show wlan0`
- [ ] `ip_forward` is still `1` → `cat /proc/sys/net/ipv4/ip_forward`
- [ ] `dnsmasq` is running → `systemctl status dnsmasq`
- [ ] nftables ruleset is actually loaded, not just the service enabled → `sudo nft list ruleset` (must show the masquerade rule, not be blank)

**On Proxmox:**

- [ ] `vmbr4` exists and has a DHCP-assigned IP → `ip addr show vmbr4`
- [ ] Default route via `192.168.100.1 dev vmbr4` is present → `ip route show`
- [ ] Management access (web UI) is still reachable on its own subnet/gateway, if you rely on one
