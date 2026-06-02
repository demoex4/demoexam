## Модуль 1 (меняйте ip адреса на нужные)

### ISP 

```bash
hostnamectl hostname ISP.au-team.irpo; exec bash

cd /etc/net/ifaces
mkdir ens18 ens19 ens20

vim ens18/options

# options
TYPE=eth
BOOTPROTO=dhcp
DISABLED=no
SYSTEMD_CONTROLLED=yes

cp ens18/options ens19/options
# меняем bootproto на static
cp ens19/options ens20/options

echo "172.16.1.1/28" > ens19/ipv4address
echo "172.16.2.1/28" > ens20/ipv4address

cd
vim /etc/net/sysctl.conf
# меняем 0 на 1

systemctl restart network
apt-get update && apt-get install iptables tzdata -y

iptables -t nat -A POSTROUTING -o ens18 -j MASQUERADE
iptables-save > /etc/sysconfig/iptables
systemctl restart iptables
systemctl enable --now iptables

timedatectl set-timezone Europe/Moscow
```
### HQ-RTR

```bash
hostname HQ-RTR.au-team.irpo

username net_admin
password P@ssw0rd
role admin
exit

ip route 0.0.0.0/0 172.16.1.1
security none
ip name-server 192.168.10.2

port te0
service-instance to-isp
encapsulation untagged
exit

port te1
service-instance to-srv100
encapsulation dot1q 100
rewrite pop 1
exit
service-instance to-cli200
encapsulation dot1q 200
rewrite pop 1
exit
service-instance VL999
encapsulation dot1q 999
rewrite pop 1
exit

interface to-isp
ip nat outside
ip address 172.16.1.2/28
connect port te0 service-instance to-isp
exit

interface to-srv100
ip nat inside
ip address 192.168.10.1/27
connect port te1 service-instance to-srv100
exit

interface to-cli200
ip nat inside
ip address 192.168.20.1/28
connect port te1 service-instance to-cli200
exit

interface VL999
ip address 192.168.99.1/29
connect port te1 service-instance VL999
exit

interface tunnel.1
ip address 10.0.0.1/30
ip tunnel 172.16.1.2 172.16.2.2 mode gre
ip ospf authentication
ip ospf authentication-key P@ssw0rd
exit

router ospf 1
router-id 10.10.10.1
passive-interface default
no passive-interface tunnel.1
network 10.0.0.0/30 area 1
network 192.168.10.0/27 area 1
network 192.168.20.0/28 area 1
network 192.168.99.0/29 area 1
exit

ip nat pool 100 192.168.10.1-192.168.10.10
ip nat pool 200 192.168.20.1-192.168.20.10
ip nat source dynamic inside-to-outside pool 100 overload interface to-isp
ip nat source dynamic inside-to-outside pool 200 overload interface to-isp

ip pool CLI 192.168.20.1-192.168.20.10
dhcp-server 1
pool CLI 1
dns 192.168.10.2
domain-search au-team.irpo
gateway 192.168.20.1
mask 28
exit

interface to-cli200
dhcp-server 1
exit

ntp timezone utc+3
exit
write memory
```
### BR-RTR

```bash
hostname BR-RTR.au-team.irpo

username net_admin
password P@ssw0rd
role admin
exit

ip route 0.0.0.0/0 172.16.2.1
security none
ip name-server 192.168.10.2

port te0
service-instance to-isp
encapsulation untagged
exit

port te1
service-instance to-srv
encapsulation untagged
exit

interface to-isp
ip nat outside
ip address 172.16.2.2/28
connect port te0 service-instance to-isp
exit

interface to-srv
ip nat inside
ip address 192.168.1.1/28
connect port te1 service-instance to-srv
exit

interface tunnel.1
ip address 10.0.0.2/30
ip tunnel 172.16.2.2 172.16.1.2 mode gre
ip ospf authentication
ip ospf authentication-key P@ssw0rd
exit

router ospf 1
router-id 10.10.10.2
passive-interface default
no passive-interface tunnel.1
network 10.0.0.0/30 area 1
network 192.168.1.0/28 area 1
exit

ip nat pool SRV 192.168.1.1-192.168.1.10
ip nat source dynamic inside-to-outside pool SRV overload interface to-isp

ntp timezone utc+3
exit
write memory
```
### BR-SRV

