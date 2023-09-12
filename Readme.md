Vagrant-стенд c OSPF

Цель домашнего задания
Создать домашнюю сетевую лабораторию. Научится настраивать протокол OSPF в Linux-based системах.


Описание домашнего задания
1. Развернуть 3 виртуальные машины
2. Объединить их разными vlan
- настроить OSPF между машинами на базе Quagga;
- изобразить ассиметричный роутинг;
- сделать один из линков "дорогим", но чтобы при этом роутинг был симметричным.

Формат сдачи: Vagrantfile + ansible

1. Разворачиваем 3 виртуальные машины c помощью приложенного Vagrantfile. В данный Vagrantfile уже добавлен модуль запуска Ansible-playbook.
С помощью Ansible мы заранее установим следующее ПО, которое нам пригодится для дальнейшей работы: vim, traceroute, tcpdump, net-tools.

```
neva@Uneva:~$ vagrant up
neva@Uneva:~$ vagrant status
Current machine states:

router1                   running (virtualbox)
router2                   running (virtualbox)
router3                   running (virtualbox)
```

2. Сначала настроим OSPF вручную, потом сделаем все настройки через Ansible.

Процесс установки FRR и настройки OSPF вручную:
1) Подключаемся к router1, переходим на пользователя root. Отключаем файерволл ufw и удаляем его из автозагрузки:

```
neva@Uneva:~$ vagrant ssh router1
vagrant@router1:~$ sudo -i
root@router1:~# systemctl stop ufw
root@router1:~# systemctl disable ufw
Synchronizing state of ufw.service with SysV service script with /lib/systemd/systemd-sysv-install.
Executing: /lib/systemd/systemd-sysv-install disable ufw
Removed /etc/systemd/system/multi-user.target.wants/ufw.service.
```

2) Добавляем gpg ключ:

```
root@router1:~# curl -s https://deb.frrouting.org/frr/keys.asc | sudo apt-key add -
OK
```

3) Добавляем репозиторий c пакетом FRR:

```
root@router1:~# echo deb https://deb.frrouting.org/frr $(lsb_release -s -c) frr-stable > /etc/apt/sources.list.d/frr.list
```

4) Обновляем пакеты и устанавливаем FRR:

```
root@router1:~# apt update
root@router1:~# apt install frr frr-pythontools
```

5) Разрешаем (включаем) маршрутизацию транзитных пакетов:

```
root@router1:~# sysctl net.ipv4.conf.all.forwarding=1
net.ipv4.conf.all.forwarding = 1
```

6) Включаем демон ospfd в FRR
Для этого открываем в редакторе файл /etc/frr/daemons и меняем в нём параметры для пакетов zebra и ospfd на yes:

```
bgpd=no
ospfd=yes
zebra=yes
ospf6d=no
ripd=no
ripngd=no
isisd=no
pimd=no
pim6d=no
ldpd=no
nhrpd=no
eigrpd=no
babeld=no
sharpd=no
pbrd=no
bfdd=no
fabricd=no
vrrpd=no
pathd=no
```

7) Настройка OSPF
Для настройки OSPF нам потребуется создать файл /etc/frr/frr.conf который будет содержать в себе информацию о требуемых интерфейсах и OSPF. Разберем пример создания файла на хосте router1. 
Для начала нам необходимо узнать имена интерфейсов и их адреса. Сделать это можно с помощью двух способов:

```
root@router1:~# ip a | grep inet
    inet 127.0.0.1/8 scope host lo
    inet6 ::1/128 scope host
    inet 10.0.2.15/24 brd 10.0.2.255 scope global dynamic enp0s3
    inet6 fe80::72:eeff:fee2:edce/64 scope link
    inet 10.0.10.1/24 brd 10.0.10.255 scope global enp0s8
    inet6 fe80::a00:27ff:fedc:af7e/64 scope link
    inet 10.0.12.1/24 brd 10.0.12.255 scope global enp0s9
    inet6 fe80::a00:27ff:fe44:7086/64 scope link
    inet 192.168.10.1/24 brd 192.168.10.255 scope global enp0s10
    inet6 fe80::a00:27ff:fe54:ecc6/64 scope link
    inet 192.168.50.10/24 brd 192.168.50.255 scope global enp0s16
    inet6 fe80::a00:27ff:fe7b:574f/64 scope link

root@router1:~# vtysh

Hello, this is FRRouting (version 9.0).
Copyright 1996-2005 Kunihiro Ishiguro, et al.

router1# sh int br
Interface       Status  VRF             Addresses
---------       ------  ---             ---------
enp0s3          up      default         10.0.2.15/24
enp0s8          up      default         10.0.10.1/24
enp0s9          up      default         10.0.12.1/24
enp0s10         up      default         192.168.10.1/24
enp0s16         up      default         192.168.50.10/24
lo              up      default
```

