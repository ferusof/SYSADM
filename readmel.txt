Методичка: Демо-экзамен Alt Linux p10 от Олежи
Сеть
 
________________________________________
1. ISP
1.1 Переименование машины и изменение часовых поясов
hostnamectl set-hostname ISP.au-team.irpo; exec bash
 
1.2 Просмотр портов чтобы понять как называются порты (enp7s1 или ens19)
ip -c a   или   ip -c -br a
 

1.3 Настройка интерфейса enp7s1 (выход в интернет)
Переходим в директорию:
cd /etc/net/ifaces/enp7s1
•	Файл ipv4address:
(IP от преподавателя)/24
Пример: 10.0.1.77/24
•	Файл ipv4route: default via 10.0.1.1
•	Файл resolv.conf: nameserver 8.8.8.8
•	Файл options:
ONBOOT=yes
TYPE=eth
DISABLED=no
CONFIG_IPV4=yes
BOOTPROTO=static
SYSTEMD_BOOTPROTO=static
SYSTEMD_CONTROLLED=no
CONFIG_WIRELESS=no
 
1.4 Перезапуск сети и проверка
systemctl restart network
ping ya.ru
 

1.5 Обновление и установка iptables
apt-get update
apt-get install iptables –y
1.6 Включение IP-forwarding
Отредактируйте файл /etc/net/sysctl.conf (меняем 0 на 1):
net.ipv4.ip_forward = 1
Применяем:
sysctl -p
1.7 Настройка остальных портов (enp7s2 и enp7s3)
Копируем настройки:
cp -rf enp7s1/ enp7s2
cp -rf enp7s1/ enp7s3
 
Для enp7s2 (HQ-RTR):
•	ipv4address: 172.16.1.1/28
•	ipv4route: 172.16.1.1/28 via 172.16.1z.2/28
 
Для enp7s3 (BR-RTR):
•	ipv4address: 172.16.2.1/28
•	ipv4route: 172.16.2.1/28 via 172.16.2.2/28
Перезапускаем сеть:
systemctl restart network
 


1.8 Настройка iptables
iptables -t nat -A POSTROUTING -o enp7s1 -j MASQUERADE
Сохраняем и включаем:
iptables-save > /etc/sysconfig/iptables
systemctl enable iptables
 
Чтобы наверняка понять что iptables включен:
systemctl status iptables
 
Если он не запущен, попробуйте:
systemctl start iptables
________________________________________
2. HQ-RTR
2.1 Переименование машины и изменение часовых поясов
hostnamectl set-hostname hq-rtr.au-team.irpo; exec bash
 
2.2 Настройка интерфейса enp7s1 (подключение к ISP)
mkdir -p /etc/net/ifaces/enp7s1
cd /etc/net/ifaces/enp7s1
•	Файл ipv4address: 172.16.1.2/28
•	Файл ipv4route: default via 172.16.1.1
•	Файл resolv.conf: nameserver 8.8.8.8
•	Файл options:
ONBOOT=yes
TYPE=eth
DISABLED=no
CONFIG_IPV4=yes
BOOTPROTO=static
SYSTEMD_BOOTPROTO=static
SYSTEMD_CONTROLLED=no
CONFIG_WIRELESS=no
NM_CONTROLLED=yes
2.3 Настройка enp7s2 (для VLAN)
•	Файл ipv4route: default via 172.16.1.1/28
•	Файл options:
ONBOOT=yes
TYPE=eth
DISABLED=no
CONFIG_IPV4=no
Перезапускаем сеть:
systemctl restart network
 
2.4 Установка пакетов 
apt-get update
apt-get install iptables frr dnsmasq sudo –y
Включаем forwarding (/etc/net/sysctl.conf):
net.ipv4.ip_forward = 1
Применяем:
sysctl -p
2.5 Настройка iptables (NAT)
Настраиваем NAT:
iptables -t nat -A POSTROUTING -o enp7s1 -j MASQUERADE


Сохраняем правила
iptables-save > /etc/sysconfig/iptables

systemctl enable iptables
systemctl status iptables
systemctl start iptables
2.6 Настройка VLAN 100, 200, 999
VLAN 100 (HQ-SRV):
mkdir -p /etc/net/ifaces/vlan100
cd /etc/net/ifaces/vlan100
•	Файл ipv4address: 192.168.1.1/27
•	Файл options:
ONBOOT=yes
TYPE=vlan
VID=100
DISABLED=no
CONFIG_IPV4=yes
BOOTPROTO=static
HOST=enp7s2

