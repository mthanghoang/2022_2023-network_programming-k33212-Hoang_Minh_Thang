University: [ITMO University](https://itmo.ru/ru/)  
Faculty: [FICT](https://fict.itmo.ru)  
Course: [Network programming](https://github.com/itmo-ict-faculty/network-programming)  
Year: 2022/2023  
Group: K34212  
Author: Hoang Minh Thang  
Lab: Lab4  
Date created: 26.10.2022  
Date finished: 
#
## ЦЕЛЬ РАБОТЫ
Изучить синтаксис языка программирования P4 и выполнить 2 задания обучающих задания от Open network foundation для ознакомления на практике с P4.
## ХОД РАБОТЫ
### 1. Установить необходимоую виртуальную машину с заданиями для ознакомления с P4
С помощью Vagrant установлена виртуальная машина.

![image](https://user-images.githubusercontent.com/61542577/204060436-98c41af4-8ee8-4e93-890f-04e982e6e077.png)

### 2. Выполнить задание Basic Forwarding
Цель задания - дописать программу P4, реализующиую базовую переадресацию.
Неполный стартовый код программы отбрасывает все пакеты.

Схема для задания

![image](https://user-images.githubusercontent.com/61542577/204061065-72d9b574-6adc-43fa-9b4b-e8ed9659bd96.png)

**Запустить неполный стартовый код**

Заходить в директориий задания
```
cd ~/tutorials/exercises/basic
```
Скомпилировать неполный файл **basic.p4**
```
make build
make run
```
После этого Mininet схема запускается и появится Mininet командная строка

![image](https://user-images.githubusercontent.com/61542577/204061422-bf0d1793-fcde-486d-b9f9-3ed8497aabec.png)

Попробовать пинг между хостами в схеме

![image](https://user-images.githubusercontent.com/61542577/204061862-3e954489-07d5-416e-9d3e-9bdab482c38b.png)

Проверка связи не удалась, как ожидалось, потому что каждый коммутатор запрограммирован в соответствии с файлом **basic.p4**, который отбрасывает все пакеты по прибытии.

**Реализовать базовую переадресацию**

Добавлены парсеры для ipv4 и ethernet

![image](https://user-images.githubusercontent.com/61542577/204062087-30b432fa-3b60-4c06-a863-e54ff409f1ec.png)

Добавлена логика для пересылки ipv4 пакетов, которая
- Устанавливает выходный порт
- Обновляет MAC адрес назначения 
- Обновляет исходный MAC адрес
- Уменьшает значение TTL

![image](https://user-images.githubusercontent.com/61542577/204062266-53f61654-dea8-4f23-89ea-5de7bfe0b4b4.png)

Добавлена проверка ipv4 пакета

![image](https://user-images.githubusercontent.com/61542577/204062456-266c4824-8bce-458a-afef-c8726c8fc1de.png)

Добавлена вставка заголовок в исходящий пакет

![image](https://user-images.githubusercontent.com/61542577/204062631-ddb9009a-1562-47f4-8ac9-6fbba926c8e3.png)

**Проверить решение**

Сохранить файл p4 и заново скомпилировать программу.
```
make build
make run
```
Теперь пинг между хостами работает

![image](https://user-images.githubusercontent.com/61542577/204062737-7754d191-4b3f-4775-b183-758fcc328444.png)

Запустить хосты h1 и h2

![image](https://user-images.githubusercontent.com/61542577/204062858-9e1d51fc-9d21-4a2b-bf95-b4abf1c96a67.png)

Запустить send.py на h1 и receive.py на h2, проверить содержание пакетов

![image](https://user-images.githubusercontent.com/61542577/204063071-b46ad26f-1381-47ee-ab02-02383e3f5aab.png)

![image](https://user-images.githubusercontent.com/61542577/204063079-188ec4ba-58d1-4c83-8721-d32c3670382f.png)

### 3. Выполнить задание Basic Tunneling
Цель задания - добавить поддержку базового протокола туннелирования в IP-маршрутизатор, который выполнили в предыдущем задании. Определить новый тип заголовка для инкапсуляции IP-пакета и изменить код программы, чтобы она могла определить порт назначения, используя новый заголовок туннеля.

Стартовым кодом является решение предыдущего задания.

Схема для этого задания.

![image](https://user-images.githubusercontent.com/61542577/204066359-5f16d5f0-f6de-4a12-ab74-6b2d0e4372e3.png)

**Реализовать базовое туннелирование**

Добавлен новый тип заголовка **myTunnel_t**, который содержит два 16-битных поля: **proto_id** и **dst_id**.

![image](https://user-images.githubusercontent.com/61542577/204066467-2498979c-290f-4d8d-8758-e67ef8b6020e.png)

![image](https://user-images.githubusercontent.com/61542577/204066470-e773abdc-3484-4c63-8ecd-13ded6646dcb.png)

Добавлен парсер **parse_myTunnel**, который извлекает **myTunnel** заголовок. Если значение поля **proto_id** равно 0x800 то следует переходить на парсер **parse_ipv4** 

![image](https://user-images.githubusercontent.com/61542577/204066586-7a86a95a-b65f-44f1-8dfa-a6c22f218eb9.png)

Обновлен парсер **parse_ethernet**, чтобы извлечь либо **ipv4** заголовок, либо **myTunnel** заголовок в зависимости от значения поля **etherType**. Значение **etherType**, которое соответствует заголовку **myTunnel**, равен 0x1212. Это значение определено в самом начале файла.

![image](https://user-images.githubusercontent.com/61542577/204066759-b09df2fc-00b6-45a9-8c02-9e57d3abedb3.png)

![image](https://user-images.githubusercontent.com/61542577/204066772-1f1a3db2-36d8-4fec-863c-e090f5006e2e.png)

Добавлена новая таблица **myTunnel_exact**, которая вызывает функцию **myTunnel_forward** при совпадении значения поля **dst_id**.
Функция **myTunnel_forward** обновляет выходной порт для пакета. При этом таблица **myTunnel_exact** еще не заполнена. Заполнение таблицы будет описано позже.

![image](https://user-images.githubusercontent.com/61542577/204066941-056e17c0-517c-41d0-afdd-728965b2c1cd.png)

![image](https://user-images.githubusercontent.com/61542577/204066952-6216cf63-b061-4891-b0bf-f4e201271bda.png)

Обновлена функция **apply()**, чтобы она могла вызывать либо таблицу **myTunnel_exact**, либо таблицу **ipv4_lpm** при необходимости.

![image](https://user-images.githubusercontent.com/61542577/204067074-96b94d8d-09c7-4087-919a-bcdf43e10695.png)

Обновлен депарсер.

![image](https://user-images.githubusercontent.com/61542577/204067090-8d5df121-5d1b-490b-90cf-9819c43a2c75.png)

**Заполнить новую таблицу**

На основе схемы задания заполнена таблица **myTunnel_exact**. Можно добавить записи в таблицу, редактировав файлы **s1-runtime.json**, **s2-runtime.json**, **s3-runtime.json**, соответствующие коммутаторам s1, s2, s3.

![image](https://user-images.githubusercontent.com/61542577/204067235-bad97c8d-c4fa-44a4-85c9-3be49d2bbc1d.png)

**Проверить решение**

Сохранить файл p4 и заново скомпилировать программу.
```
make build
make run
```

Запустить хосты h1 и h2

![image](https://user-images.githubusercontent.com/61542577/204067299-41d05300-1eaf-4e7a-984a-e0e3372741d5.png)

Запустить receive.py на хосте h2 и send.py на хосте h1. Можно указать любой ip адрес, потому что созданный туннель не проверяет ip адрес назанчения, а только id назначения.

![image](https://user-images.githubusercontent.com/61542577/204067430-f329d08a-14f8-46b1-a31f-9aae480c4073.png)

![image](https://user-images.githubusercontent.com/61542577/204067437-debec1d2-c20e-44f7-afb6-775d2955bae7.png)

Отправлять пакет с h1 на h3, используя туннель.

![image](https://user-images.githubusercontent.com/61542577/204067519-d2ba28ce-700a-4710-8d69-a96f1077585b.png)

![image](https://user-images.githubusercontent.com/61542577/204067530-3da57de1-67ab-4379-a174-9b5e9a1cb728.png)

Все пакеты, отправляемые через туннель, успешно получены.

## ВЫВОД
Были успешно реализованы базовая переадресация и базовое туннелирование с помощью языка P4. В ходе работы был изучен синтаксис языка P4.












