## Attack Detection via Windows Event Log

Prepairing steps
#### 1. Enable Directory Service Access auditing via Advanced Audit Policy on DC01
```powershell
auditpol /set /subcategory:"Directory Service Access" /success:enable /failure:enable
```
#### 2. Enable Firewall Rule for monitoring Windows Event Viewer on DC01
```powershell
Enable-NetFirewallRule -DisplayGroup 'Romote Event Log Management'
```
#### 3. Add a SACL on the domain root object
```
dsa.msc → Properties → Security → Advanced → Auditing → Add 
OR
Server Manager Tools → Active Directory Users and Computers → Properties → Security → Advanced → Auditing → Add:

Principal: Everyone
Type: Success
Applies to: This object only
Permissions: Control access
```
***This ensures that any attempt at GetChanges/GetChangesAll on the domain lands in the Security log as Event ID 4662***

#### 3. DCSync (Event Id 4662)
Reproduce the attack and find the event in the logs (On Kali Machine)
```bash
secretsdump.py labdomain.local/svc-monitor:'Monitor2024!'@<192.168.100.1>
```
On host — Event Viewer → Event Viewer (Local) → Connect To Another Computer... → set user (Administrator) → Create Custom View → Event Logs: Windows Logs Event Id **4662**.
The key fields are Access Mask and Properties (GUID list):

Account Name:		svc-monitor

![DCSync Windows Log_1](/Sandbox_SecurityLab_1/block1-attacks/src/EV_DCSync.jpg)

1131f6aa-9c07-11d1-f79f-00c04fc2dcd2
DS-Replication-Get-Changes

![DCSync Windows Log_2](/Sandbox_SecurityLab_1/block1-attacks/src/EV_DCSync1.jpg)

DS-Replication-Get-Changes
1131f6ad-9c07-11d1-f79f-00c04fc2dcd2

The presence of both GUIDs in a single event (or two consecutive events) from one Account Name that is not a Domain Controller account and is not a member of the built-in replication groups (Domain Controllers, ENTERPRISE DOMAIN CONTROLLERS) — that is the signature of a DCSync attack
GUID

#### 4. Kerberoasting (Event Id 4769)

Reproduce the attack
```bash
GetUserSPNs.py corpnet.local/j.melnyk:'P@ssw0rd!' -dc-ip 192.168.100.1 -request
```
On host — Event Viewer → Event Viewer (Local) → Connect To Another Computer... → set user (Administrator) → Create Custom View → Event Logs: Windows Logs Event Id **4769**.

![Kerberoasting Windows Log_1](/Sandbox_SecurityLab_1/block1-attacks/src/Kerberoasting.jpg)

#### 5. AS-REP Roasting (Event Id 4768)

Reproduce the attack
```bash
GetNPUsers.py corpnet.local/ -usersfile users.txt -dc-ip 192.168.100.1 -format hashcat
```
On host — Event Viewer → Event Viewer (Local) → Connect To Another Computer... → set user (Administrator) → Create Custom View → Event Logs: Windows Logs Event Id **4768**.

![Kerberoasting Windows Log_1](/Sandbox_SecurityLab_1/block1-attacks/src/AS-REP_R_EV.jpg)