Копируем настройки:
cd ..
cp -rf vlan100 vlan200
cp -rf vlan100 vlan999
 

VLAN 200 (CLI-SRV):
•	ipv4address: 192.168.2.1/28
•	Файл options:
Меняем VID=100 на VID=200
VLAN 999:
•	ipv4address: 192.168.3.1/29
•	Файл options:
Меняем VID=100 на VID=999
Перезапускаем сеть:
systemctl restart network
 
2.7 Настройка DHCP (dnsmasq) для VLAN200 / HQ-CLI
Открываем основной конфиг:
vim /etc/dnsmasq.conf
В самый конец файла(Shift+g) добавь эти 4 строки:
interface=vlan200
dhcp-range=192.168.2.10,192.168.2.14,12h
dhcp-option=3,192.168.2.1
dhcp-option=6,192.168.1.2,8.8.8.8
dhcp-option=15,au-team.irpo
Сохрани и примени:
systemctl restart dnsmasq
systemctl enable dnsmasq
systemctl status dnsmasq   # должно быть active (running)
2.8 Настройка GRE
mkdir -p /etc/net/ifaces/gre1
cd /etc/net/ifaces/gre1
•	Файл ipv4address: 10.10.10.1/30
•	Файл ipv4route: 192.164.4.0/27 via 10.10.10.2
•	Файл options:
ONBOOT=yes
TYPE=iptun
TUNTYPE=gre
TUNLOCAL=172.16.1.2
TUNREMOTE=172.16.2.2
DISABLED=no
CONFIG_IPV4=yes
BOOTPROTO=static
HOST=enp7s1
И чтобы применить GRE, перезапускаем ВМ
reboot
 
________________________________________
3. HQ-SRV
3.1 Переименование машины
hostnamectl set-hostname hq-srv.au-team.irpo; exec bash
 
3.2 Настройка интерфейса enp7s1 (подключение к HQ-RTR)
cd /etc/net/ifaces/enp7s1
•	Файл ipv4address: 192.168.1.2/28
•	Файл ipv4route: default via 192.168.1.1
•	Файл resolv.conf: nameserver 8.8.8.8
•	Файл options:
ONBOOT=yes
TYPE=eth
DISABLED=no
CONFIG_IPV4=yes
BOOTPROTO=static
SYSTEMD_BOOTPROTO=static
SYSTEMD_CONTROLLED=no
CONFIG_WIRELESS=no
 
3.3 Перезапуск сети
systemctl restart network
 












________________________________________
4. HQ-CLI 
4.1 Переименование машины и изменение часовых поясов
hostnamectl set-hostname hq-cli.au-team.irpo; exec bash
 
4.2 Отключение NetworkManager
systemctl stop NetworkManager
systemctl disable NetworkManager
Причина: NetworkManager игнорирует DNS-опции из DHCP и не записывает их в /etc/resolv.conf. Вместо него используем etcnet.
4.3 Настройка интерфейса (ens19 или enp7s1) через etcnet
mkdir -p /etc/net/ifaces/ens19
cd /etc/net/ifaces/ens19
Файл options:
ONBOOT=yes
DISABLED=no
BOOTPROTO=dhcp
TYPE=eth
NM_CONTROLLED=no
4.4 Перезапуск сети и проверка
systemctl restart network
ip -c -br a           # должен получить адрес из пула 192.168.2.10–14
cat /etc/resolv.conf  # должен появиться nameserver 192.168.1.2
ping ya.ru
 
 
________________________________________
5. BR-RTR
5.1 Переименование машины и изменение часовых поясов
hostnamectl set-hostname br-rtr.au-team.irpo; exec bash
 
5.2 Настройка интерфейса enp7s1 (подключение к ISP)
cd /etc/net/ifaces/enp7s1
•	Файл ipv4address: 172.16.2.2/28
•	Файл ipv4route: default via 172.16.2.1
•	Файл resolv.conf: nameserver 8.8.8.8
•	Файл options:
ONBOOT=yes
TYPE=eth
DISABLED=no
CONFIG_IPV4=yes
BOOTPROTO=static
SYSTEMD_BOOTPROTO=static
SYSTEMD_CONTROLLED=no
CONFIG_WIRELESS=no
 
5.3 Настройка enp7s2 (для BR-SRV)
Копируем настройки:
cp /etc/net/ifaces/enp7s1 enp7s2/
cd /etc/net/ifaces/enp7s2
 
