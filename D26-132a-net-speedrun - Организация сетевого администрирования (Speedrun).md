---
title: "`D26-132a-sys-speedrun`: Организация сетевого администрирования (Speedrun)"
aliases:
  - "`D26-132a-sys-speedrun`: Организация сетевого администрирования (Speedrun)"
  - D26-132a-sys-speedrun
para: ресурс
status: задействовано
dirpipe-id: D26-132a-sys-speedrun
disciplines:
  - МДК.02.01. Администрирование сетевых операционных систем
  - МДК.02.02. Программное обеспечение компьютерных сетей
  - МДК.02.03. Организация администрирования компьютерных систем
competencies:
  - ОК 01
  - ОК 02
  - ПК 2.2
  - ПК 2.4
  - ПК 3.1
  - ПК 4.1
cdate: 2026-04-12 03:15:00
mdate: 2026-04-12 04:05:19
---

# `D26-132a-sys-speedrun`: Организация сетевого администрирования (Speedrun)

| Узел (Hostname) | Сегмент (Домен) | IP-адрес / Маска | Роль в Модуле 2 |
| --- | --- | --- | --- |
| HQ-SRV | Серверы (VLAN 100) | `192.168.1.10/27` | Файловый сервер (RAID/NFS) |
| HQ-CLI | Клиенты (VLAN 200) | `192.168.2.10/28` | Рабочая станция (Ввод в домен) |
| BR-SRV | Филиал (BR-LAN) | `192.168.3.10/28` | Контроллер домена (AD) / Ansible |
| VPN Tunnel | GRE (HQ <-> BR) | `192.168.5.1 – .2 /30` | L3-Связность между офисами |
| Управление | VLAN 999 (HQ) | `192.168.9.1/29` | Изолированный сегмент (Резерв) |

> 📋 **Параметры работы**
>
> *   ⏱️ **Норматив времени:** 90 минут
> *   💎 **Вес работы:** 25 из 50 баллов
> *   ⭐ **Сложность:** ⭐ Базовый
> *   📌 **Статус трека:** 🔴 Экзаменационный норматив
> *   🧭 **Педагогическая модель:** ⏱️ Скоростное прохождение (Speedrun)
> *   📝 **Описание:** Экстремально сжатый алгоритм развертывания системных сервисов **Модуля 2** для стендов типа 🟢 **Greenfield**. Полная минимизация переключений окон, хардкод переменных и монолитная группировка команд по целевым узлам (VM-Centric).

## Вводная

Второй модуль кардинально отличается от первого наличием жестких межсерверных зависимостей (Race Conditions). Вы не можете ввести клиента в домен, пока контроллер не поднял базу LDAP. Вы не можете проверить связность Ansible, пока на всех узлах не сброшены настройки безопасности SSH.

**Особенности скриптов Модуля 2:**
*   **Абсолютная автоматизация:** Мастер развертывания леса Active Directory (`samba-tool`), обычно требующий интерактивного ввода паролей, переведен в жесткий пакетный режим.
*   **Откат безопасности:** В начало каждого серверного скрипта вшит сброс политик SSH (возврат на стандартный порт 22 и разрешение входа `root`) для обеспечения «коробочной» работы Ansible.
*   **Убийство конкурентов:** Скрипты автоматически и хладнокровно уничтожают старую службу `bind`, чтобы освободить 53-й порт для встроенного DNS-движка Samba AD DC.

> ⚠️ **Критически важно:** Правила выживания
>
> 1. **Строгий порядок обхода:** Выполняйте настройку узлов **строго** в той последовательности, в которой идут разделы этой тетради. Нарушение порядка приведет к неразрешимым конфликтам служб.
> 2. **Нулевой параллелизм (Zero Parallelism):** Категорически запрещено запускать скрипты на разных машинах одновременно. Вставили код -> нажали `Enter` -> **дождались полной перезагрузки и окна `login:`** -> только потом перешли к следующей ВМ.
> 3. **Одно окно — один скрипт:** Вы открываете консоль узла только один раз, вставляете блок целиком и больше не возвращаетесь к этой машине до этапа финального тестирования.
> 4. **Запрет на повтор:** Скрипты не идемпотентны. Если вы запустите один и тот же блок дважды, конфигурации задублируются, а файловые системы (fstab) могут заблокировать загрузку ОС.

