# 2 модуль — шпаргалка замен для GitHub

> Формат для GitHub: все значения, которые могут измениться на экзамене, выделены через `<mark>...</mark>`.  
> Это не файл для копипаста в терминал, а быстрая визуальная шпаргалка.

## Легенда

| Тип значения | Что искать |
|---|---|
| IP / маска | адреса из таблицы Модуля 2 |
| Домен Samba | `au-team.irpo`, `AU-TEAM.IRPO`, `AU-TEAM` |
| RAID | `/dev/md0`, `--level=0`, `--raid-devices=2`, `/dev/sdb /dev/sdc` |
| NFS | `/raid`, `/raid/nfs`, `/mnt/nfs`, сеть клиента |
| Web / Nginx / DNAT | `web.au-team.irpo`, `docker.au-team.irpo`, порты `80`, `8080`, `2026` |
| DB / Docker | логины, пароли, имена БД, порты публикации |
| Ansible | IP-адреса targets, root-пароль |

---

## Таблица адресации Модуля 2

<pre>
HQ-SRV: <mark>192.168.1.10/27</mark>  роль: RAID/NFS/LAMP
HQ-CLI: <mark>192.168.2.10/28</mark>  роль: клиент домена/NFS
BR-SRV: <mark>192.168.3.10/28</mark>  роль: Samba DC/DNS/Ansible/Docker
GRE:    <mark>192.168.5.1</mark> – <mark>192.168.5.2</mark> /30
Mgmt:   VLAN <mark>999</mark>, <mark>192.168.9.1/29</mark>
</pre>

---

## Раздел 1 — ISP: NTP, Nginx Reverse Proxy

<pre>
NAT outside interface: oifname "<mark>enp7s1</mark>"
Repository cache: <mark>10.45.33.250:3142</mark>
NTP stratum: <mark>5</mark>
NTP allow: <mark>0.0.0.0/0</mark>

Basic Auth user:     <mark>WEB</mark>
Basic Auth password: <mark>P@ssw0rd</mark>

Nginx server_name: <mark>web.au-team.irpo</mark>
Proxy to LAMP:     http://<mark>172.16.1.10</mark>:<mark>8080</mark>

Nginx server_name: <mark>docker.au-team.irpo</mark>
Proxy to Docker:   http://<mark>172.16.2.10</mark>:<mark>8080</mark>

/etc/hosts: <mark>web.au-team.irpo</mark>, <mark>docker.au-team.irpo</mark>
timezone: <mark>Asia/Vladivostok</mark>
</pre>

---

## Раздел 2 — HQ-RTR: DNS, NTP, DNAT

<pre>
Temporary DNS: <mark>77.88.8.8</mark>
NTP server:    <mark>172.16.1.1</mark>
SSH Port:      <mark>22</mark>
PermitRootLogin: <mark>yes</mark>

Old DNS: <mark>192.168.1.10</mark>
New DNS / Samba DC: <mark>192.168.3.10</mark>

DNAT incoming interface: <mark>enp7s1</mark>
Public web port: <mark>8080</mark> → <mark>192.168.1.10</mark>:<mark>80</mark>
Public SSH port: <mark>2026</mark> → <mark>192.168.1.10</mark>:<mark>22</mark>
NAT outside interface: oifname "<mark>enp7s1</mark>"
</pre>

---

## Раздел 3 — BR-RTR: DNS, NTP, DNAT

<pre>
Temporary DNS: <mark>77.88.8.8</mark>
NTP server:    <mark>172.16.2.1</mark>
SSH Port:      <mark>22</mark>
PermitRootLogin: <mark>yes</mark>

Old DNS candidates: <mark>192.168.1.10</mark>, <mark>127.0.0.1</mark>
New DNS / Samba DC: <mark>192.168.3.10</mark>