В обоих примерах мы увидим имена сетевых интерфейсов, их ip-адреса и маски подсети. Исходя из схемы мы понимаем, что для настройки OSPF нам достаточно описать интерфейсы enp0s8, enp0s9, enp0s10.

Создаём файл /etc/frr/frr.conf и вносим в него следующую информацию:

```
!Указание версии FRR
frr version 8.1
frr defaults traditional
!Указываем имя машины
hostname router1
log syslog informational
no ipv6 forwarding
service integrated-vtysh-config
!
!Добавляем информацию об интерфейсе enp0s8
interface enp0s8
 !Указываем имя интерфейса
 description r1-r2
 !Указываем ip-aдрес и маску (эту информацию мы получили в прошлом шаге)
 ip address 10.0.10.1/24
 !Указываем параметр игнорирования MTU
 ip ospf mtu-ignore
 !Если потребуется, можно указать «стоимость» интерфейса
 !ip ospf cost 1000
 !Указываем параметры hello-интервала для OSPF пакетов
 ip ospf hello-interval 10
 !Указываем параметры dead-интервала для OSPF пакетов
 !Должно быть кратно предыдущему значению
 ip ospf dead-interval 30
!
interface enp0s9
 description r1-r3
 ip address 10.0.12.1/24
 ip ospf mtu-ignore
 !ip ospf cost 45
 ip ospf hello-interval 10
 ip ospf dead-interval 30

interface enp0s10
 description net_router1
 ip address 192.168.10.1/24
 ip ospf mtu-ignore
 !ip ospf cost 45
 ip ospf hello-interval 10
 ip ospf dead-interval 30 
!
!Начало настройки OSPF
router ospf
 !Указываем router-id 
 router-id 1.1.1.1
 !Указываем сети, которые хотим анонсировать соседним роутерам
 network 10.0.10.0/24 area 0
 network 10.0.12.0/24 area 0
 network 192.168.10.0/24 area 0 
 !Указываем адреса соседних роутеров
 neighbor 10.0.10.2
 neighbor 10.0.12.2

!Указываем адрес log-файла
log file /var/log/frr/frr.log
default-information originate always
```

8) После создания файлов /etc/frr/frr.conf и /etc/frr/daemons нужно проверить, что владельцем файла является пользователь frr. Группа файла также должна быть frr. Должны быть установленны следующие права:
у владельца на чтение и запись
у группы только на чтение
ls -l /etc/frr
Если права или владелец файла указан неправильно, то нужно поменять владельца и назначить правильные права, например:
chown frr:frr /etc/frr/frr.conf 
chmod 640 /etc/frr/frr.conf 

```
root@router1:~# ls -l /etc/frr
total 24
-rw-r----- 1 frr frr 4138 Sep  8 13:59 daemons
-rw-r----- 1 frr frr 2363 Sep  8 14:20 frr.conf
-rw-r----- 1 frr frr 6170 Aug  8 22:03 support_bundle_commands.conf
-rw-r----- 1 frr frr   32 Aug  8 22:03 vtysh.conf
```

9) Перезапускаем FRR и добавляем его в автозагрузку

```
root@router1:~# systemctl restart frr
root@router1:~# systemctl enable frr
Synchronizing state of frr.service with SysV service script with /lib/systemd/systemd-sysv-install.
Executing: /lib/systemd/systemd-sysv-install enable frr
```

