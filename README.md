# altubunta2026
//







Модуль 1: Настройка сетевой инфраструктуры (с нуля)

    Цель: Полностью поднять сеть с нуля по схеме. Все IP, имена, маршруты настраиваются вручную.
    Устройства: ISP, HQ-RTR, BR-RTR, HQ-SRV, BR-SRV, HQ-CLI, HQ-SW (опционально).

1. Базовая настройка устройств
Альт (Сервер/Раб. станция): HQ-SRV, BR-SRV, HQ-CLI, ISP
bash

# Установить имя хоста и полное доменное имя
hostnamectl set-hostname <hq-srv|br-srv|hq-cli|isp>.au-team.irpo; exec bash

# Проверка
hostname -f
reboot # Рекомендуется после смены имени

EcoRouterOS: HQ-RTR, BR-RTR
bash

enable
configure terminal
hostname <hq-rtr|br-rtr>
ip domain-name au-team.irpo
write memory

# Проверка
show hostname
show running-config | include domain-name

2. Настройка IPv4-адресов
Таблица 1.2 (Пример из методички)
Имя устройства	IP-адрес	Шлюз по умолчанию
HQ-RTR	172.16.1.2/28, 192.168.100.1/27, 192.168.200.1/24, 192.168.99.1/29	172.16.1.1
BR-RTR	172.16.2.2/28, 192.168.0.1/28	172.16.2.1
HQ-SRV	192.168.100.2/27	192.168.100.1
HQ-CLI	192.168.200.2/24 (или по DHCP)	192.168.200.1
BR-SRV	192.168.0.2/28	192.168.0.1
Альт (ISP, HQ-SRV, BR-SRV, HQ-CLI)
bash

# 1. Посмотреть интерфейсы
ip -c a

# 2. Настроить статический IP (пример для HQ-SRV, интерфейс ensXX)
echo "192.168.100.2/27" > /etc/net/ifaces/ensXX/ipv4address
echo "default via 192.168.100.1" > /etc/net/ifaces/ensXX/ipv4route
echo "nameserver 77.88.8.8" > /etc/net/ifaces/ensXX/resolv.conf

# 3. Прописать TYPE и BOOTPROTO в options
echo "TYPE=eth" > /etc/net/ifaces/ensXX/options
echo "BOOTPROTO=static" >> /etc/net/ifaces/ensXX/options

# 4. Применить настройки
systemctl restart network

# Проверка
ip -c a
ip -c r

EcoRouterOS (HQ-RTR, BR-RTR)
bash

configure terminal
# 1. Создать L3-интерфейсы и дать им IP
interface <v100|v200|v999|isp|int1>
 description "VLAN 100"
 ip address 192.168.100.1/27
 exit

# 2. Настроить порт и связать его с L3-интерфейсами через service-instance
port ge1
 service-instance ge1/v100
  encapsulation dot1q 100 exact
  rewrite pop 1
  connect ip interface v100
  exit
 service-instance ge1/v200
  encapsulation dot1q 200 exact
  rewrite pop 1
  connect ip interface v200
  exit
 # ... аналогично для VLAN 999
exit

# 3. Настроить интерфейс в сторону ISP (без VLAN)
port ge2
 service-instance ge2/isp
  encapsulation untagged
  connect ip interface isp
  exit
exit

# 4. Задать маршрут по умолчанию
ip route 0.0.0.0/0 <IP_шлюза_ISP> # например, 172.16.1.1
write memory

# Проверка
show ip interface brief
show service-instance brief

3. Настройка доступа в Интернет
ISP (Альт JeOS)
bash

# 1. Настроить DHCP на внешнем интерфейсе (ensXX, который смотрит "в мир")
echo "BOOTPROTO=dhcp" > /etc/net/ifaces/ensXX/options
echo "TYPE=eth" >> /etc/net/ifaces/ensXX/options
systemctl restart network

# 2. Настроить статические IP для HQ-RTR и BR-RTR
echo "172.16.1.1/28" > /etc/net/ifaces/ensYY/ipv4address
echo "172.16.2.1/28" > /etc/net/ifaces/ensZZ/ipv4address

# 3. Включить IP-форвардинг (маршрутизацию)
sed -i 's/net.ipv4.ip_forward = 0/net.ipv4.ip_forward = 1/g' /etc/net/sysctl.conf
sysctl -p # или systemctl restart network

