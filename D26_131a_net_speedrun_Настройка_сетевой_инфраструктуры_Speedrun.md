> > 🧩 **Пример заполнения (🟢 Greenfield):**
> > Учебные стенды Greenfield имеют детерминированную аппаратную базу. Если вы работаете на официальном стенде, ваша заполненная таблица будет выглядеть именно так:
> >
> | **Узел (Hostname)** | **Роль (Сегмент)** | **Интерфейс ОС** | **IP-адрес / Маска** | **Шлюз (Gateway)** |
> | :---: | :--- | :---: | :---: | :---: |
> | **`ISP`** | WAN (Интернет) | `enp7s1` | *DHCP* | *DHCP* |
> | | LAN к `HQ-RTR` | `enp7s2` | `172.16.1.1/28` | — |
> | | LAN к `BR-RTR` | `enp7s3` | `172.16.2.1/28` | — |
> | **`HQ-RTR`** | Uplink (к `ISP`) | `enp7s1` | `172.16.1.2/28` | `172.16.1.1` |
> | **`BR-RTR`** | Uplink (к `ISP`) | `enp7s1` | `172.16.2.2/28` | `172.16.2.1` |
> 

## Раздел 1. Аппаратная коммутация (Proxmox VE)

### 1.1. Тегируем порты доступа (Access Ports)

> 🖱️ **В веб-интерфейсе Proxmox VE:**
>
> > 🗺️ **Путь:** Выберите виртуальную машину -> вкладка **Hardware** -> **Network Device**
>
> 1. **Откройте** настройки машины **`HQ-SRV`** и **дважды кликните** по сетевому адаптеру, смотрящему в локальную сеть.
> 2. **Впишите** в поле **VLAN Tag** значение `100` и **нажмите** кнопку **[OK]**.
> 3. **Откройте** настройки машины **`HQ-CLI`** и **дважды кликните** по её локальному сетевому адаптеру.
> 4. **Впишите** в поле **VLAN Tag** значение `200` и **нажмите** кнопку **[OK]**.

## Раздел 2. Маршрутизатор провайдера (`ISP`)

### 2.1. Настраиваем пограничную маршрутизацию и трансляцию адресов (NAT)

> 💻 **В терминале маршрутизатора `ISP`:**
>
> > 📥 **Скопируйте** и **выполните** блок целиком.
> > ```bash
> > hostnamectl set-hostname isp.au-team.irpo
> >
> > systemctl disable --now NetworkManager 2>/dev/null || true
> >
> > mkdir -p /etc/net/ifaces/enp7s1
> > echo "TYPE=eth" > /etc/net/ifaces/enp7s1/options
> > echo "BOOTPROTO=dhcp" >> /etc/net/ifaces/enp7s1/options
> >
> > mkdir -p /etc/net/ifaces/enp7s2
> > echo "TYPE=eth" > /etc/net/ifaces/enp7s2/options
> > echo "172.16.1.1/28" > /etc/net/ifaces/enp7s2/ipv4address
> >
> > mkdir -p /etc/net/ifaces/enp7s3
> > echo "TYPE=eth" > /etc/net/ifaces/enp7s3/options
> > echo "172.16.2.1/28" > /etc/net/ifaces/enp7s3/ipv4address
> >
> > sed -i '/net.ipv4.ip_forward/d' /etc/net/sysctl.conf
> > echo "net.ipv4.ip_forward = 1" >> /etc/net/sysctl.conf
> > sysctl -w net.ipv4.ip_forward=1
> > 
> > systemctl restart network
> > sleep 3
> > 
> > sed -i 's|http://ftp.altlinux.org/pub/distributions/ALTLinux|http://10.45.33.250:3142/altlinux|g' /etc/apt/sources.list.d/alt.list
> > apt-get update && apt-get install -y nftables tzdata
> > 
> > mkdir -p /etc/nftables
> > cat <<EOF > /etc/nftables/nftables.nft
> > #!/usr/sbin/nft -f
> > flush ruleset
> > table ip nat {
> >     chain postrouting {
> >         type nat hook postrouting priority srcnat;
 > >        oifname "enp7s1" masquerade
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
> > systemctl enable nftables
> > timedatectl set-timezone Asia/Vladivostok
> > 
> > systemctl restart nftables
> > 
> > history -a
> > sleep 10
> > reboot
> > ```

