Pfsense default IP configuration:

![](media/image1.png)

We must change the IP configuration to allow connectivity with the staff and pfsense:

![](converted/media/image2.png){width="6.268055555555556in" height="3.60625in"}

Changed the IP configuration:

![](converted/media/image3.png){width="6.268055555555556in" height="3.1152777777777776in"}

In proxmox web ui head to Datacenter/proxmox/Staff2/Console, remote into the computer

Go into staff 2 computer and login to pfsense as we should have connectivity now. This can be done by going to the browswer and typing in pfsense's IP address which we set to 192.168.0.1

![](converted/media/image4.png){width="6.268055555555556in" height="4.697916666666667in"}

Pfsense will automatically make you go through a wizard. Go thru the wizard of pfsense:

![](converted/media/image5.png){width="6.261290463692038in" height="4.688153980752406in"}

Pfsense is asking us to see if we want it to be part of a domain, we allocate cyberforge to become part of its domain. Pfsense domain setup:

![](converted/media/image6.png){width="6.268055555555556in" height="4.2444444444444445in"}Override DNS as well

Set the time server to Australia/perth:

![](converted/media/image7.png){width="6.268055555555556in" height="3.125in"}

Leave WAN as DHCP for now as we don't need internet to pass through pfsense:

![](converted/media/image8.png){width="6.268055555555556in" height="2.8409722222222222in"}

Block WAN traffic to prevent any unwanted packets to come through:

![](converted/media/image9.png){width="6.268055555555556in" height="2.1034722222222224in"}

Leave IP as 192.168.0.1 and subnet mask as 24:

![](converted/media/image10.png){width="6.268055555555556in" height="3.2493055555555554in"}

Set pfsense password as "pfsense" and leave it at that default password

![](converted/media/image11.png){width="6.268055555555556in" height="3.4458333333333333in"}

Reload everything for the configuration to be setup:

![](converted/media/image12.png){width="6.268055555555556in" height="3.225in"}

Pfsenese is now configured:

![](converted/media/image13.png){width="6.268055555555556in" height="3.4506944444444443in"}

Here, we can see pfsense home screen setup after going through the wizard:

![](converted/media/image14.png){width="6.268055555555556in" height="4.08125in"}

Theres only 2 interfaces instead of 3 because we've only configured the LAN to communicate with pfsense (within vmbr1 network), and the other is the WAN which is not in use currently. The 3^rd^ interface is not showing because we didn't assign anything to it yet

![](converted/media/image15.png){width="4.917352362204724in" height="1.156411854768154in"}

However, when we head to check the interfaces, we can see the 2 interfaces are assigned and the 3^rd^ one isn't assigned to anything as of yet.

![](converted/media/image16.png){width="6.268055555555556in" height="2.7368055555555557in"}

We can see the configuration of the network adapters of pfsense. Vmbr1 is set to em0, Vmbr2 is set to WAN, vmbr3 is set to OPT1. Since LAN is set to vmbr1, we set that network adapter's IP address to 192.168.0.1 so the staff network can communicate with pfsense (because they're all on the .0 subnet and their firewalls have been configured to allow traffic to pass through).

![](converted/media/image17.png){width="6.268055555555556in" height="1.9055555555555554in"}
