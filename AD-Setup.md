# Active Directory, DNS and Windows Workstation Setup

## Purpose

This document records the installation of Active Directory Domain Services (AD DS), creation of the `CYBERFORGE.com` domain, the DNS configuration wizard and the joining of the Staff1 Windows workstation. Both virtual machines were accessed through the Proxmox noVNC console.

## Environment

| Component | Value shown in the screenshots |
|---|---|
| Domain controller VM | AD, VM ID 100 |
| Server operating system | Windows Server 2012 R2 Standard Evaluation |
| Server hostname | `WIN-Q9A07CPFELN` |
| Server IPv4 address | `192.168.0.200` |
| Domain and forest name | `CYBERFORGE.com` |
| Domain NetBIOS name | `CYBERFORGE` |
| Forest functional level | Windows Server 2012 R2 |
| Domain functional level | Windows Server 2012 R2 |
| Workstation VM | Staff1, VM ID 101 |
| Workstation name | `Staff1` |
| Workstation full name after joining | `Staff1.CYBERFORGE.com` |

The browser address identifies the Proxmox host, not the domain controller's IP address. The domain controller address is shown separately in Server Manager.

## 1. Open the Add Roles and Features Wizard

On the AD server, open **Server Manager**, select **Manage → Add Roles and Features**, and review the **Before you begin** page.

The wizard lists preparation tasks, including configuring network settings such as a static IP address and setting an Administrator password. Check these settings before continuing. The supplied screenshots do not show the initial network configuration process.

Click **Next** and proceed through installation type and server selection to the server roles page, targeting `WIN-Q9A07CPFELN`.

<img width="960" height="564" alt="1  Add Roles and Features" src="https://github.com/user-attachments/assets/457a852b-8398-4b1d-b428-27491867c61b" />

## 2. Select Active Directory Domain Services

On **Select server roles**, select **Active Directory Domain Services**. Include its required management features when prompted, then continue through the remaining pages and start installation.

The installation progress screen lists:

- Active Directory Domain Services.
- Group Policy Management.
- Remote Server Administration Tools.
- The Active Directory module for Windows PowerShell.
- Active Directory Administrative Center.
- AD DS snap-ins and command-line tools.

Wait for role installation to finish before configuring the domain controller. The captured progress screen shows installation underway; the subsequent configuration wizard shows that the process progressed to promotion.


<img width="960" height="564" alt="2  " src="https://github.com/user-attachments/assets/4767896e-7c50-4940-94df-fa55482d9c70" />

<img width="960" height="564" alt="3  Installing Roles" src="https://github.com/user-attachments/assets/ce512eee-bf23-4d03-9fc7-5395820a07ae" />

## 3. Configure domain controller promotion

After installing AD DS, use the Server Manager notification to open **Promote this server to a domain controller**.

The first deployment screenshot shows **Add a domain controller to an existing domain** selected with `CyberForge` entered. However, the later **Review Options** screen explicitly records the server as the **first domain controller in a new forest**. The final recorded deployment therefore used the new-forest configuration.

To reproduce the configuration shown in the review screen, select **Add a new forest** and enter the root domain name:

```text
CYBERFORGE.com
```

Continue through the domain controller options, DNS options, additional options and paths. Enter a Directory Services Restore Mode password when requested. The password and the individual options pages are not captured in this screenshot set.

<img width="960" height="564" alt="4  Promoting to a Domain Controller" src="https://github.com/user-attachments/assets/dcc62968-0452-46fa-9a28-c6d9e4771e43" />

## 4. Review the domain configuration

Confirm the settings on **Review Options** before continuing:

| Option | Recorded selection |
|---|---|
| Deployment | First domain controller in a new forest |
| Domain name | `CYBERFORGE.com` |
| Forest name | `CYBERFORGE.com` |
| NetBIOS name | `CYBERFORGE` |
| Forest functional level | Windows Server 2012 R2 |
| Domain functional level | Windows Server 2012 R2 |
| Global catalog | Yes |
| DNS Server | Yes |
| Create DNS delegation | No |

Click **Next** to run the prerequisite checks.

<img width="960" height="564" alt="5  Review Options" src="https://github.com/user-attachments/assets/81c464eb-d22f-4cac-8e19-690bdd49fc51" />


## 5. Check prerequisites and install

The **Prerequisites Check** page reports that all prerequisite checks passed successfully. It also displays a DNS delegation warning: a delegation could not be created because the authoritative parent zone could not be found or did not run Windows DNS. The wizard explains that parent-zone delegation is relevant when integrating with an existing DNS infrastructure; otherwise no action is required for that warning.

