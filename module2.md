2 МОДУЛЬ

Справочная таблица

| Узел (Hostname) | Сегмент (Домен) | IP-адрес / Маска | Роль в Модуле 2 |
| --- | --- | --- | --- |
| HQ-SRV | Серверы (VLAN 100) | `192.168.1.10/27` | Файловый сервер (RAID/NFS) |
| HQ-CLI | Клиенты (VLAN 200) | `192.168.2.10/28` | Рабочая станция (Ввод в домен) |
| BR-SRV | Филиал (BR-LAN) | `192.168.3.10/28` | Контроллер домена (AD) / Ansible |
| VPN Tunnel | GRE (HQ <-> BR) | `192.168.5.1` – `.2 /30` | L3-Связность между офисами |
| Управление | VLAN 999 (HQ) | `192.168.9.1/29` | Изолированный сегмент (Резерв) |

В терминале маршрутизатора **ISP**:

```bash
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
systemctl restart nftables
sed -i 's|http://ftp.altlinux.org/pub/distributions/ALTLinux|http://10.45.33.250:3142/altlinux|g' /etc/apt/sources.list.d/alt.list
apt-get update && apt-get install -y chrony
```

В терминале маршрутизатора **HQ-RTR**:

```bash
echo "nameserver 77.88.8.8" > /etc/resolv.conf
sed -i 's|http://ftp.altlinux.org/pub/distributions/ALTLinux|http://10.45.33.250:3142/altlinux|g' /etc/apt/sources.list.d/alt.list
apt-get update && apt-get install -y chrony
sed -i 's/192.168.1.10/192.168.3.10/' /etc/dnsmasq.conf
sed -i 's/192.168.1.10/192.168.3.10/' /etc/net/ifaces/*/resolv.conf 2>/dev/null || true
cat <<EOF > /etc/openssh/sshd_config
Port 22
PermitRootLogin yes
Subsystem sftp /usr/lib/openssh/sftp-server
EOF
systemctl enable sshd
systemctl restart dnsmasq network sshd
```

На сервере **HQ-SRV**:

```bash
echo "nameserver 77.88.8.8" > /etc/resolv.conf
sed -i 's|http://ftp.altlinux.org/pub/distributions/ALTLinux|http://10.45.33.250:3142/altlinux|g' /etc/apt/sources.list.d/alt.list
apt-get update && apt-get install -y chrony
systemctl disable --now bind 2>/dev/null || true
sed -i -E 's/(192\.168\.1\.10|127\.0\.0\.1)/192.168.3.10/' /etc/net/ifaces/*/resolv.conf 2>/dev/null || true
cat <<EOF > /etc/openssh/sshd_config
Port 22
PermitRootLogin yes
Subsystem sftp /usr/lib/openssh/sftp-server
EOF
systemctl enable sshd
systemctl restart network sshd
systemctl is-active bind
```

В терминале клиента **HQ-CLI**:

```bash
cat <<EOF > /etc/openssh/sshd_config
Port 22
PermitRootLogin yes
Subsystem sftp /usr/lib/openssh/sftp-server
EOF
systemctl enable --now sshd
echo "nameserver 77.88.8.8" > /etc/resolv.conf
sed -i 's|http://ftp.altlinux.org/pub/distributions/ALTLinux|http://10.45.33.250:3142/altlinux|g' /etc/apt/sources.list.d/alt.list
apt-get update && apt-get install -y chrony task-auth-ad-sssd libnss-role
```

В терминале маршрутизатора **BR-RTR**:

```bash
echo "nameserver 77.88.8.8" > /etc/resolv.conf
sed -i 's|http://ftp.altlinux.org/pub/distributions/ALTLinux|http://10.45.33.250:3142/altlinux|g' /etc/apt/sources.list.d/alt.list
apt-get update && apt-get install -y chrony
sed -i 's/192.168.1.10/192.168.3.10/' /etc/net/ifaces/*/resolv.conf 2>/dev/null || true
cat <<EOF > /etc/openssh/sshd_config
Port 22
PermitRootLogin yes
Subsystem sftp /usr/lib/openssh/sftp-server
EOF
systemctl enable sshd
systemctl restart network sshd
```

В терминале сервера **BR-SRV**:

```bash
echo "nameserver 77.88.8.8" > /etc/resolv.conf
sed -i 's|http://ftp.altlinux.org/pub/distributions/ALTLinux|http://10.45.33.250:3142/altlinux|g' /etc/apt/sources.list.d/alt.list
apt-get update && apt-get install -y chrony ansible sshpass task-samba-dc bind-utils
systemctl disable --now bind 2>/dev/null || true
sed -i 's/192.168.1.10/192.168.3.10/' /etc/net/ifaces/*/resolv.conf 2>/dev/null || true
cat <<EOF > /etc/openssh/sshd_config
Port 22
PermitRootLogin yes
Subsystem sftp /usr/lib/openssh/sftp-server
EOF
systemctl enable sshd
systemctl restart network sshd
ansible --version
```