```bash
hostnamectl hostname BR-SRV.au-team.irpo; exec bash

vim /etc/net/ifaces/ens18/options

# options
TYPE=eth
BOOTPROTO=static
DISABLED=no
SYSTEMD_CONTROLLED=yes

echo "192.168.1.2/28" > /etc/net/ifaces/ens18/ipv4address
echo "default via 192.168.1.1" > /etc/net/ifaces/ens18/ipv4route

vim /etc/net/sysctl.conf
# меняем 0 на 1

vim /etc/resolv.conf

# resolv.conf
search au-team.irpo
nameserver 192.168.10.2

systemctl restart network

# меняйте имя и пароль на свои
useradd sshuser -u 2026 
passwd sshuser P@ssw0rd
usermod -aG wheel sshuser
echo "sshuser ALL=(ALL:ALL) NOPASSWD: ALL" >> /etc/sudoers

vim /etc/openssh/sshd_config

# sshd_config
Port 2026
AllowUsers sshuser
MaxAuthTries 2
Banner /etc/banner

echo "Authorized access only" > /etc/banner
systemctl restart sshd
```
### HQ-SRV

```bash
hostnamectl hostname HQ-SRV.au-team.irpo; exec bash

vim /etc/net/ifaces/ens18/options

# options
TYPE=eth
BOOTPROTO=static
DISABLED=no
SYSTEMD_CONTROLLED=yes

echo "192.168.10.2/27" > /etc/net/ifaces/ens18/ipv4address
echo "default via 192.168.10.1" > /etc/net/ifaces/ens18/ipv4route

vim /etc/net/sysctl.conf
# меняем 0 на 1

vim /etc/resolv.conf

# resolv.conf
nameserver 77.88.8.8

systemctl restart network

# меняйте имя и пароль на свои
useradd sshuser -u 2026
passwd sshuser  # P@ssw0rd
usermod -aG wheel sshuser
echo "sshuser ALL=(ALL:ALL) NOPASSWD: ALL" >> /etc/sudoers

vim /etc/openssh/sshd_config

# sshd_config
Port 2026
AllowUsers sshuser
MaxAuthTries 2
Banner /etc/banner

echo "Authorized access only" > /etc/banner
systemctl restart sshd

apt-get update && apt-get install bind bind-utils -y

vim /etc/bind/options.conf

# options.conf
listen-on { any; };
forwarders { 77.88.8.8; };
allow-query { any; };

vim /etc/bind/rfc1912.conf

# rfc1912.conf
zone "au-team.irpo" {
    type master;
    file "au-team";
};
zone "168.192.in-addr.arpa" {
    type master;
    file "192.168";
};

cd /etc/bind/zone
cp localdomain au-team
cp 127.in-addr.arpa 192.168

vim au-team

# au-team
IN NS HQ-SRV.au-team.irpo.
docker  IN A 172.16.1.1
web     IN A 172.16.2.1
hq-rtr  IN A 192.168.10.1
hq-rtr  IN A 192.168.20.1
br-rtr  IN A 192.168.1.1
hq-cli  IN A 192.168.20.2
hq-srv  IN A 192.168.10.2
br-srv  IN A 192.168.1.2

vim 192.168

# 192.168
IN NS HQ-SRV.au-team.irpo.
1.10   IN PTR HQ-RTR.au-team.irpo.
2.10   IN PTR HQ-SRV.au-team.irpo.
2.20   IN PTR HQ-CLI.au-team.irpo.

chown root:named au-team
chown root:named 192.168

vim /etc/resolv.conf

# resolv.conf
search au-team.irpo
nameserver 192.168.10.2

systemctl restart bind
systemctl enable --now bind
```
### HQ-CLI

```bash
su -
toor
hostnamectl hostname HQ-CLI.au-team.irpo; exec bash

vim /etc/net/sysctl.conf
# меняем 0 на 1
```
## Модуль 2 (меняйте ip адреса на нужные)

### 1. Настройка контроллера домена Samba DC на BR-SRV

### BR-SRV