•	Файл ipv4address: 192.168.4.1/28
•	Файл ipv4route: 192.168.4.0/28 via 192.168.4.2/28
 
5.4 Обновление и установка iptables
apt-get update
apt-get install iptables sudo –y
5.5 Включение IP-forwarding
Отредактируйте файл /etc/net/sysctl.conf (меняем 0 на 1):
net.ipv4.ip_forward = 1
Применяем:
sysctl -p
5.6 Настройка iptables
iptables -t nat -A POSTROUTING -o enp7s1 -j MASQUERADE
Сохраняем и включаем:
iptables-save > /etc/sysconfig/iptables
systemctl enable iptables
systemctl start iptables
 
Перезапускаем сеть:
systemctl restart network
 
2.8 Настройка GRE1
mkdir -p /etc/net/ifaces/gre1
cd /etc/net/ifaces/gre1
•	Файл ipv4address: 10.10.10.2/30
•	Файл ipv4route: 
192.164.1.0/27 via 10.10.10.1
192.164.2.0/26 via 10.10.10.1
192.164.3.0/29 via 10.10.10.1

•	Файл options:
ONBOOT=yes
TYPE=iptun
TUNTYPE=gre
TUNLOCAL=172.16.2.2
TUNREMOTE=172.16.1.2
DISABLED=no
CONFIG_IPV4=yes
BOOTPROTO=static
HOST=enp7s1
И чтобы применить GRE, перезапускаем ВМ
reboot
 
________________________________________
6. BR-SRV
6.1 Переименование машины
hostnamectl set-hostname br-srv.au-team.irpo; exec bash 
 
6.2 Настройка интерфейса enp7s1 (подключение к BR-RTR)
cd /etc/net/ifaces/enp7s1
•	Файл ipv4address: 192.168.4.2/28
•	Файл ipv4route: default via 192.168.4.1
•	Файл resolv.conf: nameserver 8.8.8.8
•	Файл options:
ONBOOT=yes
TYPE=eth
DISABLED=no
CONFIG_IPV4=yes
BOOTPROTO=static
SYSTEMD_BOOTPROTO=static
SYSTEMD_CONTROLLED=no
CONFIG_WIRELESS=no
 
Перезапускаем сеть:
systemctl restart network 
 
________________________________________
7. Локальные учётные записи
7.1 Создание sshuser на HQ-SRV и BR-SRV
Выполнить на обоих серверах.
Создаём пользователя с UID 1010:
useradd -u 1010 sshuser
passwd sshuser
Пароль: P@ssw0rd
Добавляем в группу wheel:
usermod -aG wheel sshuser
Разрешаем sudo без пароля:
visudo
Найти и раскомментировать строку (она где-то в конце файла):
# WHEEL  ALL=(ALL:ALL) NOPASSWD: ALL
Убрать # в начале строки с помощью Delet на этом знаке.
Проверка:
su - sshuser
id
sudo whoami
id должен показать uid=1010, sudo whoami вернуть root без запроса пароля.
7.2 Создание net_admin на HQ-RTR и BR-RTR
Выполнить на обоих роутерах.
useradd net_admin
passwd net_admin
usermod -aG wheel net_admin
Пароль: P@$$word
Проверка:
su - net_admin
sudo whoami
________________________________________
8. Безопасный SSH-доступ
Выполнить на HQ-SRV и BR-SRV.
8.1 Редактируем конфиг
vim /etc/openssh/sshd_config
Найти и изменить/добавить:
Port 2024
AllowUsers sshuser
MaxAuthTries 2
PasswordAuthentication yes
Banner /etc/openssh/banner
8.2 Создаём баннер
vim /etc/openssh/banner
Содержимое файла:
Authorized access only
8.3 Перезапуск и проверка
systemctl restart sshd
Проверка с другой машины:
ssh -p 2024 sshuser@192.168.1.2
Должен показать баннер Authorized access only.
Проверка что другие пользователи заблокированы:
ssh -p 2024 root@192.168.1.2
Должно вернуть Permission denied.
 