---

> 📍 **Контрольная точка:** Ожидание загрузки шлюза
>
> Узел ушел в перезагрузку. Категорически запрещено переходить к следующей машине до появления приглашения `isp login:`. Маршрутизатору `HQ-RTR` потребуется работающий шлюз для маршрутизации трафика и установки пакетов.

## Раздел 3. Главный маршрутизатор (`HQ-RTR`)

### 3.1. Развертываем ядро сети: VLAN, OSPF, GRE-туннель и DHCP

> 💻 **В терминале маршрутизатора `HQ-RTR`:**
>
> > 📥 **Скопируйте** и **выполните** блок целиком.
> > ```bash
> > hostnamectl set-hostname hq-rtr.au-team.irpo
> >
> > systemctl disable --now NetworkManager 2>/dev/null || true
> >
> > mkdir -p /etc/net/ifaces/enp7s1
> > echo "TYPE=eth" > /etc/net/ifaces/enp7s1/options
> > echo "172.16.1.2/28" > /etc/net/ifaces/enp7s1/ipv4address
> > echo "default via 172.16.1.1" > /etc/net/ifaces/enp7s1/ipv4route
> > echo "nameserver 8.8.8.8" > /etc/net/ifaces/enp7s1/resolv.conf
> >
> > mkdir -p /etc/net/ifaces/enp7s2
> > rm -f /etc/net/ifaces/enp7s2/ipv4address
> > echo "TYPE=eth" > /etc/net/ifaces/enp7s2/options
> >
> > mkdir -p /etc/net/ifaces/vlan100
> > echo "TYPE=vlan" > /etc/net/ifaces/vlan100/options
> > echo "HOST=enp7s2" >> /etc/net/ifaces/vlan100/options
> > echo "VID=100" >> /etc/net/ifaces/vlan100/options
> > echo "172.16.100.1/28" > /etc/net/ifaces/vlan100/ipv4address
> >
> > mkdir -p /etc/net/ifaces/vlan200
> > echo "TYPE=vlan" > /etc/net/ifaces/vlan200/options
> > echo "HOST=enp7s2" >> /etc/net/ifaces/vlan200/options
> > echo "VID=200" >> /etc/net/ifaces/vlan200/options
> > echo "172.16.200.1/27" > /etc/net/ifaces/vlan200/ipv4address
> >
> > mkdir -p /etc/net/ifaces/vlan999
> > echo "TYPE=vlan" > /etc/net/ifaces/vlan999/options
> > echo "HOST=enp7s2" >> /etc/net/ifaces/vlan999/options
> > echo "VID=999" >> /etc/net/ifaces/vlan999/options
> > echo "172.16.99.1/29" > /etc/net/ifaces/vlan999/ipv4address
> >
> > mkdir -p /etc/net/ifaces/gre1
> > cat << EOF > /etc/net/ifaces/gre1/options
> > TYPE=iptun
> > TUNTYPE=gre
> > TUNLOCAL=172.16.1.2
> > TUNREMOTE=172.16.2.2
> > TUNTTL=64
> > TUNOPTIONS='ttl 64'
> > EOF
> > echo "10.10.10.1/30" > /etc/net/ifaces/gre1/ipv4address
> > 
> > sed -i '/net.ipv4.ip_forward/d' /etc/net/sysctl.conf
> > echo "net.ipv4.ip_forward = 1" >> /etc/net/sysctl.conf
> > sysctl -w net.ipv4.ip_forward=1
> > 
> > systemctl restart network
> > sleep 3
> > 
> > sed -i 's|http://ftp.altlinux.org/pub/distributions/ALTLinux|http://10.45.33.250:3142/altlinux|g' /etc/apt/sources.list.d/alt.list
> > apt-get update && apt-get install -y nftables frr dnsmasq tzdata
> > 
> > mkdir -p /etc/nftables
> > cat <<EOF > /etc/nftables/nftables.nft
> > #!/usr/sbin/nft -f
> > flush ruleset
> > table ip nat {
> >     chain postrouting {
> >         type nat hook postrouting priority srcnat;
> >         oifname "enp7s1" masquerade
 > >    }
> > }
> > table ip filter {
> >     chain forward {
> >         type filter hook forward priority 0;
> >         tcp flags syn tcp option maxseg size set 1400
> >     }
> > }
> > EOF
> > 
> > cat <<EOF > /etc/dnsmasq.conf
> > port=0
> > interface=vlan200
> > dhcp-range=172.16.200.10,172.16.200.14,255.255.255.224,12h
> > dhcp-host=id:01:11:11:11:11:11:11,172.16.200.10
> > dhcp-option=3,172.16.200.1
> > dhcp-option=6,172.16.100.2
> > domain=au-team.irpo
> > EOF
> > 
> > cat <<EOF >> /etc/resolvconf.conf
> > resolv_conf_local_only=NO
> > deny_interfaces="lo.dnsmasq"
> > EOF
> > 
> > sed -i 's/ospfd=no/ospfd=yes/' /etc/frr/daemons
> > cat <<'EOF' > /etc/frr/frr.conf
> > interface gre1
> >  ip ospf area 0
> >  ip ospf authentication
> >  ip ospf authentication-key P@ssw0rd
> >  no ip ospf passive
> > exit
> > !
> > interface vlan100
> >  ip ospf area 0
> > exit
> > !
> > interface vlan200
> >  ip ospf area 0
> > exit
> > !
> > interface vlan999
> >  ip ospf area 0
> > exit
> > !
> > router ospf
 > > passive-interface default
> > exit
> > EOF
> > 
> > useradd -m -s /bin/bash net_admin
> > echo "net_admin:P@ssw0rd" | chpasswd
> > mkdir -p /etc/sudoers.d/
> > echo "net_admin ALL=(ALL) NOPASSWD: ALL" > /etc/sudoers.d/net_admin
> > 
> > timedatectl set-timezone Asia/Vladivostok
> > 
> > systemctl enable nftables frr dnsmasq
> > systemctl restart nftables frr dnsmasq
> > 
> > history -a
> > sleep 10
> > reboot
> > ```