✍️ Артефакт

Требование к сдаче: Коллаж из двух скриншотов с консолей BR-SRV и HQ-SRV.

☑️ Самопроверка:

 Вывод версии Ansible: [core 2.x.x] (или выше).

 Статус службы **bind**: inactive (dead).

📎 Ваш: [Вставьте коллаж скриншотов]

Чтоб сделать скриншот на машинах  BR-SRV и HQ-SRV пишешь сначала 
ansible --version

потом
systemctl is-active bind

должно быть инактив типо выключано

В терминале сервера **BR-SRV**:

```bash
ping -c 2 192.168.1.1
```

📤 Ожидаемый вывод: Пакеты должны благополучно пройти сквозь GRE-туннель.

```bash
64 bytes from 192.168.1.1: icmp_seq=1 ttl=63 time=1.4 ms
...
2 packets transmitted, 2 received, 0% packet loss
```

На маршрутизаторе **ISP**:

```bash
apt-get update && apt-get install -y chrony
cat <<EOF >> /etc/chrony.conf
port 123
local stratum 5
allow 0.0.0.0/0
EOF
```

.

```bash
systemctl enable --now chronyd
systemctl restart chronyd
ss -ulnp | grep 123
chronyc tracking
```

📤 Ожидаемый вывод: Команда ss должна показать 0.0.0.0:123 (порт открыт). Команда tracking должна содержать строку Stratum : 5 (или меньше, если есть внешняя связь).

Артефакт

Требование к сдаче:

Скриншот вывода chronyc tracking.

В 📋 Полном списке заданий (Модуль 2) найдите пункт, регламентирующий уровень Stratum (стратум) для сервера времени.

☑️ Самопроверка:

 Значение поля Stratum: 5 (или меньше).

💬 Трассировка требований:

Требование: [Вставьте цитату из задания]

Реализация: [Вставьте ваш вариант настройки]

📎 Ваш артефакт: [Вставьте ваш скриншот]

Для машин зоны HQ (**HQ-RTR**, **HQ-SRV**, **HQ-CLI**):

📥 Настройте синхронизацию с шлюзом.

Мы удаляем старые пулы (sed) и добавляем наш локальный эталон (ISP).

```bash
sed -i '/pool/d' /etc/chrony.conf
sed -i '/server/d' /etc/chrony.conf
echo "server 172.16.1.1 iburst prefer" >> /etc/chrony.conf
```

📥 Перезапустите и проверьте.

```bash
systemctl restart chronyd
sleep 10 && chronyc sources
```

Для машин зоны BR (**BR-RTR**, **BR-SRV**):

📥 Настройте синхронизацию с шлюзом.

Для филиала адресом эталона (ISP) является 172.16.2.1.

```bash
sed -i '/pool/d' /etc/chrony.conf
sed -i '/server/d' /etc/chrony.conf
echo "server 172.16.2.1 iburst prefer" >> /etc/chrony.conf
```

📥 Перезапустите и проверьте.

```bash
systemctl restart chronyd
sleep 10 && chronyc sources
```

Артефакт

Требование к сдаче:

Коллаж из двух скриншотов (HQ + BR) с выводом chronyc sources.

В 📋 Полном списке заданий (Модуль 2) найдите требование к списку NTP-клиентов. Докажите строкой кода, что машины синхронизируются с нужным сервером.

☑️ Самопроверка:

 Статус источника (символ слева): \* (звездочка).

 IP-адрес источника в HQ: 172.16.1.1.

 IP-адрес источника в BR: 172.16.2.1.

💬 Трассировка требований:

Требование: [Вставьте цитату из задания]

Реализация: [Вставьте ваш вариант настройки]

📎 Ваш артефакт: [Вставьте коллаж скриншотов]

На сервере **BR-SRV**:

📥 Создайте конфигурационные файлы и запустите проверку.

```bash
mkdir -p /etc/ansible
cat <<EOF > /etc/ansible/ansible.cfg
[defaults]
inventory = /etc/ansible/hosts
host_key_checking = False
EOF
cat <<EOF > /etc/ansible/hosts
[all:vars]
ansible_user=root
ansible_password=toor
ansible_python_interpreter=/usr/bin/python3
[targets]
192.168.1.10    # HQ-SRV
192.168.2.10    # HQ-CLI
192.168.1.1     # HQ-RTR (LAN)
192.168.3.1     # BR-RTR (LAN)
EOF
ansible all -m ping
```

📤 Ожидаемый вывод: Четыре зеленых SUCCESS блока с ответом pong.

```bash
192.168.3.1 | SUCCESS => {
   "changed": false,
   "ping": "pong"
```

}

```bash
192.168.1.1 | SUCCESS => {
   "changed": false,
   "ping": "pong"
```

}

```bash
...
```

Во всех открытых терминалах:

📥 Сохраните историю команд. Сессия при этом не прервётся.

```bash
history -a
```

📋 **Карта доступа к стенду:**

