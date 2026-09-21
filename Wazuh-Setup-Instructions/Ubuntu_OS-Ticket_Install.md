# For Ubuntu Machines

## Goal
The goal is to install the specific Wazuh agent tailored to the operating system running, enabling and starting the Wazuh agent, making sure the agent has connected to the manager on your Wazuh manager device.

## 1.0 Download and Installation

### 1.1 Open a Bash Shell

Open a Bash Shell anywhere on your Ubuntu machine by clicking on the terminal icon or searching for terminal.

### 1.2 Download the Wazuh Agent through CLI
Paste & configure the following commands in sequential order, replacing the ```<Wazuh-Manager-IP>``` with your Wazuh manager IP address, and replace ```<Wazuh-Agent-Name>``` with your desired name, making sure it is all 1 word:

```bash
wget https://packages.wazuh.com/4.x/apt/pool/main/w/wazuh-agent/wazuh-agent_4.14.7-1_amd64.deb && sudo WAZUH_MANAGER='192.168.0.67' WAZUH_AGENT_NAME='OS-Ticket' dpkg -i ./wazuh-agent_4.14.7-1_amd64.deb
```

<img width="1284" height="312" alt="image" src="https://github.com/user-attachments/assets/18945d15-4d5a-46e0-b76c-3e66fc17afba" />


Click enter and you should see a progress bar.

## 2.0 Configure the Agent

### 2.1 Start the service enable the agent
Input the following commands, enabling and starting the wazuh agent service: 
    
```bash
sudo systemctl daemon-reload && sudo systemctl enable wazuh-agent && sudo systemctl start wazuh-agent
```

<img width="1056" height="106" alt="image" src="https://github.com/user-attachments/assets/f3d87832-1d71-44f1-8338-fcba8c356410" />

### 2.2 Ensure agent is connected to manager
Once that is done, navigate back to your "endpoints summary page" in the Wazuh dashboard, and allow it some time to connect.
