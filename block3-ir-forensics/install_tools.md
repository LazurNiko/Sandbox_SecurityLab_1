### Install Docker
```bash
curl -fsSL https://get.docker.com/ -o get-docker.sh
sudo sh get-docker.sh
sudo usermod -aG docker $USER
```
### Install DFIR-IRIS case-management and investigation platform
```bash
git clone https://github.com/dfir-iris/iris-web.git
cd iris-web
cp .env.example .env
# set up .env (passwords, ports)
docker compose up -d
```
  