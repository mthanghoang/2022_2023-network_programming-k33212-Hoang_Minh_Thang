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
Можно посмотреть обзор информации о виртуальной машины по [ссылке](https://user-images.githubusercontent.com/61542577/191607858-f84b6882-d0de-4f81-bff8-fba97322245b.png)
### 2. Установка системы контроля конфигураций Ansible на виртуальной машине.
* Обновление операционной системы с помощью команд:
```
sudo apt update
sudo apt upgrade
```
* Установление python3 и Ansible
```
sudo apt install python3-pip
sudo pip3 install ansible
```
### 3. Установка CHR на VirtualBox
* Загрузка VDI файл на <https://mikrotik.com>
* Установка CHR на VirtualBox с помощью загруженного файла
### 4. Создание Wireguard сервера на машине Ubuntu (далее - VPN сервер)
* Установка Wireguard
```
sudo apt install wireguard
```
