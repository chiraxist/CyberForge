# Active Directory

## Adding Roles and Features

Here I clicked on the tools menu the clicked on 'Add Roles and Features'.

![](media/image1.png)

Selected Active Directory Domain Services and DNS Server then click Next. Skip features.

![](media/image2.png)

It will start installing. When it's finished, select Close.

![](media/image3.png)

## Setting up Domain

Promote the Windows Server 2012 machine to be domain controller.

![](media/image4.png)

Below is a screenshot of all the selected options to set. Click next.

![](media/image5.png)

Confirm the correct prerequisites are checked and select Install.

![](media/image6.png)

The Machine will restart. After a restart, the Domain will appear on the logon screen.

![](media/image7.png)

The domain will be up and running.

![](media/image8.png)


## Configuring DNS Server

Configuring DNS Server

![](media/image9.png)

Configuring DNS server and creating forward lookup zone.

![](media/image10.png)


## Joining Domain on Workstation PC

Joining the CYBERFORGE domain

![](media/image11.png)

It will ask to log on with an administrator account then appear with the message "Welcome to the CYBERFORGE.com domain. The workstation will then require a restart.

![](media/image12.png)

After logging into the workstation using one of the AD user accounts, we can check and see we are in the domain.

![](media/image13.png)

