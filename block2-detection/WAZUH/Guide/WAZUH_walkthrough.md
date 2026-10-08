Host-Level Detection Analysis — Wazuh (corpnet.local Lab)
Scope
This document walks through host-level detection in Wazuh for the four Active Directory attacks executed against corpnet.local:


---
|#|Attack|Windows Event Source|Wazuh Rule ID|Suricata Correlate|
|---|---|---|---|---|
1|Kerberoasting|Security 4769|100201 / 100202|sid:1000001|
2|AS-REP Roasting|Security 4768|100203|(host-only, see Section 2)|
3|DCSync|Security 4662|100204|sid:1000002|
4|LSASS Credential Dump|Sysmon 10 (ProcessAccess)|100205|(host-only)|
---

Lab topology: DC01 (domain controller), KALI-ATK (attacker), SENSOR (Suricata + Wazuh manager), VICTIM (Wazuh agent), all on the isolated 192.168.100.0/24 segment (VMnet2).


0. Prerequisites
0.1 Wazuh agent on DC01
Confirm the agent is registered and active before generating any events on Sensor:

```bash
sudo /var/ossec/bin/agent_control -l
```

0.2 Windows Audit Policy on DC01
Without these categories enabled, events 4768/4769/4662 are never generated, regardless of how the Wazuh rules are written:

```powershell
auditpol /set /subcategory:"Kerberos Authentication Service" /success:enable /failure:enable

auditpol /set /subcategory:"Kerberos Service Ticket Operations" /success:enable /failure:enable

auditpol /set /subcategory:"Directory Service Access" /success:enable
```

0.3 SACL on the domain object (required for 4662 / DCSync)
Audit policy alone is not enough for 4662 — the domain object itself needs a System Access Control List entry for "Everyone" (or specifically for replication rights), configured via ADSI Edit or dsacls, before DS-Replication-Get-Changes attempts are logged.
0.4 Sysmon on DC01 (required for LSASS detection)
Standard Windows Security log does not capture process memory access. Sysmon's Event ID 10 (ProcessAccess) does:

```powershell
Invoke-WebRequest -Uri https://live.sysinternals.com/Sysmon64.exe -OutFile Sysmon64.exe

Invoke-WebRequest -Uri https://raw.github.com/SwiftOnSecurity/sysmon-config/master/sysmonconfig-export.xml -OutFile sysmonconfig.xml

.\Sysmon64.exe -accepteula -i sysmonconfig.xml
```
(Use a config that logs TargetImage containing lsass.exe, e.g. a SwiftOnSecurity base config.)

1. Kerberoasting — Event 4769, RC4 Ticket
1.1 Attack execution (KALI-ATK)
GetUserSPNs.py corpnet.local/user:password -dc-ip 192.168.100.1 -request
1.2 What Windows logs
Event 4769 (A Kerberos service ticket was requested) with TicketEncryptionType = 0x17 (RC4-HMAC) and Status = 0x0 (success).

1.3 Wazuh rule

![Wazuh rule.id 100201, 100202](/Sandbox_SecurityLab_1/block2-detection/wazuh_rules.md)

Why ServiceName is excluded for krbtgt and machine accounts ($): both routinely request/renew tickets as part of normal domain operation and would otherwise generate noise.
1.4 Verification in Wazuh Dashboard
Open Security Events → filter rule.id: 100201.
Confirm fields present: data.win.eventdata.TicketEncryptionType, data.win.eventdata.ServiceName, data.win.eventdata.TargetUserName.
Re-run the attack 3+ times within 60s from the same source to trigger 100202 and confirm the frequency rule fires.
1.5 Sample decoded event (fill in from your run)
```json
{

  "timestamp": "",

  "rule": { "id": "100201", "level": 10, "description": "" },

  "data": {

    "win": {

      "system": { "eventID": "4769" },

      "eventdata": {

        "TicketEncryptionType": "0x17",

        "ServiceName": "",

        "TargetUserName": "",

        "IpAddress": ""

      }

    }

  }

}
```

2. AS-REP Roasting — Event 4768, No Pre-Authentication
2.1 Attack execution (KALI-ATK)
GetNPUsers.py corpnet.local/ -usersfile users.txt -dc-ip 192.168.100.1 -no-pass

