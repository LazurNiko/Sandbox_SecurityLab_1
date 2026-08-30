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