---

> 📍 **Контрольная точка:** Ожидание загрузки ядра сети
>
> Узел ушел в перезагрузку. Дождитесь появления приглашения `hq-rtr login:`. Без работающего маршрутизатора центральный сервер не сможет получить доступ к сети.

## Раздел 4. Центральный сервер (`HQ-SRV`)

### 4.1. Развертываем авторитетный DNS-сервер и применяем политики безопасности (SSH)

> 💻 **В терминале сервера `HQ-SRV`:**
>
> > 📥 **Скопируйте** и **выполните** блок целиком.
> > ```bash
> > hostnamectl set-hostname hq-srv.au-team.irpo
> >
> > systemctl disable --now NetworkManager 2>/dev/null || true
> >
> > mkdir -p /etc/net/ifaces/enp7s1
> > echo "TYPE=eth" > /etc/net/ifaces/enp7s1/options
> > echo "172.16.100.2/28" > /etc/net/ifaces/enp7s1/ipv4address
> > echo "default via 172.16.100.1" > /etc/net/ifaces/enp7s1/ipv4route
> > echo "nameserver 127.0.0.1" > /etc/net/ifaces/enp7s1/resolv.conf
> > echo "search au-team.irpo" >> /etc/net/ifaces/enp7s1/resolv.conf
> > 
> > systemctl restart network
> > sleep 3
> > echo "nameserver 8.8.8.8" > /etc/resolv.conf
> > 
> > sed -i 's|http://ftp.altlinux.org/pub/distributions/ALTLinux|http://10.45.33.250:3142/altlinux|g' /etc/apt/sources.list.d/alt.list
> > apt-get update && apt-get install -y bind bind-utils tzdata
> > 
> > rndc-confgen -a -c /etc/bind/rndc.key
> > mkdir -p /etc/bind/zone
> >
> > cat <<EOF > /etc/bind/options.conf
> > options {
> >     recursion yes;
> >     allow-query { any; };
> >     forwarders { 8.8.8.8; 77.88.8.8; };
> >     dnssec-validation no;
> >     listen-on-v6 { none; };
> > };
> > EOF
> >
> > cat <<EOF > /etc/bind/local.conf
> > zone "au-team.irpo" {
> >     type master;
> >     file "/etc/bind/zone/au-team.irpo";
> > };
> > zone "16.172.in-addr.arpa" {
> >     type master;
> >     file "/etc/bind/zone/16.172.in-addr.arpa";
> > };
> > EOF
> >
> > cat <<EOF > /etc/bind/zone/au-team.irpo
> > \$TTL    604800
> > @       IN      SOA     hq-srv.au-team.irpo. root.au-team.irpo. (
> >                               1         ; Serial
> >                          604800         ; Refresh
> >                           86400         ; Retry
> >                         2419200         ; Expire
> >                          604800 )       ; Negative Cache TTL
> > ;
> > @       IN      NS      hq-srv.au-team.irpo.
> > @       IN      A       172.16.100.2
> > ;
> > hq-srv  IN      A       172.16.100.2
> > hq-rtr  IN      A       172.16.100.1
> > hq-cli  IN      A       172.16.200.10
> > br-rtr  IN      A       172.16.2.2
> > br-srv  IN      A       172.16.10.2
> > docker  IN      A       172.16.1.1
> > web     IN      A       172.16.2.1
> > EOF
> >
> > cat <<EOF > /etc/bind/zone/16.172.in-addr.arpa
> > \$TTL    604800
> > @       IN      SOA     hq-srv.au-team.irpo. root.au-team.irpo. (
> >                               1         ; Serial
> >                          604800         ; Refresh
> >                           86400         ; Retry
> >                         2419200         ; Expire
> >                          604800 )       ; Negative Cache TTL
> > ;
> > @       IN      NS      hq-srv.au-team.irpo.
> > ;
> > 2.100   IN      PTR     hq-srv.au-team.irpo.
> > 1.100   IN      PTR     hq-rtr.au-team.irpo.
> > 10.200  IN      PTR     hq-cli.au-team.irpo.
> > EOF
> >
> > chown root:named /etc/bind/zone/*
> > chmod 640 /etc/bind/zone/*
> > systemctl enable bind
> >
> > useradd -u 2026 -m -s /bin/bash sshuser
> > echo "sshuser:P@ssw0rd" | chpasswd
> > mkdir -p /etc/sudoers.d/
> > echo "sshuser ALL=(ALL) NOPASSWD: ALL" > /etc/sudoers.d/sshuser
> >
> > echo "Authorized access only" > /etc/issue.net
> > cat <<EOF >> /etc/openssh/sshd_config
> >
> > # Security Hardening Rules
> > Port 2026
> > PermitRootLogin no
> > MaxAuthTries 2
> > Banner /etc/issue.net
> > AllowUsers sshuser
> > EOF
> > systemctl enable sshd
> >
> > timedatectl set-timezone Asia/Vladivostok
> >
> > history -a
> > sleep 10
> > reboot
> > ```

