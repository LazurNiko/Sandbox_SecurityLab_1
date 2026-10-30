# Suricata — Simplified Architecture

```text
                    ┌──────────────────────┐
                    │    Windows Target    │
                    │                      │
                    │  DC01.corpnet.local  │
                    └──────────┬───────────┘
                               │
                               │ Network Traffic
                               ▼
                    ┌──────────────────────┐
                    │        SENSOR        |
                    |       Suricata       │
                    │     Ubuntu Server    |
                    |                      |
                    |     IDS / Network    │
                    │       Detection      │
                    │                      │
                    │        Rules         │
                    │        PCAP          |
                    └──────────┬───────────┘
                               │
                    ┌──────────┴───────────┐
                    │                      │
                    ▼                      ▼
                fast.log                 eve.json
                    │                      │
                    └──────────┬───────────┘
                               ▼
                    ┌──────────────────────┐
                    │       Analyst        │
                    │                      │
                    │ Detection            │
                    │ Investigation        │
                    │ IOC extraction       │
                    └──────────────────────┘


Example detection flow:

      KALI
  Network attack
       │
       ▼
  Windows Target
DC01.corpnet.local
       │
     SENSOR
    Suricata
       │
       ├── Signature match
       │       │
       │       ▼
       │    fast.log
       │
       └── Network event
               │
               ▼
            eve.json
               │
               ▼
           Investigation
```

## Main purpose

**Attack → Network Traffic → Detection → Evidence → Investigation**

### Main data sources

- Live network interface
- PCAP
- `fast.log`
- `eve.json`

### Detection examples

- Network scans
- Exploit traffic
- Suspicious HTTP traffic
- Command-and-control indicators
- Custom IDS rules

## Lab principle

Suricata focuses on **network-level visibility**.

Wazuh focuses on **endpoint-level visibility**.

Together they provide complementary evidence for DFIR:

```text
              ATTACK
                 │
        ┌────────┴────────┐
        ▼                 ▼
     Network            Host
        │                 │
    Suricata            Wazuh
        │                 │
    Network IOC       Host IOC
        │                 │
        └────────┬────────┘
                 ▼
           Investigation
```
