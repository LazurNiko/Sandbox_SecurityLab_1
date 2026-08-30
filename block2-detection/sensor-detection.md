## Block 2 — SENSOR-Side Detection
Capturing and Detecting Kerberoasting, AS-REP Roasting, and DCSync with Suricata
Lab: corpnet.local (isolated VMnet2, 192.168.100.0/24) Machine: SENSOR (Suricata IDS) Traffic source: KALI-ATK executing the attacks from <b>block1-attacks/four-attacks-chain.md</b>

Objective: Capture the relevant Kerberos/DRSUAPI traffic and write Suricata rules that reliably fire on each of the three attacks, then validate against real lab traffic.

### Prerequisites — SENSOR Placement and Interface Mode
Suricata only sees what actually reaches its interface. In VMnet2 this means SENSOR's virtual NIC must be able to observe traffic between KALI-ATK and DC01.

#### Enable promiscuous mode
```bash
ip a

sudo ip link set <interface> promisc on
```
*Without promiscuous mode, SENSOR only sees frames addressed directly to it, not the KALI-ATK ↔ DC01 traffic passing through the shared segment.*

### Install and Configure Suricata
```bash
sudo apt update

sudo apt install suricata -y

suricata --build-info | grep "AF_PACKET"   # confirm AF_PACKET support
```
#### Point Suricata at the correct interface

Edit /etc/suricata/suricata.yaml:
```bash
af-packet:

  - interface: <interface>

    cluster-id: 99

    cluster-type: cluster_flow

    defrag: yes

vars:

  address-groups:

    HOME_NET: "[192.168.100.0/24]"

    EXTERNAL_NET: "!$HOME_NET"

    DC_SERVERS: "[192.168.100.1]"
```

### Capture Ground-Truth Traffic (Baseline for Rule Writing)
Before writing rules blind, capture the actual attack traffic once, so rules are built against real packets rather than guessed field values.

Step 1 — Start a raw capture on SENSOR
```bash
sudo tcpdump -i <interface> -w /home/user/block2_capture.pcap host 192.168.100.1
```
Step 2 — On KALI-ATK, run each attack in sequence (from block1-attacks/four-attack-chain.md), while the capture is running.

Step 3 — Stop the capture and inspect

# Ctrl+C to stop tcpdump, then inspect with Wireshark or tshark

tshark -r block2_capture.pcap -Y "kerberos"

Confirm you can see:

AS-REQ packets (message type 10) for AS-REP Roasting
TGS-REQ packets (message type 12) for Kerberoasting
DRSUAPI IDL_DRSGetNCChanges calls for DCSync

This pcap becomes the regression-test artifact — every rule you write gets validated by replaying it (Section 5).


3. Writing Detection Rules
Create a dedicated rule file:

sudo nano /etc/suricata/rules/block2-ad-attacks.rules
3.1 Kerberoasting — RC4 TGS-REQ Detection
Logic: Kerberoasting tools (impacket, Rubeus) request TGS tickets using RC4 (etype 23) encryption because RC4 hashes are far easier to crack than AES. Modern AD environments default to AES-256; a burst of RC4 TGS-REQs is anomalous.

alert kerberos any any -> $DC_SERVERS 88 (msg:"AD Possible Kerberoasting - RC4 TGS-REQ"; \

  kerberos.msgtype:12; \

  kerberos.encryption:0x17; \

  threshold: type threshold, track by_src, count 3, seconds 60; \

  classtype:attempted-recon; sid:1000001; rev:1;)

kerberos.msgtype:12 — matches TGS-REQ specifically.
kerberos.encryption:0x17 — etype 23 (RC4-HMAC) in hex.
threshold — fires only after 3+ RC4 TGS-REQs from the same source within 60s, reducing noise from legacy-but-legitimate services.
3.2 AS-REP Roasting — Missing Pre-Authentication
Logic: a normal AS-REQ includes a PA-ENC-TIMESTAMP pre-auth field. AS-REP Roasting targets accounts with DONT_REQ_PREAUTH set, so the AS-REQ from the attacker's tool lacks this field entirely.

alert kerberos any any -> $DC_SERVERS 88 (msg:"AD Possible AS-REP Roasting - AS-REQ without PA-DATA"; \

  kerberos.msgtype:10; \

  kerberos.preauth:0; \

  threshold: type threshold, track by_src, count 1, seconds 10; \

  classtype:attempted-recon; sid:1000002; rev:1;)

