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