# Scenario 01 · Spear Phishing

**Setup.** Use the Scenario 1 baseline with the simulated external segment (vmbr5), Staff1, Mailcow, osTicket, Admin and JumpHost. The facilitator prepares a mock employee identity and the approved email and ticket interactions. Blue team opens Wazuh before the first inject.

| Step | Red team task | Flag | Blue team evidence to investigate |
| --- | --- | --- | --- |
| 1 | Use the supplied mock identity and staged email interaction. After the facilitator confirms the Staff1 foothold, inspect the assigned exercise artifacts on Staff1. | S1-F1 · Staff1 | Staff process activity and callback timing |
| 2 | Locate the lab help desk and use the staged ticket interaction. After the Admin foothold, inspect the Admin exercise artifacts. | S1-F2 · Admin | Ticket timing correlated with Admin execution and connections |
| 3 | Follow the approved path from Admin into JumpHost. Inspect the JumpHost exercise artifacts and record the OT boundary reached. | S1-F3 · JumpHost | Authentication failures and success, source host and time |

Submit the three recovered values privately. If a checkpoint is missing, have the facilitator verify the baseline and inject before continuing. See the [existing facilitator walkthrough](../Scenario1/Senario_1_RedTeam_Code.md) for the original implementation.