kerberos.msgtype:10 — AS-REQ.
kerberos.preauth:0 — no pre-authentication data present (Suricata's Kerberos keyword; confirm exact keyword syntax against your Suricata version's kerberos app-layer parser docs, as field naming has changed across releases).
3.3 DCSync — Replication Call From a Non-DC Host
Logic: IDL_DRSGetNCChanges (part of MS-DRSR / DRSUAPI) should only ever be called by a genuine Domain Controller replicating with another DC. A call originating from KALI-ATK's IP is definitive, not probabilistic.

alert tcp any any -> $DC_SERVERS 135 (msg:"AD Possible DCSync - DRSUAPI GetNCChanges from non-DC host"; \

  flow:to_server,established; \

  dce_iface:e3514235-4b06-11d1-ab04-00c04fc2dcd2; \

  dce_opnum:3; \

  classtype:attempted-admin; sid:1000003; rev:1;)

dce_iface: e3514235-4b06-11d1-ab04-00c04fc2dcd2 — the DRSUAPI interface UUID.
dce_opnum: 3 — opnum for IDL_DRSGetNCChanges.
No $DC_SERVERS as source means this rule as written watches inbound calls to the DC — combine with a second rule or a PCAP/BloodHound cross-check confirming the source is NOT another element of $DC_SERVERS, since DC-to-DC replication is legitimate and would otherwise false-positive if you have >1 DC in scope.


4. Load Rules Into Suricata
Step 1 — reference the rule file in suricata.yaml

rule-files:

  - block2-ad-attacks.rules

Step 2 — validate rule syntax

sudo suricata -T -c /etc/suricata/suricata.yaml -v

Look for Configuration provided was successfully loaded — any syntax error in the .rules file will show up here before you ever run live.

Step 3 — run Suricata live

sudo suricata -c /etc/suricata/suricata.yaml -i <interface>


5. Validation — Replay the Captured Attack Traffic
Rather than re-running the live attacks every time, replay the pcap from Section 2 to regression-test rule changes quickly.

sudo suricata -c /etc/suricata/suricata.yaml -r block2_capture.pcap -l /var/log/suricata/

Step — inspect alerts

tail -f /var/log/suricata/fast.log

# or, for structured output:

cat /var/log/suricata/eve.json | jq 'select(.event_type=="alert")'

Confirm three distinct alerts fire, one per attack, matching sid:1000001, 1000002, 1000003.

Step — tune for false positives

Run the same ruleset against a period of normal domain traffic (e.g. a legitimate user logging in, a real service using its SPN normally) and confirm no alerts fire. If legitimate AES-based Kerberos auth is triggering the Kerberoasting rule, the kerberos.encryption:0x17 filter isn't working as intended — check the field syntax against the installed Suricata version.


6. Log Review Workflow (Analyst Perspective)
Once rules are live, the working analyst loop on SENSOR is:

# Watch for anything firing in real time

sudo tail -f /var/log/suricata/fast.log

# Pull full JSON detail on a specific alert for write-up screenshots

jq 'select(.alert.signature_id==1000003)' /var/log/suricata/eve.json

Each alert's eve.json entry should be cross-referenced against:

Source IP → confirm it's KALI-ATK, not a legitimate DC/service.
Timestamp → correlate with the attack timeline from KALI-ATK's side for the write-up.


7. Summary Table — Detection Mapping
Attack
Suricata Signature Basis
Key Field(s)
SID
Kerberoasting
RC4-encrypted TGS-REQ burst
msgtype:12, encryption:0x17
1000001
AS-REP Roasting
AS-REQ missing pre-auth
msgtype:10, preauth:0
1000002
DCSync
DRSUAPI GetNCChanges from non-DC
dce_iface, dce_opnum:3
1000003



8. Notes / Lessons Learned
Building the pcap baseline (Section 2) before writing rules turns rule-writing from guesswork into verification against real field values — critical since Suricata's kerberos keyword names have shifted between versions.
The DCSync rule is the most "definitive" of the three (a real DC should never see this from a workstation), while Kerberoasting/AS-REP rules are inherently probabilistic and need thresholds/tuning to avoid false positives in environments with legacy RC4 usage.
This file complements block2-ad-attack-chain.md: that file documents the attacker's decision process, this one documents the defender's verification process — together they form the full Recon → Exploit → Detect loop for the write-up template.

