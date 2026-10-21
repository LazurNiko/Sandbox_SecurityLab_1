                   Network Attack
                         │
                         ▼
                       PCAP
                         │
              ┌──────────┴──────────┐
              ▼                     ▼
           Zeek                 Suricata
              │                     │
       conn.log                 eve.json
       kerberos.log             fast.log
              │                     │
              └──────────┬──────────┘
                         ▼
                    CORRELATION
                         │
              ┌──────────┼──────────┐
              ▼          ▼          ▼
           4769       Wazuh      Timeline
              │          │          │
              └──────────┴──────────┘
                         ▼
                      REPORT


### Start capture with tcpdump
```bash
sudo tcpdump -i ens33 -w /home/sensor/block2_capture.pcap host 192.168.100.1
```

#### ZEEK
```bash
mkdir -p /forensics/CASE-2026-001/working/zeek_analysis
cd /forensics/CASE-2026-001/working/zeek_analysis
zeek -C -r /forensics/CASE-2026-001/evidence/pcaps/kerberoasting.pcap
```

#### SURICATA
```bash
sudo mkdir -p /forensics/CASE-2026-001/working/suricata
sudo suricata -r /forensics/CASE-2026-001/evidence/pcaps/kerberoasting.pcap -l /forensics/CASE-2026-001/working/suricata   
```
Logs to analyze:
```text
eve.json
fast.log
stats.log
```
With custom Rules ID:
```bash
sudo suricata -c /etc/suricata/suricata.yaml -r /forensics/CASE-2026-001/evidence/pcaps/kerberoasting.pcap -l /forensics/CASE-2026-001/working/suricata

cd /forensics/CASE-2026-001/working/suricata
sudo sha256sum * | sudo tee ../../evidence_log/suricata-sha256sums.txt
```
Find Rule ID's alerts:
```bash
sudo jq 'select(.event_type=="alert")' /forensics/CASE-2026-001/working/suricata/eve.json
```
