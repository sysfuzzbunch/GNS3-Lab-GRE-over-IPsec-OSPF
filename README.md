# GNS3-Lab-GRE-over-IPsec-OSPF
Первая лаба с реальным оборудованием + GNS3.

В этой лабораторной работе реализован защищенный туннель между физическим маршрутизатором (Cisco 3825) и виртуальным филиалом (GNS3). Поверх туннеля запущен протокол динамической маршрутизации OSPF для автоматического обмена маршрутами локальных сетей. Цель лабораторной работы, изучить особенности настройки GRE-over-IPsec и применить теоретические знания о протоколе динамической маршрутизации OSPF на практике.

Физическая топология выглядит так:

<img width="745" height="745" alt="изображение" src="https://github.com/user-attachments/assets/1548c058-1a77-401a-8be0-d1c5d38c31ff" />

<img width="1280" height="960" alt="изображение" src="https://github.com/user-attachments/assets/0b2d1951-fdb2-4486-8234-a5b45692e887" />

Виртуальная топология:

<img width="1030" height="703" alt="изображение" src="https://github.com/user-attachments/assets/1a5d47f2-af90-4443-a15e-89239f987344" />

Задача: Объединить две локальные сети через незащищенную среду (эмуляция WAN), зашифровав трафик стандартом IPsec, и настроить динамическую маршрутизацию без статических маршрутов.

|Устройство|Роль|Внешний IP (WAN)|Внутренний IP (LAN)|Tunnel IP|
|---|---|---|---|---|
|R1 (Physical)|Office 1 Gateway|10.0.1.1|172.20.0.1/16|172.16.0.1|
|R2 (GNS3)|Office 2 Gateway|10.0.1.2|172.25.0.1/16|172.16.0.2|

### Принцип работы

Используется классическая схема GRE over IPsec (Map-based):

1. OSPF принимает решение о маршрутизации и отправляет трафик в виртуальный интерфейс Tunnel0.
2. GRE инкапсулирует пакет, добавляя новый заголовок (транспортный).
3. Crypto Map на физическом интерфейсе отслеживает GRE-трафик (по ACL) и шифрует его в IPsec (ESP) перед отправкой в канал.
---

## 2. Конфигурация R1 (Cisco ISR 3825)

Это реальный маршрутизатор, шлюз центрального офиса.

### Шаг 1: Настройка ISAKMP (Phase 1)

Договариваемся с соседом о ключах шифрования и методе аутентификации. Это создание защищенного канала управления.

Cisco CLI

```
crypto isakmp policy 10
 encryption aes 256        ! Алгоритм шифрования (самый надежный)
 hash sha                  ! Хеширование для проверки целостности
 authentication pre-share  ! Аутентификация по паролю
 group 5                   ! Группа Диффи-Хеллмана (сложность ключа)
 lifetime 86400            ! Время жизни сессии (сутки)

! Задаем пароль для соседа (WAN IP)
crypto isakmp key cisco address 10.0.1.2
```

### Шаг 2: Настройка IPsec (Phase 2) и ACL

Определяем, _как_ шифровать данные и _что именно_ шифровать.

Cisco CLI

```
! Набор алгоритмов для шифрования полезной нагрузки
crypto ipsec transform-set MY_SET esp-aes 256 esp-sha-hmac
 mode tunnel

! ACL, который "вылавливает" GRE трафик для шифрования
! Важно: разрешаем именно протокол GRE от внешнего IP к внешнему IP
ip access-list extended 100
 permit gre host 10.0.1.1 host 10.0.1.2
```

 ### Шаг 3: Crypto Map (Привязка к интерфейсу)

Создаем карту, которая объединяет настройки шифрования и ACL, и вешаем её на "выход".

Cisco CLI

```
crypto map MY_MAP 10 ipsec-isakmp
 set peer 10.0.1.2
 set transform-set MY_SET
 match address 100

! Применяем на физический WAN интерфейс
interface GigabitEthernet0/0
 ip address 10.0.1.1 255.255.255.0
 crypto map MY_MAP
 no shutdown
```

### Шаг 4: Настройка Туннеля (GRE)

Создаем виртуальный интерфейс. Это "мост" между офисами.

Cisco CLI

```
interface Tunnel0
 ip address 172.16.0.1 255.255.255.252
 tunnel source 10.0.1.1      ! Отправляем с нашего внешнего IP
 tunnel destination 10.0.1.2 ! Шлем на внешний IP соседа
 
 ! Критически важно для VPN! Снижаем размер пакета, чтобы влезли заголовки GRE+IPsec
 ip mtu 1400
 ip tcp adjust-mss 1360
```