- **Пользователь:** `root`
- **Пароль:** `toor` (или `P@ssw0rd`)
- **Домен (FQDN):** `au-team.irpo`
- **Домен (NetBIOS):** `AU-TEAM`
- **Domain Admin Password:** `P@ssw0rd`

2 тетрадь

В терминале сервера **BR-SRV**:

📥 Подготовьте среду, подавив демоны-конкуренты.

```bash
systemctl disable --now smb nmb krb5kdc slapd bind 2>/dev/null || true
rm -f /etc/samba/smb.conf
```

📥 Инициализируйте мастер генерации домена

!!!!!!!!!!!!!!!!!!!!!!
В мастере настройки **samba-tool**:( чуть позже начнется вы поймете)

Realm: Введите AU-TEAM.IRPO (строго заглавными буквами) и нажмите Enter.

Domain: Значение [AU-TEAM] установлено по умолчанию. Нажмите Enter.

Server Role: Значение [dc] установлено по умолчанию. Нажмите Enter.

DNS backend: Значение [SAMBA_INTERNAL] установлено по умолчанию. Нажмите Enter.

DNS forwarder IP address: Введите 77.88.8.8 и нажмите Enter.

Administrator password: Введите P@ssw0rd и нажмите Enter. Подтвердите пароль.

.

```bash
samba-tool domain provision
```

В терминале сервера **BR-SRV**:

```bash
\cp -f /var/lib/samba/private/krb5.conf /etc/krb5.conf
systemctl enable --now samba
```

📤 Ожидаемый вывод:

```bash
Created symbolic link...
```

артефакт

Требование к сдаче:

Скриншот финального вывода команды provision.

В 📋 Полном списке заданий (Модуль 2) найдите пункт, регламентирующий имя домена, и свяжите его с реализацией.

☑️ Самопроверка:

 Роль сервера: В выводе присутствует строка active directory domain controller.

 Ошибки в выводе: Отсутствуют (скрипт не оборвался на середине с ошибкой Python).

💬 Трассировка требований:

Требование: [Вставьте цитату из задания]

Реализация: [Вставьте ваш вариант настройки]

В терминале сервера **BR-SRV**:

📥 Замкните внутренний DNS на Localhost.

```bash
echo "nameserver 127.0.0.1" > /etc/net/ifaces/enp7s1/resolv.conf
echo "search au-team.irpo" >> /etc/net/ifaces/enp7s1/resolv.conf
systemctl restart network
```

⌨️ Впишите пароль P@ssw0rd, когда терминал запросит авторизацию Password for... (текст не будет отображаться на экране).

```bash
host -t A au-team.irpo
kinit administrator@AU-TEAM.IRPO
```

📥 Убедитесь, что центр безопасности (KDC) выдал вам криптографический пропуск.

```bash
klist
```

📤 Ожидаемый вывод: Вы должны увидеть таблицу билетов, где первой строкой значится Default principal: administrator@AU-TEAM.IRPO. Если её нет, значит пароль на предыдущем шаге введен неверно.

📥 Сгенерируйте группы и аккаунты сотрудников.

```bash
samba-tool group add hq
for i in {1..5}; do
  samba-tool user add hquser$i P@ssw0rd
  samba-tool user setexpiry hquser$i --noexpiry
  samba-tool group addmembers "hq" hquser$i
done
samba-tool group listmembers hq
```

Артефакт

Требование к сдаче:

Скриншот консоли с билетом Kerberos и списком пользователей.

Зафиксируйте имя сетевого интерфейса, на котором вы замкнули DNS, и обоснуйте выбор.

На клиенте **HQ-CLI**:

📥 Узнайте имя вашего профиля конфигурации.

Утилита nmcli работает не с «железом» (порт DEVICE), а с логическими профилями сети. Выполните команду ниже и скопируйте значение строго из первого столбца — NAME (например, System enp7s1 или DHCP-CLI). Не перепутайте!

```bash
nmcli c show --active
```

у нас имя: (дальше оно уже вписано)

```bash
DHCP-CLI
```

📥 Зафиксируйте DNS и введите ПК в домен.

```bash
nmcli con mod "DHCP-CLI" ipv4.dns "192.168.3.10"
nmcli con mod "DHCP-CLI" ipv4.ignore-auto-dns yes
nmcli con up "DHCP-CLI"
system-auth write ad au-team.irpo hq-cli au-team administrator P@ssw0rd
systemctl restart sssd
sleep 3
```

📤 Ожидаемый вывод: Возможно предупреждение No credentials cache found — это безопасность утилиты, игнорируйте его. Ошибок быть не должно. Задержка в 3 секунды позволит системе загрузить группы из домена.

📥 Настройте делегирование прав и ограничения.

```bash
control libnss-role enabled
roleadd hq wheel
cat <<EOF > /etc/sudoers.d/hq_restrictions
Cmnd_Alias HQ_CMDS = /bin/cat, /bin/grep, /usr/bin/id
%wheel ALL=(ALL) NOPASSWD: HQ_CMDS
EOF
```

