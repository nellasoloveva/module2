# 1 модуль — шпаргалка замен для GitHub

> Формат для GitHub: все значения, которые могут измениться на экзамене, выделены через `<mark>...</mark>`.  
> Перед вводом команд в терминал подставляй значения из ТЗ.

## Легенда

| Тип значения | Что искать |
|---|---|
| VLAN | номера VLAN в Proxmox и в `VID=` |
| IP / маска | адреса вида `172.16.../28`, `10.10.../30` |
| Интерфейс | `enp7s1`, `enp7s2`, `enp7s3` — проверять через `ip -br a` |
| Домен | `au-team.irpo` и DNS-зоны |
| SSH | порт, пользователь, UID, пароль |
| GRE/OSPF/DHCP/DNS | связанные адреса и интерфейсы |

---

## Раздел 1 — Proxmox: VLAN Tag

<pre>
HQ-SRV → Hardware → Network Device → VLAN Tag: <mark>100</mark>
HQ-CLI → Hardware → Network Device → VLAN Tag: <mark>200</mark>
</pre>

Важно: эти VLAN должны совпадать с `VID=` на `HQ-RTR`.

---

## Раздел 2 — ISP

<pre>
hostnamectl set-hostname isp.<mark>au-team.irpo</mark>

WAN interface: <mark>enp7s1</mark> → DHCP
To HQ-RTR:     <mark>enp7s2</mark> → <mark>172.16.1.1/28</mark>
To BR-RTR:     <mark>enp7s3</mark> → <mark>172.16.2.1/28</mark>

NAT outside interface: oifname "<mark>enp7s1</mark>"
timezone: <mark>Asia/Vladivostok</mark>
</pre>

---

## Раздел 3 — HQ-RTR

<pre>
hostnamectl set-hostname hq-rtr.<mark>au-team.irpo</mark>

Uplink interface: <mark>enp7s1</mark>
HQ-RTR uplink IP: <mark>172.16.1.2/28</mark>
Gateway:          <mark>172.16.1.1</mark>
DNS temporary:    <mark>8.8.8.8</mark>

Trunk interface to LAN: <mark>enp7s2</mark>

Server VLAN interface: vlan<mark>100</mark>
VID:                   <mark>100</mark>
Server VLAN gateway:   <mark>172.16.100.1/28</mark>

Client VLAN interface: vlan<mark>200</mark>
VID:                   <mark>200</mark>
Client VLAN gateway:   <mark>172.16.200.1/27</mark>

Mgmt VLAN interface: vlan<mark>999</mark>
VID:                 <mark>999</mark>
Mgmt VLAN gateway:   <mark>172.16.99.1/29</mark>

GRE local:  <mark>172.16.1.2</mark>
GRE remote: <mark>172.16.2.2</mark>
GRE tunnel: <mark>10.10.10.1/30</mark>

NAT outside interface: oifname "<mark>enp7s1</mark>"
</pre>

### DHCP на HQ-RTR

<pre>
DHCP interface: vlan<mark>200</mark>
DHCP range:     <mark>172.16.200.10</mark> – <mark>172.16.200.14</mark>
DHCP mask:      <mark>255.255.255.224</mark>
Client-ID:      <mark>01:11:11:11:11:11:11</mark>
Reserved IP:    <mark>172.16.200.10</mark>
Client gateway: <mark>172.16.200.1</mark>
Client DNS:     <mark>172.16.100.2</mark>
DHCP domain:    <mark>au-team.irpo</mark>
</pre>

### OSPF на HQ-RTR

<pre>
OSPF interfaces:
- gre1
- vlan<mark>100</mark>
- vlan<mark>200</mark>
- vlan<mark>999</mark>

OSPF auth-key: <mark>P@ssw0rd</mark>
</pre>

### Локальный админ на HQ-RTR

<pre>
user:     <mark>net_admin</mark>
password: <mark>P@ssw0rd</mark>
timezone: <mark>Asia/Vladivostok</mark>
</pre>

---

## Раздел 4 — HQ-SRV