---

> 🎯 **Цель работы**
>
> 1. **Выполнить** сквозную публикацию ресурсов через Nginx (Reverse Proxy) и статический проброс портов (DNAT).
> 2. **Развернуть** инфраструктурные сервисы (NFS, RAID 0, NTP).
> 3. **Инициализировать** службу каталогов Samba AD DC, монолитный веб-сервер (LAMP) и микросервисное приложение (Docker Compose).
> 4. **Интегрировать** клиентскую рабочую станцию в домен и применить ролевую модель доступа (RBAC).

## Раздел 1. Маршрутизатор провайдера (`ISP`)

### 1.1. Инициализируем L7-шлюз и NTP-эталон

> 💻 **В терминале маршрутизатора `ISP`:**
>
> > 📥 **Скопируйте** и **выполните** блок целиком.
> > ```bash
> > mkdir -p /etc/nftables
> > cat <<EOF > /etc/nftables/nftables.nft
> > #!/usr/sbin/nft -f
> > flush ruleset
> > table ip nat {
> >     chain postrouting {
> >         type nat hook postrouting priority srcnat;
> >         oifname "enp7s1" masquerade
> >     }
> > }
> > table ip filter {
> >     chain forward {
> >         type filter hook forward priority 0;
> >         tcp flags syn tcp option maxseg size set 1400
> >     }
> > }
> > EOF
> >
> > systemctl enable --now nftables
> > systemctl restart nftables
> >
> > sed -i 's|http://ftp.altlinux.org/pub/distributions/ALTLinux|http://10.45.33.250:3142/altlinux|g' /etc/apt/sources.list.d/alt.list
> > apt-get update && apt-get install -y chrony nginx apache2 curl
> >
> > cat <<EOF >> /etc/chrony.conf
> > port 123
> > local stratum 5
> > allow 0.0.0.0/0
> > EOF
> >
> > htpasswd -bc /etc/nginx/.htpasswd WEB P@ssw0rd
> >
> > cat <<EOF > /etc/nginx/sites-available.d/web.conf
> > server {
> >     listen 80;
> >     server_name web.au-team.irpo;
> >     location / {
> >         proxy_pass http://172.16.1.10:8080;
> >         proxy_set_header Host \$host;
> >         auth_basic "Restricted Area";
> >         auth_basic_user_file /etc/nginx/.htpasswd;
> >     }
> > }
> > EOF
> >
> > cat <<EOF > /etc/nginx/sites-available.d/docker.conf
> > server {
> >     listen 80;
> >     server_name docker.au-team.irpo;
> >     location / {
> >         proxy_pass http://172.16.2.10:8080;
> >         proxy_set_header Host \$host;
> >     }
> > }
> > EOF
> >
> > ln -sf /etc/nginx/sites-available.d/web.conf /etc/nginx/sites-enabled.d/web.conf
> > ln -sf /etc/nginx/sites-available.d/docker.conf /etc/nginx/sites-enabled.d/docker.conf
> >
> > echo "127.0.0.1 web.au-team.irpo docker.au-team.irpo" >> /etc/hosts
> >
> > timedatectl set-timezone Asia/Vladivostok
> > systemctl enable --now chronyd nginx
> > systemctl restart chronyd nginx
> > 
> > history -a
> > sleep 10
> > reboot
> > ```

---

> 📍 **Контрольная точка:** Ожидание загрузки шлюза
>
> Сервер ушел в перезагрузку. Дождитесь появления приглашения `isp login:`. Если вы запустите скрипт на роутерах раньше, они не смогут выйти в Интернет для скачивания пакетов и синхронизации времени.

## Раздел 2. Главный маршрутизатор (`HQ-RTR`)

### 2.1. Обновляем маршрутизацию, DHCP и проброс портов