Review the results, then click **Install**. The wizard states that the server automatically restarts at the end of the operation.

<img width="960" height="564" alt="6" src="https://github.com/user-attachments/assets/b3ab0786-d927-4ae2-8171-30cb1ad71ba0" />

## 6. Verify the domain controller configuration

After promotion, the server sign-in screen displays:

```text
CYBERFORGE\Administrator
```

Sign in and open **Server Manager → Local Server**. The captured properties show:

| Property | Recorded value |
|---|---|
| Computer name | `WIN-Q9A07CPFELN` |
| Domain | `CYBERFORGE.com` |
| Ethernet IPv4 address | `192.168.0.200` |
| Available role pages | AD DS and DNS |

These screens establish that the domain configuration was applied and that the AD DS and DNS role pages were available. They do not constitute a complete domain controller health check.

<img width="960" height="564" alt="7  Domain Exists" src="https://github.com/user-attachments/assets/c2fa19ab-5d68-4b8d-9741-be0e98f90847" />
<img width="960" height="564" alt="8  Domain Up" src="https://github.com/user-attachments/assets/29addb30-bde2-484a-8551-f2d6c36d4f0e" />


## 7. Open the DNS configuration wizard

From Server Manager, open **Tools → DNS**. In DNS Manager, select the server and launch **Configure a DNS Server**.

The wizard explains that it can configure lookup zones, root hints and forwarders. Click **Next** to proceed through the required configuration pages.

<img width="960" height="564" alt="9  Configuring DNS Server" src="https://github.com/user-attachments/assets/029ec8a2-dc77-4ecb-a3dc-bcdc5f4b9600" />

## 8. Review the DNS wizard summary

The final DNS wizard page records:

| Setting | Value shown |
|---|---|
| DNS server to configure | `WIN-Q9A07CPFELN` |
| Forward lookup zone to create | `cyberforge` |

Click **Finish** to save the wizard settings.

The zone name shown here is `cyberforge`, while the Active Directory domain is `CYBERFORGE.com`. Preserve this distinction when describing the evidence: this screenshot does not show the records within the AD domain zone or demonstrate that the additional short-name zone is required for domain membership. The intermediate DNS wizard choices, saved zone contents and lookup tests are not captured.
<img width="960" height="564" alt="10" src="https://github.com/user-attachments/assets/3c1ae3d1-3aa9-4bf0-830e-f1dcc019f223" />

## 9. Join the Staff1 workstation to the domain

On the Staff1 workstation, open **System Properties → Computer Name → Change** to access **Computer Name/Domain Changes**.

Confirm the computer name is `Staff1`. Under **Member of**, select **Domain**, enter the domain name and click **OK**. The earlier screenshot contains `CYBERFORGE`; the successful join screenshot shows the full domain name:

```text
CYBERFORGE.com
```

When prompted, provide credentials for an account authorised to join computers to the domain. The credential prompt and account used are not included in these screenshots.

For this join to work, the workstation must be able to reach the domain controller and resolve the domain through the internal DNS service. The workstation's DNS and IP configuration are not shown in this set.

<img width="960" height="564" alt="9  Joining Domain on Workstation PC" src="https://github.com/user-attachments/assets/5a4f5217-c7cb-479f-926e-fa5301450d0b" />
<img width="960" height="564" alt="11  Joined Domain" src="https://github.com/user-attachments/assets/1d37c0f1-a6cf-4362-94a2-64c7230b4501" />


## 10. Confirm the join and restart the workstation

The workstation displays the confirmation:

```text
Welcome to the CYBERFORGE.com domain.
```

Click **OK** and close the remaining dialogs. System Properties then shows:

```text
Full computer name: Staff1.CYBERFORGE.com
Domain: CYBERFORGE.com
```

A message states that the changes take effect after restarting the computer. Restart Staff1 to complete the process. The restart and a subsequent domain-user login are not captured in the supplied screenshots.

<img width="960" height="564" alt="11  Joined Domain" src="https://github.com/user-attachments/assets/4f39d67c-c223-4b57-86ef-870cdf0e363d" />
<img width="960" height="564" alt="12" src="https://github.com/user-attachments/assets/dd9890fc-2469-4b0c-8868-ad9b32ae1a2c" />


## Recorded outcome

The screenshots document creation of the `CYBERFORGE.com` forest and domain on Windows Server 2012 R2, with DNS and Global Catalog selected. The server properties show the domain and address `192.168.0.200`. Staff1 received a successful domain-join confirmation and displayed the full computer name `Staff1.CYBERFORGE.com`, with a restart still required in the final captured screen.