# 4. Настроить NAT (MASQUERADE) для локальных сетей
apt-get update && apt-get install -y iptables
iptables -t nat -A POSTROUTING -s 172.16.1.0/28 -o ensXX -j MASQUERADE
iptables -t nat -A POSTROUTING -s 172.16.2.0/28 -o ensXX -j MASQUERADE
iptables-save > /etc/sysconfig/iptables
systemctl enable --now iptables

# Проверка
ping 77.88.8.8

4. Создание локальных пользователей
Альт (HQ-SRV, BR-SRV)
bash

# Пользователь sshuser с UID 2026 и паролем P@ssw0rd
useradd sshuser -u 2026
passwd sshuser # Ввести P@ssw0rd

# Добавить в группу wheel и дать sudo без пароля
usermod -aG wheel sshuser
echo "sshuser ALL=(ALL:ALL) NOPASSWD: ALL" >> /etc/sudoers

# Проверка
su - sshuser
sudo -i # Должен сработать без пароля

EcoRouterOS (HQ-RTR, BR-RTR)
bash

configure terminal
username net_admin
password P@ssw0rd
role admin
write memory

# Проверка
# Выйти и зайти за net_admin
show users localdb

5. Настройка безопасного SSH
Альт (HQ-SRV, BR-SRV)
bash

# Редактируем /etc/openssh/sshd_config
vim /etc/openssh/sshd_config

# Меняем/добавляем строки:
Port 2026
AllowUsers sshuser
MaxAuthTries 2
Banner /etc/openssh/banner

# Создаем баннер
echo "Authorized access only" > /etc/openssh/banner

# Перезапускаем SSH
systemctl restart sshd

# Проверка
ssh -p 2026 sshuser@localhost

6. Настройка IP-туннеля (на выбор: GRE или IP-in-IP)
EcoRouterOS (HQ-RTR и BR-RTR)
bash

configure terminal
interface tunnel.0
 description "GRE or IP-in-IP"
 ip address 10.10.10.1/30 # HQ: .1, BR: .2
 ip mtu 1400 # Только для IP-in-IP!
 ip tunnel <source_IP> <dest_IP> mode <gre|ipip>
 exit
write memory

# Проверка
show interface tunnel.0
ping 10.10.10.2 # С HQ на BR

7. Настройка динамической маршрутизации (OSPF)
EcoRouterOS (HQ-RTR и BR-RTR)
bash

configure terminal
router ospf 1
 ospf router-id 10.10.10.1 # HQ: .1, BR: .2
 passive-interface default
 no passive-interface tunnel.0
 network 192.168.100.0/27 area 0
 network 192.168.200.0/24 area 0
 network 192.168.0.0/28 area 0
 # Аутентификация
 area 0 authentication
 interface tunnel.0
  ip ospf authentication-key P@ssw0rd
  exit
exit
write memory

# Проверка
show ip ospf neighbor
show ip route ospf

8. Настройка NAT для офисов
EcoRouterOS (HQ-RTR, BR-RTR)
bash

configure terminal
# Определить inside и outside интерфейсы
interface <v100|v200|v999|int1>
 ip nat inside
 exit
interface isp
 ip nat outside
 exit

# Настроить динамический NAT (PAT)
ip nat source dynamic inside-to-outside pool VLAN100 overload interface isp
# ... аналогично для VLAN200, VLAN999, BR-Net
write memory

# Проверка
show ip nat translations

9. Настройка DHCP-сервера для HQ-CLI
EcoRouterOS (HQ-RTR)
bash

configure terminal
# Создать пул адресов (исключая адрес шлюза .1)
ip pool VLAN200 192.168.200.2-192.168.200.254

# Настроить DHCP-сервер
dhcp-server 1
 pool VLAN200 1
  mask 24
  gateway 192.168.200.1
  dns 192.168.100.2 # HQ-SRV как DNS
  domain-name au-team.irpo
  exit
exit

# Привязать к интерфейсу
interface v200
 dhcp-server 1
 exit
write memory

# Проверка на HQ-CLI
# Настроить интерфейс на получение IP по DHCP
echo "BOOTPROTO=dhcp" > /etc/net/ifaces/ensXX/options
systemctl restart network
ip -c a

10. Настройка DNS-сервера (BIND) на HQ-SRV
Альт (HQ-SRV)
bash

# Установка
apt-get update && apt-get install -y bind bind-utils