> 💻 **В терминале маршрутизатора `HQ-RTR`:**
>
> > 📥 **Скопируйте** и **выполните** блок целиком.
> > ```bash
> > echo "nameserver 77.88.8.8" > /etc/resolv.conf
> >
> > sed -i 's|http://ftp.altlinux.org/pub/distributions/ALTLinux|http://10.45.33.250:3142/altlinux|g' /etc/apt/sources.list.d/alt.list
> > apt-get update && apt-get install -y chrony
> >
> > sed -i '/pool/d' /etc/chrony.conf
> > sed -i '/server/d' /etc/chrony.conf
> > echo "server 172.16.1.1 iburst prefer" >> /etc/chrony.conf
> >
> > cat <<EOF > /etc/openssh/sshd_config
> > Port 22
> > PermitRootLogin yes
> > Subsystem sftp /usr/lib/openssh/sftp-server
> > EOF
> >
> > sed -i 's/192.168.1.10/192.168.3.10/' /etc/dnsmasq.conf
> > sed -i -E 's/(192\.168\.1\.10|127\.0\.0\.1)/192.168.3.10/' /etc/net/ifaces/*/resolv.conf 2>/dev/null || true
> >
> > mkdir -p /etc/nftables
> > cat <<EOF > /etc/nftables/nftables.nft
> > #!/usr/sbin/nft -f
> > flush ruleset
> > table ip nat {
> >     chain prerouting {
> >         type nat hook prerouting priority dstnat;
> >         iifname "enp7s1" tcp dport 8080 dnat to 192.168.1.10:80
> >         iifname "enp7s1" tcp dport 2026 dnat to 192.168.1.10:22
> >     }
> >     chain postrouting {
> >         type nat hook postrouting priority srcnat;
> >         oifname "enp7s1" masquerade
> >     }
> > }
> > table ip filter {
> >     chain forward {
> >         type filter hook forward priority 0;
> >         tcp flags syn tcp option maxseg size set 1400
> >     }
> > }
> > EOF
> >
> > systemctl enable sshd chronyd nftables
> > 
> > history -a
> > sleep 10
> > reboot
> > ```

---

> 📍 **Контрольная точка:** Ожидание загрузки маршрутизатора
>
> Узел ушел в перезагрузку. Дождитесь появления приглашения `hq-rtr login:`, чтобы серверы главного офиса могли получить доступ к Интернету через настроенный NAT.

## Раздел 3. Маршрутизатор филиала (`BR-RTR`)

### 3.1. Обновляем маршрутизацию и проброс портов

> 💻 **В терминале маршрутизатора `BR-RTR`:**
>
> > 📥 **Скопируйте** и **выполните** блок целиком.
> > ```bash
> > echo "nameserver 77.88.8.8" > /etc/resolv.conf
> >
> > sed -i 's|http://ftp.altlinux.org/pub/distributions/ALTLinux|http://10.45.33.250:3142/altlinux|g' /etc/apt/sources.list.d/alt.list
> > apt-get update && apt-get install -y chrony
> >
> > sed -i '/pool/d' /etc/chrony.conf
> > sed -i '/server/d' /etc/chrony.conf
> > echo "server 172.16.2.1 iburst prefer" >> /etc/chrony.conf
> >
> > cat <<EOF > /etc/openssh/sshd_config
> > Port 22
> > PermitRootLogin yes
> > Subsystem sftp /usr/lib/openssh/sftp-server
> > EOF
> >
> > sed -i -E 's/(192\.168\.1\.10|127\.0\.0\.1)/192.168.3.10/' /etc/net/ifaces/*/resolv.conf 2>/dev/null || true
> >
> > mkdir -p /etc/nftables
> > cat <<EOF > /etc/nftables/nftables.nft
> > #!/usr/sbin/nft -f
> > flush ruleset
> > table ip nat {
> >     chain prerouting {
> >         type nat hook prerouting priority dstnat;
> >         iifname "enp7s1" tcp dport 8080 dnat to 192.168.3.10:8080
> >         iifname "enp7s1" tcp dport 2026 dnat to 192.168.3.10:22
> >     }
> >     chain postrouting {
> >         type nat hook postrouting priority srcnat;
> >         oifname "enp7s1" masquerade
> >     }
> > }
> > table ip filter {
> >     chain forward {
> >         type filter hook forward priority 0;
> >         tcp flags syn tcp option maxseg size set 1400
> >     }
> > }
> > EOF
> >
> > systemctl enable sshd chronyd nftables
> > 
> > history -a
> > sleep 10
> > reboot
> > ```

