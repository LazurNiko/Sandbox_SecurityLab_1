### Install kerbrute
```bash
wget https://github.com/ropnop/kerbrute/releases/latest/download/kerbrute_linux_amd64 -O /usr/local/bin/kerbrute

chmod +x /usr/local/bin/kerbrute
```
### Install Bloodhound
```bash
sudo apt install bloodhound -y
```
### Install Pypykatz
```bash
pip install pypykatz --break-sysem-packages
```
###  Install procdump.exe (Sysinternals)
```bash
wget https://download.sysinternals.com/files/Procdump.zip
unzip Procdump.zip -d procdump/
```
### Confirm all nessesary tools are installed (crackmap -> now nxc!)
```bash
which nmap hashcat nxc smbclient impacket-GetUserSPNs impacket-GetNPUsers impacket-secresdump impacket-psexec kerbrute bloodhound-python pypykatz
``` 
### Install Portable OpenCL
```bash
sudo apt install pocl-opencl-icd -y
```
### Install ntpdate for sync time with DC01
```bash
sudo apt install ntpsec-ntpdate
```