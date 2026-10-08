### Extend logic space on SENSOR VM (if not enough space error)
```bash
sudo lvextend -r -l +100%FREE /dev/mapper/ubuntu--vg-ubuntu--lv
```
### Install xrdp for remote connection from host to wazuh access
```bash
sudo apt install xrdp -y
```
### Install chromium browser
```bash
sudo apt install chromium-browser -y
```
### Install ntpdate for tyme sync
```bash
sudo apt install ntpsec-ntpdate
```
---

