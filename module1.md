1 МОДУЛЬ
Пример заполнения (🟢 Greenfield): Учебные стенды Greenfield имеют детерминированную аппаратную базу. Если вы работаете на официальном стенде, ваша заполненная таблица будет выглядеть именно так:

| Узел (Hostname) | Роль (Сегмент) | Интерфейс ОС | IP-адрес / Маска | Шлюз (Gateway) |
| --- | --- | --- | --- | --- |
| **ISP** | WAN (Интернет) | enp7s1 | DHCP | DHCP |
|  | LAN к HQ-RTR | enp7s2 | 172.16.1.1/28 | — |
|  | LAN к BR-RTR | enp7s3 | 172.16.2.1/28 | — |
| **HQ-RTR** | Uplink (к ISP) | enp7s1 | 172.16.1.2/28 | 172.16.1.1 |
| **BR-RTR** | Uplink (к ISP) | enp7s1 | 172.16.2.2/28 | 172.16.2.1 |


| Узел (Hostname) | Роль (Сегмент) | Интерфейс ОС | IP-адрес / Маска | Шлюз (Gateway) |
| --- | --- | --- | --- | --- |
| **ISP** | WAN (Интернет) | enp7s1 | DHCP | DHCP |
|  | LAN к HQ-RTR | enp7s2 | 1.16.1.1/28 | — |
|  | LAN к BR-RTR | enp7s3 | 172.16.2.1/28 | — |
| **HQ-RTR** | Uplink (к ISP) | enp7s1 | 172.16.1.2/28 | 172.16.1.1 |
| **BR-RTR** | Uplink (к ISP) | enp7s1 | 172.16.2.2/28 | 172.16.2.1 |


Матрица идентификации (Узнай свой стенд)