📥 Подготовьте профиль и смените пользователя.

⚠️ Внимание: Смена контекста

После этой команды вы перестанете быть root’ом. Оболочка изменится, и скопировать оставшийся скрипт целиком уже не получится. Вводите оставшиеся команды строго по одной строке!

```bash
mkdir -p /home/AU-TEAM.IRPO/hquser1
\cp -af /etc/skel/. /home/AU-TEAM.IRPO/hquser1/
chown -R hquser1:hq /home/AU-TEAM.IRPO/hquser1
su - hquser1
```

📥 Проверьте доступность разрешенных команд.

```bash
sudo id
```

📥 Атакуйте собственные настройки (Стресс-тест).

```bash
sudo reboot
```

если выйдет ввести пароль ничего писать не надо выходиим и все!!!

📤 Ожидаемый вывод: Вы успешно зайдете под пользователем (появится привычное приглашение [hquser1@hq-cli ~]$). Команда id выполнится успешно (вы увидите UID hquser1), а reboot вернет жёсткий отказ: Sorry, user hquser1 is not allowed to execute '/sbin/reboot'. То, что нужно.

📥 Покиньте ограниченную оболочку.

```bash
exit
```

Во всех открытых терминалах:

```bash
history -a
```

Артефакт

Требование к сдаче:

Скриншот консоли клиента с демонстрацией работы ограничений sudo.

В Полном списке заданий (Модуль 2) найдите пункт, регламентирующий ограниченный набор команд для группы hq.

☑️ Самопроверка:

 Текущий пользователь: hquser1.

 Реакция на **reboot**: Sorry, user hquser1 is not allowed to execute....

💬 Трассировка требований:

Требование: [Вставьте цитату из задания]

Реализация: [Вставьте ваш вариант настройки]

📎 Ваш артефакт: [Вставьте скриншот]

3 тетрадка///

 На сервере **HQ-SRV**:

📥 Установите утилиту и соберите физический массив.

```bash
apt-get update && apt-get install -y mdadm
mdadm --create /dev/md0 --level=0 --raid-devices=2 /dev/sdb /dev/sdc --run
cat /proc/mdstat
```

📤 Ожидаемый вывод: В статусе ядра должно быть указано md0 : active raid0 sdc... sdb....

📥 Зафиксируйте метаданные и отформатируйте диск.

```bash
mdadm --detail --scan | tee -a /etc/mdadm.conf
mkfs.ext4 /dev/md0
```

📤 Ожидаемый вывод: Сообщение о создании суперблоков и таблиц файловой системы (Writing superblocks... done).

📥 Настройте автоматическое монтирование.

```bash
mkdir -p /raid
echo "/dev/md0 /raid ext4 defaults 0 0" >> /etc/fstab
mount -a
df -h | grep /raid
```

📤 Ожидаемый вывод: Вы должны увидеть строку с диском /dev/md0 размером около 2.0G, смонтированным в папку /raid.

 Артефакт

Требование к сдаче:

Скриншот вывода df -h с точкой монтирования RAID.

В Полном списке заданий (Модуль 2) найдите пункт, регламентирующий автоматическое монтирование раздела.

☑️ Самопроверка:

 Устройство: /dev/md0.

 Точка монтирования: /raid.

 Общий объем (Size): ~1.7G.

💬 Трассировка требований:

Требование: [Вставьте цитату из задания]

Реализация: [Вставьте ваш вариант настройки]

📎 Ваш артефакт: [Вставьте скриншот]

В терминале сервера **HQ-SRV**:

📥 Установите службы и подготовьте целевую папку.

```bash
apt-get update && apt-get install -y nfs-server nfs-utils
mkdir -p /raid/nfs
chmod 777 /raid/nfs
```

📥 Сконфигурируйте права сетевого доступа.

```bash
echo "/raid/nfs 192.168.2.0/28(rw,sync,no_root_squash,no_subtree_check)" > /etc/exports
```

📥 Активируйте экспорт сетевых ресурсов.

```bash
systemctl enable --now nfs-server
exportfs -arv
```

📤 Ожидаемый вывод: Система должна отрапортовать exporting 192.168.2.0/28:/raid/nfs.

✍️ Артефакт

Требование к сдаче:

Скриншот таблицы экспорта NFS.

В Полном списке заданий (Модуль 2) найдите пункт, регламентирующий права доступа и целевую сеть.

☑️ Самопроверка:

 Экспортируемый путь: /raid/nfs.

 Разрешенная сеть: 192.168.2.0/28.

 Права доступа: rw.

💬 Трассировка требований:

Требование: [Вставьте цитату из задания]

Реализация: [Вставьте ваш вариант настройки]

📎 Ваш артефакт: [Вставьте скриншот]

В терминале клиента **HQ-CLI**:

📥 Настройте автоматическое монтирование сетевого диска.

