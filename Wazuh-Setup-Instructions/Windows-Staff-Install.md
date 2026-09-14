# For Windows Staff Machines

1. Go into a PowerShell terminal with Administrator privileges
2. Paste  & configure these following commands in sequential order:

   ```Invoke-WebRequest -Uri https://packages.wazuh.com/4.x/windows/wazuh-agent-4.14.7-1.msi -OutFile $env:tmp\wazuh-agent.msi```

<img width="1192" height="212" alt="image" src="https://github.com/user-attachments/assets/5e58f73b-bd82-4a50-9e6e-60a596b250c4" />

3. Click enter and you should see a progress bar of some sort.

4. Next, input this following command, replacing the <Wazuh-Manager-IP> with your Wazuh manager IP address, and replace <Wazuh-Agent-Name> with your desired name, making sure it is all 1 word: 
    
    ```msiexec.exe /i $env:tmp\wazuh-agent.msi /q WAZUH_MANAGER='<Wazuh-Manager-IP>' WAZUH_AGENT_NAME='<Wazuh-Agent-Name>'```

<img width="1192" height="98" alt="image" src="https://github.com/user-attachments/assets/fe8f8156-550b-4662-a57f-0b0a36c3cbd7" />

5. Finally, you must start the service:
  
    ```NET Start Wazuh```

<img width="1166" height="150" alt="image" src="https://github.com/user-attachments/assets/ac5385da-f22a-4171-9743-aa13879d1e28" />

6. Once that is done, navigate back to your "endpoints summary page" in the Wazuh dashboard, and allow it some time to connect.