# Редактируем /var/lib/bind/etc/options.conf
vim /var/lib/bind/etc/options.conf
# Добавить/проверить:
# listen-on { 192.168.100.2; };
# forwarders { 77.88.8.8; 77.88.8.3; };
# allow-query { 192.168.100.0/27; 192.168.200.0/24; 192.168.0.0/28; 127.0.0.1; };

# Редактируем /var/lib/bind/etc/rfc1912.conf - добавляем зоны
vim /var/lib/bind/etc/rfc1912.conf
# zone "au-team.irpo" { type master; file "master/au-team.irpo"; };
# zone "100.168.192.in-addr.arpa" { type master; file "master/100.168.192.in-addr.arpa"; };
# zone "200.168.192.in-addr.arpa" { type master; file "master/200.168.192.in-addr.arpa"; };

# Создаем файлы зон, копируя из empty
cp /var/lib/bind/etc/zone/empty /var/lib/bind/etc/zone/au-team.irpo
cp /var/lib/bind/etc/zone/empty /var/lib/bind/etc/zone/100.168.192.in-addr.arpa
cp /var/lib/bind/etc/zone/empty /var/lib/bind/etc/zone/200.168.192.in-addr.arpa

# Редактируем файлы зон, добавляя A и PTR записи по таблице 1.3
vim /var/lib/bind/etc/zone/au-team.irpo
# hq-srv  A 192.168.100.2
# hq-rtr  A 192.168.100.1
# и т.д.