```bash
apt-get update && apt-get install -y nfs-clients
mkdir -p /mnt/nfs
echo "192.168.1.10:/raid/nfs /mnt/nfs nfs defaults 0 0" >> /etc/fstab
mount -a
df -h | grep /mnt/nfs
```

📤 Ожидаемый вывод: Диск размером ~2.0G должен появиться в списке активных устройств.

📥 Проведите нагрузочный тест с правами на запись.

```bash
touch /mnt/nfs/test_file.txt
ls -l /mnt/nfs
```

📤 Ожидаемый вывод: Наличие файла test_file.txt. Это доказывает, что NFS сервер разрешил вам запись (rw), а не только чтение.

💻 Во всех открытых терминалах:

```bash
history -a
```

✍️ Артефакт

Требование к сдаче:

Скриншот успешного монтирования и записи файла на клиенте.

В 📋 Полном списке заданий (Модуль 2) найдите пункт, регламентирующий каталог автомонтирования.

☑️ Самопроверка:

 Источник монтирования (Filesystem): 192.168.1.10:/raid/nfs.

 Наличие тестового файла: test_file.txt.

💬 Трассировка требований:

Требование: [Вставьте цитату из задания]

Реализация: [Вставьте ваш вариант настройки]

📎 Ваш артефакт: [Вставьте скриншот]

В терминале сервера **HQ-SRV**:

📥 Разверните стек и подключите носитель.

```bash
apt-get update && apt-get install -y lamp-server
systemctl enable --now mysqld
mkdir -p /mnt/cdrom
mount /dev/sr0 /mnt/cdrom
```

📥 Сконфигурируйте безопасность и создайте контур данных.

```bash
mariadb -e "CREATE DATABASE webdb;"
mariadb -e "CREATE USER 'web'@'localhost' IDENTIFIED BY 'P@ssw0rd';"
mariadb -e "GRANT ALL PRIVILEGES ON webdb.* TO 'web'@'localhost';"
mariadb -e "FLUSH PRIVILEGES;"
```

📥 Импортируйте данные и проведите Smoke-тест СУБД.

```bash
mariadb webdb < /mnt/cdrom/web/dump.sql
mariadb -e "SHOW TABLES FROM webdb;"
```

📤 Ожидаемый вывод: Команда SHOW TABLES доказывает, что файл дампа был успешно прочитан и структура таблиц сформирована. Вы увидите ASCII-таблицу.

+-----------------+

| Tables_in_webdb |

+-----------------+

| employees       |

+-----------------+

✍️ Артефакт

Требование к сдаче:

Скриншот консоли с ASCII-таблицей списка импортированных баз данных.

В 📋 Полном списке заданий (Модуль 2) найдите пункт, регламентирующий имя пользователя СУБД, и свяжите его с вашей реализацией.

☑️ Самопроверка:

 Наличие таблиц (**Tables_in_webdb**): employees.

 Ошибки подключения: Отсутствуют.

💬 Трассировка требований:

Требование: [Вставьте цитату из задания]

Реализация: [Вставьте ваш вариант настройки]

📎 Ваш артефакт: [Вставьте скриншот]

 В терминале сервера **HQ-SRV**:

📥 Разместите исходный код и настройте безопасность ФС.

```bash
rm -f /var/www/html/index.html
cp -r /mnt/cdrom/web/* /var/www/html/
chown -R apache2:apache2 /var/www/html/
```

📥 Сконфигурируйте строку подключения к БД.

```bash
sed -i 's/$username = .*/$username = "web";/' /var/www/html/index.php
sed -i 's/$password = .*/$password = "P@ssw0rd";/' /var/www/html/index.php
sed -i 's/$dbname = .*/$dbname = "webdb";/' /var/www/html/index.php
grep -E "username|password|dbname" /var/www/html/index.php
```

📤 Ожидаемый вывод: На экране должны появиться три строки из файла с вашими правильными переменными (web, P@ssw0rd).

📥 Активируйте Frontend и проведите Smoke-тест (curl).

```bash
systemctl enable --now httpd2
curl -s localhost | grep -i "td" | head -n 5
```

📤 Ожидаемый вывод: Вы должны увидеть куски HTML-разметки таблицы (теги <td>), внутри которых будут находиться имена или данные из дампа базы данных (Александр, Иванов и т.д). Если вы видите ошибку Access denied или SQL Error — настройки в index.php были заменены неверно.

 Артефакт

Требование к сдаче:

Скриншот ответа веб-сервера (HTML-код).

В Полном списке заданий (Модуль 2) найдите пункт, регламентирующий конфигурацию файла index.php.

☑️ Самопроверка:

 Содержимое ответа: HTML-теги <td>.

 Данные внутри тегов: Кириллица (Имена из БД).

 Ошибки PHP/SQL: Отсутствуют.

💬 Трассировка требований:

Требование: [Вставьте цитату из задания]

Реализация: [Вставьте ваш вариант настройки]

📎 Ваш артефакт: [Вставьте скриншот]

