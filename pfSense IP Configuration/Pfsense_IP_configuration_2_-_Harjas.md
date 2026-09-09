# Downloading iso:

1.  Head to the website: <https://archive.org/details/pfSense-CE-2.6.0-RELEASE-amd64>

2.  Click on the "ISO IMAGE" option and the file will start downloading

![](media/image1.png)

-   Now that we have the iso image file, we can upload that onto proxmox

3.  Type in this address on the search bar in your search engine: <https://192.168.0.137:8006/>

-   You will have this website:

![](media/image2.png)

4.  On the left hand side, you will see the file system

5.  Underneath "Datacenter", click on "proxmox" to expand it, click on "local (proxmox)"

6.  Once you're in, you will be directed to the summary. Click on the ISO images within the node you're in

7.  Then select upload

8.  Select the ISO image you downloaded after clicking select file

9.  Then select upload

![](media/image3.png)

10. After it has been uploaded, select create VM on the top right hand side:\
    Name: pfSense

11. Click next

![](media/image4.png)

12. Click next

13. Select the pfsense-CE-2.6.0-RELE image for the ISO image:

![](media/image5.png)

14. Click next:

![](media/image6.png)

15. Assign 8GB of RAM for pfSense:

![](media/image7.png)

16. Click next

17. Assign 1 socket and 2 cores for CPU

18. Click next

![](media/image8.png)

19. Change the memory to 4096

20. Click next

![](media/image9.png)

21. Connect network to vmbr1

22. Model: Intel E1000

23. Click next

![](media/image10.png)

24. Confirm everything to finalise the setup:

![](media/image11.png)

25. Click finish

26. Within the pfsnense node, select hardware and add 2 network devices

27. First add "vmbr 2" and set the model to "paravirtualised"

28. Then select add

29. Then add "vmbr 3" and set the model to "paravirtualised"

30. Then select add

![](media/image12.png)

31. Click on console and start pfsense

32. Select start now

![](media/image13.png)

# Installing pfSense:

1.  Press \[Enter\] to accept the notices.

2.  Press \[Enter\] to proceed with installation.

![](media/image14.png)

![](media/image15.png)

3.  ![](media/image16.png)Press {Enter}

4.  Press {Enter}

![](media/image17.png)

5.  Press {Enter}

![](media/image18.png)

6.  Press {Enter}

![](media/image19.png)

7.  Press \[Space\] to tick the checkbox, and press \[Enter\].

![](media/image20.png)

8.  Press \[Enter\]. Please wait for the installation to complete.

![](media/image21.png)

9.  Select No and Press \[Enter\].

![](media/image22.png)

10. Press \[Enter\] to reboot.

![](media/image23.png)

11. pfSense will ask to setup the VLANs.

12. Press y

![](media/image24.png)

13. When prompted to ask for WAN interface name, type in "vtnet1"

14. When prompted to ask for LAN interface name, type in "vtnet0"

15. When prompted to ask for optional 1 interface name, type in "em0"

![](media/image25.png)

16. You will be prompted to confirm the interfaces, select y

![](media/image26.png)

-   Pfsense default IP configuration is "192.168.1.1":

![](media/image27.png)

-   We must change the IP configuration to allow connectivity with the staff and pfsense:

17. Type in 2

18. Type in 2

19. Type in 192.168.0.1

![](media/image28.png)

-   IP configuration has been changed

![](media/image29.png)

# Pfsense GUI setup

Go into staff 2 computer and login to pfsense as we should have connectivity now. This can be done by going to the browser and typing in pfsense's IP address which we set to 192.168.0.1

1.  Log into staff 2's computer

2.  Go to the browser

3.  Type in 192.168.0.1

![](media/image30.png)

Pfsense will automatically make you go through a wizard. Go thru the wizard of pfSense:

4.  Click next

![](media/image31.png)

Pfsense is asking us to see if we want it to be part of a domain, we allocate cyberforge to become part of its domain. Pfsense domain setup:

5.  Type "pfSense" in the hostname

6.  Type in "CYBERFORGE.com" in the domain

![](media/image32.png)

7.  Click on "Override DNS"

8.  Set the time server to Australia/perth:

![](media/image33.png)

9.  Leave WAN as DHCP for now as we don't need internet to pass through pfsense:

![](media/image34.png)

10. Block WAN traffic to prevent any unwanted packets to come through:

![](media/image35.png)

11. Leave IP as 192.168.0.1

12. Leave subnet mask as 24

![](media/image36.png)

13. Set pfSense password as "pfsense" and leave it at that default password

![](media/image37.png)

14. Reload everything for the configuration to be setup:

![](media/image38.png)

pfSense is now configured:

![](media/image39.png)

Here, we can see pfsense home screen setup after going through the wizard:

![](media/image40.png)

Theres only 2 interfaces instead of 3 because we've only configured the LAN to communicate with pfSense (within vmbr1 network), and the other is the WAN which is not in use currently. The 3^rd^ interface is not showing because we didn't assign anything to it yet

![](media/image41.png)

However, when we head to check the interfaces, we can see the 2 interfaces are assigned and the 3^rd^ one isn't assigned to anything as of yet.

![](media/image42.png)

We can see the configuration of the network adapters of pfSense. Vmbr1 is set to em0, Vmbr2 is set to WAN, vmbr3 is set to OPT1. Since LAN is set to vmbr1, we set that network adapter's IP address to 192.168.0.1 so the staff network can communicate with pfSense (because they're all on the 192.168.0.\* network)

![](media/image43.png)