# Даем права и запускаем
chown -R root:named /var/lib/bind/etc/zone/*
systemctl enable --now bind

# Проверка
host hq-srv.au-team.irpo 127.0.0.1

11. Настройка часового пояса
Альт (все)
bash

timedatectl set-timezone Europe/Moscow
timedatectl

EcoRouterOS (все)
bash

configure terminal
ntp timezone utc+3
write memory
show ntp timezone

Модуль 2: Организация сетевого администрирования (преднастроено)

    Цель: Развернуть домен, файловые службы, веб-приложения, Ansible.
    Устройства: те же, но с преднастроенной сетью, маршрутизацией, NAT, DNS, DHCP.
    Важно: У HQ-SRV есть два дополнительных диска по 1 ГБ.

1. Настройка Samba DC (контроллер домена)
BR-SRV (Альт Сервер)
bash

# Установка
apt-get update && apt-get install -y task-samba-dc

# Остановка конфликтующих служб
systemctl stop krb5kdc slapd bind
systemctl disable krb5kdc slapd bind

# Очистка предыдущих конфигураций
rm -f /etc/samba/smb.conf
rm -rf /var/lib/samba

# Развертывание домена
samba-tool domain provision --use-rfc2307 --interactive
# Внимательно ответить на вопросы:
# - Realm: AU-TEAM.IRPO
# - Domain: AU-TEAM
# - DNS forwarder: 77.88.8.8
# - Administrator password: <пароль не менее 7 символов>

# Настроить Kerberos
cp /var/lib/samba/private/krb5.conf /etc/krb5.conf

# Запустить Samba
systemctl enable --now samba

# Проверка
samba-tool domain info 127.0.0.1
kinit administrator@AU-TEAM.IRPO
klist

2. Создание пользователей и групп в домене
BR-SRV
bash

# Создать группу hq
samba-tool group add hq

# Создать 5 пользователей и добавить в группу
samba-tool user add hquser1 P@ssw0rd
samba-tool user setexpiry hquser1 --noexpiry
samba-tool group addmembers hq hquser1
# ... для hquser2, hquser3 и т.д. (можно циклом)

3. Ввод клиента (HQ-CLI) в домен и настройка sudo
HQ-CLI (Альт Раб. станция)
bash

# 1. Указать DNS-сервером BR-SRV (192.168.0.2)
echo "nameserver 192.168.0.2" > /etc/net/ifaces/ensXX/resolv.conf
systemctl restart network

# 2. Установить SSSD
apt-get update && apt-get install -y task-auth-ad-sssd

# 3. Ввести в домен (удобнее через Центр Управления Системой (ЦУС))
# Пользователи -> Аутентификация -> Домен Active Directory
# SSSD (в единственном домене), указать домен AU-TEAM.IRPO, ввести admin@au-team.irpo

# 4. Установить libnss-role
apt-get install -y libnss-role

# 5. Связать доменную группу 'hq' с локальной группой 'wheel'
roleadd hq wheel

# 6. Настроить sudo для wheel (без пароля, только cat, grep, id)
echo "%wheel ALL=(ALL:ALL) NOPASSWD: /bin/cat, /bin/grep, /usr/bin/id" >> /etc/sudoers

# Проверка
# Перезагрузиться и войти как AU-TEAM\hquser1
sudo cat /etc/hosts # должно сработать
sudo useradd test # должно НЕ сработать

4. Настройка RAID 0 и NFS
HQ-SRV
bash

# 1. Создать RAID 0 из дисков /dev/sdb и /dev/sdc
apt-get install -y mdadm
mdadm --create --verbose /dev/md0 -l 0 -n 2 /dev/sdb /dev/sdc
mdadm --detail --scan --verbose | tee -a /etc/mdadm.conf

# 2. Создать ФС и смонтировать
mkfs.ext4 /dev/md0
mkdir /raid
echo "/dev/md0 /raid ext4 defaults 0 0" >> /etc/fstab
mount -av

# 3. Настроить NFS-сервер
apt-get install -y nfs-server nfs-utils
mkdir /raid/nfs
chmod 777 /raid/nfs
echo "/raid/nfs 192.168.200.0/24(rw,no_root_squash)" >> /etc/exports
systemctl enable --now nfs-server

# Проверка на HQ-CLI
apt-get install -y nfs-utils nfs-clients
mkdir /mnt/nfs
echo "192.168.100.2:/raid/nfs /mnt/nfs nfs defaults 0 0" >> /etc/fstab
mount -av
touch /mnt/nfs/test.txt

5. Настройка Ansible на BR-SRV
BR-SRV
bash

# Установка
apt-get update && apt-get install -y ansible sshpass
ansible-galaxy collection install ansible.netcommon cisco.ios
apt-get install -y python3-module-pip
pip3 install ansible-pylibssh

# Настроить инвентарь /etc/ansible/hosts
echo "[servers]" > /etc/ansible/hosts
echo "hq-srv ansible_host=192.168.100.2" >> /etc/ansible/hosts
echo "hq-cli ansible_host=192.168.200.2" >> /etc/ansible/hosts
echo "[routers]" >> /etc/ansible/hosts
echo "hq-rtr ansible_host=192.168.100.1" >> /etc/ansible/hosts
echo "br-rtr ansible_host=192.168.0.1" >> /etc/ansible/hosts
echo "[all:vars]" >> /etc/ansible/hosts
echo "ansible_user=sshuser" >> /etc/ansible/hosts
echo "ansible_ssh_common_args='-o StrictHostKeyChecking=no'" >> /etc/ansible/hosts

# Для EcoRouter разрешить SSH без профиля безопасности (временно!)
# На HQ-RTR и BR-RTR: configure terminal -> security none

# Проверка
ansible all -m ping -k # введем пароль P@ssw0rd

6. Развертывание Docker-приложения (testapp) на BR-SRV
BR-SRV
bash

# Установка Docker
apt-get install -y docker-engine docker-compose-v2
systemctl enable --now docker

# Монтируем ISO с доп. материалами
mount /dev/cdrom /mnt # или /dev/sr0

# Импортируем образы
docker load < /mnt/docker/site_latest.tar
docker load < /mnt/docker/mariadb_latest.tar

# Создаем docker-compose.yml
vim compose.yaml

yaml

version: '3'
services:
  db:
    image: mariadb_latest
    container_name: db
    environment:
      MYSQL_ROOT_PASSWORD: rootpass
      MYSQL_DATABASE: mariadb
      MYSQL_USER: maria
      MYSQL_PASSWORD: Password
    ports:
      - "3306:3306"
  web:
    image: site_latest
    container_name: testapp
    ports:
      - "8080:80"
    depends_on:
      - db
    environment:
      DB_HOST: db
      DB_USER: maria
      DB_PASSWORD: Password
      DB_NAME: mariadb

bash

# Запуск
docker-compose up -d

# Проверка
docker ps
curl localhost:8080

7. Развертывание веб-приложения (Apache/PHP) на HQ-SRV
HQ-SRV
bash

# Установка LAMP
apt-get install -y lamp-server

# Копируем файлы приложения из ISO
cp /mnt/web/index.php /var/www/html/
cp /mnt/web/logo.png /var/www/html/

# Настройка MariaDB
systemctl enable --now mariadb
mysql -u root
CREATE DATABASE webdb;
CREATE USER 'web'@'localhost' IDENTIFIED BY 'P@ssw0rd';
GRANT ALL PRIVILEGES ON webdb.* TO 'web'@'localhost';
FLUSH PRIVILEGES;
EXIT;

# Импорт БД
mysql -u web -p webdb < /mnt/web/dump.sql # пароль P@ssw0rd

# Указать пароль в БД в файле index.php
vim /var/www/html/index.php
# Найти строки подключения и ввести: пользователь web, пароль P@ssw0rd, БД webdb

# Запуск Apache
systemctl enable --now httpd2

# Проверка
curl localhost

8. Настройка статического NAT (проброс портов)
EcoRouterOS (HQ-RTR, BR-RTR)
bash

configure terminal
# На HQ-RTR: проброс 8080->80 на HQ-SRV и 2026->2026
ip nat source static tcp 192.168.100.2 80 172.16.1.2 8080
ip nat source static tcp 192.168.100.2 2026 172.16.1.2 2026

# На BR-RTR: проброс 8080->8080 (testapp) и 2026->2026
ip nat source static tcp 192.168.0.2 8080 172.16.2.2 8080
ip nat source static tcp 192.168.0.2 2026 172.16.2.2 2026

write memory
show ip nat translations

9. Настройка обратного прокси (Nginx) на ISP
ISP
bash

# Установка Nginx
apt-get update && apt-get install -y nginx

# Настройка виртуальных хостов в /etc/nginx/sites-available.d/default.conf
vim /etc/nginx/sites-available.d/default.conf

nginx

server {
    listen 80;
    server_name web.au-team.irpo;
    location / {
        proxy_pass http://172.16.1.2:8080;
    }
}
server {
    listen 80;
    server_name docker.au-team.irpo;
    location / {
        proxy_pass http://172.16.2.2:8080;
    }
}

bash

# Включить сайт и перезапустить
systemctl enable --now nginx

# На HQ-CLI прописать в /etc/hosts для теста
echo "172.16.1.1 web.au-team.irpo" >> /etc/hosts
echo "172.16.1.1 docker.au-team.irpo" >> /etc/hosts

10. Настройка web-аутентификации на ISP
ISP
bash

# Создать файл паролей
apt-get install -y apache2-utils # утилита htpasswd
htpasswd -c /etc/nginx/.htpasswd WEB
# Ввести пароль P@ssw0rd

# Добавить аутентификацию в location / для web.au-team.irpo
vim /etc/nginx/sites-available.d/default.conf
# Добавить внутрь location / для web.au-team.irpo:
# auth_basic "Authorized access";
# auth_basic_user_file /etc/nginx/.htpasswd;

# Проверить конфиг и перезагрузить
nginx -t
systemctl reload nginx

11. Установка Яндекс.Браузера на HQ-CLI
HQ-CLI
bash

apt-get update && apt-get install -y yandex-browser-stable
# Или через Центр Приложений

Модуль 3: Организация сетевого администрирования (продвинутый уровень)

    Цель: Безопасность, мониторинг, сертификаты ГОСТ, логирование, резервное копирование.
    Устройства: те же, что и в Модуле 2.

1. Массовый импорт пользователей в домен из CSV
BR-SRV
bash

# Смонтировать ISO и скопировать users.csv
mount /dev/cdrom /mnt
cp /mnt/Users.csv /root
iconv -f UTF-8 -t UTF-8//IGNORE /root/Users.csv > /root/Users_fixed.csv

# Создать скрипт import.sh
vim /root/import.sh

bash

#!/bin/bash
while IFS=';' read -r first last role phone ou street zip city country pass; do
  username="${first:0:1}$last"
  samba-tool user create "$username" "$pass" --given-name="$first" --surname="$last" --job-title="$role" --telephone-number="$phone"
  [[ -n "$ou" ]] && samba-tool ou create "OU=$ou" 2>/dev/null
  [[ -n "$ou" ]] && samba-tool user move "$username" "OU=$ou" 2>/dev/null
done < /root/Users_fixed.csv

bash

# Запустить
bash /root/import.sh

2. Настройка ГОСТ-сертификатов (ЦС на HQ-SRV)
HQ-SRV
bash

# 1. Включить поддержку ГОСТ
control openssl-gost enabled

# 2. Создать Центр Сертификации (ЦС)
openssl genpkey -algorithm gost2012_256 -pkeyopt paramset:TCB -out ca.key
openssl req -new -x509 -md_gost12_256 -days 90 -key ca.key -out ca.crt

# 3. Создать ключи и сертификаты для web.au-team.irpo
openssl genpkey -algorithm gost2012_256 -pkeyopt paramset:A -out web.au-team.irpo.key
openssl req -new -md_gost12_256 -key web.au-team.irpo.key -out web.au-team.irpo.csr
openssl x509 -req -in web.au-team.irpo.csr -CA ca.crt -CAkey ca.key -CAcreateserial -out web.au-team.irpo.crt -days 30

# 4. Аналогично для docker.au-team.irpo
# ... (заменить web на docker)

# 5. Скопировать ключи и серты на ISP
scp *.crt *.key root@172.16.1.1:/etc/nginx/

# 6. Настроить HTTPS на ISP (см. след. пункт)

ISP
bash

# Изменить конфиг /etc/nginx/sites-available.d/default.conf, добавив server-блоки на 443 порту
vim /etc/nginx/sites-available.d/default.conf

nginx

server {
    listen 443 ssl;
    server_name web.au-team.irpo;
    ssl_certificate /etc/nginx/web.au-team.irpo.crt;
    ssl_certificate_key /etc/nginx/web.au-team.irpo.key;
    ssl_ciphers GOST2012-KUZNYECHIK-KUZNYECHIKOMAC;
    ssl_protocols TLSv1.2;
    # ... location / с проксированием и auth_basic ...
}
# Аналогичный блок для docker.au-team.irpo

bash

systemctl reload nginx

HQ-CLI
bash

# 1. Установить КриптоПро CSP (из ISO)
# 2. Скопировать сертификат ЦС
scp root@192.168.100.2:/root/ca.crt /etc/pki/ca-trust/source/anchors/
update-ca-trust
# 3. Добавить сертификат в браузер
# Открыть в Яндекс.Браузере https://web.au-team.irpo

3. Настройка IPSec (поверх существующего GRE-туннеля)
EcoRouterOS (HQ-RTR и BR-RTR)
bash

configure terminal
crypto-ipsec ike enable

crypto-ipsec profile IPSEC ike-v2
 mode tunnel
 nat-traversal
 ike-phase1
  proposal aes256-sha256-modp2048
  auth pre-shared-key P@ssw0rd
  exit
 ike-phase2
  protocol esp
  proposal aes256-sha256
  exit
 local-ts 172.16.1.2
 remote-ts 172.16.2.2
exit

crypto-map CMAP 10
 match peer <IP_другого_маршрутизатора>
 set crypto-ipsec profile IPSEC
exit

filter-map ipv4 FMAP 10
 match gre host <IP_local> host <IP_удаленного>
 set crypto-map CMAP peer <IP_удаленного>
exit

interface tunnel.0
 ip mtu 1360
 set filter-map in FMAP 10
exit
write memory
# Проверка
show crypto-ipsec ike security-associations

4. Настройка межсетевого экрана (Filewall) на EcoRouter
EcoRouterOS (HQ-RTR, BR-RTR)
bash

configure terminal
# Удалить правило allow all
no filter-map ipv4 FMAP 30

# Добавить правила разрешения для нужных протоколов
filter-map ipv4 FMAP 21
 match tcp any any eq 88 # Kerberos
 match udp any any eq 88
 match udp any any eq dns
 # ... и т.д. по списку из методички (для AD, ICMP, HTTP/S, NTP, SSH)
 set accept
exit

filter-map ipv4 FMAP 22
 match icmp any any
 match tcp any any eq 8080
 match tcp any any eq http
 # ... и т.д.
 set accept
exit
# Неявный deny в конце
write memory

5. Настройка NTP-сервера (chrony) на ISP
ISP
bash

apt-get update && apt-get install -y chrony
vim /etc/chrony.conf
# Закомментировать pool, добавить:
server ntp.ideco.ru iburst prefer
local stratum 5
allow 0.0.0.0/0

systemctl restart chronyd

Клиенты (HQ-SRV, HQ-CLI, BR-RTR, BR-SRV)
bash

# В /etc/chrony.conf добавить:
server 172.16.1.1 iburst
systemctl restart chronyd
chronyc sources

6. Настройка CUPS (принт-сервер)
HQ-SRV
bash

apt-get update && apt-get install -y cups cups-pdf
# Отредактировать /etc/cups/cupsd.conf, разрешить доступ
vim /etc/cups/cupsd.conf
# В секциях <Location />, <Location /admin>, <Location /admin/conf> добавить "Allow all"
systemctl enable --now cups

HQ-CLI
bash

# В браузере открыть https://hq-srv:631/admin
# Добавить принтер Cups-PDF как сетевой (IPP)

7. Настройка Rsyslog (логи с BR-RTR, BR-SRV на HQ-SRV)
HQ-SRV (сервер логов)
bash

# В /etc/rsyslog.conf раскомментировать/добавить:
module(load="imudp")
input(type="imudp" port="514")

# Создать правила для приема логов
if $fromhost-ip == '10.10.10.2' then /opt/br-rtr/router.log
if $fromhost-ip == '192.168.0.2' then /opt/br-srv/server.log
# & stop

systemctl restart rsyslog

BR-SRV (клиент)
bash

# В /etc/rsyslog.conf добавить в конце:
*.warn @@192.168.100.2:514
systemctl restart rsyslog

8. Настройка мониторинга (Prometheus + Grafana) на HQ-SRV
HQ-SRV
bash

apt-get install -y grafana prometheus prometheus-node_exporter
# Настроить /etc/prometheus/prometheus.yml
vim /etc/prometheus/prometheus.yml
# В секцию scrape_configs -> static_configs -> targets добавить:
# - 'localhost:9090'
# - '192.168.100.2:9100' # сам HQ-SRV
# - '192.168.0.2:9100'   # BR-SRV

systemctl enable --now prometheus grafana-server prometheus-node_exporter

BR-SRV
bash

apt-get install -y prometheus-node_exporter
systemctl enable --now prometheus-node_exporter

Настройка Grafana

    Войти на http://192.168.100.2:3000 (admin/admin, сменить на P@ssw0rd).

    Добавить Data Source: Prometheus (http://localhost:9090).

    Импортировать Dashboard (например, ID 11074).

    Проверить отображение метрик CPU, RAM, HDD.

9. Инвентаризация через Ansible Playbook
BR-SRV
bash

# Создать плейбук /etc/ansible/get_info.yml
vim /etc/ansible/get_info.yml

yaml

- name: Get hosts info
  hosts: hq-srv, hq-cli
  tasks:
    - name: Save info to file
      copy:
        dest: "/etc/ansible/PC-INFO/{{ ansible_hostname }}.yml"
        content: |
          Hostname: {{ ansible_hostname }}
          IP_Address: {{ ansible_default_ipv4.address }}
      delegate_to: localhost

bash

mkdir /etc/ansible/PC-INFO
ansible-playbook get_info.yml
cat /etc/ansible/PC-INFO/*.yml

10. Настройка Fail2ban для защиты SSH
HQ-SRV
bash

apt-get install -y fail2ban
# Создать /etc/fail2ban/jail.local
echo "[sshd]" > /etc/fail2ban/jail.local
echo "enabled = true" >> /etc/fail2ban/jail.local
echo "port = 2026" >> /etc/fail2ban/jail.local
echo "maxretry = 3" >> /etc/fail2ban/jail.local
echo "bantime = 60" >> /etc/fail2ban/jail.local
echo "logpath = %(sshd_log)s" >> /etc/fail2ban/jail.local

systemctl enable --now fail2ban
fail2ban-client status sshd

11. Настройка "Кибер Бэкап" (Cyber Backup)

    Важно: Перед установкой требуется обновить ядро на HQ-SRV! update-kernel && reboot

HQ-SRV (Сервер управления + Агент)
bash

# Монтируем ISO с Cyber Backup 18
mount /dev/cdrom /mnt
bash /mnt/CyberBackup_18_64-bit.x86_64
# В установщике выбрать компоненты: Management Server, Agent for Linux, Agent for MySQL/MariaDB
# После установки создать пользователя iproadmin с паролем P@ssw0rd

HQ-CLI (Узел-хранилище)
bash

mkdir /backup
bash /mnt/CyberBackup_18_64-bit.x86_64
# Выбрать: Agent for Linux, Storage Node
# В веб-интерфейсе сервера (http://hq-srv:9877) добавить этот узел как хранилище