---

> 📍 **Контрольная точка:** Ожидание инициализации DNS
>
> Сервер ушел в перезагрузку. Дождитесь появления приглашения `hq-srv login:`, прежде чем переходить к настройке филиала. Соблюдение строгой очередности гарантирует корректное построение таблиц маршрутизации.

## Раздел 5. Маршрутизатор филиала (`BR-RTR`)

### 5.1. Интегрируем филиал: Маршрутизация, OSPF и инкапсуляция (GRE)

> 💻 **В терминале маршрутизатора `BR-RTR`:**
>
> > 📥 **Скопируйте** и **выполните** блок целиком.
> > ```bash
> > hostnamectl set-hostname br-rtr.au-team.irpo
> >
> > systemctl disable --now NetworkManager 2>/dev/null || true
> >
> > mkdir -p /etc/net/ifaces/enp7s1
> > echo "TYPE=eth" > /etc/net/ifaces/enp7s1/options
> > echo "172.16.2.2/28" > /etc/net/ifaces/enp7s1/ipv4address
> > echo "default via 172.16.2.1" > /etc/net/ifaces/enp7s1/ipv4route
> > echo "nameserver 8.8.8.8" > /etc/net/ifaces/enp7s1/resolv.conf
> >
> > mkdir -p /etc/net/ifaces/enp7s2
> > echo "TYPE=eth" > /etc/net/ifaces/enp7s2/options
> > echo "172.16.10.1/28" > /etc/net/ifaces/enp7s2/ipv4address
> >
> > mkdir -p /etc/net/ifaces/gre1
> > cat << EOF > /etc/net/ifaces/gre1/options
> > TYPE=iptun
> > TUNTYPE=gre
> > TUNLOCAL=172.16.2.2
> > TUNREMOTE=172.16.1.2
> > TUNTTL=64
> > TUNOPTIONS='ttl 64'
> > EOF
> > echo "10.10.10.2/30" > /etc/net/ifaces/gre1/ipv4address
> > 
> > sed -i '/net.ipv4.ip_forward/d' /etc/net/sysctl.conf
> > echo "net.ipv4.ip_forward = 1" >> /etc/net/sysctl.conf
> > sysctl -w net.ipv4.ip_forward=1
> > 
> > systemctl restart network
> > sleep 3
> > 
> > sed -i 's|http://ftp.altlinux.org/pub/distributions/ALTLinux|http://10.45.33.250:3142/altlinux|g' /etc/apt/sources.list.d/alt.list
> > apt-get update && apt-get install -y nftables frr tzdata
> > 
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
> > sed -i 's/ospfd=no/ospfd=yes/' /etc/frr/daemons
> > cat <<'EOF' > /etc/frr/frr.conf
> > interface gre1
> >  ip ospf area 0
> >  ip ospf authentication
> >  ip ospf authentication-key P@ssw0rd
> >  no ip ospf passive
> > exit
> > !
> > interface enp7s2
> >  ip ospf area 0
> > exit
> > !
> > router ospf
> >  passive-interface default
> > exit
> > EOF
> >
> > useradd -m -s /bin/bash net_admin
> > echo "net_admin:P@ssw0rd" | chpasswd
> > mkdir -p /etc/sudoers.d/
> > echo "net_admin ALL=(ALL) NOPASSWD: ALL" > /etc/sudoers.d/net_admin
> >
> > timedatectl set-timezone Asia/Vladivostok
> >
> > systemctl enable nftables frr
> > systemctl restart network nftables frr
> > 
> > history -a
> > sleep 10
> > reboot
> > ```

