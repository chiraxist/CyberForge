# Cyberforge 4.0 - Proxmox Virtual Environment Setup Instructions
``01/09/2026``


1.  Navigate to: <https://www.proxmox.com/en/downloads/proxmox-virtual-environment>

2.  Download "Proxmox VE 9.2 ISO Installer" or later

3.  Flash the ISO to a USB for installation:

    a.  **Linux**:

        i.  Go to the file location of the download and open a terminal in that folder by right clicking -\> open terminal here

        ii. Within terminal, enter:

```sudo dd if=/path/to/proxmox-ve.iso of =/dev/sdX bs=4M status=progress```
> **AND** **replace:**
```path/to/proxmox-ve.iso```
> with your path and file name
> **AND** **replace:**
```/dev/sdX/```
> with your USB Drive identifier name. (Example below)

![](media/083691d30cca6251c8fffdcc43308472bf3bf6c2.png)

b.  **Windows**:

    i.  Navigate to: <https://rufus.ie/en/#download>

    ii. Download "rufus-4.15.exe" or newer and install

    iii. Open rufus, and under "Device", select your USB device as the target device

    iv. Under "Boot Selection", select your ISO file to flash

    v.  Click Start

![](media/f8dcd0f77d7123446c926ac267ec51bdd31aafbe.png)

4.  Eject your USB safely from your device

5.  Plug your USB with the flashed ISO into your target device you would like Proxmox VE to be installed in

6.  Enter the boot menu of the target device through the BIOS, and select your USB as the boot device

![](media/e8a67f4aba60d51b266fbfe1da02d9b59a384b54.png "screenshot/pve-grub-menu.png")

7.  Once booted in, click on "Install Proxmox VE (Graphical)"

8.  At the bottom of the screen make sure the "Target Hard disk" is the device's main storage location

![](media/95a1d6512fef6b7ae328bf95996ed442b41d00d7.png "https://pve.proxmox.com/pve-docs/images/screenshot/pve-select-target-disk.png")

9.  Click Next

10. Set your "Country, Time zone and Keyboard layout" to your preferences

![](media/cb6aa14fd6fcfa4d8d22b3319bc2087544cf6c71.png "https://pve.proxmox.com/pve-docs/images/screenshot/pve-select-location.png")

11. Click Next

![](media/0d61521c9cabe6c161ab86a110d2b5e000e632ce.png "https://pve.proxmox.com/pve-docs/images/screenshot/pve-set-password.png")

12. Enter a valid email address and password for your proxmox root password, then note these pieces of information

13. Click Next

![](media/589e85240cfda3d33a59e515b19e253a1511a1fa.png "https://pve.proxmox.com/pve-docs/images/screenshot/pve-setup-network.png")

14. If familiar with IP address subnets of your home network, configure your "IP Address, Gateway and DNS Server" accordingly, otherwise leave these options by default

15. For management interface, if there are multiple options, click on the first network interface not starting with "WL\....", as that is the wireless interface

16. Click Next

![](media/dd0ae276fef490276c1ab3e2a017dd5f12d2f415.png "https://pve.proxmox.com/pve-docs/images/screenshot/pve-install-summary.png")

17. Make sure the information is correct and check the "Automatically reboot after successful installation" box and click Install

![](media/3b1e0f734d9013ad2e8c04c348e6a090ec2deb12.png "https://pve.proxmox.com/pve-docs/images/screenshot/pve-installation.png")

18. After device has rebooted, unplug your USB with the ISO, so that the device can boot from Proxmox installation

19. Once the screen boots up, it will display the chosen or default IP range you set along with the port to configure the Proxmox VE server

![](media/2ce49ecaef21e8d2db1219fcf79bb1979d8df25c.png "Install Proxmox VE [A Step By Step Guide] - OSTechNix")

20. For "pve login" / "proxmox login", enter ```root```

21. For "password", enter ```your configured password```

22. With a separate device, that is connected via ethernet to the Proxmox machine, statically apply the laptop an IP address in the same range as the Proxmox machine:

    a.  **Windows**:

        i.  Navigate to your network adapter for ethernet connections

        ii. Right click -\> Properties

        iii. Configure Internet Protocol Version 4 (TCP/IPv4) options

        iv. Check the "Use the following IP Address" (SKIP if Proxmox can dynamically assign IP with its own DHCP capabilities, we had to as it could not at the time):

> (If machine is: "192.168.0.100:8006" (As shown on the Proxmox VE) login page before logging in)

1.  IP Address:

    a.  Enter ```192.168.0.XX``` (replace XX with any number)

2.  Subnet Mask:

    a.  Enter ```255.255.255.0```

3.  Default Gateway:

    a.  Enter ```192.168.0.1```

```{=html}
<!-- -->
```
23. Navigate to Proxmox's web browser IP and port shown on the login page via a web browser. E.g: ```192.168.0.100:8006```

> ![](media/1efb881b2b21b9a92da19da9461add93a898027d.png "screenshot/gui-login-window.png")

24. Enter ```root``` as username

25. Enter ```your chosen password``` as password

26. Click "Okay" once the valid subscription dialogue box pops up

![](media/3668e676ab9757e7e01080f4d5f86bb16a4f6b72.png)

27. The dashboard will be present now

28. You have successfully setup Proxmox and logged into the configuration server