DNAT incoming interface: <mark>enp7s1</mark>
Public Docker port: <mark>8080</mark> → <mark>192.168.3.10</mark>:<mark>8080</mark>
Public SSH port:    <mark>2026</mark> → <mark>192.168.3.10</mark>:<mark>22</mark>
NAT outside interface: oifname "<mark>enp7s1</mark>"
</pre>

---

## Раздел 4 — HQ-SRV: RAID, NFS, LAMP, MariaDB

<pre>
Temporary DNS: <mark>77.88.8.8</mark>
Disable old DNS service: <mark>bind</mark>
NTP server: <mark>172.16.1.1</mark>
SSH Port: <mark>22</mark>
PermitRootLogin: <mark>yes</mark>

Old DNS candidates: <mark>172.16.100.2</mark>, <mark>127.0.0.1</mark>
New DNS / Samba DC: <mark>192.168.3.10</mark>
</pre>

### RAID

<pre>
RAID device:       <mark>/dev/md0</mark>
RAID level:        <mark>0</mark>
Number of disks:   <mark>2</mark>
Disks:             <mark>/dev/sdb</mark> <mark>/dev/sdc</mark>
RAID config file:  <mark>/etc/mdadm.conf</mark>
Filesystem:        <mark>ext4</mark>
Mount point:       <mark>/raid</mark>
fstab entry:       <mark>/dev/md0</mark> <mark>/raid</mark> <mark>ext4</mark> defaults 0 0
</pre>

Если в ТЗ строго сказано “создать раздел на массиве”, тогда после RAID нужен раздел, и в fstab будет не `/dev/md0`, а, например, `<mark>/dev/md0p1</mark>`.

### NFS

<pre>
NFS directory: <mark>/raid/nfs</mark>
Permissions:   <mark>777</mark>
Client network allowed: <mark>192.168.2.0/28</mark>
NFS options: <mark>rw,sync,no_root_squash,no_subtree_check</mark>
</pre>

### LAMP / MariaDB

<pre>
CD-ROM mount: <mark>/mnt/cdrom</mark>
Database:     <mark>webdb</mark>
DB user:      <mark>web</mark>@<mark>localhost</mark>
DB password:  <mark>P@ssw0rd</mark>
DB dump:      <mark>/mnt/cdrom/web/dump.sql</mark>
Web root:     <mark>/var/www/html/</mark>
Web config:   <mark>/var/www/html/index.php</mark>
Web owner:    <mark>apache2:apache2</mark>
</pre>

---

## Раздел 5 — BR-SRV: Samba AD DC, Ansible, Docker Compose

<pre>
Temporary DNS: <mark>77.88.8.8</mark>
NTP server:    <mark>172.16.2.1</mark>
SSH Port:      <mark>22</mark>
PermitRootLogin: <mark>yes</mark>

Stop conflicting services:
<mark>smb</mark> <mark>nmb</mark> <mark>krb5kdc</mark> <mark>slapd</mark> <mark>bind</mark> <mark>ahttpd</mark> <mark>httpd2</mark> <mark>nginx</mark>

DNS client replacement:
old candidates: <mark>192.168.1.10</mark>, <mark>127.0.0.1</mark>, <mark>172.16.100.2</mark>
new DNS: <mark>127.0.0.1</mark>
search domain: <mark>au-team.irpo</mark>
</pre>

### Samba domain provision

<pre>
Realm:        <mark>AU-TEAM.IRPO</mark>
Domain:       <mark>AU-TEAM</mark>
Server role:  <mark>dc</mark>
DNS backend:  <mark>SAMBA_INTERNAL</mark>
Admin pass:   <mark>P@ssw0rd</mark>
DNS forwarder:<mark>77.88.8.8</mark>
Host IP:      <mark>192.168.3.10</mark>
</pre>

### Domain users

<pre>
Domain group: <mark>hq</mark>
Users pattern: <mark>hquser</mark>1–<mark>hquser</mark>5
User password: <mark>P@ssw0rd</mark>
No expiry: <mark>--noexpiry</mark>
</pre>

### Ansible

