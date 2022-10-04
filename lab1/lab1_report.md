University: [ITMO University](https://itmo.ru/ru/)  
Faculty: [FICT](https://fict.itmo.ru)  
Course: [Network programming](https://github.com/itmo-ict-faculty/network-programming)  
Year: 2022/2023  
Group: K34212  
Author: Hoang Minh Thang  
Lab: Lab1  
Date created: 21.09.2022  
Date finished:
# 

## ЦЕЛЬ РАБОТЫ:
1. Развертывание виртуальной машины Ubuntu с установленной Ansible на платформы Яндекс Облако
2. Установка CHR на VirtualBox
3. Поднять VPN туннель между машиной Ubuntu и RouterOS
## ХОД РАБОТЫ:
### 1. Развертывание виртуальной машины Ubuntu на платформе Яндекс Облако.
Можно посмотреть обзор информации о виртуальной машины
![image](https://user-images.githubusercontent.com/61542577/193770134-312cdc1d-a188-4757-bc57-d1799a4bef43.png)

### 2. Установка системы контроля конфигураций Ansible на виртуальной машине.
* **Обновление операционной системы с помощью команд:**
```bash
sudo apt update
sudo apt upgrade
```
* **Установление python3 и Ansible**
```bash
sudo apt install python3-pip
sudo pip3 install ansible
```
### 3. Установка CHR на VirtualBox
* **Загрузка VDI файл на <https://mikrotik.com>**
* **Установка CHR на VirtualBox с помощью загруженного файла**
### 4. Создание Wireguard сервера на машине Ubuntu (далее - VPN сервер)
* **Установка Wireguard**
```bash
sudo apt install wireguard
```
* **Далее необходимо сгенерировать пару закрытого и открытого ключей для сервера**
> Закрытый ключ сгенирирован и скопирован в файл /etc/wireguard/private.key
```bash
wg genkey | sudo tee /etc/wireguard/private.key
```
> Соответственный открытый ключ получен из закрытого ключа и скопирован в файл /etc/wireguard/public.key
```bash
sudo cat /etc/wireguard/private.key | wg pubkey | sudo tee /etc/wireguard/public.key
```
* **Конфигурация Wireguard сервера**
> Создание файла конфигурации для сервера
```bash
sudo nano /etc/wireguard/wg0.conf
```
> Теперь необходимо выбрать диапазон адресов для сервера и клиента. В данной работе были выбраны адресы из диапазон 10.8.0.0/24.  
  Номер прослушивания (Listen Port) - 51820, что является значением по умолчанию для Wireguard сервера.  
  После определения IP адреса и номера прослушивания необходимо внести изменения в файл конфигураций /etc/wireguard/wg0.conf
  
```
[Interface]
Address = 10.8.0.1/24
ListenPort = 51820
PrivateKey = INPDG1SdQjqBMtQ01cTmLTFKqsMkJVra6DwnxbvPLEw=
SaveConfig = true
```
![image](https://user-images.githubusercontent.com/61542577/193769952-dce127cf-4e58-428f-9498-052e75ee089f.png)

> **PrivateKey** - закрытый ключ сервера, который мы скопировали в файл /etc/wireguard/private.key.  
> Строка **SaveConfig** гарантирует, что при выключении интерфейса WireGuard любые изменения будут сохранены в файле конфигурации.
* **Запуск сервера Wireguard**
> Необходимо включить интерфейс при перезагрузке и запускать его.
```bash
sudo systemctl enable wg-quick@wg0
sudo systemctl start wg-quick@wg0
```
### 5. Настройка Wireguard клиента на RouterOS
* **Добавление нового интерфейса Wireguard и назначение ему IP-адреса 10.8.0.2/30**
```mikrotik
[admin@Mikrotik] > interface/wireguard add name=wg0
[admin@Mikrotik] > ip/address add address=10.8.0.2/24 interface=wg0
```
> При этом пара закрытого и открытого ключа будет автоматически сгенерирована, их можно посмотреть с помощью команды:
```
[admin@Mikrotik] > interface/wireguard print
```
![image](https://user-images.githubusercontent.com/61542577/193770500-723938a4-c735-4861-a2d7-9e4c01fafb7f.png)

* **Далее необходимо добавить Wireguard пир, указав нужные параметры сервера.**
```
[admin@Mikrotik] > interface/wireguard/peers add public-key="j4MFO922JTRyx3JCfVTy8OGiJaBirh/90d2s6nwdLn4=" allowed-address=10.8.0.1/32 endpoint-address=178.154.202.203 endpoint-port=51820 interface=wg0
```
После этого пир будет добавлен и информацию о пире можно посмотреть с помощью команды:
```
[admin@Mikrotik] > interface/wireguard/peers print detail
```
![image](https://user-images.githubusercontent.com/61542577/193771153-b484af06-8227-4b11-b2d8-681232d0478e.png)

> **public-key** - открытый ключ сервера  
> **allowed-address** - разрешеный адрес через Wireguard туннель  
> **endpoint-address** - публичный IP-адрес сервера  
> **endpoint-port** - номер порта прослушивания сервера
> **interface** - Wireguard интерфейс на RouterOS
* **Добавление открытого ключа Wireguard клинета на Wireguard сервер**
> 
```bash
sudo wg set wg0 peer 4C6ntaWzrJPNmsIyg5HsJxd5EGdb9FZ+9rQjmJMwqhU= allowed-ips 10.8.0.2
```
> Теперь Wireguard туннель между нашим VPN сервером на Ubuntu и VPN клиентом на RouterOS был настроен. При этом клиент должен инициировать соединение со сервером.
![image](https://user-images.githubusercontent.com/61542577/193772129-325b9090-ab52-412e-8b88-4bbba9f82796.png)
## ВЫВОД:
В ходе работы были изучены основные протоколы VPN и как настроить VPN туннель.

