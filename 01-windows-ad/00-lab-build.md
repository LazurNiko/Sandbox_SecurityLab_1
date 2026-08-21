### Create AD DS in Windows Server
```powershell

Rename-Computer -NewName "DC01"

Install-WindowsFeature -name AD-Domain-Services -IncludeManagementTools

Install-ADDSForest -DomainName "corpnet.local"

Get-ADDomain

Get-ADForest

Get-ADDomainController
```
### Create new OU
```powershell
New-ADOrganizationalUnit -Name "CorpNet-Lab"
```
### Create a service account with SPN & weak password
```powershell
New-ADUser -Name "svc-backup" -AccountPassword (ConvertTo-SecureString "Summer2024!" -AsPlainText -Force) -Enabled $true -Path "OU=CorpNet-Lab,DC=corpnet,DC=local"

setspn -A MSSQLSvc/db01.corpnet.local:1433 svc-backup

setspn -L svc-backup
```
### Create a new user & enables Kerberos pre-authentication bypass
```powershell
New-ADUser -Name "j.melnyk" -AccountPassword (ConvertTo-SecureString "P@ssw0rd123" -AsPlainText -Force) -Enabled $true -Path "OU=CorpNet-Lab,DC=corpnet,DC=local"

Set-ADAccountControl -Identity "j.melnyk" -DoesNotRequirePreAuth $true
```
### Create service account with Replication Rights
```powershell
New-ADUser -Name "svc-monitor" -AccountPassword (ConvertTo-SecureString "Monitor2024!" -AsPlainText -Force) -Enabled $true -Path "OU=CorpNet-Lab,DC=corpnet,DC=local"

dsacls "DC=corpnet,DC=local" /G "corpnet\svc-monitor:CA;Replicating Directory Changes All"

dsacls "DC=corpnet,DC=local" /G "corpnet\svc-monitor:CA;Replicating Directory Changes"
```
## Setting up Network Interfaces
### Windows Server (D01)
```powershell
Get-NetAdapter

New-NetIPAddress -InterfaceAlias "Ethernet0" -IPAddress 192.168.100.1 -PrefixLength 24
```
### Ubuntu server (SENSOR)
```bash
ip a

sudo nano /etc/netplan/01-sensor-static.yaml

#01-sensor-static.yaml
network:
  version: 2
  ethernets:
    ens33:
      dhcp4: no
      addresses:
        - 192.168.100.10/24

sudo chmod 600 /etc/netplan/01-sensor-static.yaml

sudo netplan apply
```
### Kali Linux (ATTACKER)
```bash
ip a

sudo nmcli con add type ethernet ifname eth0 con-name kali-static ip4 192.168.100.20/24

sudo nmcli con up kali-static
```
### Metasploitable2 (VICTIM)
```bash
ip a

sudo nano /etc/network/interfaces

#/etc/network/interfaces
auto lo
iface lo inet loopback

auto eth0
iface eth0 inet static
    address 192.168.100.30
    netmask 255.255.255.0

sudo /etc/init.d/networking restart
```