Проделываем аналогичные настройки на router2, router3, но вместо ручного создания файла конфигурации frr можно задать данные параметры вручную из vtysh. Vtysh использует cisco-like команды.
Сделаем это на router2 и router3, предварительно включив на обоих роутерах демон ospfd в FRR.

```
root@router2:~# vim /etc/frr/daemons
root@router2:~# vtysh

Hello, this is FRRouting (version 9.0.1).
Copyright 1996-2005 Kunihiro Ishiguro, et al.

router2# sh interface br
Interface       Status  VRF             Addresses
---------       ------  ---             ---------
enp0s3          up      default         10.0.2.15/24
enp0s8          up      default         10.0.10.2/24
enp0s9          up      default         10.0.11.2/24
enp0s10         up      default         192.168.20.1/24
enp0s16         up      default         192.168.50.11/24
lo              up      default

router2# conf t
router2(config)# hostname router2
router2(config)# log syslog informational
router2(config)# no ipv6 forwarding
router2(config)# service integrated-vtysh-config
router2(config)# interface enp0s8
router2(config-if)# description r2-r1
router2(config-if)# ip address 10.0.10.2/24
router2(config-if)# ip ospf mtu-ignore
router2(config-if)# ip ospf hello-interval 10
router2(config-if)# ip ospf dead-interval 30
router2(config)# int enp0s9
router2(config-if)# description r2-r3
router2(config-if)# ip address  10.0.11.2/24
router2(config-if)# ip ospf mtu-ignore
router2(config-if)# ip ospf mtu-ignore
router2(config-if)# ip ospf hello-interval 10
router2(config-if)# ip ospf dead-interval 30
router2(config-if)# exit
router2(config)# interface enp0s10
router2(config-if)# description net2-router2
router2(config-if)# ip address 192.168.20.1/24
router2(config-if)# ip ospf mtu-ignore
router2(config-if)# ip ospf hello-interval 10
router2(config-if)# ip ospf dead-interval 30
router2(config-if)# exit
router2(config)# router ospf
router2(config-router)# router-id 2.2.2.2
router2(config-router)# network 10.0.10.0/24 area 0
router2(config-router)# network 10.0.11.0/24 area 0
router2(config-router)# network 192.168.20.0/24 area 0
router2(config-router)# neighbor 10.0.10.1
router2(config-router)# neighbor 10.0.11.1
router2(config)# exit
router2# write
Note: this version of vtysh never writes vtysh.conf
Building Configuration...
Integrated configuration saved to /etc/frr/frr.conf
[OK]
```

Аналогичные настройки сделаем для router3, указав его адреса на интерфейсах и сети.
Проверим доступность сетей с хоста router1:
попробуем сделать ping до ip-адреса 192.168.30.1

```
root@router1:~# ping 192.168.30.1
PING 192.168.30.1 (192.168.30.1) 56(84) bytes of data.
64 bytes from 192.168.30.1: icmp_seq=1 ttl=64 time=0.686 ms
64 bytes from 192.168.30.1: icmp_seq=2 ttl=64 time=0.669 ms
64 bytes from 192.168.30.1: icmp_seq=3 ttl=64 time=0.667 ms
64 bytes from 192.168.30.1: icmp_seq=4 ttl=64 time=0.758 ms
^C

root@router1:~# traceroute 192.168.30.1
traceroute to 192.168.30.1 (192.168.30.1), 30 hops max, 60 byte packets
traceroute to 192.168.30.1 (192.168.30.1), 30 hops max, 60 byte packets
 1  * * *
 2  * * *
 3  * * *
 4  * * *
 5  * * *
 6  * 192.168.30.1 (192.168.30.1)  1.070 ms  1.026 ms
```

Попробуем отключить интерфейс enp0s9 и немного подождем и снова запустим трассировку до ip-адреса 192.168.30.1

