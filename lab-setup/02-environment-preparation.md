### Environment Preparation (On ATTACKER)
```bash
ping -c 3 192.168.100.1

echo "192.168.100.1 corpnet.local dc01.corpnet.local" | sudo tee -a /etc/hosts
ping -c 1 corpnet.local
ping -c 1 dc01.corpnet.local

sudo ntpdate 192.168.100.1
```
### Install Server Manager for Windows
[Server Manager Install Link here](https://www.microsoft.com/en-us/download/details.aspx?id=45520)
 