В терминале сервера **BR-SRV**:

📥 Подготовьте среду, подключив диск и установив ПО.

```bash
mkdir -p /mnt/cdrom
mount /dev/sr0 /mnt/cdrom
apt-get update && apt-get install -y docker-engine docker-compose-v2
systemctl enable --now docker
```

📥 Загрузите образы в реестр («Распаковка»).

```bash
docker load -i /mnt/cdrom/docker/site_latest.tar
docker load -i /mnt/cdrom/docker/mariadb_latest.tar
```

📥 Проверьте состояние реестра.

```bash
docker images
```

📤 Ожидаемый вывод: Вы должны увидеть таблицу с двумя образами: site (tag: latest) и mariadb (tag: latest или версия).

✍️ Артефакт

Требование к сдаче: Скриншот локального репозитория образов.

☑️ Самопроверка:

 Образ приложения: site.

 Образ СУБД: mariadb.

📎 Ваш артефакт: [Вставьте скриншот]

В терминале сервера **BR-SRV**:

📥 Создайте манифест инфраструктуры.

```bash
mkdir -p /opt/testapp
cd /opt/testapp
cat <<EOF > docker-compose.yaml
services:
  db:
    container_name: db
    image: mariadb:latest
    environment:
      MYSQL_DATABASE: testdb
      MYSQL_USER: test
      MYSQL_PASSWORD: P@ssw0rd
      MYSQL_ROOT_PASSWORD: toor
    volumes:
      - db_data:/var/lib/mysql
    restart: always
  testapp:
    container_name: testapp
    image: site:latest
    ports:
      - "8080:8000"
    environment:
      DB_TYPE: "maria"
      DB_HOST: "db"
      DB_PORT: "3306"
      DB_NAME: "testdb"
      DB_USER: "test"
      DB_PASS: "P@ssw0rd"
    depends_on:
      - db
    restart: always
volumes:
  db_data:
EOF
```

📥 Очистите порт и активируйте микросервисный стек.

```bash
systemctl disable --now ahttpd httpd2 nginx 2>/dev/null || true
cd /opt/testapp
docker compose up -d
sleep 10
```

📤 Ожидаемый вывод: Docker сообщит о создании внутренних сетей и двух контейнеров (Created -> Started).

📥 Выполните проверку отклика (Smoke-тест).

```bash
cd /opt/testapp
docker compose ps
curl localhost:8080 | head -n 20
```

📤 Ожидаемый вывод: Вывод ps доказывает, что оба контейнера (база и сайт) работают и прослушивают порты.

```bash
NAME      IMAGE            COMMAND                  SERVICE   CREATED          STATUS          PORTS
db        mariadb:latest   "docker-entrypoint.s…"   db        10 seconds ago   Up 10 seconds   3306/tcp
testapp   site:latest      "sh -c 'python3 -m a"    testapp   10 seconds ago   Up 10 seconds   0.0.0.0:8080->8000/tcp, [::]:8080->8000/tcp
```

Вывод curl доказывает, что Nginx внутри контейнера жив и отдает код веб-страницы. Если вы видите Connection refused — контейнер упал (изучайте docker logs testapp).

```bash
...
<!DOCTYPE html>
<html lang="ru">
 <head>
...
 </head>
 <body>
   <header class="hero is-primary">
     <div class="hero-body">
       <div class="container">
```

         <h1 class="title is-1">Очень нужный и важный сайт</h1>

```bash
...
```

 Артефакт

Требование к сдаче:

Скриншот статуса контейнеров и HTTP-ответа.

В Полном списке заданий (Модуль 2) найдите пункт, регламентирующий параметры подключения к СУБД.

☑️ Самопроверка:

 Статус контейнеров (State): Up.

 Порты приложения **testapp**: 0.0.0.0:8080->8000/tcp.

 Ответ **curl**: Тег <title>Все студенты</title>.

💬 Трассировка требований:

Требование: [Вставьте цитату из задания]

Реализация: [Вставьте ваш вариант настройки]

📎 Ваш артефакт: [Вставьте скриншот]

В терминале клиента **HQ-CLI**

📥 Установите браузер.

```bash
apt-get update && apt-get install -y yandex-browser-stable
```

🖱️ В графическом интерфейсе (GUI):

Чтобы зайти в граф интерфейс надо в проксмоксе где вы открываете терминалы щас скрин вставалю короч

> 🖼️ **Скриншот из исходного docx (Proxmox VE 9.1.6):** в левой панели выбрана ВМ `19702 (BR-RTR)`, открыта вкладка **Summary**. В правой части видно: Status — `running`, Node — `none` (пример), Memory usage — `182.69 MiB`, Bootdisk size — `120.00 GiB`, на верхней панели — кнопка раскрывающегося меню **Console** (отсюда открывается **noVNC**).

Вот тут сверху справа где написано CONSOLE есть стрелка, ее нажать там нажать noVNS(вылезет в списке)

