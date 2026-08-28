| VM | Role | RAM | vCPU | Storage | Network Interface |
|---|---|---|---|---|---|
| DC01 | Windows Server 2022 Evaluation (Administartor) | 3 GB | 2 | 40 GB dynamic | VMnet2 (Host-Only) |
| KALI-ATK | Kali Linux (ATTACKER)  | 2 GB | 2 | 30 GB dynamic | VMnet2 (Host-Only) |
| VICTIM | Metasploitable2 (VICTIM) | 1 GB | 1 | 8 GB | VMnet2 (Host-Only) |
| SENSOR | Ubuntu Server + Zeek/Suricata (SENSOR) | 1.5 GB | 1 | 15 GB | VMnet2 (Host-Only) |