```bash
apt-get update && apt-get install task-samba-dc chrony ansible sshpass docker-engine docker-compose-v2 python3-module-pip -y
rm -f /etc/samba/smb.conf
samba-tool domain provision
# AU-TEAM.IRPO, AU-TEAM, dc, SAMBA_INTERNAL, 192.168.0.10, P@ssw0rd

systemctl enable --now samba
cp /var/lib/samba/private/krb5.conf /etc/krb5.conf
systemctl restart samba

# проверка
samba-tool domain info 127.0.0.1

# в /etc/resolv.conf поменять nameserver на 127.0.0.1

kinit administrator@AU-TEAM.IRPO
# P@ssw0rd

samba-tool group add hq
samba-tool user add hquser1 P@ssw0rd
samba-tool user add hquser2 P@ssw0rd
samba-tool user add hquser3 P@ssw0rd
samba-tool user add hquser4 P@ssw0rd
samba-tool user add hquser5 P@ssw0rd
samba-tool group addmembers hq hquser1,hquser2,hquser3,hquser4,hquser5

# проверка
samba-tool group listmembers hq
```
### HQ-CLI

```bash
# меню → центр управления → центр управления системой (toor) → Ethernet-интерфейсы → конфигурация вручную
# 192.168.0.72/28, DNS: 192.168.10.10

systemctl restart NetworkManager

# меню → центр управления → центр управления системой (toor) → аутентификация → поставить галочку на «Домен AD» → применить (P@ssw0rd)

roleadd hq wheel

vim /etc/sudoers

# /etc/sudoers
Cmnd_Alias SHELLCMD = /bin/cat, /bin/grep, /usr/bin/id
WHEEL_USERS ALL=(ALL:ALL) SHELLCMD

systemctl restart network
reboot

# проверка: заходим с hquser и пробуем
sudo id | sudo cat /etc/hosts | sudo grep '127.0.0.1' /etc/hosts | sudo su --
```
### 2. Дисковый массив RAID 0 

### HQ-SRV

```bash
lsblk
mdadm --create /dev/md0 -l 0 -n 2 /dev/sdb /dev/sdc
mkfs.ext4 /dev/md0

vim /etc/fstab

# /etc/fstab
/dev/md0 /raid ext4 defaults 0 0

mkdir /raid
mount -av
df -h
```
### 3. Настройка NFS-сервера 

### HQ-SRV

```bash
apt-get update && apt-get install nfs-server nfs-utils chrony lamp-server -y

mkdir /raid/nfs
chmod 777 /raid/nfs

vim /etc/exports

# /etc/exports
/raid/nfs 192.168.0.64/28(rw,no_subtree_check,no_root_squash)

exportfs -arv
systemctl enable --now nfs-server
```
### HQ-CLI

```bash
apt-get update && apt-get install nfs-clients nfs-utils chrony yandex-browser -y

su -
toor
vim /etc/net/sysctl.conf
# меняем 0 на 1

apt-get update && apt-get install nfs-clients nfs-utils -y

mkdir /mnt/nfs
chmod 777 /mnt/nfs

vim /etc/fstab

# /etc/fstab
192.168.0.10:/raid/nfs /mnt/nfs nfs defaults 0 0

mount -av
df -h
```
### 4. Настройка NTP (chrony) 

### ISP

```bash
apt-get update && apt-get install nginx apache2-htpasswd -y

vim /etc/chrony.conf

# комментируем pool
allow 0.0.0.0/0
local stratum 5

systemctl restart chronyd
chronyc tracking
```
### HQ-RTR

```bash
ntp server 172.16.4.1
exit
write memory
```
### BR-RTR

```bash
ntp server 172.16.5.1
exit
write memory
```
### HQ-SRV, HQ-CLI, BR-SRV

```bash
apt-get update && apt-get install chrony -y

vim /etc/chrony.conf

# комментируем pool
server 172.16.4.1 iburst   # или 172.16.5.1

systemctl restart chronyd
chronyc tracking
```
### 5. Настройка Ansible

### BR-SRV

```bash
apt-get update && apt-get install ansible sshpass -y

vim /etc/ansible/hosts

[all:vars]
ansible_python_interpreter=/usr/bin/python3

HQ-SRV ansible_host=192.168.0.10 ansible_user=sshuser ansible_password=P@ssw0rd ansible_port=2026
HQ-CLI ansible_host=192.168.0.72 ansible_user=user ansible_password=P@ssw0rd
HQ-RTR ansible_host=192.168.0.1 ansible_user=net_admin ansible_password=P@ssw0rd ansible_connection=network_cli ansible_network_os=ios
BR-RTR ansible_host=192.168.10.1 ansible_user=net_admin ansible_password=P@ssw0rd ansible_connection=network_cli ansible_network_os=ios

vim /etc/ansible/ansible.cfg

inventory = /etc/ansible/hosts
host_key_checking = False

ansible-galaxy collection install cisco.ios
apt-get install -y python3-module-pip
pip3 install ansible-pylibssh

# на HQ и BR SRV в /etc/openssh/sshd_config поменять порт с 2024 и 2026
# на HQ-CLI и BR-SRV: systemctl enable --now sshd
# на HQ-RTR, BR-RTR: security none

ansible -m ping all
```
### 6. Docker-контейнеры 