У вас откроется вм там надо ввести логин пароль( user resu) и дальше как на обычном компе заходите в яндекс и в строке поиска вводите адреса которые дальше прописаны

Запустите Yandex Browser.

Откройте первую вкладку и перейдите: **http://192.168.1.10** (LAMP, порт 80 по умолчанию).

Откройте вторую вкладку и перейдите: **http://192.168.3.10:8080** (Docker, порт, который мы открыли).

Если сайты открываются значит все супер ну и можно закрывать

💻 Во всех открытых терминалах:

```bash
history -a
```

✍️ Артефакт

Требование к сдаче:

Скриншот браузера с открытыми вкладками приложений.

В Полном списке заданий (Модуль 2) найдите пункт, регламентирующий установку приложения.

☑️ Самопроверка:

 Вкладка 1 (LAMP): Загружена успешно.

 Вкладка 2 (Docker): Загружена успешно.

💬 Трассировка требований:

Требование: [Вставьте цитату из задания]

Реализация: [Вставьте ваш вариант настройки]

📎 Ваш артефакт: [Вставьте скриншот]

В терминале маршрутизатора **HQ-RTR**:

```bash
mkdir -p /etc/nftables
cat <<EOF > /etc/nftables/nftables.nft
#!/usr/sbin/nft -f
flush ruleset
table ip nat {
    chain prerouting {
        type nat hook prerouting priority dstnat;
        iifname "enp7s1" tcp dport 8080 dnat to 192.168.1.10:80
        iifname "enp7s1" tcp dport 2026 dnat to 192.168.1.10:22
    }
    chain postrouting {
        type nat hook postrouting priority srcnat;
        oifname "enp7s1" masquerade
    }
}
EOF
systemctl restart nftables
nft list ruleset
```

📤 Ожидаемый вывод: Ищите глазами цепочку prerouting. Наличие адреса 192.168.1.10 подтверждает, что ядро Linux успешно загрузило ваши правила публикации (Port Forwarding) в память.

```bash
table ip nat {
        chain prerouting {
                type nat hook prerouting priority dstnat; policy accept;
```

                iifname "[Ваш WAN-порт]" tcp dport 8080 dnat to 192.168.1.10:80

                iifname "[Ваш WAN-порт]" tcp dport 2026 dnat to 192.168.1.10:22

        }

```bash
...
```

Артефакт

Требование к сдаче:

Скриншот вывода nft list ruleset (таблица NAT).

Зафиксируйте выбранный вами внешний интерфейс (WAN) и объясните, по какому признаку вы его определили.

☑️ Самопроверка:

 Правило Web: tcp dport 8080 dnat to 192.168.1.10:80.

 Правило SSH: tcp dport 2026 dnat to 192.168.1.10:22.

💬 Фиксация решения (WAN-интерфейс):

Ваш выбор: [Впишите значение]

Обоснование: [Кратко поясните причину]

📎 Ваш артефакт: [Вставьте скриншот]

В терминале маршрутизатора **BR-RTR**:

📥 Примените зеркальную конфигурацию для филиала (DNAT).

```bash
mkdir -p /etc/nftables
cat <<EOF > /etc/nftables/nftables.nft
#!/usr/sbin/nft -f
flush ruleset
table ip nat {
    chain prerouting {
        type nat hook prerouting priority dstnat;
        iifname "enp7s1" tcp dport 8080 dnat to 192.168.3.10:8080
        iifname "enp7s1" tcp dport 2026 dnat to 192.168.3.10:22
    }
    chain postrouting {
        type nat hook postrouting priority srcnat;
        oifname "enp7s1" masquerade
    }
}
EOF
systemctl restart nftables
nft list ruleset
```

 Артефакт

Требование к сдаче:

Скриншот правил NAT на BR-RTR.

Зафиксируйте выбранный WAN-интерфейс и обоснуйте выбор.

☑️ Самопроверка:

 Правило Web: tcp dport 8080 dnat to 192.168.3.10:8080.

 Правило SSH: tcp dport 2026 dnat to 192.168.3.10:22.

💬 Фиксация решения (WAN-интерфейс):

Ваш выбор: [Впишите значение]

Обоснование: [Кратко поясните причину]

📎 Ваш артефакт: [Вставьте скриншот]

В терминале сервера **ISP**:

📥 Протестируйте перенаправление порта.

```bash
ssh -p 2026 root@172.16.1.10
```

📤 Ожидаемый вывод: Вы увидите предупреждение безопасности (о новом устройстве) и запрос пароля. Сам факт того, что удаленная система спросила у вас пароль, доказывает: порт 2026 проброшен верно, TCP-сессия установлена.

```bash
The authenticity of host '[172.16.1.10]:2026 ([172.16.1.10]:2026)' can't be established.
...
Are you sure you want to continue connecting (yes/no)?
```

Нам не нужно заходить внутрь сервера. Нажмите комбинацию Ctrl + C, чтобы сбросить запрос пароля и вернуться в командную оболочку ISP.

Если вы все таки зашли, напишите exit