### Шаг 5: Динамическая маршрутизация (OSPF)

Запускаем OSPF внутри туннеля. Внешнюю сеть не анонсируем, чтобы избежать рекурсии.

Cisco CLI

```
router ospf 1
 router-id 1.1.1.1
 ! Анонсируем локальную сеть офиса
 network 172.20.0.0 0.0.255.255 area 0
 ! Анонсируем сеть туннеля (для связности)
 network 172.16.0.0 0.0.0.3 area 0
 ! Анонсируем Loopback (для тестов)
 network 1.1.1.1 0.0.0.0 area 0
 passive-interface Loopback0
```
---

## 3. Конфигурация R2 (Virtual GNS3)

Виртуальный роутер филиала. Конфигурация зеркальна R1.


Cisco CLI

```
! --- Phase 1 ---
crypto isakmp policy 10
 encryption aes 256
 hash sha
 authentication pre-share
 group 5
 lifetime 86400
crypto isakmp key cisco address 10.0.1.1

! --- Phase 2 ---
crypto ipsec transform-set MY_SET esp-aes 256 esp-sha-hmac
 mode tunnel

! --- ACL (Зеркальный!) ---
! Здесь Source - это R2, Destination - это R1
ip access-list extended 100
 permit gre host 10.0.1.2 host 10.0.1.1

! --- Crypto Map ---
crypto map MY_MAP 10 ipsec-isakmp
 set peer 10.0.1.1
 set transform-set MY_SET
 match address 100

! --- Interfaces ---
interface GigabitEthernet0/0
 ip address 10.0.1.2 255.255.255.0
 crypto map MY_MAP
 no shutdown

interface Tunnel0
 ip address 172.16.0.2 255.255.255.252
 tunnel source 10.0.1.2
 tunnel destination 10.0.1.1
 ip mtu 1400
 ip tcp adjust-mss 1360
 ! Принудительный тип сети, чтобы OSPF быстрее сходился
 ip ospf network point-to-point

! --- OSPF ---
router ospf 1
router-id 9.9.9.9
passive-interface Loopback0
network 9.9.9.9 0.0.0.0 area 0 ! loopback0
network 10.0.2.0 0.0.0.3 area 0 ! Point-to-Point R2
network 10.0.2.4 0.0.0.3 area 0 ! Point-to-Point R3
network 172.16.0.0 0.0.0.3 area 0 ! Point-to-Point tun0
passive-interface Loopback0
```

---

## 4. Верификация и Траблшутинг

### Основные команды проверки

Чтобы убедиться, что VPN работает, используем следующие команды:

1. Проверка соседства OSPF:

    Cisco CLI
    

    `show ip ospf neighbor`
    
    _Ожидаемый результат:_ Состояние FULL через интерфейс Tunnel0.
    
2. Проверка шифрования:

    
    Cisco CLI
    

    `show crypto ipsec sa`
    
    _Ожидаемый результат:_ Счетчики pkts encaps и pkts decaps должны расти при отправке пинга.
    
1. Проверка 1-й фазы (ISAKMP):

    Cisco CLI
    

    `show crypto isakmp sa`
    
    _Ожидаемый результат:_ Статус QM_IDLE.

### Проблемы, с которыми столкнулся (Lessons Learned)

> Проблема 1: Односторонняя видимость
> 
> Пинг с хоста проходил в одну сторону, но не возвращался.
> 
> Решение: Проблема была в брандмауэре Windows и настройках promiscuous mode на стыке реальной сетевой карты и GNS3. Также важно убедиться, что на хосте корректно прописан Default Gateway.

> Проблема 2: MTU и Фрагментация
> 
> OSPF поднимался, но сайты/тяжелые пакеты не открывались.
> 
> Решение: Добавление команд ip mtu 1400 и ip tcp adjust-mss 1360. Это необходимо, так как заголовки GRE и IPsec "отъедают" место в пакете, и стандартный 1500-байтный пакет не пролезает.

> Проблема 3: Рекурсивная маршрутизация
> 
> Если добавить внешний IP (WAN) в OSPF, туннель начинает падать (flapping).
> 
> Решение: OSPF должен работать _только_ внутри туннеля. Внешние адреса должны быть доступны через статику или connected.

> Loppback интерфейсы и интерфейсы, к которому не подключены маршрутизаторы находятся в пассивном режиме. На L3 свитчах, G0/2 интерфейсы переведены в пассивный режим, так как к ним подключены конечные хосты.