---

> 📍 **Контрольная точка:** Ожидание загрузки маршрутизатора
>
> Узел ушел в перезагрузку. Дождитесь появления приглашения `br-rtr login:`, чтобы сервер филиала получил выход в сеть.

## Раздел 4. Сервер главного офиса (`HQ-SRV`)

### 4.1. Развертываем файловое хранилище (RAID/NFS) и веб-монолит (LAMP)

> 💻 **В терминале сервера `HQ-SRV`:**
>
> > 📥 **Скопируйте** и **выполните** блок целиком.
> > ```bash
> > echo "nameserver 77.88.8.8" > /etc/resolv.conf
> >
> > systemctl disable --now bind 2>/dev/null || true
> >
> > sed -i 's|http://ftp.altlinux.org/pub/distributions/ALTLinux|http://10.45.33.250:3142/altlinux|g' /etc/apt/sources.list.d/alt.list
> > apt-get update && apt-get install -y chrony mdadm nfs-server nfs-utils lamp-server
> >
> > sed -i '/pool/d' /etc/chrony.conf
> > sed -i '/server/d' /etc/chrony.conf
> > echo "server 172.16.1.1 iburst prefer" >> /etc/chrony.conf
> >
> > cat <<EOF > /etc/openssh/sshd_config
> > Port 22
> > PermitRootLogin yes
> > Subsystem sftp /usr/lib/openssh/sftp-server
> > EOF
> >
> > sed -i -E 's/(172\.16\.100\.2|127\.0\.0\.1)/192.168.3.10/' /etc/net/ifaces/*/resolv.conf 2>/dev/null || true
> >
> > mdadm --create /dev/md0 --level=0 --raid-devices=2 /dev/sdb /dev/sdc --run
> > mdadm --detail --scan | tee -a /etc/mdadm.conf
> > mkfs.ext4 /dev/md0
> >
> > mkdir -p /raid
> > echo "/dev/md0 /raid ext4 defaults 0 0" >> /etc/fstab
> > mount -a
> >
> > mkdir -p /raid/nfs
> > chmod 777 /raid/nfs
> > echo "/raid/nfs 192.168.2.0/28(rw,sync,no_root_squash,no_subtree_check)" > /etc/exports
> >
> > mkdir -p /mnt/cdrom
> > mount /dev/sr0 /mnt/cdrom
> >
> > systemctl enable --now mysqld
> > sleep 3
> >
> > mariadb -e "CREATE DATABASE webdb;"
> > mariadb -e "CREATE USER 'web'@'localhost' IDENTIFIED BY 'P@ssw0rd';"
> > mariadb -e "GRANT ALL PRIVILEGES ON webdb.* TO 'web'@'localhost';"
> > mariadb -e "FLUSH PRIVILEGES;"
> >
> > mariadb webdb < /mnt/cdrom/web/dump.sql
> >
> > rm -f /var/www/html/index.html
> > cp -r /mnt/cdrom/web/* /var/www/html/
> > chown -R apache2:apache2 /var/www/html/
> >
> > sed -i 's/$username = .*/$username = "web";/' /var/www/html/index.php
> > sed -i 's/$password = .*/$password = "P@ssw0rd";/' /var/www/html/index.php
> > sed -i 's/$dbname = .*/$dbname = "webdb";/' /var/www/html/index.php
> >
> > systemctl enable sshd chronyd nfs-server httpd2
> >
> > history -a
> > sleep 10
> > reboot
> > ```