```
root@router1:~# ifconfig enp0s9 down
root@router1:~# ip a | grep enp0s9
4: enp0s9: <BROADCAST,MULTICAST> mtu 1500 qdisc fq_codel state DOWN group default qlen 1000
root@router1:~# traceroute 192.168.30.1
traceroute to 192.168.30.1 (192.168.30.1), 30 hops max, 60 byte packets
 1  10.0.10.2 (10.0.10.2)  1.330 ms  1.268 ms  1.147 ms
 2  * * *
 3  * * *
 4  * * *
 5  * * *
 6  * * *
 7  * * *
 8  * * *
 9  * * *
10  * * *
11  * * *
12  * * 192.168.30.1 (192.168.30.1)  2.503 ms
```

Как мы видим, после отключения интерфейса сеть 192.168.30.0/24 нам остаётся доступна.

```
root@router1:~# vtysh

Hello, this is FRRouting (version 9.0).
Copyright 1996-2005 Kunihiro Ishiguro, et al.

router1# show ip route ospf
Codes: K - kernel route, C - connected, S - static, R - RIP,
       O - OSPF, I - IS-IS, B - BGP, E - EIGRP, N - NHRP,
       T - Table, v - VNC, V - VNC-Direct, A - Babel, F - PBR,
       f - OpenFabric,
       > - selected route, * - FIB route, q - queued, r - rejected, b - backup
       t - trapped, o - offload failure

O   10.0.10.0/24 [110/100] is directly connected, enp0s8, weight 1, 00:16:39
O>* 10.0.11.0/24 [110/200] via 10.0.10.2, enp0s8, weight 1, 00:04:36
O>* 10.0.12.0/24 [110/300] via 10.0.10.2, enp0s8, weight 1, 00:04:36
O   192.168.10.0/24 [110/100] is directly connected, enp0s10, weight 1, 00:16:39
O>* 192.168.20.0/24 [110/200] via 10.0.10.2, enp0s8, weight 1, 00:15:41
O>* 192.168.30.0/24 [110/300] via 10.0.10.2, enp0s8, weight 1, 00:04:36
```

2.2 Настройка ассиметричного роутинга

Для настройки ассиметричного роутинга нам необходимо выключить блокировку ассиметричной маршрутизации:

```
root@router1:~# sysctl net.ipv4.conf.all.rp_filter=0
net.ipv4.conf.all.rp_filter = 0
```

Далее, выбираем один из роутеров, на котором изменим «стоимость интерфейса». Например поменяем стоимость интерфейса enp0s8 на router1:

```
router1# conf t
router1(config)# int enp0s8
router1(config-if)# ip ospf cost 1000
```

После внесения данных настроек, мы видим, что маршрут до сети 192.168.20.0/30  теперь пойдёт через router2, но обратный трафик от router2 пойдёт по другому пути.