<pre>
hostnamectl set-hostname hq-srv.<mark>au-team.irpo</mark>

Interface: <mark>enp7s1</mark>
IP:        <mark>172.16.100.2/28</mark>
Gateway:   <mark>172.16.100.1</mark>
DNS:       <mark>127.0.0.1</mark>
Search:    <mark>au-team.irpo</mark>
</pre>

### BIND DNS

<pre>
Forwarders: <mark>8.8.8.8</mark>, <mark>77.88.8.8</mark>
Forward zone: <mark>au-team.irpo</mark>
Reverse zone: <mark>16.172.in-addr.arpa</mark>

A records:
hq-srv  → <mark>172.16.100.2</mark>
hq-rtr  → <mark>172.16.100.1</mark>
hq-cli  → <mark>172.16.200.10</mark>
br-rtr  → <mark>172.16.2.2</mark>
br-srv  → <mark>172.16.10.2</mark>
docker  → <mark>172.16.1.1</mark>
web     → <mark>172.16.2.1</mark>
</pre>

### SSH hardening на HQ-SRV

<pre>
user:     <mark>sshuser</mark>
UID:      <mark>2026</mark>
password: <mark>P@ssw0rd</mark>
SSH Port: <mark>2026</mark>
Banner:   <mark>Authorized access only</mark>
AllowUsers: <mark>sshuser</mark>
timezone: <mark>Asia/Vladivostok</mark>
</pre>

---

## Раздел 5 — BR-RTR

<pre>
hostnamectl set-hostname br-rtr.<mark>au-team.irpo</mark>

Uplink interface: <mark>enp7s1</mark>
BR-RTR uplink IP: <mark>172.16.2.2/28</mark>
Gateway:          <mark>172.16.2.1</mark>
DNS temporary:    <mark>8.8.8.8</mark>

BR LAN interface: <mark>enp7s2</mark>
BR LAN gateway:   <mark>172.16.10.1/28</mark>

GRE local:  <mark>172.16.2.2</mark>
GRE remote: <mark>172.16.1.2</mark>
GRE tunnel: <mark>10.10.10.2/30</mark>

NAT outside interface: oifname "<mark>enp7s1</mark>"
OSPF auth-key: <mark>P@ssw0rd</mark>
user: <mark>net_admin</mark>, password: <mark>P@ssw0rd</mark>
timezone: <mark>Asia/Vladivostok</mark>
</pre>

---

## Раздел 6 — BR-SRV

<pre>
hostnamectl set-hostname br-srv.<mark>au-team.irpo</mark>

Interface: <mark>enp7s1</mark>
IP:        <mark>172.16.10.2/28</mark>
Gateway:   <mark>172.16.10.1</mark>
DNS:       <mark>8.8.8.8</mark>

user:     <mark>sshuser</mark>
UID:      <mark>2026</mark>
password: <mark>P@ssw0rd</mark>
SSH Port: <mark>2026</mark>
AllowUsers: <mark>sshuser</mark>
timezone: <mark>Asia/Vladivostok</mark>
</pre>

---

## Раздел 7 — HQ-CLI

<pre>
hostnamectl set-hostname hq-cli.<mark>au-team.irpo</mark>

Interface: <mark>enp7s1</mark>
Connection name: <mark>lan</mark>
DHCP Client-ID: <mark>01:11:11:11:11:11:11</mark>
timezone: <mark>Asia/Vladivostok</mark>
</pre>

---

## Итоговая проверка

<pre>
ip -br a show <mark>enp7s1</mark>
ping -c 2 <mark>ya.ru</mark>
host br-srv.<mark>au-team.irpo</mark>
ssh -p <mark>2026</mark> <mark>sshuser</mark>@br-srv.<mark>au-team.irpo</mark> "date"
password: <mark>P@ssw0rd</mark>
</pre>

---

## Быстрый поиск перед экзаменом

Ищи по файлу: `<mark>` или конкретные значения: `100`, `200`, `999`, `2026`, `au-team.irpo`, `172.16`, `10.10.10`, `sshuser`, `P@ssw0rd`, `enp7s`.
