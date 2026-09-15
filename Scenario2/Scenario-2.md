# Scenario 2
Scenario 2 entailed the red teaming in the Cyberforge CTF

## Scenario 2 contains the following methodology:
  1.	Hacker finds WiFi Network “FT_Smart_Factort”
  2.	Hacker uses WIFITE to crack password
  3.	Hacker authenticates onto the Vmbr1 network
  4.	Hacker does recon and discovers devices on network
  5.	Hacker discovers Windows Server 2012 AD Server
  6.	Hacker exploits SMB Eternal Blue CVE to dump credentials
  7.	Hacker authenticates as “Admin”, allowing for access to Vmbr2
  8.	Hacker authenticates into jumphost
  9.	Hacker accesses OT
  10.	 Hacker completes CTF

## Tools and Resources used:
  1. AirCrack
  2. Nmap
  3. Metasploit Framework
  4. Meterpreter
  5. Hashcat
  6. RockYou Wordlist

## WiFi Cracking

## Reconnaissance
After gaining access to the Vmbr1 we perform the following reconnaissance
```bash
sudo nmap -sS -O -p- 192.168.0.0/24
```
This was to scan the network for hosts and to see any open ports and the operating systems.

<img width="891" height="1245" alt="Screenshot From 2026-09-15 17-31-01" src="https://github.com/user-attachments/assets/eab2ca18-5a5b-4254-a6d2-d49521f7c446" />
