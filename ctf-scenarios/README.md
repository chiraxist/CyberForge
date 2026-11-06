# Run the CyberForge CTF

1. Pick a scenario and start from the approved baseline using the [operator guide](../operator-guide/).
2. The facilitator confirms the red team scope, blue-team telemetry, and the three live flag values from the private exercise answer sheet. Record the start time, scenario and team names.
3. Red team submits each recovered flag privately with the host, timestamp and artifact location. Blue team records the affected host, time, Wazuh event or supporting evidence, and proposed containment.
4. Suggested scoring is one point per independently verified flag, up to three per scenario. Debrief using the red and blue timelines, then preserve evidence and restore the baseline.

| Scenario | Entry and checkpoints |
| --- | --- |
| [01 · Spear Phishing](01-spear-phishing.md) | Simulated external segment → Staff1 → Admin → JumpHost |
| [02 · Network Discovery](02-network-discovery.md) | Authorized lab network → AD Server → Admin → JumpHost |

End the exercise at the JumpHost checkpoint. Any physical OT exercise requires separate approval, a defined safe state and supervision. Pause if unexpected traffic or physical behavior occurs.