```
router1# sh ip route ospf
Codes: K - kernel route, C - connected, S - static, R - RIP,
       O - OSPF, I - IS-IS, B - BGP, E - EIGRP, N - NHRP,
       T - Table, v - VNC, V - VNC-Direct, A - Babel, F - PBR,
       f - OpenFabric,
       > - selected route, * - FIB route, q - queued, r - rejected, b - backup
       t - trapped, o - offload failure

O   10.0.10.0/24 [110/300] via 10.0.12.2, enp0s9, weight 1, 00:00:29
O>* 10.0.11.0/24 [110/200] via 10.0.12.2, enp0s9, weight 1, 00:00:29
O   10.0.12.0/24 [110/100] is directly connected, enp0s9, weight 1, 00:06:52
O   192.168.10.0/24 [110/100] is directly connected, enp0s10, weight 1, 00:06:52
O>* 192.168.20.0/24 [110/300] via 10.0.12.2, enp0s9, weight 1, 00:00:29
O>* 192.168.30.0/24 [110/200] via 10.0.12.2, enp0s9, weight 1, 00:06:43


router2# show ip route ospf
Codes: K - kernel route, C - connected, S - static, R - RIP,
       O - OSPF, I - IS-IS, B - BGP, E - EIGRP, N - NHRP,
       T - Table, v - VNC, V - VNC-Direct, A - Babel, F - PBR,
       f - OpenFabric,
       > - selected route, * - FIB route, q - queued, r - rejected, b - backup
       t - trapped, o - offload failure

O   10.0.10.0/24 [110/100] is directly connected, enp0s8, weight 1, 19:58:33
O   10.0.11.0/24 [110/100] is directly connected, enp0s9, weight 1, 20:02:09
O>* 10.0.12.0/24 [110/200] via 10.0.10.1, enp0s8, weight 1, 00:04:11
  *                        via 10.0.11.1, enp0s9, weight 1, 00:04:11
O>* 192.168.10.0/24 [110/200] via 10.0.10.1, enp0s8, weight 1, 00:04:11
O   192.168.20.0/24 [110/100] is directly connected, enp0s10, weight 1, 20:18:06
O>* 192.168.30.0/24 [110/200] via 10.0.11.1, enp0s9, weight 1, 20:01:50
````

Давайте это проверим. На router1 запускаем пинг от 192.168.10.1 до 192.168.20.1:

```
ping -I 192.168.10.1 192.168.20.1
```

На router2 запускаем tcpdump, который будет смотреть трафик только на порту enp0s9:

```
root@router2:~#  tcpdump -i enp0s9
tcpdump: verbose output suppressed, use -v or -vv for full protocol decode
listening on enp0s9, link-type EN10MB (Ethernet), capture size 262144 bytes
12:40:50.421220 IP 192.168.10.1 > router2: ICMP echo request, id 9, seq 94, length 64
12:40:51.423224 IP 192.168.10.1 > router2: ICMP echo request, id 9, seq 95, length 64
12:40:52.425107 IP 192.168.10.1 > router2: ICMP echo request, id 9, seq 96, length 64
12:40:53.012360 IP router2 > ospf-all.mcast.net: OSPFv2, Hello, length 48
12:40:53.426517 IP 192.168.10.1 > router2: ICMP echo request, id 9, seq 97, length 64
12:40:54.428276 IP 192.168.10.1 > router2: ICMP echo request, id 9, seq 98, length 64
12:40:55.430686 IP 192.168.10.1 > router2: ICMP echo request, id 9, seq 99, length 64
12:40:56.432458 IP 192.168.10.1 > router2: ICMP echo request, id 9, seq 100, length 64
```

Видим что данный порт только получает ICMP-трафик с адреса 192.168.10.1
На router2 запускаем tcpdump, который будет смотреть трафик только на порту enp0s8:

```
root@router2:~# tcpdump -i enp0s8
tcpdump: verbose output suppressed, use -v or -vv for full protocol decode
listening on enp0s8, link-type EN10MB (Ethernet), capture size 262144 bytes
12:49:03.189907 IP router2 > ospf-all.mcast.net: OSPFv2, Hello, length 48
12:49:03.438317 IP router2 > 192.168.10.1: ICMP echo reply, id 10, seq 37, length 64
12:49:04.445847 IP router2 > 192.168.10.1: ICMP echo reply, id 10, seq 38, length 64
12:49:04.557966 IP 10.0.10.1 > ospf-all.mcast.net: OSPFv2, Hello, length 48
12:49:05.447669 IP router2 > 192.168.10.1: ICMP echo reply, id 10, seq 39, length 64
12:49:05.646261 ARP, Request who-has 10.0.10.1 tell router2, length 28
12:49:05.646991 ARP, Reply 10.0.10.1 is-at 08:00:27:dc:af:7e (oui Unknown), length 46
12:49:06.450064 IP router2 > 192.168.10.1: ICMP echo reply, id 10, seq 40, length 64
12:49:07.451529 IP router2 > 192.168.10.1: ICMP echo reply, id 10, seq 41, length 64
12:49:08.452722 IP router2 > 192.168.10.1: ICMP echo reply, id 10, seq 42, length 64
12:49:09.456221 IP router2 > 192.168.10.1: ICMP echo reply, id 10, seq 43, length 64
```

Видим что данный порт только отправляет ICMP-трафик на адрес 192.168.10.1

Таким образом мы видим ассиметричный роутинг.

Настройка симметичного роутинга
Так как у нас уже есть один «дорогой» интерфейс, нам потребуется добавить ещё один дорогой интерфейс, чтобы у нас перестала работать ассиметричная маршрутизация. 

Так как в прошлом задании мы заметили что router2 будет отправлять обратно трафик через порт enp0s8, мы также должны сделать его дорогим и далее проверить, что теперь используется симметричная маршрутизация:

Поменяем стоимость интерфейса enp0s8 на router2:

```
router2# conf t
router2(config)# int enp0s8
router2(config-if)# ip ospf cost 1000
router2(config-if)# exit
router2(config)# exit
router2# show ip route ospf
Codes: K - kernel route, C - connected, S - static, R - RIP,
       O - OSPF, I - IS-IS, B - BGP, E - EIGRP, N - NHRP,
       T - Table, v - VNC, V - VNC-Direct, A - Babel, F - PBR,
       f - OpenFabric,
       > - selected route, * - FIB route, q - queued, r - rejected, b - backup
       t - trapped, o - offload failure

