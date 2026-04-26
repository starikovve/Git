Домашнее задание VPN

Цель домашнего задания

- Создать домашнюю сетевую лабораторию. Научится настраивать VPN-сервер в Linux-based системах.
Описание домашнего задания
- Настроить VPN между двумя ВМ в tun/tap режимах, замерить скорость в туннелях, сделать вывод об отличающихся показателях
- Поднять RAS на базе OpenVPN с клиентскими сертификатами, подключиться с локальной машины на ВМ
- (*) Самостоятельно изучить и настроить ocserv, подключиться с хоста к ВМ


1. TUN/TAP режимы VPN


Для выполнения первого пункта необходимо написать Vagrantfile, который будет поднимать 2 виртуальные машины server и client.


После запуска машин из Vagrantfile необходимо выполнить следующие действия на server и client машинах:

# Устанавливаем нужные пакеты и отключаем SELinux  
```
apt update
apt install openvpn iperf3 selinux-utils
setenforce 0
```
Настройка хоста 1:

 	# Cоздаем файл-ключ 
       openvpn --genkey secret /etc/openvpn/static.key
	# Cоздаем конфигурационный файл OpenVPN 
	vim /etc/openvpn/server.conf


![alt text](image.png)


# Создаем service unit для запуска OpenVPN

 # Создаем service unit для запуска OpenVPN
     vim /etc/systemd/system/openvpn@.service
     # Содержимое файла-юнита

![alt text](image-1.png)

# Запускаем сервис 
```
systemctl start openvpn@server 
systemctl enable openvpn@server
```

![alt text](image-2.png)

Настройка хоста 2: 

# Cоздаем конфигурационный файл OpenVPN 
vim /etc/openvpn/server.conf

# Содержимое конфигурационного файла  
dev tap 
remote 192.168.56.10 
ifconfig 10.10.10.2 255.255.255.0 
topology subnet 
route 192.168.56.0 255.255.255.0 
secret /etc/openvpn/static.key
comp-lzo
status /var/log/openvpn-status.log 
log /var/log/openvpn.log 
verb 3 


На хост 2 в директорию /etc/openvpn необходимо скопировать файл-ключ static.key, который был создан на хосте 1.  

	# Создаем service unit для запуска OpenVPN
     vim /etc/systemd/system/openvpn@.service
     # Содержимое файла-юнита
[Unit] 
Description=OpenVPN Tunneling Application On %I 
After=network.target 
[Service] 
Type=notify 
PrivateTmp=true 
ExecStart=/usr/sbin/openvpn --cd /etc/openvpn/ --config %i.conf 
[Install] 
WantedBy=multi-user.target
     # Запускаем сервис 
systemctl start openvpn@server 
systemctl enable openvpn@server

Далее необходимо замерить скорость в туннеле: 

На хосте 1 запускаем iperf3 в режиме сервера: iperf3 -s & 
На хосте 2 запускаем iperf3 в режиме клиента и замеряем  скорость в туннеле: iperf3 -c 10.10.10.1 -t 40 -i 5 

![alt text](image-3.png)

![alt text](image-4.png)

Повторяем пункты 1-2 для режима работы tun. 
Конфигурационные файлы сервера и клиента изменятся только в директиве dev. Делаем выводы о режимах, их достоинствах и недостатках.

![alt text](image-5.png)


2. RAS на базе OpenVPN 

Для выполнения данного задания можно воспользоваться Vagrantfile из  1 задания, только убрать одну ВМ. После запуска отключаем SELinux (setenforce 0) или создаём правило для него. 