---

> 📍 **Контрольная точка:** Ожидание загрузки хранилища
>
> Сервер ушел в перезагрузку. Дождитесь появления приглашения `hq-srv login:`. Массив RAID 0 и служба NFS должны быть полностью собраны и готовы к работе до того, как мы подключим клиентскую станцию.

## Раздел 5. Контроллер домена и оркестрация (`BR-SRV`)

### 5.1. Развертываем Samba AD, Ansible и Docker Compose

> 💻 **В терминале сервера `BR-SRV`:**
>
> > 📥 **Скопируйте** и **выполните** блок целиком.
> > ```bash
> > echo "nameserver 77.88.8.8" > /etc/resolv.conf
> >
> > sed -i 's|http://ftp.altlinux.org/pub/distributions/ALTLinux|http://10.45.33.250:3142/altlinux|g' /etc/apt/sources.list.d/alt.list
> > apt-get update && apt-get install -y chrony ansible sshpass task-samba-dc bind-utils
> >
> > systemctl disable --now smb nmb krb5kdc slapd bind ahttpd httpd2 nginx 2>/dev/null || true
> > rm -f /etc/samba/smb.conf
> >
> > sed -i '/pool/d' /etc/chrony.conf
> > sed -i '/server/d' /etc/chrony.conf
> > echo "server 172.16.2.1 iburst prefer" >> /etc/chrony.conf
> >
> > cat <<EOF > /etc/openssh/sshd_config
> > Port 22
> > PermitRootLogin yes
> > Subsystem sftp /usr/lib/openssh/sftp-server
> > EOF
> >
> > sed -i -E 's/(192\.168\.1\.10|127\.0\.0\.1|172\.16\.100\.2)/127.0.0.1/' /etc/net/ifaces/*/resolv.conf 2>/dev/null || true
> > echo "search au-team.irpo" >> /etc/net/ifaces/*/resolv.conf
> >
> > samba-tool domain provision --realm=AU-TEAM.IRPO --domain=AU-TEAM --server-role=dc --dns-backend=SAMBA_INTERNAL --adminpass='P@ssw0rd' --option="dns forwarder=77.88.8.8" --host-ip=192.168.3.10
> >
> > \cp -f /var/lib/samba/private/krb5.conf /etc/krb5.conf
> > systemctl enable --now samba
> >
> > samba-tool group add hq
> > for i in {1..5}; do
> >   samba-tool user add hquser$i P@ssw0rd
> >   samba-tool user setexpiry hquser$i --noexpiry
> >   samba-tool group addmembers "hq" hquser$i
> > done
> >
> > mkdir -p /etc/ansible
> > cat <<EOF > /etc/ansible/ansible.cfg
> > [defaults]
> > inventory = /etc/ansible/hosts
> > host_key_checking = False
> > EOF
> >
> > cat <<EOF > /etc/ansible/hosts
> > [all:vars]
> > ansible_user=root
> > ansible_password=toor
> > ansible_python_interpreter=/usr/bin/python3
> >
> > [targets]
> > 192.168.1.10
> > 192.168.2.10
> > 192.168.1.1
> > 192.168.3.1
> > EOF
> >
> > apt-get install -y docker-engine docker-compose-v2
> > systemctl enable --now docker
> >
> > mkdir -p /mnt/cdrom
> > mount /dev/sr0 /mnt/cdrom
> >
> > docker load -i /mnt/cdrom/docker/site_latest.tar
> > docker load -i /mnt/cdrom/docker/mariadb_latest.tar
> >
> > mkdir -p /opt/testapp
> > cat <<EOF > /opt/testapp/docker-compose.yaml
> > services:
> >   db:
> >     container_name: db
> >     image: mariadb:latest
> >     environment:
> >       MYSQL_DATABASE: testdb
> >       MYSQL_USER: test
> >       MYSQL_PASSWORD: P@ssw0rd
> >       MYSQL_ROOT_PASSWORD: toor
> >     volumes:
> >       - db_data:/var/lib/mysql
> >     restart: always
> >
> >   testapp:
> >     container_name: testapp
> >     image: site:latest
> >     ports:
> >       - "8080:8000"
> >     environment:
> >       DB_TYPE: "maria"
> >       DB_HOST: "db"
> >       DB_PORT: "3306"
> >       DB_NAME: "testdb"
> >       DB_USER: "test"
> >       DB_PASS: "P@ssw0rd"
> >     depends_on:
> >       - db
> >     restart: always
> >
> > volumes:
> >   db_data:
> > EOF
> >
> > cd /opt/testapp && docker compose up -d
> >
> > systemctl enable sshd chronyd
> >
> > history -a
> > sleep 20
> > reboot
> > ```

---

> 📍 **Контрольная точка:** Ожидание инициализации домена
>
> Сервер ушел в перезагрузку. Дождитесь появления приглашения `br-srv login:`. 
> 
> > ⚠️ **Важно:** 
> > После перезагрузки службе Samba AD DC требуется дополнительно около 1-2 минут для старта базы данных LDAP. Клиент (`HQ-CLI`) на следующем этапе будет дожидаться открытия порта.

## Раздел 6. Клиентская рабочая станция (`HQ-CLI`)

### 6.1. Вводим в домен, монтируем хранилище и настраиваем RBAC

> 💻 **В терминале клиента `HQ-CLI`:**
>
> > 📥 **Скопируйте** и **выполните** блок целиком.
> > ```bash
> > echo "nameserver 77.88.8.8" > /etc/resolv.conf
> >
> > sed -i 's|http://ftp.altlinux.org/pub/distributions/ALTLinux|http://10.45.33.250:3142/altlinux|g' /etc/apt/sources.list.d/alt.list
> > apt-get update && apt-get install -y chrony nfs-clients task-auth-ad-sssd libnss-role yandex-browser-stable
> >
> > sed -i '/pool/d' /etc/chrony.conf
> > sed -i '/server/d' /etc/chrony.conf
> > echo "server 192.168.2.1 iburst prefer" >> /etc/chrony.conf
> >
> > cat <<EOF > /etc/openssh/sshd_config
> > Port 22
> > PermitRootLogin yes
> > Subsystem sftp /usr/lib/openssh/sftp-server
> > EOF
> >
> > CON_NAME=$(nmcli -t -f NAME connection show | head -n 1)
> > nmcli con mod "$CON_NAME" ipv4.dns "192.168.3.10" ipv4.ignore-auto-dns yes ipv4.dhcp-client-id 01:11:11:11:11:11:11
> > nmcli con up "$CON_NAME"
> > sleep 5
> >
> > system-auth write ad au-team.irpo hq-cli au-team administrator P@ssw0rd
> > systemctl restart sssd
> >
> > control libnss-role enabled
> > roleadd hq wheel
> >
> > mkdir -p /etc/sudoers.d
> > cat <<EOF > /etc/sudoers.d/hq_restrictions
> > Cmnd_Alias HQ_CMDS = /bin/cat, /bin/grep, /usr/bin/id
> > %wheel ALL=(ALL) NOPASSWD: HQ_CMDS
> > EOF
> >
> > mkdir -p /mnt/nfs
> > sed -i '/\/mnt\/nfs/d' /etc/fstab
> > echo "192.168.1.10:/raid/nfs /mnt/nfs nfs defaults 0 0" >> /etc/fstab
> > mount -a
> >
> > systemctl enable sshd chronyd
> >
> > history -a
> > sleep 10
> > reboot
> > ```

---

> 📍 **Контрольная точка:** Ожидание загрузки клиента
>
> Узел ушел в перезагрузку. Дождитесь появления приглашения `hq-cli login:` перед началом финального тестирования.

## Заключение

> 💻 **В терминале клиента `HQ-CLI`:**
>
> > 📥 **Выполните** проверку внутренней инфраструктуры.
> >
> > > ⚠️ **Внимание:**
> > > После перезагрузки сервера (`BR-SRV`) дождитесь паузы в 1–2 минуты. Контроллеру домена Samba AD DC требуется время для инициализации базы данных.
> >
> > Скопируйте и вставьте блок команд целиком.
> >
> > > ⌨️ **Впишите** ваш пароль суперпользователя `root` (по умолчанию `toor`), когда скрипт приостановит работу для запроса SSH-авторизацию.
> >
> > ```bash
> > id hquser1
> > echo "NFS OK" > /mnt/nfs/test.txt && cat /mnt/nfs/test.txt
> > ssh -o StrictHostKeyChecking=no root@192.168.3.10 "ansible all -m ping | grep SUCCESS"
> > ```
>
> > 📤 **Ожидаемый вывод:**
> > Изучите сырой ответ терминала. Сравните его с эталоном ниже:
> >
> > 1. **Проверка Samba AD DC:**<br>Вывод содержит доменные группы пользователя.
> > 	```text
> > 	uid=... (hquser1) gid=... (domain users) groups=...
> > 	```
> > 2. **Проверка RAID 0 и NFS:**<br>Файл успешно записан в сетевую папку и прочитан.
> > 	```text
> > 	NFS OK
> > 	```
> > 3. **Проверка Ansible и OSPF:**<br>Команда управления вернула 4 успешных ответа от целевых узлов.
> > 	```text
> > 	root@192.168.3.10's password:
> > 	192.168.2.10 | SUCCESS
> > 	192.168.1.1 | SUCCESS
> > 	192.168.3.1 | SUCCESS
> > 	192.168.1.10 | SUCCESS
> > 	```

---

> 💻 **В терминале маршрутизатора `ISP`:**
>
> > 📥 **Выполните** проверку публикации сервисов.
> >
> > Скопируйте и вставьте блок команд целиком.
> >
> > ```bash
> > curl -sI -m 2 -u WEB:P@ssw0rd http://web.au-team.irpo | grep -iE "HTTP/.*200"
> > curl -s -m 5 http://docker.au-team.irpo | grep -i "Все студенты"
> > ```
>
> > 📤 **Ожидаемый вывод:**
> > Изучите сырой ответ терминала. Сравните его с эталоном ниже:
> >
> > 1. **Проверка Nginx, DNAT и LAMP:**<br>Запрос ушел на локальный Nginx, прошел авторизацию и вернулся через DNAT.
> > 	```text
> > 	HTTP/1.1 200 OK
> > 	```
> > 2. **Проверка Docker Compose и Nginx:**<br>Запрос ушел на локальный Nginx по доменному имени и получил HTML-ответ от Docker-контейнера.
> > 	```text
> > 	<title>Все студенты</title>
> > 	```

---

> ✍️ **Артефакт**
>
> ☑️ **Самопроверка (Анализ логов):**
> - [ ] **Samba AD DC:** Вывод `id hquser1` содержит доменные группы.<br>*(Доказывает, что контроллер домена поднялся, DNS-записи верны, а клиент `HQ-CLI` успешно интегрирован в Active Directory).*
> - [ ] **RAID 0 и NFS:** Файл `test.txt` успешно записан в сетевую папку и прочитан.<br>*(Доказывает, что дисковый массив `md0` собран, автомонтирование `fstab` отработало при загрузке, а сервер `HQ-SRV` корректно выдал права `rw` для вашей подсети).*
> - [ ] **Nginx, DNAT и LAMP:** Локальный `curl` на провайдере вернул статус `HTTP/1.1 200 OK`.<br>*(Доказывает сквозную интеграцию: сработала защита Basic Auth, Nginx успешно проксировал запрос по доменному имени, шлюз выполнил проброс портов (Port Forwarding), а база MariaDB отдала данные веб-серверу Apache).*
> - [ ] **Ansible и SSH:** Команда вернула 4 строки `SUCCESS`.<br>*(Доказывает абсолютную связность сети (OSPF/GRE), готовность инвентарного файла `hosts` и успешный сброс настроек безопасности SSH (Port 22, `PermitRootLogin yes`) на всех целевых узлах инфраструктуры).*