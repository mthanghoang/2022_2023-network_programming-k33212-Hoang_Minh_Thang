University: [ITMO University](https://itmo.ru/ru/)  
Faculty: [FICT](https://fict.itmo.ru)  
Course: [Network programming](https://github.com/itmo-ict-faculty/network-programming)  
Year: 2022/2023  
Group: K34212  
Author: Hoang Minh Thang  
Lab: Lab3  
Date created: 21.10.2022  
Date finished: 
#
## Цель работы
С помощью Ansible и Netbox собрать всю возможную информацию об устройствах и сохранить их в отдельном файле.
## Ход работы
### 1. Поднять Ubuntu 20.04 на дополнительной виртуальной машине
Соединить эту машину с машиной на Яндексе (на котором запускается OVPN сервер), используя OVPN туннель по аналогии с второй лабораторной работой. На этой машине будет установлено Netbox.
### 2. Установить Netbox на дополнительной виртуальной машине
**Установка Postgresql**

С помощью следующих команд установить Postgresql.
```
sudo apt update
sudo apt install -y postgresql
```
После этого запустить службу и ее включить при перезагрузке машины.
```
sudo systemctl start postgresql
sudo systemctl enable postgresql
```
Перейти в оболочку Postgresql.
```
sudo -u postgres psql
```
Создать базу данных для Netbox.
```
CREATE DATABASE netbox;
CREATE USER netbox WITH PASSWORD '12345679';
GRANT ALL PRIVILEGES ON DATABASE netbox TO netbox;
```
Проверить статус службы.

![image](https://user-images.githubusercontent.com/61542577/203171295-bf9f13d6-7034-4845-a8c0-04a639a24d5d.png)

**Установка Redis**

Установить Redis с помощью команды.
```
sudo apt install -y redis-server
```
**Установка Netbox**

Установить все системные пакеты, необходимые для NetBox и его зависимостей.
```
sudo apt install -y python3 python3-pip python3-venv python3-dev build-essential libxml2-dev libxslt1-dev libffi-dev libpq-dev libssl-dev zlib1g-dev
```
Клонировать основную ветку репозитория NetBox GitHub.
```
sudo mkdir -p /opt/netbox/
cd /opt/netbox/
sudo git clone -b master --depth 1 https://github.com/netbox-community/netbox.git .
```
Создать системную учетную запись пользователя с именем Netbox.
```
sudo adduser --system --group netbox
sudo chown --recursive netbox /opt/netbox/netbox/media/
```
Создать виртуальную среду Python.
```
sudo /opt/netbox/upgrade.sh
```
Войти в созданную виртуальную среду и создать суперпользователя чтобы авторировать в Netbox.
```
source /opt/netbox/venv/bin/activate
cd /opt/netbox/netbox
python3 manage.py createsuperuser
```
Запустить сервер Netbox.
```
python3 manage.py runserver 0.0.0.0:8000 --insecure
```
После этого можно подключиться к Netbox по протоколу HTTP.

![image](https://user-images.githubusercontent.com/61542577/203173912-a38a8ef4-474b-48b8-b2ce-dc853707884b.png)

**Установка nginx**

Генерировать самоподписанный сертификат SSL.
```
sudo openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
-keyout /etc/ssl/private/netbox.key \
-out /etc/ssl/certs/netbox.crt
```
Установить nginx и запустить службу.
```
sudo apt install -y nginx
sudo cp /opt/netbox/contrib/nginx.conf /etc/nginx/sites-available/netbox
sudo rm /etc/nginx/sites-enabled/default
sudo ln -s /etc/nginx/sites-available/netbox /etc/nginx/sites-enabled/netbox
sudo systemctl restart nginx
```
Теперь можно подключиться к Netbox по протоколу HTTPS.

![image](https://user-images.githubusercontent.com/61542577/203174510-d2aa3ac9-cf05-4f03-bb38-37755ff30834.png)

### 3. Заполнить информацию о наших CHR в Netbox.
Информация о первом CHR

![image](https://user-images.githubusercontent.com/61542577/203174868-8bade9fe-cf25-48a3-b562-f7926b14e439.png)

![image](https://user-images.githubusercontent.com/61542577/203174983-94a4a32a-4732-4385-9f4f-c7ae72f51c4b.png)

Информация о втором CHR

![image](https://user-images.githubusercontent.com/61542577/203175560-d7be9d26-1def-4b13-a20f-66452343c42c.png)

![image](https://user-images.githubusercontent.com/61542577/203175591-f3e64fad-6441-43c0-afa7-29f2ae699be5.png)

### 4. Используя Ansible сохранить все данные из Netbox в отдельный файл
Файл Inventory
```
[routers]
10.10.0.50
10.10.0.60
[Netbox]
10.10.0.2
[routers:vars]
ansible_connection=network_cli
ansible_network_os=routeros
ansible_network_cli_ssh_type=paramiko
ansible_user=admin
[Netbox:vars]
ansible_user=mthanghoang
```
Сценарий netbox_gather_facts.yml для собрания данных из Netbox и сохранения их в файл.
```
- name: Gather data from netbox
  hosts: Netbox
  gather_facts: false
  tasks:

    - name: GATHER DEVICES
      set_fact:
        devices: "{{ query('netbox.netbox.nb_lookup', 'devices', api_endpoint='https://10.10.0.2',
                 token='46e343c43c98592f21ac2de449df1d33ec914501',
                 validate_certs=false) }}"
    - name: GATHER INTERFACES
      set_fact:
        interfaces: "{{ query('netbox.netbox.nb_lookup', 'interfaces', api_endpoint='https://10.10.0.2',
                 token='46e343c43c98592f21ac2de449df1d33ec914501',
                 validate_certs=false) }}"

    - name: SAVE TO FILE
      delegate_to: 127.0.0.1
      copy:
        content:
          - "{{ devices }}"
          - "{{ interfaces }}"
        dest: ~/netbox_data.txt
```
Запускать сценарий netbox_gather_facts.yml
``` bash
ansible-playbook -i ~/ansible/inventory ~/ansible/netbox_gather_facts.yml
```
![image](https://user-images.githubusercontent.com/61542577/203176857-f6dccc45-1713-432c-b141-3f59e682edeb.png)

Данные из Netbox сохранены в файл ~/netbox_data.txt

### 5. Написать сценарий, при котором на основе данных из Netbox можно настроить 2 CHR
- Изменить имя устройства
- Добавить IP адрес на устройство

Сценарий netbox_config_routers.yml для настроения устройств. Сценарий также позволяет проверить имени и IP адресы до и после запуски.
```
- name: Change Router Name and add IP-address based on  Netbox
  hosts: routers
  gather_facts: false
  tasks:

    - name: Getting devices data from Netbox
      set_fact:
        devices: "{{ query('netbox.netbox.nb_lookup', 'ip-addresses',
                        api_endpoint='https://10.10.0.2',
                        api_filter='tag=add',
                        token='46e343c43c98592f21ac2de449df1d33ec914501',
                        validate_certs=false) }}"
        index: "{{ groups['routers'].index(inventory_hostname) | int }}"

    - name: SAVE DATA TO VARIABLES
      set_fact:
        router_name: "{{ devices[index | int].value.assigned_object.device.name }}"
        router_ip: "{{ devices[index | int].value.address }}"

    - name: SET ROUTERS' NAME AND ADD IPs ACCORDINGLY
      routeros_command:
        commands:
          - /ip/address/print detail
          - /ip/address/add address={{ router_ip }} interface=ether2
          - /ip/address/print detail
          - /system/identity/set name="{{ router_name }}"
          - /system/identity/print
       register: output

    - name: CHECK FOR NEW NAME AND IP
      debug:
        var: output.stdout_lines
```
Запускать сценарий netbox_config_routers.yml
```bash
ansible-playbook -i ~/ansible/inventory ~/ansible/netbox_config_routers.yml
```
![image](https://user-images.githubusercontent.com/61542577/203181930-5deb7389-b048-4251-a1cb-c0c2e1ce32e4.png)

![image](https://user-images.githubusercontent.com/61542577/203181987-5dde65c1-cbb4-48b1-acab-ca0400306112.png)

![image](https://user-images.githubusercontent.com/61542577/203182021-6bf20ef4-177f-419e-9dd9-2048b78f97ac.png)

### 6. Написать сценарий, позволяющий собрать серийный номер устройства и вносящий серийный номер в Netbox
Сценарий serial_num.yml для собрания серийного номера устройсва и добавления его в Netbox
```
- name: Get Devices' Serial Number and Populate into Netbox
  hosts: routers
  gather_facts: false
  tasks:

  - name: Get Serial Number
    community.routeros.command:
      commands:
        - /system/license/print
    register: output

  - name: Print Serial Number
    debug:
      var: output.stdout_lines
  - name: SAVE DATA TO VARIABLES
    set_fact:
      index: "{{ groups['routers'].index(inventory_hostname) | int }}"
      devices: "{{ query('netbox.netbox.nb_lookup', 'devices',
                        api_endpoint='https://10.10.0.2',
                        token='46e343c43c98592f21ac2de449df1d33ec914501',
                        validate_certs=false) }}"

  - name: Populate Netbox with serial numbers
    netbox_device:
      netbox_url: https://10.10.0.2
      netbox_token: 46e343c43c98592f21ac2de449df1d33ec914501
      data:
        name: "{{ devices[index | int].value.name }}"
        serial: "{{ output.stdout_lines[0][0] | regex_replace('system-id:', '')}}"
      validate_certs: false
```
Запускать сценарий serial_num.yml

![image](https://user-images.githubusercontent.com/61542577/203182703-9c5b73a9-0f6b-4c19-8013-9507427fa596.png)

Теперь в информации устройств на Netbox появился серийный номер.

![image](https://user-images.githubusercontent.com/61542577/203182923-2fba1901-4acf-4b7f-ad1e-6d7d3c3895a3.png)

![image](https://user-images.githubusercontent.com/61542577/203182888-7dd319d2-c1e8-4f2c-a7c6-537d5d83b260.png)

## Результаты
В результате у нас файл с данными, собранными из Netbox сервера netbox_data.txt

Схема связи

![image](https://user-images.githubusercontent.com/61542577/203185026-77c68eb6-21e1-47c1-8ffa-83583bd140de.png)

Пинг между машиной Netbox с VPN сервером

![image](https://user-images.githubusercontent.com/61542577/203185173-b8d546b3-7a76-4a0c-8feb-598031a9c4ca.png)

Пинг между машиной Netbox c CHR

![image](https://user-images.githubusercontent.com/61542577/203185308-660b034f-d236-4c86-956d-d147a354314f.png)

## Вывод
Познакомились с инструментом документирования сетей Netbox. С помощью Netbox и Ansible были настроены оба CHR. Также добавления информации в Netbox осуществляется Ansible.





