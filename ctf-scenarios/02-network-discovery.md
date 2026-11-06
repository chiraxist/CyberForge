# Scenario 02 · Network Discovery

**Setup.** Use the Scenario 2 baseline. The facilitator identifies the authorized access point or provides a preconnected red endpoint. Verify AD, Admin and JumpHost are available and Wazuh is receiving events.

| Step | Red team task | Flag | Blue team evidence to investigate |
| --- | --- | --- | --- |
| 1 | Discover services on the designated enterprise segment. Identify the staged SMB vulnerability on the legacy AD Server and complete the facilitator-approved initial-access exercise. Search the AD exercise artifacts. | S2-F1 · AD Server | Service discovery, SMB activity and unusual logons |
| 2 | Use the scenario-issued or recovered lab identity to reach Admin and locate its exercise artifact. | S2-F2 · Admin | New remote session and related authentication records |
| 3 | Follow the approved administration path to JumpHost and locate its exercise artifact. Record the boundary reached and end the exercise. | S2-F3 · JumpHost | Authentication sequence, source and any boundary events |

Submit flags and evidence privately. See the [existing facilitator walkthrough](../Scenario2/Scenario-2.md) for the original implementation.