<pre>
Ansible user: <mark>root</mark>
Ansible password: <mark>toor</mark>
Python: <mark>/usr/bin/python3</mark>

Targets:
<mark>192.168.1.10</mark>  HQ-SRV
<mark>192.168.2.10</mark>  HQ-CLI
<mark>192.168.1.1</mark>   HQ-RTR
<mark>192.168.3.1</mark>   BR-RTR
</pre>

### Docker Compose

<pre>
Images:
<mark>/mnt/cdrom/docker/site_latest.tar</mark>
<mark>/mnt/cdrom/docker/mariadb_latest.tar</mark>

Compose directory: <mark>/opt/testapp</mark>

DB container: <mark>db</mark>
DB image:     <mark>mariadb:latest</mark>
MYSQL_DATABASE:      <mark>testdb</mark>
MYSQL_USER:          <mark>test</mark>
MYSQL_PASSWORD:      <mark>P@ssw0rd</mark>
MYSQL_ROOT_PASSWORD: <mark>toor</mark>

App container: <mark>testapp</mark>
App image:     <mark>site:latest</mark>
Published port: <mark>8080</mark>:<mark>8000</mark>
DB_TYPE: <mark>maria</mark>
DB_HOST: <mark>db</mark>
DB_PORT: <mark>3306</mark>
DB_NAME: <mark>testdb</mark>
DB_USER: <mark>test</mark>
DB_PASS: <mark>P@ssw0rd</mark>
</pre>

---

## Раздел 6 — HQ-CLI: домен, RBAC, NFS mount

<pre>
Temporary DNS: <mark>77.88.8.8</mark>
NTP server:    <mark>192.168.2.1</mark>
SSH Port:      <mark>22</mark>
PermitRootLogin: <mark>yes</mark>

NetworkManager connection: <mark>first connection from nmcli</mark>
DNS for client: <mark>192.168.3.10</mark>
DHCP Client-ID: <mark>01:11:11:11:11:11:11</mark>
</pre>

### Join domain

<pre>
system-auth write ad <mark>au-team.irpo</mark> <mark>hq-cli</mark> <mark>au-team</mark> <mark>administrator</mark> <mark>P@ssw0rd</mark>
</pre>

### RBAC / sudo

<pre>
Domain group: <mark>hq</mark>
Local role/group: <mark>wheel</mark>
Allowed sudo commands:
<mark>/bin/cat</mark>, <mark>/bin/grep</mark>, <mark>/usr/bin/id</mark>
Sudoers file: <mark>/etc/sudoers.d/hq_restrictions</mark>
</pre>

### NFS client mount

<pre>
Mount point: <mark>/mnt/nfs</mark>
NFS server:  <mark>192.168.1.10</mark>
NFS export:  <mark>/raid/nfs</mark>
fstab:       <mark>192.168.1.10:/raid/nfs</mark> <mark>/mnt/nfs</mark> nfs defaults 0 0
</pre>

---

## Финальная проверка на HQ-CLI

<pre>
id <mark>hquser1</mark>
echo "NFS OK" > <mark>/mnt/nfs/test.txt</mark> && cat <mark>/mnt/nfs/test.txt</mark>
ssh root@<mark>192.168.3.10</mark> "ansible all -m ping | grep SUCCESS"
root password: <mark>toor</mark>
</pre>

---

## Финальная проверка на ISP

<pre>
curl -sI -m 2 -u <mark>WEB</mark>:<mark>P@ssw0rd</mark> http://<mark>web.au-team.irpo</mark> | grep -iE "HTTP/.*200"
curl -s -m 5 http://<mark>docker.au-team.irpo</mark> | grep -i "<mark>Все студенты</mark>"
</pre>

---

## Быстрый поиск перед экзаменом

Ищи по файлу: `<mark>`, `au-team`, `P@ss`, `192.168`, `172.16`, `8080`, `2026`, `md0`, `sdb`, `sdc`, `webdb`, `testdb`, `hquser`, `hq`, `wheel`, `nfs`.