________________________________________
9. DNS (BIND) на HQ-SRV
9.1 Установка
apt-get install bind -y
9.2 Настройка options.conf
vim /var/lib/bind/etc/options.conf
Важно: в файле найти строку include "/etc/bind/resolvconf-options.conf"; и закомментировать её (поставить // в начало). Иначе BIND не запустится с ошибкой 'forwarders' redefined.

Содержимое файла (найти эти строки в файле и заменить/изменить) options { ... }:
options {
    listen-on { 192.168.1.2; 127.0.0.1; };
    forwarders { 8.8.8.8; };
    allow-query { any; };
    recursion yes;
};
9.3 Добавляем зоны в rfc1912.conf
vim /var/lib/bind/etc/rfc1912.conf
Добавить в конец файла:
zone "au-team.irpo" {
    type master;
    file "au-team.irpo";
    allow-update { none; };
};

zone "1.168.192.in-addr.arpa" {
    type master;
    file "rev";
    allow-update { none; };
};
 
9.4 Создаём файл прямой зоны
cd /var/lib/bind/etc/zone
cp localhost au-team.irpo
vim au-team.irpo
Важно — требования к синтаксису файлов зон BIND:
•	$TTL 1D — между ними должен быть пробел, а не табуляция
•	Каждая запись (hq-rtr, moodle и т.д.) должна начинаться с первой колонки, без пробелов и табов в начале строки
•	Между колонками имя, IN, A, IP — табуляции или несколько пробелов
•	Точка . в конце hq-srv.au-team.irpo. обязательна
Содержимое файла:
$TTL 1D
@   IN SOA hq-srv.au-team.irpo. root. (1 1H 15M 1W 1H)
    IN NS  hq-srv.au-team.irpo.

hq-rtr  IN A 192.168.1.1
hq-srv  IN A 192.168.1.2
hq-cli  IN A 192.168.2.10
br-rtr  IN A 172.16.2.2
br-srv  IN A 192.168.4.2
 
9.5 Создаём файл обратной зоны
cp localhost rev
vim rev
Содержимое:
$TTL 1D
@   IN SOA hq-srv.au-team.irpo. root. (1 1H 15M 1W 1H)
    IN NS  hq-srv.au-team.irpo.

1   IN PTR hq-rtr.au-team.irpo.
2   IN PTR hq-srv.au-team.irpo.
10  IN PTR hq-cli.au-team.irpo.
 

9.6 Запуск
chgrp named /var/lib/bind/etc/zone/*
chmod 644 /var/lib/bind/etc/zone/*
named-checkconf (Не обязательно)
named-checkconf -z (Не обязательно)

systemctl enable bind
systemctl restart bind
systemctl status bind
9.9 Проверка DNS
С HQ-CLI:
host hq-rtr.au-team.irpo
host moodle.au-team.irpo
nslookup 192.168.1.1
ping moodle.au-team.irpo
Всё должно резолвиться без указания DNS-сервера явно.



 

НЕРАБОТАЕТ
2. HQ 
2.1 Настройка GRE
mkdir -p /etc/net/ifaces/gre1
cd /etc/net/ifaces/gre1
2.2 Настройка интерфейса enp7s1 (подключение к ISP)
mkdir -p /etc/net/ifaces/enp7s1
cd /etc/net/ifaces/enp7s1
•	Файл ipv4address: 172.16.1.2/28
•	Файл ipv4route: default via 172.16.1.1
•	Файл resolv.conf: nameserver 8.8.8.8
•	Файл options:
ONBOOT=yes
TYPE=eth
DISABLED=no
CONFIG_IPV4=yes
BOOTPROTO=static
SYSTEMD_BOOTPROTO=static
SYSTEMD_CONTROLLED=no
CONFIG_WIRELESS=no
NM_CONTROLLED=yes
 

OSPF - HQ
cat > /etc/frr/frr.conf << 'EOF'
frr version 9.0.2
frr defaults traditional
no ipv6 forwarding
!
interface gre1
 ip ospf network point-to-point
 ip ospf hello-interval 5
 ip ospf dead-interval 20
!
router ospf
 ospf router-id 1.1.1.1
 network 10.0.0.0/30 area 0
 network 192.168.1.0/27 area 0
 network 192.168.2.0/28 area 0
 network 192.168.3.0/29 area 0
!
EOF

OSPF - BR
cat > /etc/frr/frr.conf << 'EOF'
frr version 9.0.2
frr defaults traditional
no ipv6 forwarding
!
interface gre1
 ip ospf network point-to-point
 ip ospf hello-interval 5
 ip ospf dead-interval 20
!
router ospf
 ospf router-id 2.2.2.2
 network 10.0.0.0/30 area 0
 network 192.168.4.0/27 area 0
!
EOF

