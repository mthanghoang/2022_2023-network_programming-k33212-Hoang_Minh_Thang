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

topology subnet

server 10.10.0.0 255.255.255.0

client-config-dir ccd
client-to-client

;tls-auth ta.key 0

;explicit-exit-notify 1
```

Импортируем сертификаты и ключ на CHR используя Winbox
![image](https://user-images.githubusercontent.com/61542577/196706089-22a4a9d6-e926-46e7-b50a-35329ba23414.png)

Задание статического IP-адреса клиентам
``` bash
echo "ifconfig-push 10.10.0.50 255.255.255.0" >> /etc/openvpn/ccd/OVPNclient
echo "ifconfig-push 10.10.0.60 255.255.255.0" >> /etc/openvpn/ccd/OVPNclient2
```

Запускать OVPN сервер на Ubuntu
``` bash
sudo systemctl enable openvpn@server
sudo systemctl start openvpn@server
```

Настройка интерфейса OVPN Client на CHR используя Winbox
![image](https://user-images.githubusercontent.com/61542577/196709278-d7accf29-a5c8-4aba-a9c2-b4a084edce84.png)
![image](https://user-images.githubusercontent.com/61542577/196709368-625a6dad-d25d-40d4-9c1c-d38e3d437c2b.png)

Теперь OVPN туннель между нашим Ubuntu сервером и CHR был настроен.
### 3. Используя Ansible, настроить на 2-х CHR:
- Логин/пароль
- NTP клиент
- OSPF с указанием RouterID
- Собрать данные по OSPF топологии и полный конфиг устройств

Файл Inventory
```
[routers]
10.10.0.60
10.10.0.50
[routers:vars]
ansible_connection=network_cli
ansible_network_os=routeros
ansible_network_cli_ssh_type=libssh
ansible_user=admin
```

Сценарии routers_config.yml для настройки CHR
```
- name: CHRs config
  hosts: routers
  gather_facts: false
  tasks:

  - name: Add login/password
    community.routeros.command:
      commands:
        - /user/add name=mthanghoang group=read password=12345679

  - name: Configure NTP client
    community.routeros.command:
      commands:
        - /system/ntp/client/set enabled=yes servers=216.239.32.15

  - name: Gather facts
    community.routeros.facts:
      gather_subset:
        - interfaces

  - name: Configure OSPF
    community.routeros.command:
      commands:
        - /routing/ospf/instance/add name=instance1 version=2 router-id={{ ansible_net_all_ipv4_addresses[0] }}
        - /routing/ospf/area/add name=area0 area-id=0.0.0.0 instance=instance1
        - /routing/ospf/interface-template/add interfaces=all area=area0 type=ptp
```
Запускаем сценарии
``` bash
ansible-playbook -i ~/ansible/inventory ~/ansible/routers_config.yml
```
![image](https://user-images.githubusercontent.com/61542577/197294392-ddbad8b6-2947-47e0-8dc0-bbe9d8c578b1.png)

Сценарии gather_facts.yml для собрания данных о устройствах
```
- name: OSPF topology and device configuration
  hosts: routers
  gather_facts: false
  tasks:

  - name: Gather facts
    community.routeros.facts:
      gather_subset:
        - routing
        - interfaces

  - name: Get config
    community.routeros.command:
      commands:
        - /export
    register: config

  - name: Print config
    debug:
      var: config.stdout_lines

  - name: Copy config to file conf1
    copy:
      content: "{{ config.stdout_lines }}"
      dest: ~/conf1.txt
    when: ansible_net_all_ipv4_addresses[0] == "1.1.1.1"

  - name: Copy config to file conf2
    copy:
      content: "{{ config.stdout_lines }}"
      dest: ~/conf2.txt
    when: ansible_net_all_ipv4_addresses[0] == "2.2.2.2"

  - name: Show OSPF instance
    debug:
      var: ansible_net_ospf_instance

  - name: Show OSPF neighbor
    debug:
      var: ansible_net_ospf_neighbor
```

Запускаем сценарии
``` bash
ansible-playbook -i ~/ansible/inventory ~/ansible/gather_facts.yml
```
![image](https://user-images.githubusercontent.com/61542577/197299313-a8ce38ee-22aa-4c23-a977-b2e9654ca96e.png)

### 4. Результат
В результате у нас 2 файла с конфигурациями устройств conf1 и conf2. 


Схема связи


![image](https://user-images.githubusercontent.com/61542577/197304435-f3ef7581-8c5b-4275-aa46-78da26e9831c.png)

Пинг между клиентом и сервером 


![image](https://user-images.githubusercontent.com/61542577/197304545-e1227c58-4ec5-419c-9c83-e8465aa35ca7.png)

Пинг между клиентами

![image](https://user-images.githubusercontent.com/61542577/197304591-77f40b4b-1006-4602-9f69-8b61e2131d86.png)
* ****
