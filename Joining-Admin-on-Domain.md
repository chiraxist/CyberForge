# Adding the Ubuntu Admin Machine to the Active Directory Domain

## Purpose

The Ubuntu Admin virtual machine was joined to the `cyberforge.com` Active Directory domain so that domain users could sign in using their existing credentials. The machine was accessed through the Proxmox noVNC console.

## Environment

| Setting | Value shown in screenshots |
|---|---|
| Virtual machine | Admin, VM ID 105 |
| Operating system | Ubuntu; package repositories show Jammy (22.04) |
| Local account | `admin-computer` |
| Domain | `cyberforge.com` |
| Kerberos realm | `CYBERFORGE.COM` |
| Domain DNS server | `192.168.0.200` |
| Domain-facing interface | `ens18` |
| Domain join account | `Administrator` |

## 1. Update the package index

Open a terminal while signed in to the local administrator account and run:

```bash
sudo apt update
```

Enter the local account password when prompted. The package index update completed successfully. This refreshes the available package information; it does not upgrade the installed packages.



## 2. Install the domain integration packages

The following installation command was entered:

```bash
sudo apt install realmd sssd-tools adcli libnss-sss libpam-sss samba-common-bin
```

These packages support domain discovery and joining, SSSD identity integration, and domain authentication through the Linux account and login systems. The later successful domain operations demonstrate that the required integration components were available.


## 3. Identify the DNS issue

The initial attempt to discover the domain failed:

```bash
realm discover CYBERFORGE.com
```

The terminal returned:

```text
realm: No such realm found: CYBERFORGE.com
```

The DNS configuration was then inspected:

```bash
resolvectl status
```

Interface `ens18` was using the public DNS servers `1.1.1.1` and `9.9.9.9`. Domain discovery subsequently succeeded after this interface was directed to the internal domain DNS server.


## 4. Configure DNS for domain discovery

Set the DNS server and search domain for the domain-facing interface, then clear the DNS cache:

```bash
sudo resolvectl dns ens18 192.168.0.200
sudo resolvectl domain ens18 CYBERFORGE.com
sudo resolvectl flush-caches
```

Confirm the interface configuration:

```bash
resolvectl status ens18
```

The output showed:

```text
DNS Servers: 192.168.0.200
DNS Domain: CYBERFORGE.com
```

**Configuration note:** These `resolvectl` commands apply runtime settings. The screenshots do not show a persistent DNS configuration. For a repeatable deployment, configure the same DNS server and search domain in the connection's NetworkManager or Netplan settings and verify them after restarting.


## 5. Verify the domain controller DNS record

Query the Active Directory LDAP service record:

```bash
resolvectl query -t SRV _ldap._tcp.dc._msdcs.CYBERFORGE.com
```

The query returned a domain controller hostname under `cyberforge.com` and LDAP port `389`. This confirmed that the internal DNS server could supply the service record needed to locate a domain controller.



## 6. Discover the domain

Repeat domain discovery:

```bash
realm discover CYBERFORGE.com
```

The output identified the following properties:

```text
type: kerberos
realm-name: CYBERFORGE.COM
server-software: active-directory
client-software: sssd
configured: no
```

The domain was now discoverable. At this stage, `configured: no` indicated that the Ubuntu machine had not yet joined it.


## 7. Join the Ubuntu machine to the domain

Join the domain using the domain Administrator account:

```bash
sudo realm join CYBERFORGE.com -U Administrator
```

Enter the domain Administrator password when prompted. The command returned to the terminal without displaying an error.

Verify membership:

```bash
realm list
```

The output included:

```text
configured: kerberos-member
server-software: active-directory
client-software: sssd
login-formats: %U@cyberforge.com
login-policy: allow-realm-logins
```

The `kerberos-member` status confirmed that the domain join had completed.


## 8. Enable home directory creation and check a domain user

Enable automatic home directory creation for users at login:

```bash
sudo pam-auth-update --enable mkhomedir
```

Check that Ubuntu can resolve a domain account:

```bash
id 'Jsmith@CYBERFORGE.com'
```

The command returned the user's UID, GID and Domain Users membership. This verified domain account lookup; the graphical login was checked separately in the next step.

Restart the machine:

```bash
sudo reboot
```


## 9. Test a domain login

At the Ubuntu sign-in screen, select the option to enter another username and use:

```text
Administrator@cyberforge.com
```

Enter the domain account password. Once the desktop loads, open a terminal and run:

```bash
id
```

The terminal prompt and command output identified the signed-in user as the domain Administrator and displayed domain group memberships, including Domain Users and Domain Admins. This confirmed a successful graphical login using a domain account.


## Additional configuration attempt: local sudo access

The final screenshots show a separate attempt to grant the domain Administrator local sudo privileges:

```bash
sudo visudo -f /etc/sudoers.d/domain-admin
```

The entered rule contained `ALL=(ALL:) ALL`, which produced a syntax error. The error screen also shows that the invalid changes were saved using the `Q` option. These screenshots do not demonstrate working sudo access.

Domain membership and local sudo privileges are separate configurations. If local administrative access is required, correct the rule through `visudo`, using the exact username returned by `id`:

```sudoers
administrator@CYBERFORGE.com ALL=(ALL:ALL) ALL
```

Validate the configuration before relying on it:

```bash
sudo visudo -c
```

Then test from the domain user's session:

```bash
sudo -l
```

If the saved syntax error prevents use of sudo, repair the file from a root console or recovery session. The corrected rule and validation commands above are follow-up guidance; they are not shown as successfully completed in the supplied evidence.


## Recorded outcome

The Ubuntu Admin machine successfully joined `cyberforge.com`, resolved a domain user and accepted a graphical login using the domain Administrator account. The main issue during discovery was resolved by directing `ens18` to the domain DNS server at `192.168.0.200`. Persistent DNS configuration and successful local sudo access are not established by the supplied screenshots.