Target account must have Do not require Kerberos preauthentication enabled.
2.2 What Windows logs
Event 4768 (A Kerberos authentication ticket (TGT) was requested) with PreAuthType = 0.
2.3 Why there is no Suricata correlate
Suricata's native krb5_* keyword set (as of 8.0.x) does not expose the PA-DATA field in AS-REQ — only msg_type, cname, sname, err_code, and ticket_encryption. This was confirmed empirically: no keyword matches PA-DATA presence/absence, so this attack is host-detection only in this lab. This asymmetry is itself a documented finding — see Section 4.

2.4 Wazuh rule

![Wazuh rule.id 100203](/Sandbox_SecurityLab_1/block2-detection/wazuh_rules.md)

2.5 Verification
Filter rule.id: 100203 in Wazuh Dashboard, confirm data.win.eventdata.PreAuthType = 0 and TargetUserName matches the account configured without pre-auth.

3. DCSync — Event 4662, Replication Rights
3.1 Attack execution (KALI-ATK)
secretsdump.py -just-dc corpnet.local/user:password@192.168.100.1
3.2 What Windows logs
Event 4662 (An operation was performed on an object) with Properties containing either:

*DS-Replication-Get-Changes — 1131f6aa-9c07-11d1-f79f-00c04fc2dcd2
DS-Replication-Get-Changes-All — 1131f6ad-9c07-11d1-f79f-00c04fc2dcd2*

3.3 Wazuh rule

![Wazuh rule.id 100204](/Sandbox_SecurityLab_1/block2-detection/wazuh_rules.md)

Why SubjectUserName excludes machine accounts ($): legitimate DC-to-DC replication uses the computer account of each domain controller and would otherwise be a constant false positive.
3.4 Network-side cross-check (DCERPC opnum)
The Suricata rule for this same attack (sid:1000002) matches dcerpc.opnum:3 (IDL_DRSGetNCChanges) on the DRSUAPI interface bind (e3514235-4b06-11d1-ab04-00c04fc2dcd2). During initial testing in this lab, a captured DRSUAPI session showed only opnum 0 (DSBind), 12 (DSCrackNames), 13 (DSWriteSPN), and 1 (DSUnbind) — opnum 3 never appeared, confirming that session was reconnaissance/SPN-related traffic, not an actual DCSync call. This is a useful negative-result finding: it shows the Suricata rule correctly distinguishes DCSync from adjacent, benign DRSUAPI activity.
3.5 Verification
Filter rule.id: 100204, confirm Properties contains one of the two GUIDs above and SubjectUserName is a real user account, not a $ machine account.


4. LSASS Credential Dumping — Sysmon Event 10
4.1 Attack execution (on DC01 or VICTIM, post-compromise)
Example via a tool like procdump or mimikatz sekurlsa::logonpasswords
```bash
procdump64.exe -accepteula -ma lsass.exe lsass.dmp
```
4.2 What Sysmon logs
Event ID 10 (ProcessAccess) with TargetImage ending in lsass.exe and a GrantedAccess value associated with memory-read access rights (e.g. 0x1010, 0x1410, 0x1438, 0x143a, 0x1fffff).

4.3 Wazuh rule

![Wazuh rule.id 100205](/Sandbox_SecurityLab_1/block2-detection/wazuh_rules.md)

4.4 Verification
Filter rule.id: 100205, confirm SourceImage (the process that opened the handle) is not a trusted AV/EDR process — tune the rule with a negate="yes" exclusion list if your lab generates false positives from legitimate security tooling.


5. Cross-Validation (Network vs. Host)
For Kerberoasting and DCSync, both layers fire independently for the same event. Methodology, results table, and timestamp-delta notes are tracked separately in block2-cross-validation.md (Section 3.3 of the write-up) — re-run each attack against a freshly restored snapshot, tail both eve.json and alerts.json in parallel, and record the delta. Expect Wazuh to lag Suricata by roughly the log-collector polling interval, not real time — this is expected behavior, not a detection failure.


6. Syntax Verification Checklist (before every manager restart)
```bash
sudo /var/ossec/bin/wazuh-analysisd -t

sudo systemctl restart wazuh-manager

sudo /var/ossec/bin/agent_control -l

# If wazuh-logtest is available, validate a captured raw event against the rule before relying on a live attack run:

sudo /var/ossec/bin/wazuh-logtest
```