Артефакт

Требование к сдаче: Скриншот консоли ISP, демонстрирующий запрос пароля от целевого узла при подключении по нестандартному порту.

☑️ Самопроверка:

 Установка сессии: Консоль вернула строку password: (соединение успешно проброшено на порт 22 сервера).

 Диагностика: Если команда зависла или вернула Connection refused, вернитесь на Этап 1.1 и проверьте, не перепутали ли вы имя интерфейса в ___ при создании правил nftables.

📎 Ваш артефакт: [Вставьте скриншот]

В терминале маршрутизатора **ISP**:

📥 Подготовьте ПО и секреты.

```bash
apt-get update && apt-get install -y nginx apache2 curl
htpasswd -bc /etc/nginx/.htpasswd WEB P@ssw0rd
```

📥 Сконфигурируйте защищенный портал (HQ).

```bash
cat <<EOF > /etc/nginx/sites-available.d/web.conf
server {
    listen 80;
    server_name web.au-team.irpo;
    location / {
        proxy_pass http://172.16.1.10:8080;
        proxy_set_header Host \$host;
        auth_basic "Restricted Area";
        auth_basic_user_file /etc/nginx/.htpasswd;
    }
}
EOF
```

📥 Сконфигурируйте публичное приложение (Docker).

```bash
cat <<EOF > /etc/nginx/sites-available.d/docker.conf
server {
    listen 80;
    server_name docker.au-team.irpo;
    location / {
        proxy_pass http://172.16.2.10:8080;
        proxy_set_header Host \$host;
    }
}
EOF
```

📥 Включите конфигурацию.

```bash
ln -s /etc/nginx/sites-available.d/web.conf /etc/nginx/sites-enabled.d/web.conf
ln -s /etc/nginx/sites-available.d/docker.conf /etc/nginx/sites-enabled.d/docker.conf
systemctl enable --now nginx
nginx -t
systemctl restart nginx
ls -l /etc/nginx/sites-enabled.d/
```

📤 Ожидаемый вывод: syntax is ok и список из двух активных симлинков.

✍️ Артефакт

Требование к сдаче:

Скриншот активных конфигураций Nginx.

В 📋 Полном списке заданий (Модуль 2) найдите пункт, регламентирующий web-based аутентификацию.

☑️ Самопроверка:

 Симлинк 1: docker.conf.

 Симлинк 2: web.conf.

 Результат **nginx -t**: syntax is ok.

💬 Трассировка требований:

Требование: [Вставьте цитату из задания]

Реализация: [Вставьте ваш вариант настройки]

📎 Ваш артефакт: [Вставьте скриншот]

В терминале сервера **ISP**:

📥 Подготовьте локальный резолвер.

```bash
echo "127.0.0.1 web.au-team.irpo docker.au-team.irpo" >> /etc/hosts
```

📥 Выстреливаем первой пулей (Публичный доступ).

```bash
curl -s http://docker.au-team.irpo | head -n 10
```

📤 Ожидаемый вывод: Nginx расшифровал заголовок Host, перенаправил трафик в контейнер и вернул вам HTML-код сайта.

```bash
<!DOCTYPE html>
<html lang="ru">
...
```

    <title>Все студенты</title>

```bash
...
```

📥 Выстреливаем второй пулей (Атака без пароля).

```bash
curl -I -s http://web.au-team.irpo
```

📤 Ожидаемый вывод: Запрос жестко заблокирован Nginx'ом на этапе проверки файла .htpasswd.

```bash
HTTP/1.1 401 Unauthorized
...
WWW-Authenticate: Basic realm="Restricted Area"
```

📥 Выстреливаем третьей пулей (Легитимный вход).

```bash
curl -u WEB:P@ssw0rd -s http://web.au-team.irpo | head -n 10
```

📤 Ожидаемый вывод: Пароль прошел проверку. Nginx пустил вас внутрь и вернул HTML-код (монолит LAMP).

```bash
<!DOCTYPE html>
<html>
<head>
    <meta charset="utf-8">
```

    <title>Задание 7 модуль 2</title>

```bash
...
```

💻 Во всех открытых терминалах:

```bash
history -a
```

✍️ Артефакт

Требование к сдаче:

Скриншот консоли ISP, демонстрирующий успешное завершение трех интеграционных тестов (публичный доступ, отказ в авторизации 401, успешный вход).

В 📋 Полном списке заданий (Модуль 2) найдите пункт, регламентирующий логику маршрутизации для корпоративного портала, и свяжите его с вашей конфигурацией.

☑️ Самопроверка:

 Домен **docker**: Ответ содержит HTML-теги.

 Домен **web** (без пароля): Ответ содержит 401 Unauthorized.

 Домен **web** (с паролем): Ответ содержит HTML-теги.

💬 Трассировка требований:

Требование: [Вставьте цитату из задания]

Реализация: [Вставьте ваш вариант настройки]

📎 Ваш артефакт: [Вставьте скриншот]
