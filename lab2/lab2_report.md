University: [ITMO University](https://itmo.ru/ru/)  
Faculty: [FICT](https://fict.itmo.ru)  
Course: [Network programming](https://github.com/itmo-ict-faculty/network-programming)  
Year: 2022/2023  
Group: K34212  
Author: Hoang Minh Thang  
Lab: Lab2
Date created: 19.10.2022  
Date finished: 
# 

## Цель работы
С помощью Ansible настроить несколько сетевых устройств и собрать информацию о них. Правильно собрать файл Inventory.
## Ход работы
### 1. Установить второй CHR на ПК
### 2. Организовать OVPN клиент на втором CHR.
Установка Openvpn на Ubuntu с помощью команд:
``` bash
sudo apt update
sudo apt upgrade
sudo apt install openvpn
```

Установка Easy-RSA:
``` bash
wget -P ~/ https://github.com/OpenVPN/easy-rsa/releases/download/v3.0.8/EasyRSA-3.0.8.tgz
tar xvf EaseyRSA-3.0.8.tgz
```

Настройка центра сертификации (CA)
``` bash
cd ~/EasyRSA-3.0.8.tgz
cp vars.example vars
./easyrsa init-pki
./easyrsa build-ca nopass
```

Генерация ключей сервера и сертификата
``` bash
./easyrsa gen-req OVPNserver nopass
./easyrsa gen-dh
./easyrsa sign-req server OVPNserver
```

После этого сертификаты и ключи были генерированы в подкаталогиях. Скопируем в /etc/openvpn/
``` bash
sudo cp ~/EasyRSA-3.0.8/pki/ca.crt /etc/openvpn/
sudo cp ~/EasyRSA-3.0.8/pki/dh.pem /etc/openvpn/
sudo cp ~/EasyRSA-3.0.8/pki/issued/OVPNserver.crt /etc/openvpn/
sudo cp ~/EasyRSA-3.0.8/pki/private/OVPNserver.key /etc/openvpn/
```
Генерация сертификата клиента
``` bash
./easyrsa gen-req OVPNclient nopass
./easyrsa sign-req client OVPNclient
```
После этого скопируем файлы **ca.crt**, **OVPNclient.crt** и **OVPNclient.key** на OVPN клиент (CHR) используя Winbox.
![image](https://user-images.githubusercontent.com/61542577/196701156-8f19587c-82c3-49d9-8b60-712b673cc34e.png)

На OVPN сервере внести следующие изменения в файл server.conf:
```
proto tcp
;proto udp

ca ca.crt
cert OVPNserver.crt
key OVPNserver.key
dh dh.pem

server 10.0.0.0 255.255.255.0

;tls-auth ta.key 0

;explicit-exit-notify 1
```

Импортируем сертификаты и ключ на CHR используя Winbox
![image](https://user-images.githubusercontent.com/61542577/196706089-22a4a9d6-e926-46e7-b50a-35329ba23414.png)

Запускать OVPN сервер на Ubuntu
``` bash
sudo systemctl enable openvpn@server
sudo systemctl start openvpn@server
```

Настройка интерфейса OVPN Client на CHR используя Winbox
![image](https://user-images.githubusercontent.com/61542577/196709278-d7accf29-a5c8-4aba-a9c2-b4a084edce84.png)
![image](https://user-images.githubusercontent.com/61542577/196709368-625a6dad-d25d-40d4-9c1c-d38e3d437c2b.png)

Теперь OVPN туннель между нашим Ubuntu сервером и втором CHR был настроен.
### 3. Настройка 2-х CHR, используя Ansible
* ****