---

> 📍 **Контрольная точка:** Ожидание загрузки маршрутизатора филиала
>
> Узел ушел в перезагрузку. Дождитесь появления приглашения `br-rtr login:`, чтобы сервер филиала получил рабочий шлюз для доступа в сеть.

## Раздел 6. Сервер филиала (`BR-SRV`)

### 6.1. Настраиваем сервер филиала и защищенный удаленный доступ

> 💻 **В терминале сервера `BR-SRV`:**
>
> > 📥 **Скопируйте** и **выполните** блок целиком.
> > ```bash
> > hostnamectl set-hostname br-srv.au-team.irpo
> >
> > systemctl disable --now NetworkManager 2>/dev/null || true
> >
> > mkdir -p /etc/net/ifaces/enp7s1
> > echo "TYPE=eth" > /etc/net/ifaces/enp7s1/options
> > echo "172.16.10.2/28" > /etc/net/ifaces/enp7s1/ipv4address
> > echo "default via 172.16.10.1" > /etc/net/ifaces/enp7s1/ipv4route
> > echo "nameserver 8.8.8.8" > /etc/net/ifaces/enp7s1/resolv.conf
> >
> > useradd -u 2026 -m -s /bin/bash sshuser
> > echo "sshuser:P@ssw0rd" | chpasswd
> > mkdir -p /etc/sudoers.d/
> > echo "sshuser ALL=(ALL) NOPASSWD: ALL" > /etc/sudoers.d/sshuser
> >
> > echo "Authorized access only" > /etc/issue.net
> > cat <<EOF >> /etc/openssh/sshd_config
> >
> > # Security Hardening Rules
> > Port 2026
> > PermitRootLogin no
> > MaxAuthTries 2
> > Banner /etc/issue.net
> > AllowUsers sshuser
> > EOF
> > systemctl enable sshd
> >
> > timedatectl set-timezone Asia/Vladivostok
> >
> > history -a
> > sleep 10
> > reboot
> > ```

