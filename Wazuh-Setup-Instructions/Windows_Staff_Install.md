# For Windows Machines

## Goal
The goal is to install the specific Wazuh agent tailored to the operating system running, enabling and starting the Wazuh agent, making sure the agent has connected to the manager on your Wazuh manager device.

## 1.0 Download and Installation

### 1.1 Go into a PowerShell terminal with Administrator privileges

Open a Powershell terminal by searching for "powershell" in the search box.

### 1.2 Download the Wazuh Agent through CLI
Paste & configure the following commands in sequential order

```powershell
Invoke-WebRequest -Uri https://packages.wazuh.com/4.x/windows/wazuh-agent-4.14.7-1.msi -OutFile $env:tmp\wazuh-agent.msi
```

<img width="1192" height="212" alt="image" src="https://github.com/user-attachments/assets/5e58f73b-bd82-4a50-9e6e-60a596b250c4" />

Click enter and you should see a progress bar of some sort.

### 1.3 Install the Wazuh Agent
Next, input the following command, replacing the ```<Wazuh-Manager-IP>``` with your Wazuh manager IP address, and replace ```<Wazuh-Agent-Name>``` with your desired name, making sure it is all 1 word: 
    
```powershell
msiexec.exe /i $env:tmp\wazuh-agent.msi /q WAZUH_MANAGER='<Wazuh-Manager-IP>' WAZUH_AGENT_NAME='<Wazuh-Agent-Name>'
```

<img width="1192" height="98" alt="image" src="https://github.com/user-attachments/assets/fe8f8156-550b-4662-a57f-0b0a36c3cbd7" />

## 2.0 Configuring the Agent

### 2.1 Start the service and enable the agent

Start the Wazuh agent service through CLI:
  
```powershell
NET Start Wazuh
```

<img width="1166" height="150" alt="image" src="https://github.com/user-attachments/assets/ac5385da-f22a-4171-9743-aa13879d1e28" />

### 2.2 Ensure Agent is connected to the manager
Once that is done, navigate back to your "endpoints summary page" in the Wazuh dashboard, and allow it some time to connect.
