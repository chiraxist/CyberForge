# Linking Vmbr3 to the OT Network
This documentation details how Vmbr3 on proxmox was linked to the OT network.

## Creating the Linux Bridge in Proxmox
In proxmox a linux bridge was created:
<img width="1101" height="388" alt="image" src="https://github.com/user-attachments/assets/c18b09d1-45ce-4c19-9223-ac48f6f05315" />
Named Vmbr3 and the CIDR was set to 192.168.0.1/24
<img width="1038" height="529" alt="image" src="https://github.com/user-attachments/assets/3426a0c1-4851-47ca-a211-36a36d61e562" />
## Physically Connecting Vmbr3 to the OT Network
All of the OT devices connected to a swith, in that same switch an ethernet cable was used to connect it to the proxmox:
<img width="3024" height="4032" alt="IMG_6466" src="https://github.com/user-attachments/assets/727e0979-95f8-443d-b16c-1564e3894b9f" />
<img width="3024" height="4032" alt="IMG_6467" src="https://github.com/user-attachments/assets/fe11bb10-1072-4489-b859-7c388955f01f" />
## Adding the Physical Ethernet Cable to the Linux Bridge

