# Wazuh Lab — Simplified Architecture

```text
                    ┌──────────────────────┐
                    │   Windows Endpoint   │
                    │                      │
                    │  Sysmon              │
                    │  Security Events      │
                    │  PowerShell           │
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

  ProcDump / LSASS activity
          │
          ▼
       Sysmon
          │ Event ID 1 / 11 / 13
          ▼
    Wazuh Agent
          │
          ▼
    Wazuh Manager
          │
          ▼
   Custom Rule 100205
          │
          ▼
       Alert
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
