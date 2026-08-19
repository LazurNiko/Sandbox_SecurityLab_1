```mermaid
flowchart TD
    subgraph HOST["Host-machine"]
        DC01["<b>DC01</b><br/>Domain Controller<br/>corpnet.local"]
        NIC["Physical host adapter<br/>Wi-Fi / Ethernet"]
        subgraph VMNET["VMnet2 — Host-only Network"]
            KALI["<b>KALI-ATK</b><br/>Attacker<br/>Kali Linux"]
            VICTIM["<b>VICTIM</b><br/>Target<br/>Metasploitable2"]
            SENSOR["<b>SENSOR</b><br/>Montoring<br/>Ubuntu Server (Zeek + Suricata)"]
            DC01 --- KALI
            KALI --- SENSOR
            SENSOR --- VICTIM
            VICTIM --- DC01
        end
    end
    NIC -->|"host-only"| WAN["Home Network / Internet"]

    style DC01 fill:#E6F1FB,stroke:#185FA5,color:#0C447C
    style KALI fill:#FAECE7,stroke:#993C1D,color:#712B13
    style VICTIM fill:#FAEEDA,stroke:#854F0B,color:#633806
    style SENSOR fill:#E1F5EE,stroke:#0F6E56,color:#085041
    style NIC fill:#F1EFE8,stroke:#5F5E5A,color:#2C2C2A
    style WAN fill:#F1EFE8,stroke:#5F5E5A,color:#2C2C2A
    style VMNET fill:none,stroke:#888780,stroke-dasharray: 4 4
    style HOST fill:none,stroke:#5F5E5A
```
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
### Create a service domain account svc-backup with SPN & weak password
```powershell
New-ADUser -Name "svc-backup" -AccountPassword (ConvertTo-SecureString "Summer2024!" -AsPlainText -Force) -Enabled $true -Path "OU=CorpNet-Lab,DC=corpnet,DC=local"

setspn -A MSSQLSvc/db01.corpnet.local:1433 svc-backup

setspn -L svc-backup
```