O   10.0.10.0/24 [110/1000] is directly connected, enp0s8, weight 1, 00:05:24
O   10.0.11.0/24 [110/100] is directly connected, enp0s9, weight 1, 21:01:40
O>* 10.0.12.0/24 [110/200] via 10.0.11.1, enp0s9, weight 1, 00:05:24
O>* 192.168.10.0/24 [110/300] via 10.0.11.1, enp0s9, weight 1, 00:05:24
O   192.168.20.0/24 [110/100] is directly connected, enp0s10, weight 1, 21:17:37
O>* 192.168.30.0/24 [110/200] via 10.0.11.1, enp0s9, weight 1, 21:01:21
```

После внесения данных настроек, мы видим, что маршрут до сети 192.168.10.0/30  пойдёт через 10.0.11.1.
Давайте это проверим:
На router1 запускаем пинг от 192.168.10.1 до 192.168.20.1: 

```
ping -I 192.168.10.1 192.168.20.1
```

На router2 запускаем tcpdump, который будет смотреть трафик только на порту enp0s9:

```
root@router2:~# tcpdump -i enp0s9
tcpdump: verbose output suppressed, use -v or -vv for full protocol decode
listening on enp0s9, link-type EN10MB (Ethernet), capture size 262144 bytes
13:18:20.160320 IP 192.168.10.1 > router2: ICMP echo request, id 11, seq 95, length 64
13:18:20.160376 IP router2 > 192.168.10.1: ICMP echo reply, id 11, seq 95, length 64
13:18:20.257582 ARP, Request who-has 10.0.11.1 tell router2, length 28
13:18:20.258269 ARP, Reply 10.0.11.1 is-at 08:00:27:df:87:c0 (oui Unknown), length 46
13:18:21.162203 IP 192.168.10.1 > router2: ICMP echo request, id 11, seq 96, length 64
13:18:21.162253 IP router2 > 192.168.10.1: ICMP echo reply, id 11, seq 96, length 64
13:18:22.164541 IP 192.168.10.1 > router2: ICMP echo request, id 11, seq 97, length 64
13:18:22.164620 IP router2 > 192.168.10.1: ICMP echo reply, id 11, seq 97, length 64
13:18:23.189676 IP 192.168.10.1 > router2: ICMP echo request, id 11, seq 98, length 64
13:18:23.189726 IP router2 > 192.168.10.1: ICMP echo reply, id 11, seq 98, length 64
13:18:23.685311 IP router2 > ospf-all.mcast.net: OSPFv2, Hello, length 48
13:18:24.192395 IP 192.168.10.1 > router2: ICMP echo request, id 11, seq 99, length 64
13:18:24.192459 IP router2 > 192.168.10.1: ICMP echo reply, id 11, seq 99, length 64
13:18:25.193901 IP 192.168.10.1 > router2: ICMP echo request, id 11, seq 100, length 64
13:18:25.194024 IP router2 > 192.168.10.1: ICMP echo reply, id 11, seq 100, length 64
13:18:26.195680 IP 192.168.10.1 > router2: ICMP echo request, id 11, seq 101, length 64
13:18:26.195732 IP router2 > 192.168.10.1: ICMP echo reply, id 11, seq 101, length 64
```

Теперь мы видим, что трафик между роутерами ходит симметрично.