### BR-SRV

```bash
apt-get install -y docker-engine docker-compose-v2
systemctl enable --now docker.service

mount /dev/sr0 /mnt/
docker load < /mnt/docker/site_latest.tar
docker load < /mnt/docker/mariadb_latest.tar
docker image ls

vim compose.yaml

services:
  database:
    container_name: db
    image: mariadb:10.11
    restart: always
    ports:
      - "3306:3306"
    environment:
      MARIADB_DATABASE: "testdb"
      MARIADB_USER: "testc"
      MARIADB_PASSWORD: "P@ssw0rd"
      MARIADB_ROOT_PASSWORD: "toor"
  app:
    container_name: testapp
    image: site:latest
    restart: always
    ports:
      - "8080:8000"
    environment:
      DB_TYPE: "maria"
      DB_HOST: "192.168.10.10"
      DB_PORT: "3306"
      DB_NAME: "testdb"
      DB_USER: "testc"
      DB_PASS: "P@ssw0rd"
    depends_on:
      - database

docker compose up -d

# Проверка: в браузере HQ-CLI открыть 192.168.10.10:8080
```
### 7. Веб-приложение (Apache + MariaDB)

### HQ-SRV

```bash
apt-get update && apt-get install -y lamp-server

mount /dev/sr0 /mnt/
cp /mnt/web/index.php /var/www/html
cp /mnt/web/logo.png /var/www/html

vim /var/www/html/index.php

$servername = "localhost";
$username = "webc";
$password = "P@ssw0rd";
$dbname = "webdb";

systemctl enable --now mariadb

mariadb -u root

CREATE DATABASE webdb;
CREATE USER 'webc'@'localhost' IDENTIFIED BY 'P@ssw0rd';
GRANT ALL PRIVILEGES ON webdb.* TO 'webc'@'localhost' WITH GRANT OPTION;
EXIT;

mariadb webdb < /mnt/web/dump.sql

# проверка
mariadb -u root -e "USE webdb; SHOW TABLES;"

systemctl enable --now httpd2

# Проверка: в браузере HQ-CLI открыть 192.168.0.10
```
### 8. Статическая трансляция портов (NAT) на маршрутизаторах

### HQ-RTR

```bash
ip nat source static tcp 192.168.0.10 80 172.16.4.2 8080
ip nat source static tcp 192.168.0.10 2026 172.16.4.2 2026
write memory
```
### BR-RTR

```bash
ip nat source static tcp 192.168.10.10 8080 172.16.5.2 8080
ip nat source static tcp 192.168.10.10 2026 172.16.5.2 2026
write memory
```
### 9. Обратный прокси (nginx)

### ISP

```bash
apt-get update && apt-get install nginx -y

vim /etc/nginx/sites-available.d/default.conf

server {
    listen 80;
    server_name web.au-team.irpo;
    location / {
        proxy_pass http://172.16.4.2:8080;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}

server {
    listen 80;
    server_name docker.au-team.irpo;
    location / {
        proxy_pass http://172.16.5.2:8080;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}

ln -s /etc/nginx/sites-available.d/default.conf /etc/nginx/sites-enabled.d/
nginx -t
systemctl restart nginx
systemctl enable --now nginx
```
### BR-RTR

```bash
vim /etc/hosts

# /etc/hosts
172.16.4.1 web.au-team.irpo
172.16.5.1 docker.au-team.irpo

systemctl restart network
# в браузере: http://web.au-team.irpo и http://docker.au-team.irpo
```
### 10. Web-based аутентификация

### ISP

```bash
apt-get update && apt-get install apache2-htpasswd -y

htpasswd -c /etc/nginx/.htpasswd WEB
# P@ssw0rd

vim /etc/nginx/sites-available.d/default.conf
# добавить в секцию server web.au-team.irpo:

auth_basic "Restricted area";
auth_basic_user_file /etc/nginx/.htpasswd;

nginx -t
systemctl restart nginx

# Проверка: в браузере HQ-CLI http://web.au-team.irpo → ввести WEB / P@ssw0rd
```