| Параметр | 🟢 Greenfield (Чистовой стенд) | 🟤 Brownfield (Самосборный стенд) |
| --- | --- | --- |
| Пароль **root** | toor | P@ssw0rd (из шаблонов *207*/*208*) |
| Имена сетевых карт | enp7s1, enp7s2 | ens18, ens19, ens20 (зависит от PVE) |
| Служба сети | Настроена | Требует отключения NetworkManager |
| Номера VLAN | 100, 200, 999 | Ваши уникальные ID из прошлого модуля |


В терминале сервера **ISP**:
```bash
ip -br -c link
```
📤 Ожидаемый вывод:
```bash
lo               UNKNOWN        00:00:00:00:00:00 <LOOPBACK,UP,LOWER_UP>
enp... / ens...  UP             bc:24:11:xx:xx:01 <BROADCAST,MULTICAST,UP,LOWER_UP>
enp... / ens...  UP             bc:24:11:xx:xx:02 <BROADCAST,MULTICAST,UP,LOWER_UP>
enp... / ens...  UP             bc:24:11:xx:xx:03 <BROADCAST,MULTICAST,UP,LOWER_UP>
```

На сервере **ISP**:
```bash
hostnamectl set-hostname isp.au-team.irpo && exec bash
```
На сервере **HQ-RTR**:
```bash
hostnamectl set-hostname hq-rtr.au-team.irpo && exec bash
```

На сервере **HQ-SRV**:
```bash
hostnamectl set-hostname hq-srv.au-team.irpo && exec bash
```

На клиенте **HQ-CLI**:
```bash
hostnamectl set-hostname hq-cli.au-team.irpo && exec bash
```
На сервере **BR-RTR**:
```bash
hostnamectl set-hostname br-rtr.au-team.irpo && exec bash
```
На сервере **BR-SRV**:
```bash
hostnamectl set-hostname br-srv.au-team.irpo && exec bash
```
В терминале сервера **ISP**:
📥 Обновите список аппаратных устройств на экране.
```bash
ip -br -c link
```
📤 Ожидаемый вывод: Операционная система нумерует сетевые карты в том же порядке, в котором они были подключены к виртуальной машине в гипервизоре. Опираясь на Справочную таблицу из начала тетради, сопоставьте порты:
1. Первый адаптер (например, enp7s1 или ens18) = Интерфейс в Интернет (WAN).
1. Второй адаптер (например, enp7s2 или ens19) = Интерфейс в сторону HQ-RTR.
1. Третий адаптер (например, enp7s3 или ens20) = Интерфейс в сторону BR-RTR.
```bash
lo               UNKNOWN        00:00:00:00:00:00 <LOOPBACK,UP,LOWER_UP>
[Первый адаптер] UP             bc:24:11:xx:xx:01 <BROADCAST,MULTICAST,UP,LOWER_UP>
[Второй адаптер] UP             bc:24:11:xx:xx:02 <BROADCAST,MULTICAST,UP,LOWER_UP>
[Третий адаптер] UP             bc:24:11:xx:xx:03 <BROADCAST,MULTICAST,UP,LOWER_UP>
```

В терминале маршрутизатора **ISP**:
📥 Отключите службу NetworkManager.
```bash
systemctl disable --now NetworkManager 2>/dev/null || true
```
📥 Настройте WAN-интерфейс (DHCP).
```bash
mkdir -p /etc/net/ifaces/enp7s1
echo "TYPE=eth" > /etc/net/ifaces/enp7s1/options
echo "BOOTPROTO=dhcp" >> /etc/net/ifaces/enp7s1/options
```
📥 Настройте LAN-интерфейс к офису HQ.

```bash
mkdir -p /etc/net/ifaces/enp7s2
echo "TYPE=eth" > /etc/net/ifaces/enp7s2/options
echo "172.16.1.1/28" > /etc/net/ifaces/enp7s2/ipv4address
```
📥 Настройте LAN-интерфейс к офису BR.

```bash
mkdir -p /etc/net/ifaces/enp7s3
echo "TYPE=eth" > /etc/net/ifaces/enp7s3/options
echo "172.16.2.1/28" > /etc/net/ifaces/enp7s3/ipv4address
```
📥 Примените настройки и проверьте адреса.
```bash
systemctl restart network && ip -c a
```
📤 Ожидаемый вывод:
```bash
2: enp7s1: ... inet 10.x.x.x ...
3: enp7s2: ... inet 172.16.1.1/28 ...
4: enp7s3: ... inet 172.16.2.1/28 ...
```

В терминале сервера **ISP**:
```bash
sed -i '/net.ipv4.ip_forward/d' /etc/net/sysctl.conf
echo "net.ipv4.ip_forward = 1" >> /etc/net/sysctl.conf
sysctl -w net.ipv4.ip_forward=1
sysctl net.ipv4.ip_forward
```
📤 Ожидаемый вывод:
```bash
net.ipv4.ip_forward = 1
```
В терминале сервера **ISP**:
📥 Ускорьте загрузку и установите ПО.

```bash
sed -i 's|http://ftp.altlinux.org/pub/distributions/ALTLinux|http://10.45.33.250:3142/altlinux|g' /etc/apt/sources.list.d/alt.list
apt-get update && apt-get install -y nftables tzdata

mkdir -p /etc/nftables

cat <<EOF > /etc/nftables/nftables.nft
#!/usr/sbin/nft -f
flush ruleset

table ip nat {
    chain postrouting {
        type nat hook postrouting priority srcnat;
        oifname "enp7s1" masquerade
    }
}

table ip filter {
    chain forward {
        type filter hook forward priority 0;
        tcp flags syn tcp option maxseg size set 1400
    }
}
EOF
```
📥 Запускаем службу и проверяем таблицу маршрутизации.
```bash
systemctl enable --now nftables
nft list ruleset
```
📤 Ожидаемый вывод:
```bash
table ip nat {
        chain postrouting {
                type nat hook postrouting priority srcnat; policy accept;
                oifname "[Ваш интерфейс]" masquerade
        }
}
table ip filter { ... }
```
В терминале сервера **ISP**:
📥 Синхронизируйте часы ОС.
```bash
timedatectl set-timezone Asia/Vladivostok
date
```
📤 Ожидаемый вывод:
```bash
[День недели] [Месяц] [Число] [ЧЧ:ММ:СС] +10 2026
```
В терминале сервера **ISP**:
📥 Проверьте доступность шлюза Google DNS.
```bash
ping -c 2 8.8.8.8
```
📤 Ожидаемый вывод: Нулевой процент потерь (0% packet loss) доказывает, что L3-маршрут от вашего шлюза ISP до серверов Google физически проходим в обе стороны.
```bash
64 bytes from 8.8.8.8: icmp_seq=1 ttl=113 time=38.4 ms
64 bytes from 8.8.8.8: icmp_seq=2 ttl=113 time=38.9 ms

--- 8.8.8.8 ping statistics ---
2 packets transmitted, 2 received, 0% packet loss, time 1002ms
```

В терминалах **HQ-RTR** и **BR-RTR**:
📥 Просканируйте аппаратные устройства серверов.
```bash
ip -br -c link
```
📤 Ожидаемый вывод:
```bash
lo               UNKNOWN        00:00:00:00:00:00 <LOOPBACK,UP,LOWER_UP>
[Ваш WAN-порт]   UP             bc:24:11:xx:xx:01 <BROADCAST,MULTICAST,UP,LOWER_UP>
```

В терминале маршрутизатора **HQ-RTR**:
📥 Сконфигурируйте сеть, скачайте пакеты и настройте часы.
⌨️ Впишите вместо ___ найденное в разведке физическое имя вашей внешней сетевой карты, смотрящей в сторону провайдера ISP (например, enp7s1 или ens19).
```bash
systemctl disable --now NetworkManager 2>/dev/null || true

mkdir -p /etc/net/ifaces/enp7s1
echo "TYPE=eth" > /etc/net/ifaces/enp7s1/options
echo "172.16.1.2/28" > /etc/net/ifaces/enp7s1/ipv4address
echo "default via 172.16.1.1" > /etc/net/ifaces/enp7s1/ipv4route
echo "nameserver 8.8.8.8" > /etc/net/ifaces/enp7s1/resolv.conf

systemctl restart network

sed -i 's|http://ftp.altlinux.org/pub/distributions/ALTLinux|http://10.45.33.250:3142/altlinux|g' /etc/apt/sources.list.d/alt.list
apt-get update && apt-get install -y nftables frr dnsmasq tzdata

timedatectl set-timezone Asia/Vladivostok
```

В терминале маршрутизатора **BR-RTR**:
📥 Сконфигурируйте сеть, скачайте пакеты и настройте часы.
⌨️ Впишите вместо ___ физическое имя вашей внешней сетевой карты в филиале, смотрящей в сторону ISP.
```bash
systemctl disable --now NetworkManager 2>/dev/null || true

mkdir -p /etc/net/ifaces/enp7s1
echo "TYPE=eth" > /etc/net/ifaces/enp7s1/options
echo "172.16.2.2/28" > /etc/net/ifaces/enp7s1/ipv4address
echo "default via 172.16.2.1" > /etc/net/ifaces/enp7s1/ipv4route
echo "nameserver 8.8.8.8" > /etc/net/ifaces/enp7s1/resolv.conf

systemctl restart network

sed -i 's|http://ftp.altlinux.org/pub/distributions/ALTLinux|http://10.45.33.250:3142/altlinux|g' /etc/apt/sources.list.d/alt.list
apt-get update && apt-get install -y nftables frr dnsmasq tzdata

timedatectl set-timezone Asia/Vladivostok
```
В терминалах обоих маршрутизаторов (**HQ-RTR** и **BR-RTR**):
📥 Включите и сохраните разрешение на транзит пакетов.
```bash
sed -i '/net.ipv4.ip_forward/d' /etc/net/sysctl.conf
echo "net.ipv4.ip_forward = 1" >> /etc/net/sysctl.conf
sysctl -w net.ipv4.ip_forward=1
```
📥 Сформируйте правила фаервола (NAT и MSS Clamping).
⌨️ Впишите в строку oifname физическое имя вашего Uplink-интерфейса (например, enp7s1 или ens19), смотрящего в сторону провайдера ISP. Вы определяли его на Этапе 4.1.
```bash
mkdir -p /etc/nftables

cat <<EOF > /etc/nftables/nftables.nft
#!/usr/sbin/nft -f
flush ruleset

table ip nat {
    chain postrouting {
        type nat hook postrouting priority srcnat;
        oifname "enp7s1" masquerade
    }
}

table ip filter {
    chain forward {
        type filter hook forward priority 0;
        tcp flags syn tcp option maxseg size set 1400
    }
}
EOF
```
📥 Активируйте службу и проверьте правила.
```bash
systemctl enable --now nftables
nft list ruleset
```
📤 Ожидаемый вывод: Как и на шлюзе ISP, ключевым маркером успеха является наличие правила подмены адресов. Убедитесь, что оно привязалось именно к вашему внешнему Uplink-интерфейсу.
...
```bash
                oifname "[Ваш интерфейс]" masquerade
```
...
**2 тетрадка**

Матрица идентификации (Узнай свой стенд)
Демонстрационный экзамен проводится в двух параллельных реальностях. Ваша задача — понять, чем вы будете заполнять пропуски ___ в коде команд.

| Параметр | 🟢 Greenfield (Чистовой стенд) | 🟤 Brownfield (Самосборный стенд) |
| --- | --- | --- |
| Внутренний порт маршрутизатора | enp7s2 | Ваш второй адаптер (например, ens20) |
| Тег серверов (SRV-Net) | 100 | Ваш уникальный ID из личного пула |
| Тег клиентов (CLI-Net) | 200 | Ваш уникальный ID из личного пула |
| Тег управления (Mngmt) | 999 | Ваш уникальный ID из личного пула |


1. В дереве ресурсов слева выберите виртуальную машину **HQ-SRV**.
1. Перейдите во вкладку Hardware.
1. Найдите сетевое устройство, смотрящее во внутреннюю сеть (обычно это Network Device (net6) или (net1)). Выделите его и нажмите кнопку [Edit].
1. В поле VLAN Tag впишите 100
1. Нажмите кнопку [OK].
1. Повторите шаги 1–5 для виртуальной машины HQ-CLI, вписав в её сетевую карту тег 200

 В терминале маршрутизатора **HQ-RTR**:
📥 Настройте подсеть для серверов (SRV-Net).
```bash
mkdir -p /etc/net/ifaces/vlan100

echo "TYPE=vlan" > /etc/net/ifaces/vlan100/options
echo "HOST=enp7s2" >> /etc/net/ifaces/vlan100/options
echo "VID=100" >> /etc/net/ifaces/vlan100/options

echo "172.16.100.1/28" > /etc/net/ifaces/vlan100/ipv4address
```

Настройте подсеть для клиентов (CLI-Net).
```bash
mkdir -p /etc/net/ifaces/vlan200

echo "TYPE=vlan" > /etc/net/ifaces/vlan200/options
echo "HOST=enp7s2" >> /etc/net/ifaces/vlan200/options
echo "VID=200" >> /etc/net/ifaces/vlan200/options

echo "172.16.200.1/27" > /etc/net/ifaces/vlan200/ipv4address
```

Настройте подсеть управления (Management).
```bash
mkdir -p /etc/net/ifaces/vlan999

echo "TYPE=vlan" > /etc/net/ifaces/vlan999/options
echo "HOST=enp7s2" >> /etc/net/ifaces/vlan999/options
echo "VID=999" >> /etc/net/ifaces/vlan999/options

echo "172.16.99.1/29" > /etc/net/ifaces/vlan999/ipv4address
```

В терминале маршрутизатора **HQ-RTR**:
📥 Очистите и поднимите магистральный порт.
```bash
mkdir -p /etc/net/ifaces/enp7s2
rm -f /etc/net/ifaces/enp7s2/ipv4address

echo "TYPE=eth" > /etc/net/ifaces/enp7s2/options
```
📥 Примените конфигурацию и проверьте интерфейсы.
```bash
systemctl restart network && ip -c a
```
📤 Ожидаемый вывод: Наличие символа @ означает, что ядро успешно создало логическую привязку VLAN (802.1Q) к физическому кабелю. Убедитесь, что сам базовый порт [Внутренний порт] находится в состоянии UP, но не имеет строки inet (IP-адреса).
```bash
3: [Внутренний порт]: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 ... state UP ...
```
...
```bash
4: vlan[Тег серверов]@[Внутренний порт]: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 ...
    inet 172.16.100.1/28 scope global vlan[Тег серверов]
5: vlan[Тег клиентов]@[Внутренний порт]: ... inet 172.16.200.1/27 ...
6: vlan[Тег управления]@[Внутренний порт]: ... inet 172.16.99.1/29 ...
```
 В терминале маршрутизатора **HQ-RTR**:
📥 Сконфигурируйте пул адресов и резервацию.
```bash
cat <<EOF > /etc/dnsmasq.conf
port=0
interface=vlan200
dhcp-range=172.16.200.10,172.16.200.14,255.255.255.224,12h
dhcp-host=id:01:11:11:11:11:11:11,172.16.200.10
dhcp-option=3,172.16.200.1
dhcp-option=6,172.16.100.2
domain=au-team.irpo
EOF
```
📥 Заблокируйте попытки системы перехватить DNS.
```bash
cat <<EOF >> /etc/resolvconf.conf
resolv_conf_local_only=NO
deny_interfaces="lo.dnsmasq"
EOF
```
📥 Активируйте службу и проверьте статус.
```bash
systemctl enable --now dnsmasq
systemctl status dnsmasq --no-pager -l
```
📤 Ожидаемый вывод: Зеленый статус active доказывает, что синтаксический парсер демона принял ваш конфигурационный файл, и DHCP-сервер начал прослушивать клиентский VLAN.
```bash
Active: active (running) since ...
```

В терминале маршрутизатора **HQ-RTR**:
📥 Запустите встроенный тест конфигурации. Утилита прочитает файл и укажет точный номер строки с ошибкой.
```bash
dnsmasq --test
```
В терминале сервера **HQ-SRV**:
📥 Отключите службу автоматической конфигурации сети.
```bash
systemctl disable --now NetworkManager 2>/dev/null || true
```
📥 Настройте статические параметры интерфейса.
⌨️ Впишите физическое имя вашей единственной сетевой карты (например, enp7s1 или ens19) вместо всех пропусков ___. Уточните его командой ip -br link, если забыли.
```bash
mkdir -p /etc/net/ifaces/enp7s1

echo "TYPE=eth" > /etc/net/ifaces/enp7s1/options
echo "172.16.100.2/28" > /etc/net/ifaces/enp7s1/ipv4address
echo "default via 172.16.100.1" > /etc/net/ifaces/enp7s1/ipv4route
echo "nameserver 8.8.8.8" > /etc/net/ifaces/enp7s1/resolv.conf
```
📥 Синхронизируйте системное время.
```bash
timedatectl set-timezone Asia/Vladivostok
date
```
📤 Ожидаемый вывод: Наличие правильного часового пояса — критическое условие для работы будущей службы каталогов (Active Directory). Без него сервер откажется принимать билеты авторизации.
```bash
[День недели] [Месяц] [Число] [ЧЧ:ММ:СС] +10 2026
```
📥 Примените настройки и проверьте доступность шлюза.
```bash
systemctl restart network && ping -c 2 172.16.100.1
```
📤 Ожидаемый вывод
```bash
64 bytes from 172.16.100.1: icmp_seq=1 ttl=64 time=0.4 ms
64 bytes from 172.16.100.1: icmp_seq=2 ttl=64 time=0.2 ms

--- 172.16.100.1 ping statistics ---
2 packets transmitted, 2 received, 0% packet loss, time 1002ms
```
 В терминале клиента **HQ-CLI**:
📥 Инвертируйте системные службы.
```bash
systemctl disable --now network
systemctl enable --now NetworkManager
```
📥 Создайте профиль соединения с резервацией.
```bash
nmcli con add type ethernet ifname enp7s1 con-name lan \
ipv4.method auto \
ipv4.dhcp-client-id 01:11:11:11:11:11:11
```
📥 Синхронизируйте системное время.
```bash
timedatectl set-timezone Asia/Vladivostok
```
📥 Активируйте соединение и проверьте полученные параметры.
```bash
nmcli con up lan

ip -br -c a
date
ping -c 2 8.8.8.8
```
📤 Ожидаемый вывод:
1. Проверка резервации: Вывод ip a обязан показать строго адрес .10. Это доказывает, что роутер узнал вас по синтетическому client-id и выдал забронированный IP.
1. Проверка времени: Вывод date должен показать временной сдвиг.
1. Проверка маршрутизации: Вывод ping обязан показать 0% packet loss. Это доказывает, что DHCP корректно передал вам опцию шлюза (172.16.200.1), и вы смогли выйти через NAT провайдера.
```bash
[Ваш_интерфейс]   UP             172.16.200.10/27 ...
[День недели] [Месяц] [Число] [ЧЧ:ММ:СС] +10 2026
```
...
```bash
2 packets transmitted, 2 received, 0% packet loss
```

**//3 тетрадка**
В терминале маршрутизатора BR-RTR:
📥 Отключите службу автоматической конфигурации и задайте параметры внутренней сети.
```bash
systemctl disable --now NetworkManager 2>/dev/null || true

mkdir -p /etc/net/ifaces/enp7s2
echo "TYPE=eth" > /etc/net/ifaces/enp7s2/options
echo "172.16.10.1/28" > /etc/net/ifaces/enp7s2/ipv4address

systemctl restart network && ip -c a
```
📤 Ожидаемый вывод: Наличие строки inet 172.16.10.1 на вашем LAN-интерфейсе означает, что маршрутизатор филиала успешно создал шлюз по умолчанию и готов принимать трафик от локальных серверов.
```bash
3: [Ваш LAN-порт]: ... state UP ...
    inet 172.16.10.1/28 scope global [Ваш LAN-порт]
```

В терминале сервера **BR-SRV**:
📥 Отключите автоматику, задайте статические параметры сети и синхронизируйте часы.
```bash
systemctl disable --now NetworkManager 2>/dev/null || true

mkdir -p /etc/net/ifaces/___
echo "TYPE=eth" > /etc/net/ifaces/enp7s1/options
echo "172.16.10.2/28" > /etc/net/ifaces/enp7s1/ipv4address
echo "default via 172.16.10.1" > /etc/net/ifaces/enp7s1/ipv4route
echo "nameserver 8.8.8.8" > /etc/net/ifaces/___/resolv.conf

timedatectl set-timezone Asia/Vladivostok
```
📥 Примените настройки и проверьте локальную связь со шлюзом.
```bash
systemctl restart network && ping -c 2 172.16.10.1
```
📤 Ожидаемый вывод: Нулевой процент потерь (0% packet loss) доказывает, что сервер успешно связался со шлюзом филиала по локальной сети 172.16.10.x. Физический кабель и L3-адресация исправны.
```bash
64 bytes from 172.16.10.1: icmp_seq=1 ttl=64 time=0.527 ms
64 bytes from 172.16.10.1: icmp_seq=2 ttl=64 time=0.301 ms

--- 172.16.10.1 ping statistics ---
2 packets transmitted, 2 received, 0% packet loss, time 1002ms
```

 В терминале маршрутизатора **HQ-RTR**:
📥 Создайте туннель и задайте внешний конверт.
Интерфейс поднимется, даже если на том конце никого нет. GRE — протокол без состояния (stateless). Он не проверяет соединение, он просто шлет пакеты в темноту.
```bash
mkdir -p /etc/net/ifaces/gre1

cat << EOF > /etc/net/ifaces/gre1/options
TYPE=iptun
TUNTYPE=gre
TUNLOCAL=172.16.1.2
TUNREMOTE=172.16.2.2
TUNTTL=64
TUNOPTIONS='ttl 64'
EOF
```
📥 Задайте внутренний IP-адрес и поднимите интерфейс.
```bash
echo "10.10.10.1/30" > /etc/net/ifaces/gre1/ipv4address

systemctl restart network && ip -c a show gre1
```
📤 Ожидаемый вывод: Флаги POINTOPOINT и NOARP доказывают, что интерфейс работает в режиме изолированного туннеля. В отличие от обычных сетевых карт, здесь нет аппаратных MAC-адресов (ARP не нужен), так как это закрытая «труба», на другом конце которой может находиться строго один получатель.
```bash
13: gre1@NONE: <POINTOPOINT,NOARP,UP,LOWER_UP> mtu 1476 ...
    link/gre 172.16.1.2 peer 172.16.2.2
    inet 10.10.10.1/30 scope global gre1
```
    ...
 В терминале маршрутизатора **BR-RTR**:
📥 Замкните туннель со стороны филиала.
Настройки строго зеркальны главному офису: то, что было удаленным, становится локальным.
⌨️ Впишите вместо [Внешний IP BR] ваш собственный внешний (WAN) IP-адрес роутера BR-RTR (172.16.2.2).
⌨️ Впишите вместо [Внешний IP HQ] внешний IP-адрес главного офиса HQ-RTR (172.16.1.2).
```bash
mkdir -p /etc/net/ifaces/gre1

cat << EOF > /etc/net/ifaces/gre1/options
TYPE=iptun
TUNTYPE=gre
TUNLOCAL=172.16.2.2
TUNREMOTE=172.16.1.2
TUNTTL=64
TUNOPTIONS='ttl 64'
EOF
```
📥 Задайте внутренний IP, поднимите интерфейс и проверьте связность.
Команда ping попытается достучаться до внутреннего адреса туннеля главного офиса.
```bash
echo "10.10.10.2/30" > /etc/net/ifaces/gre1/ipv4address

systemctl restart network && ping -c 2 10.10.10.1
```
📤 Ожидаемый вывод: Успешный пинг на этот адрес означает, что ваш ICMP-запрос был «спрятан» в GRE-конверт, прошел сквозь публичную сеть провайдера ISP, распаковался на HQ-RTR и благополучно вернулся обратно. Магистральный туннель между офисами физически пробит.
```bash
64 bytes from 10.10.10.1: icmp_seq=1 ttl=64 time=1.2 ms
64 bytes from 10.10.10.1: icmp_seq=2 ttl=64 time=1.1 ms

--- 10.10.10.1 ping statistics ---
2 packets transmitted, 2 received, 0% packet loss, time 1002ms
```
 В терминалах обоих маршрутизаторов (**HQ-RTR** и **BR-RTR**):
📥 Активируйте демон OSPF.
По умолчанию все протоколы динамической маршрутизации в ALT Linux выключены. Утилита sed найдет строку ospfd=no и изменит её на yes. Выполните эту команду по очереди в обеих консолях.
```bash
sed -i 's/ospfd=no/ospfd=yes/' /etc/frr/daemons
```
 В терминале маршрутизатора **HQ-RTR**:
📥 Запишите рабочий конфиг OSPF для главного офиса.
Мы создаем конфигурационный файл с нуля, используя синтаксис командной оболочки Cisco (vtysh).
```bash
cat <<'EOF' > /etc/frr/frr.conf
interface gre
 no ip ospf passive
exit
!
interface gre1
 ip ospf area 0
 ip ospf authentication
 ip ospf authentication-key P@ssw0rd
 no ip ospf passive
exit
!
interface vlan100
 ip ospf area 0
exit
!
interface vlan200
 ip ospf area 0
exit
!
interface vlan999
 ip ospf area 0
exit
!
router ospf
 passive-interface default
exit
EOF
```
📥 Запустите службу маршрутизации.
```bash
systemctl enable --now frr
```

💻 В терминале маршрутизатора **BR-RTR**:
📥 Запишите рабочий конфиг OSPF для филиала.
```bash
cat <<'EOF' > /etc/frr/frr.conf
interface gre1
 ip ospf area 0
 ip ospf authentication
 ip ospf authentication-key P@ssw0rd
 no ip ospf passive
exit
!
interface enp7s2
 ip ospf area 0
exit
!
router ospf
 passive-interface default
exit
EOF
```
📥 Запустите службу маршрутизации.
```bash
systemctl enable --now frr
```
В терминале маршрутизатора **BR-RTR**:
📥 Проверьте статус соседства.
⚠ Внимание: После запуска службы frr маршрутизаторам нужно 10-15 секунд, чтобы «найти» друг друга. Немного подождите перед вводом команды. Утилита vtysh -c позволяет выполнить команду во внутреннем интерпретаторе frr без входа в него.

```bash
vtysh -c "show ip ospf neighbor"
```

📤 Ожидаемый вывод: Ищите глазами состояние **Full** (или Full/DR, Full/Backup, Full/-). Это означает, что маршрутизаторы полностью обменялись картами сетей. Если там написано Init или пустота — OSPF мертв (проверьте пароли или опечатки в frr.conf).
```bash
10.10.10.1      1 Full/-          ... 10.10.10.2      gre1:10.10.10.1
```
📥 Убедитесь, что ядро Linux получило новые маршруты.
```bash
ip route | grep 172.16
```
📤 Ожидаемый вывод: Буква O (в Cisco) или протокол zebra (в FRR Linux) в выводе означает, что маршрут добавлен в таблицу не руками системного администратора, а динамически «выучен» от соседа через OSPF. Маршрутизатор филиала узнал, что для связи с сетями HQ (172.16.100.0/28 и 172.16.200.0/27) нужно отправлять пакеты в туннель gre1.
```bash
172.16.100.0/28 nhid 18 via 10.10.10.1 dev gre1 proto zebra metric 20
172.16.200.0/27 nhid 18 via 10.10.10.1 dev gre1 proto zebra metric 20
172.16.99.0/29 nhid 18 via 10.10.10.1 dev gre1 proto zebra metric 20
```

💻 В терминале сервера **HQ-SRV**:
📥 Выполните финальный тест сквозной маршрутизации (Момент истины).
Запустите проверку связи с сервером филиала (172.16.10.2). Ваш пакет пройдет через коммутатор Proxmox, поднимется на маршрутизатор HQ-RTR, «нырнёт» в GRE-туннель, пролетит сквозь провайдера ISP (незамеченным), выйдет из туннеля на BR-RTR и достигнет цели в филиале.
```bash
ping -c 2 172.16.10.2
```
📤 Ожидаемый вывод: Нулевой процент потерь доказывает, что OSPF успешно проложил маршрут сквозь GRE-туннель. Локальные (серые) сети двух удаленных городов физически объединены в единую корпоративную L3-магистраль.
```bash
64 bytes from 172.16.10.2: icmp_seq=1 ttl=62 time=2.1 ms
64 bytes from 172.16.10.2: icmp_seq=2 ttl=62 time=1.9 ms

--- 172.16.10.2 ping statistics ---
2 packets transmitted, 2 received, 0% packet loss, time 1002ms
```

**//// 4 тетрадка**

В терминале сервера **HQ-SRV**:
📥 Ускорьте загрузку и установите ПО.
Команда sed перехватит обращения к медленным серверам ALT Linux и перенаправит их на локальный кэш 10.45.33.250. Установка пройдет мгновенно.
```bash
sed -i 's|http://ftp.altlinux.org/pub/distributions/ALTLinux|http://10.45.33.250:3142/altlinux|g' /etc/apt/sources.list.d/alt.list
apt-get update && apt-get install -y bind bind-utils
```
📥 Сгенерируйте ключ управления и подготовьте хранилище зон.
```bash
rndc-confgen -a -c /etc/bind/rndc.key
mkdir -p /etc/bind/zone
```
📥 Настройте глобальные параметры BIND.
```bash
cat <<EOF > /etc/bind/options.conf
options {
    recursion yes;
    allow-query { any; };
    forwarders { 8.8.8.8; 77.88.8.8; };

    dnssec-validation no;
    listen-on-v6 { none; };
};
EOF
```

 В терминале сервера **HQ-SRV**:
📥 Объявите зоны в главном реестре.
```bash
cat <<EOF > /etc/bind/local.conf
zone "au-team.irpo" {
    type master;
    file "/etc/bind/zone/au-team.irpo";
};

zone "16.172.in-addr.arpa" {
    type master;
    file "/etc/bind/zone/16.172.in-addr.arpa";
};
EOF
```
📥 Создайте файл прямой зоны (Forward Zone).
Этот файл переводит человекочитаемые доменные имена (FQDN) в IP-адреса с помощью записей типа A (Address Record):
- Синтаксис имен: Если слева указано короткое имя (например, hq-rtr), сервер автоматически допишет к нему ваш домен (.au-team.irpo).
- Закон точки: Обратите внимание на точки (.) в конце полных доменных имен (например, root.au-team.irpo.). Точка обозначает «Корень Интернета» (абсолютное имя). Если вы забудете поставить точку в конце, BIND посчитает имя относительным и ошибочно приклеит к нему ваш домен еще раз (получится root.au-team.irpo.au-team.irpo), что приведет к фатальному сбою разрешения имен.
```bash
cat <<EOF > /etc/bind/zone/au-team.irpo
\$TTL    604800
@       IN      SOA     hq-srv.au-team.irpo. root.au-team.irpo. (
                              1         ; Serial
                         604800         ; Refresh
                          86400         ; Retry
                        2419200         ; Expire
                         604800 )       ; Negative Cache TTL
;
@       IN      NS      hq-srv.au-team.irpo.
@       IN      A       172.16.100.2
;
hq-srv  IN      A       172.16.100.2
hq-rtr  IN      A       172.16.100.1
hq-cli  IN      A       172.16.200.10
br-rtr  IN      A       172.16.2.2
br-srv  IN      A       172.16.10.2
docker  IN      A       172.16.1.1
web     IN      A       172.16.2.1
EOF
```
📥 Создайте файл обратной зоны (Reverse Zone).
Обратная зона делает наоборот — переводит известные IP-адреса обратно в имена хостов с помощью записей типа PTR(Pointer Record):
- Синтаксис сети: Имя самой зоны (16.172.in-addr.arpa) читается как сеть 172.16.0.0/16, записанная задом наперед. Это международный стандарт архитектуры DNS.
- Запись хостов: В левой колонке мы указываем только недостающие октеты (цифры) конкретного IP-адреса (в обратном порядке), так как первые два октета уже зашифрованы в названии зоны. Например, цифры 2.100 ядро сложит с названием зоны и превратит в адрес 172.16.100.2.
```bash
cat <<EOF > /etc/bind/zone/16.172.in-addr.arpa
\$TTL    604800
@       IN      SOA     hq-srv.au-team.irpo. root.au-team.irpo. (
                              1         ; Serial
                         604800         ; Refresh
                          86400         ; Retry
                        2419200         ; Expire
                         604800 )       ; Negative Cache TTL
;
@       IN      NS      hq-srv.au-team.irpo.
;
2.100   IN      PTR     hq-srv.au-team.irpo.
1.100   IN      PTR     hq-rtr.au-team.irpo.
10.200  IN      PTR     hq-cli.au-team.irpo.
EOF
```
📥 Исправьте права доступа и запустите службу.
```bash
chown root:named /etc/bind/zone/*
chmod 640 /etc/bind/zone/*

systemctl enable --now bind
systemctl status bind --no-pager
```
📤 Ожидаемый вывод: Вы должны увидеть зеленый индикатор Active: active (running).
💡 Примечание: Если в логах службы вы видите предупреждение the working directory is not writable — это нормально. Мы выдали демону права только на чтение файлов (Read-Only) ради безопасности. Главное — убедитесь в наличии строк loaded serial 1 для обеих зон и финальной надписи all zones loaded.
```bash
Active: active (running) since ...
```
...
```bash
zone 16.172.in-addr.arpa/IN: loaded serial 1
zone au-team.irpo/IN: loaded serial 1
all zones loaded
running
```

В терминалах **HQ-SRV**, **HQ-RTR**, **BR-RTR** и **BR-SRV**:
ℹ️ Справка: Символ подстановки (Wildcard)
📥 Выполните команду на сервере **HQ-SRV** (Сервер имен).
```bash
sed -i 's/8.8.8.8/127.0.0.1/g' /etc/net/ifaces/*/resolv.conf
reboot
```

📥 Выполните команду на трёх остальных узлах (HQ-RTR, BR-RTR, BR-SRV).
```bash
sed -i 's/8.8.8.8/172.16.100.2/g' /etc/net/ifaces/*/resolv.conf && systemctl restart network
```

В терминале клиента **HQ-CLI**:
📥 Протестируйте резолвинг имен.
Утилита host генерирует DNS-запрос к серверу, указанному в системных настройках. Ошибок connection timed outбыть не должно.
```bash
host hq-srv.au-team.irpo
host 172.16.100.2
host ya.ru
```
📤 Ожидаемый вывод: Изучите ответ консоли. Каждый успешный ответ доказывает работоспособность конкретного сегмента архитектуры BIND:
- Вывод адреса **172.16.100.2** подтверждает, что ваша Прямая зона загружена в память и синтаксически верна.
- Вывод строки **domain name pointer** доказывает, что Обратная зона (PTR) успешно связывает IP-адрес с именем узла.
- Вывод адресов Яндекса (например, 5.255.255.242) означает, что ваш сервер успешно перехватил неизвестный ему запрос и сработал как прозрачный прокси-сервер (Рекурсия), переслав его публичным DNS через директиву forwarders.
```bash
hq-srv.au-team.irpo has address 172.16.100.2

2.100.16.172.in-addr.arpa domain name pointer hq-srv.au-team.irpo.

ya.ru has address 213.180.193.56
ya.ru mail is handled by 10 mx.yandex.ru.
```
В терминалах серверов **HQ-SRV** и **BR-SRV**:
📥 Создайте серверных администраторов.
```bash
useradd -u 2026 -m -s /bin/bash sshuser
echo "sshuser:P@ssw0rd" | chpasswd

mkdir -p /etc/sudoers.d/
echo "sshuser ALL=(ALL) NOPASSWD: ALL" > /etc/sudoers.d/sshuser
```

💻 В терминалах маршрутизаторов **HQ-RTR** и **BR-RTR**:
📥 Создайте сетевых инженеров.
```bash
useradd -m -s /bin/bash net_admin
echo "net_admin:P@ssw0rd" | chpasswd

mkdir -p /etc/sudoers.d/
echo "net_admin ALL=(ALL) NOPASSWD: ALL" > /etc/sudoers.d/net_admin
```

В терминалах серверов **HQ-SRV** и **BR-SRV**:
📥 Проверьте наличие учетной записи серверного администратора.
```bash
id sshuser
```
📤 Ожидаемый вывод: Убедитесь, что система присвоила пользователю строго запрошенный идентификатор. Если вы видите no such user — вы опечатались в имени при создании.
```bash
uid=2026(sshuser) gid=2026(sshuser) groups=2026(sshuser)
```
💻 В терминалах маршрутизаторов **HQ-RTR** и **BR-RTR**:
📥 Проверьте наличие учетной записи сетевого инженера.
```bash
id net_admin
```
📤 Ожидаемый вывод: Поскольку мы не задавали идентификатор жестко, ядро выдало первый свободный номер. Главное — убедиться, что пользователь существует.
```bash
uid=[Число](net_admin) gid=[Число](net_admin) groups=[Число](net_admin)
```
В терминалах серверов **HQ-SRV** и **BR-SRV**:
📥 Создайте текстовый баннер безопасности.
Этот текст будет выводиться каждому пользователю еще до запроса пароля, юридически предупреждая о недопустимости несанкционированного доступа.
```bash
echo "Authorized access only" > /etc/issue.net
```
📥 Внедрите политики Hardening в конфигурацию демона.
Мы добавляем в конец системного файла sshd_config блок правил:
- Port 2026 — Скрывает службу от базовых автоматических сканеров.
- PermitRootLogin no — Уничтожает главный вектор брутфорс-атак (взломщика пароля рута).
- MaxAuthTries 2 — Ограничивает количество попыток подбора пароля за одну сессию.
- AllowUsers sshuser — Жесткий White-list (белый список). Любой другой пользователь (даже с правильным паролем) получит отказ в доступе.
```bash
cat <<EOF >> /etc/openssh/sshd_config

# Security Hardening Rules
Port 2026
PermitRootLogin no
MaxAuthTries 2
Banner /etc/issue.net
AllowUsers sshuser
EOF
```
📥 Примените политики безопасности.
```bash
systemctl restart sshd
```
В терминале клиента **HQ-CLI**:
📥 Выстрелите «Серебряной пулей».
⌨️ Впишите пароль P@ssw0rd, когда терминал запросит авторизацию.
```bash
ssh -p 2026 sshuser@br-srv.au-team.irpo "ping -c 2 ya.ru && date"
```
📤 Ожидаемый вывод (Анатомия чуда): Вас встретит установленный ранее баннер. Сразу после ввода пароля вы получите ответы от Яндекса с правильным часовым сдвигом (ваше локальное время).
```bash
Authorized access only
sshuser@br-srv.au-team.irpo's password:

PING ya.ru (213.180.193.56) ...
64 bytes from familysearch.yandex.ru (213.180.193.56): icmp_seq=1 ttl=48 time=118 ms

[День недели] [Месяц] [Число] [ЧЧ:ММ:СС] +10 2026
```
