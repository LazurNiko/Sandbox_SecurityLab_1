# Wazuh — Simplified Architecture

```text
                    ┌──────────────────────┐
                    │    Windows Target    │
                    │                      │
                    │  DC01.corpnet.local  │
                    └──────────┬───────────┘
                               │
                               │ Wazuh Agent
                               ▼
                    ┌──────────────────────┐
                    │    Wazuh Manager     │
                    │                      │
                    │  Decoders            │
                    │  Rules               │
                    │  Custom Rules        │
                    │  Alert Generation    │
                    └──────────┬───────────┘
                               │
                               │ Alerts
                               ▼
                    ┌──────────────────────┐
                    │   Wazuh Indexer      │
                    │      + Filebeat      │
                    └──────────┬───────────┘
                               │
                               ▼
                    ┌──────────────────────┐
                    │   Wazuh Dashboard    │
                    │                      │
                    │  Detection           │
                    │  Investigation       │
                    │  Visualization       │
                    └──────────────────────┘


Example detection flow:

                         KALI-ATK
  Host attack  ---------------------------  Network attack             
          │                                        |
          ▼                                        |
  ProcDump / LSASS activity                        |
          │                                        |
          ▼                                        |
       Sysmon                                      |
          │ Event ID 1 / 11 / 13                   |         
          ▼                                        ▼
    Wazuh Agent (WORKSTATION)       Wazuh Agent (DC01.corpnet.local)
          │                                        |
          ▼                                        ▼
          --------------  Wazuh Manager ------------                             
          │                                        |
          ▼                                        |
   Custom Rule 100205                  Custom Rules 100201-100204
          │                                        |
          --------------  Alert  -------------------    
                            │
                            ▼
                        Dashboard
```

## Main purpose

**Attack → Telemetry → Detection → Alert → Investigation**

### Main data sources

- Windows Security Event Log
- Sysmon
- PowerShell
- Wazuh Agent

### Detection examples

- LSASS credential dumping
- Suspicious process execution
- PowerShell activity
- File creation
- Registry modification
- Suspicious Sysmon events
