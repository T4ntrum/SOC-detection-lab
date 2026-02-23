# Network Enumeration

### Prerequisites
- [ELK stack](https://github.com/T4ntrum/SOC-detection-lab/blob/main/elk-stack.md)
- [Atomic Red Team setup](https://www.youtube.com/watch?v=_xW3fAumh1c&t=878s)

---

### Objective

To simulate adversary reconnaissance behavior using Atomic Red Team's MITRE ATT&CK - T1016 and validate telemetry within an ELK stack. 

### Technique Overview

MITRE ATT&CK; T1016 (System Network Configuration Discovery) involves adversaries collecting network configuration information to better understand the environment. 

### Test execution

To start the Atomic Red Team module, use the command:
 ```ps1
 Invoke-Module T1016 -TestNumbers 1 
 ``` 

### Telemetry Observed 

Event ID 1 (Process Creation) 
```CMD
process.name: cmd.exe
process.parent.name: explorer.exe
process.command_line: cmd.exe /c set & arp -a & ipconfig /all & net view /all
```

### Log Analysis 

In Elasticsearch, the following query was used to identify relevant activity: 
```kql
event.code: 1
``` 

Analysis revealed `cmd.exe` being spawned from `Explorer.exe` with chained enumeration commands. 

This pattern is consistent with automated reconnaissance rather than normal user behavior.

### Assessment

Enumeration may be legitimate when performed by system administrators; however, the combination of commands and their execution context indicate active reconnaissance conducted by an attacker.

Within the Cyber Kill Chain framework, this activity aligns with the Reconnaissance phase. 

During this phase, an attacker gathers information about the environment, leading into the later stages of the attack. 

Reconnaissance -> Privilege escalation -> Attack on objectives


---
### Resources
- [Atomic Red Team T1016 - System Network Configuration Discovery](https://github.com/redcanaryco/atomic-red-team/blob/master/atomics/T1016/T1016.md)
- [Cyber Kill Chain](https://en.wikipedia.org/wiki/Cyber_kill_chain)