---

> 📍 **Контрольная точка:** Ожидание загрузки сервера филиала
>
> Узел ушел в перезагрузку. Дождитесь появления приглашения `br-srv login:`. Узлы инфраструктуры должны быть полностью готовы до включения клиента.

## Раздел 7. Клиентская рабочая станция (`HQ-CLI`)

### 7.1. Автоматизируем клиентскую станцию через NetworkManager (DHCP)

> 💻 **В терминале клиента `HQ-CLI`:**
>
> > 📥 **Скопируйте** и **выполните** блок целиком.
> > ```bash
> > hostnamectl set-hostname hq-cli.au-team.irpo
> >
> > systemctl disable --now network 2>/dev/null || true
> > systemctl enable --now NetworkManager
> >
> > nmcli con delete lan 2>/dev/null || true
> > nmcli con add type ethernet ifname enp7s1 con-name lan ipv4.method auto ipv4.dhcp-client-id 01:11:11:11:11:11:11
> > nmcli con up lan
> >
> > timedatectl set-timezone Asia/Vladivostok
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
> > 📥 **Выполните** комплексную проверку инфраструктуры.
> >
> > Дождитесь полной загрузки всех виртуальных машин. Скопируйте и вставьте весь блок команд целиком.
> >
> > > ⌨️ **Впишите** пароль `P@ssw0rd`, когда скрипт приостановит работу для запроса SSH-авторизации на удаленном узле.
> >
> > ```bash
> > ip -br a show enp7s1
> > ping -c 2 ya.ru
> > host br-srv.au-team.irpo
> > ssh -p 2026 sshuser@br-srv.au-team.irpo "date"
> > ```
>
> > 📤 **Ожидаемый вывод:**
> > Изучите сырой ответ терминала. Сравните его с эталонной декомпозицией ниже:
> >
> > 1. **Проверка DHCP:**<br>Узел получил строго зарезервированный IP-адрес.
> > 	```text
> > 	enp7s1           UP             172.16.200.10/27 ...
> > 	```
> > 2. **Проверка NAT и Forwarders:**<br>Пинг до внешнего узла прошел без потерь.
> > 	```text
> > 	...
> > 	2 packets transmitted, 2 received, 0% packet loss ...
> > 	```
> > 3. **Проверка BIND9:**<br>Имя сервера филиала успешно разрешилось во внутренний IP.
> > 	```text
> > 	br-srv.au-team.irpo has address 172.16.10.2
> > 	```
> > 4. **Проверка OSPF, GRE и SSH:**<br>Удаленное подключение состоялось, отобразился баннер и правильный сдвиг времени.
> > 	```text
> > 	Authorized access only
> > 	sshuser@br-srv.au-team.irpo's password:
> > 	[День недели] [Месяц] [Число] [ЧЧ:ММ:СС] +10 2026
> > 	```

---

> ✍️ **Артефакт**
>
> ☑️ **Самопроверка (Анализ логов):**
> - [ ] **DHCP:** В первой строке выдан строго зарезервированный IP `172.16.200.10`.<br>*(Доказывает работу `dnsmasq` и привязку по Client-ID).*
> - [ ] **NAT и Forwarders:** Пинг до `ya.ru` прошел без потерь.<br>*(Доказывает маршрутизацию, `ip_forward` и `masquerade` на `ISP`).*
> - [ ] **BIND9:** Имя сервера филиала успешно разрешилось в адрес `172.16.10.2`.<br>*(Доказывает работу локального DNS-сервера и корректность файлов зон).*
> - [ ] **OSPF, GRE и SSH Hardening:** Удаленное подключение по нестандартному порту 2026 состоялось, отобразился юридический баннер.<br>*(Доказывает, что туннель поднят и маршруты переданы ядру).*
> - [ ] **NTP:** Дата на удаленном сервере содержит правильный часовой сдвиг (например, `+10` для Владивостока).