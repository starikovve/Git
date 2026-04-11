Vagrant-стенд c OSPF

Цель домашнего задания
Создать домашнюю сетевую лабораторию. Научится настраивать протокол OSPF в Linux-based системах.


Описание домашнего задания
1. Развернуть 3 виртуальные машины
2. Объединить их разными vlan
- настроить OSPF между машинами на базе Quagga;
- изобразить ассиметричный роутинг;
- сделать один из линков "дорогим", но что бы при этом роутинг был симметричным.



1. Разворачиваем 3 виртуальные машины

Так как мы планируем настроить OSPF, все 3 виртуальные машины должны быть соединены между собой (разными VLAN), а также иметь одну (или несколько) доолнительных сетей, к которым, далее OSPF сформирует маршруты. Исходя из данных требований, мы можем нарисовать топологию сети:

![alt text](image.png)

Обратите внимание, сети, указанные на схеме не должны использоваться в Oracle Virtualbox, иначе Vagrant не сможет собрать стенд и зависнет. По умолчанию Virtualbox использует сеть 10.0.2.0/24. Если была настроена другая сеть, то проверить её можно в настройках программы: VirtualBox — File — Preferences — Network — щёлкаем по созданной сети

Результатом выполнения данной команды будут 3 созданные виртуальные машины, которые соединены между собой сетями (10.0.10.0/30, 10.0.11.0/30 и 10.0.12.0/30). У каждого роутера есть дополнительная сеть:

- на router1 — 192.168.10.0/24
- на router2 — 192.168.20.0/24
- на router3 — 192.168.30.0/24


# Установка пакетов для тестирования и настройки OSPF

Перед настройкой FRR рекомендуется поставить базовые программы для изменения конфигурационных файлов (vim) и изучения сети (traceroute, tcpdump, net-tools):

sudo apt update

sudo apt install vim traceroute tcpdump net-tools -y

# 2.1 Настройка OSPF между машинами на базе Quagga

Пакет Quagga перестал развиваться в 2018 году. Ему на смену пришёл пакет FRR, он построен на базе Quagga и продолжает своё развитие. В данном руководстве настойка OSPF будет осуществляться в FRR. 

Процесс установки FRR и настройки OSPF вручную:
1) Отключаем файерволл ufw и удаляем его из автозагрузки:

   sudo systemctl stop ufw 

   sudo systemctl disable ufw

![alt text](image-1.png)

2) Добавляем gpg ключ:

   curl -s https://deb.frrouting.org/frr/keys.asc | sudo apt-key add -

3) Добавляем репозиторий c пакетом FRR:

   echo "deb https://deb.frrouting.org/frr $(lsb_release -s -c) frr-stable" | sudo tee /etc/apt/sources.list.d/frr.list

   curl -s https://deb.frrouting.org/frr/keys.asc | sudo apt-key add -

   sudo apt update


4) Обновляем пакеты и устанавливаем FRR:

   sudo apt update
   
   sudo apt install frr frr-pythontools

5) Разрешаем (включаем) маршрутизацию транзитных пакетов:

sudo sysctl net.ipv4.conf.all.forwarding=1

6) Включаем демон ospfd в FRR
Для этого открываем в редакторе файл /etc/frr/daemons и меняем в нём параметры для пакетов zebra и ospfd на yes:

sudo vim /etc/frr/daemons

zebra=yes
ospfd=yes
bgpd=no
ospf6d=no
ripd=no
ripngd=no
isisd=no
pimd=no
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

В примере показана только часть файла

![alt text](image-2.png)


7) Настройка OSPF
Для настройки OSPF нам потребуется создать файл /etc/frr/frr.conf который будет содержать в себе информацию о требуемых интерфейсах и OSPF. Разберем пример создания файла на хосте router1. 


Для начала нам необходимо узнать имена интерфейсов и их адреса. Сделать это можно с помощью двух способов:
Посмотреть в linux: ip a | grep inet 
```
root@router1:~# ip a | grep "inet " 
    vagrant@router1:~$ ip a | grep inet
    inet 127.0.0.1/8 scope host lo
    inet 10.0.2.15/24 metric 100 brd 10.0.2.255 scope global dynamic eth0
    inet6 fd17:625c:f037:2:a00:27ff:fee8:487c/64 scope global dynamic mngtmpaddr noprefixroute
    inet6 fe80::a00:27ff:fee8:487c/64 scope link
    inet 10.0.10.1/30 brd 10.0.10.3 scope global eth1
    inet6 fe80::a00:27ff:feae:83b1/64 scope link
    inet 10.0.12.1/30 brd 10.0.12.3 scope global eth2
    inet6 fe80::a00:27ff:fe4f:73db/64 scope link
    inet 192.168.10.1/24 brd 192.168.10.255 scope global eth3
    inet6 fe80::a00:27ff:feb3:ada5/64 scope link
    inet 192.168.50.10/24 brd 192.168.50.255 scope global eth4
    inet6 fe80::a00:27ff:fe8a:e81e/64 scope link
root@router1:~# 
```

Зайти в интерфейс FRR и посмотреть информацию об интерфейсах

```
vtysh
```

![alt text](image-3.png)

В обоих примерах мы увидем имена сетевых интерфейсов, их ip-адреса и маски подсети. Исходя из схемы мы понимаем, что для настройки OSPF нам достаточно описать интерфейсы eth1, eth2, eth3


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
interface eth1
 !Указываем имя интерфейса
 description r1-r2
 !Указываем ip-aдрес и маску (эту информацию мы получили в прошлом шаге)
 ip address 10.0.10.1/30
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
interface eth2
 description r1-r3
 ip address 10.0.12.1/30
 ip ospf mtu-ignore
 !ip ospf cost 45
 ip ospf hello-interval 10
 ip ospf dead-interval 30

interface eth3
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
 network 10.0.10.0/30 area 0
 network 10.0.12.0/30 area 0
 network 192.168.10.0/24 area 0 
 !Указываем адреса соседних роутеров
 neighbor 10.0.10.2
 neighbor 10.0.12.2

!Указываем адрес log-файла
log file /var/log/frr/frr.log
default-information originate always
```

![alt text](image-4.png)

![alt text](image-5.png)

Сохраняем изменения и выходим из данного файла. 

Вместо файла frr.conf мы можем задать данные параметры вручную из vtysh. Vtysh использует cisco-like команды.
 
На хостах router2 и router3 также потребуется настроить конфигруационные файлы, предварительно поменяв ip -адреса интерфейсов. 

В ходе создания файла мы видим несколько OSPF-параметров, которые требуются для настройки:
