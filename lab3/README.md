# Лабораторная работа: Настройка IS-IS в Underlay сети Clos-топологии

## **Цель работы**
Настроить протокол динамической маршрутизации IS-IS в Underlay сети для обеспечения IP-связанности между всеми сетевыми устройствами Clos-топологии.

### **Домашнее задание**
**Цель:** Настроить IS-IS для Underlay сети.

**Задачи:**
1) Настроить ISIS в Underlay сети, для IP связанности между всеми сетевыми устройствами.
2) Зафиксировать в документации - план работы, адресное пространство, схему сети, конфигурацию устройств.
3) Убедиться в наличии IP связанности между устройствами в ISIS домене.

## **Топология сети**
![Схема Clos-топологии с IS-IS](lab.png)

### **Архитектура:**
- 2 spine коммутатора (spine-1, spine-2)
- 3 leaf коммутатора (leaf-1, leaf-2, leaf-3)
- Клиентские устройства (Client-1, Client-2, Client-3, Client-4)

## **Схема IP-адресации**

### **Loopback интерфейсы (Router ID):**
- `spine-1`: 10.0.0.1/32
- `spine-2`: 10.0.0.2/32
- `leaf-1`: 10.0.1.1/32
- `leaf-2`: 10.0.2.1/32
- `leaf-3`: 10.0.3.1/32

### **Spine-Leaf линки (/31):**
```
spine-1 <-> leaf-1: 10.1.1.0/31 (spine .0, leaf .1)
spine-1 <-> leaf-2: 10.1.2.0/31
spine-1 <-> leaf-3: 10.1.3.0/31

spine-2 <-> leaf-1: 10.2.1.0/31
spine-2 <-> leaf-2: 10.2.2.0/31
spine-2 <-> leaf-3: 10.2.3.0/31
```

### **Клиентские сети:**
- `leaf-1`: 192.168.1.0/24 (Client-1)
- `leaf-2`: 192.168.2.0/24 (Client-2)
- `leaf-3`: 192.168.3.0/24 (Client-3), 192.168.4.0/24 (Client-4)

## **Схема NET-адресов IS-IS**
```
spine-1: 49.0001.0000.0000.0001.00
spine-2: 49.0001.0000.0000.0002.00
leaf-1:  49.0001.0000.0000.0101.00
leaf-2:  49.0001.0000.0000.0201.00
leaf-3:  49.0001.0000.0000.0301.00

Где:
49 - ISO Country Code (частное использование)
0001 - Area ID
0000.0000.XXXX - System ID (уникальный для каждого устройства)
00 - SEL (всегда 00 для маршрутизаторов)
```

## **План настройки IS-IS**

### **1. Базовая настройка IS-IS на всех устройствах**
- Настройка NET-адресов (Network Entity Titles)
- Настройка всех устройств как Level-2 (backbone)
- Активация IS-IS на соответствующих интерфейсах
- Настройка passive-интерфейсов для Loopback

### **2. Оптимизация конфигурации IS-IS**
- Настройка интерфейсов spine-leaf как level-2 circuits
- Установка одинаковой метрики (10) для всех линков
- Настройка maximum-paths для ECMP
- Логирование изменений соседств

### **3. Верификация работы IS-IS**
- Проверка установления соседства (adjacency)
- Проверка таблицы маршрутизации IS-IS
- Тестирование связности между Loopback интерфейсами
- Проверка работы ECMP

## **Конфигурация IS-IS на устройствах**

### **1. SPINE-1 конфигурация IS-IS**

```bash
! spine-1.cfg
interface Ethernet1
   description to-leaf-1
   no switchport
   ip address 10.1.1.0/31
   isis enable UNDERLAY
   isis circuit-type level-2
   isis metric 10

interface Ethernet2
   description to-leaf-2
   no switchport
   ip address 10.1.2.0/31
   isis enable UNDERLAY
   isis circuit-type level-2
   isis metric 10

interface Ethernet3
   description to-leaf-3
   no switchport
   ip address 10.1.3.0/31
   isis enable UNDERLAY
   isis circuit-type level-2
   isis metric 10

interface Loopback0
   ip address 10.0.0.1/32
   isis enable UNDERLAY
   isis passive

router isis UNDERLAY
   net 49.0001.0000.0000.0001.00
   is-type level-2
   log-adjacency-changes
   max-lsp-lifetime 65535
   !
   address-family ipv4 unicast
      maximum-paths 64
```

### **2. SPINE-2 конфигурация IS-IS**

```bash
! spine-2.cfg
interface Ethernet1
   description to-leaf-1
   no switchport
   ip address 10.2.1.0/31
   isis enable UNDERLAY
   isis circuit-type level-2
   isis metric 10

interface Ethernet2
   description to-leaf-2
   no switchport
   ip address 10.2.2.0/31
   isis enable UNDERLAY
   isis circuit-type level-2
   isis metric 10

interface Ethernet3
   description to-leaf-3
   no switchport
   ip address 10.2.3.0/31
   isis enable UNDERLAY
   isis circuit-type level-2
   isis metric 10

interface Loopback0
   ip address 10.0.0.2/32
   isis enable UNDERLAY
   isis passive

router isis UNDERLAY
   net 49.0001.0000.0000.0002.00
   is-type level-2
   log-adjacency-changes
   max-lsp-lifetime 65535
   !
   address-family ipv4 unicast
      maximum-paths 64
```

### **3. LEAF-1 конфигурация IS-IS**

```bash
! leaf-1.cfg
interface Ethernet1
   description to-spine-1
   no switchport
   ip address 10.1.1.1/31
   isis enable UNDERLAY
   isis circuit-type level-2
   isis metric 10

interface Ethernet2
   description to-spine-2
   no switchport
   ip address 10.2.1.1/31
   isis enable UNDERLAY
   isis circuit-type level-2
   isis metric 10

interface Loopback0
   ip address 10.0.1.1/32
   isis enable UNDERLAY
   isis passive

router isis UNDERLAY
   net 49.0001.0000.0000.0101.00
   is-type level-2
   log-adjacency-changes
   max-lsp-lifetime 65535
   !
   address-family ipv4 unicast
      maximum-paths 64
```

### **4. LEAF-2 конфигурация IS-IS**

```bash
! leaf-2.cfg
interface Ethernet1
   description to-spine-1
   no switchport
   ip address 10.1.2.1/31
   isis enable UNDERLAY
   isis circuit-type level-2
   isis metric 10

interface Ethernet2
   description to-spine-2
   no switchport
   ip address 10.2.2.1/31
   isis enable UNDERLAY
   isis circuit-type level-2
   isis metric 10

interface Loopback0
   ip address 10.0.2.1/32
   isis enable UNDERLAY
   isis passive

router isis UNDERLAY
   net 49.0001.0000.0000.0201.00
   is-type level-2
   log-adjacency-changes
   max-lsp-lifetime 65535
   !
   address-family ipv4 unicast
      maximum-paths 64
```

### **5. LEAF-3 конфигурация IS-IS**

```bash
! leaf-3.cfg
interface Ethernet1
   description to-spine-1
   no switchport
   ip address 10.1.3.1/31
   isis enable UNDERLAY
   isis circuit-type level-2
   isis metric 10

interface Ethernet2
   description to-spine-2
   no switchport
   ip address 10.2.3.1/31
   isis enable UNDERLAY
   isis circuit-type level-2
   isis metric 10

interface Loopback0
   ip address 10.0.3.1/32
   isis enable UNDERLAY
   isis passive

router isis UNDERLAY
   net 49.0001.0000.0000.0301.00
   is-type level-2
   log-adjacency-changes
   max-lsp-lifetime 65535
   !
   address-family ipv4 unicast
      maximum-paths 64
```

## **Диагностика и проверка IS-IS**

### **1. Проверка соседств IS-IS на spine-1**

```bash
spine-1#show isis neighbors

Instance  VRF      System Id        Type Interface          SNPA              State Hold time   Circuit Id
UNDERLAY  default  leaf-1           L2   Ethernet1          50:0:0:d7:ee:b    UP    8           leaf-1.08
UNDERLAY  default  leaf-2           L2   Ethernet2          50:0:0:3:37:66    UP    25          spine-1.0a
UNDERLAY  default  leaf-3           L2   Ethernet3          50:0:0:d5:5d:c0   UP    9           leaf-3.0a
```

**Результат:** Spine-1 установил L2 соседство со всеми тремя leaf коммутаторами.

### **2. Проверка соседств IS-IS на spine-2**

```bash
spine-2#show isis neighbors

Instance  VRF      System Id        Type Interface          SNPA              State Hold time   Circuit Id
UNDERLAY  default  leaf-1           L2   Ethernet1          50:0:0:d7:ee:b    UP    8           leaf-1.09
UNDERLAY  default  leaf-2           L2   Ethernet2          50:0:0:3:37:66    UP    29          spine-2.0a
UNDERLAY  default  leaf-3           L2   Ethernet3          50:0:0:d5:5d:c0   UP    8           leaf-3.0b
```

**Результат:** Spine-2 установил L2 соседство со всеми тремя leaf коммутаторами.

### **3. Проверка таблицы маршрутизации IS-IS на leaf-1**

```bash
leaf-1#show ip route isis

VRF: default
Codes: C - connected, S - static, K - kernel,
       O - OSPF, IA - OSPF inter area, E1 - OSPF external type 1,
       E2 - OSPF external type 2, N1 - OSPF NSSA external type 1,
       N2 - OSPF NSSA external type2, B - Other BGP Routes,
       B I - iBGP, B E - eBGP, R - RIP, I L1 - IS-IS level 1,
       I L2 - IS-IS level 2, O3 - OSPFv3, A B - BGP Aggregate,
       A O - OSPF Summary, NG - Nexthop Group Static Route,
       V - VXLAN Control Service, M - Martian,
       DH - DHCP client installed default route,
       DP - Dynamic Policy Route, L - VRF Leaked,
       G  - gRIBI, RC - Route Cache Route

 I L2     10.0.0.1/32 [115/20] via 10.1.1.0, Ethernet1
 I L2     10.0.0.2/32 [115/20] via 10.2.1.0, Ethernet2
 I L2     10.0.2.1/32 [115/30] via 10.1.1.0, Ethernet1
                               via 10.2.1.0, Ethernet2
 I L2     10.0.3.1/32 [115/30] via 10.1.1.0, Ethernet1
                               via 10.2.1.0, Ethernet2
 I L2     10.1.2.0/31 [115/20] via 10.1.1.0, Ethernet1
 I L2     10.1.3.0/31 [115/20] via 10.1.1.0, Ethernet1
 I L2     10.2.2.0/31 [115/20] via 10.2.1.0, Ethernet2
 I L2     10.2.3.0/31 [115/20] via 10.2.1.0, Ethernet2
```

**Результат:** Leaf-1 узнал через IS-IS маршруты ко всем Loopback интерфейсам и spine-leaf сетям. Наблюдается **ECMP (Equal-Cost Multi-Path) для маршрутов до других leaf устройств**.

### **4. Проверка связности между Loopback интерфейсами**

#### **4.1 Проверка с leaf-1 до spine-1 (10.0.0.1)**

```bash
leaf-1#ping 10.0.0.1 source 10.0.1.1
PING 10.0.0.1 (10.0.0.1) from 10.0.1.1 : 72(100) bytes of data.
80 bytes from 10.0.0.1: icmp_seq=1 ttl=64 time=20.5 ms
80 bytes from 10.0.0.1: icmp_seq=2 ttl=64 time=20.0 ms
80 bytes from 10.0.0.1: icmp_seq=3 ttl=64 time=9.67 ms
80 bytes from 10.0.0.1: icmp_seq=4 ttl=64 time=7.47 ms
80 bytes from 10.0.0.1: icmp_seq=5 ttl=64 time=7.13 ms

--- 10.0.0.1 ping statistics ---
5 packets transmitted, 5 received, 0% packet loss, time 69ms
rtt min/avg/max/mdev = 7.138/12.984/20.549/6.056 ms, pipe 2, ipg/ewma 17.275/16.363 ms
```

#### **4.2 Проверка с leaf-1 до leaf-2 (10.0.2.1)**

```bash
leaf-1#ping 10.0.2.1 source 10.0.1.1
PING 10.0.2.1 (10.0.2.1) from 10.0.1.1 : 72(100) bytes of data.
80 bytes from 10.0.2.1: icmp_seq=1 ttl=63 time=30.5 ms
80 bytes from 10.0.2.1: icmp_seq=2 ttl=63 time=16.8 ms
80 bytes from 10.0.2.1: icmp_seq=3 ttl=63 time=16.7 ms
80 bytes from 10.0.2.1: icmp_seq=4 ttl=63 time=15.2 ms
80 bytes from 10.0.2.1: icmp_seq=5 ttl=63 time=15.2 ms

--- 10.0.2.1 ping statistics ---
5 packets transmitted, 5 received, 0% packet loss, time 99ms
rtt min/avg/max/mdev = 15.232/18.918/30.501/5.833 ms, pipe 2, ipg/ewma 24.793/24.466 ms
```

#### **4.3 Проверка с leaf-1 до leaf-3 (10.0.3.1)**

```bash
leaf-1#ping 10.0.3.1 source 10.0.1.1
PING 10.0.3.1 (10.0.3.1) from 10.0.1.1 : 72(100) bytes of data.
80 bytes from 10.0.3.1: icmp_seq=1 ttl=63 time=39.3 ms
80 bytes from 10.0.3.1: icmp_seq=2 ttl=63 time=29.4 ms
80 bytes from 10.0.3.1: icmp_seq=3 ttl=63 time=27.2 ms
80 bytes from 10.0.3.1: icmp_seq=4 ttl=63 time=30.5 ms
80 bytes from 10.0.3.1: icmp_seq=5 ttl=63 time=20.7 ms

--- 10.0.3.1 ping statistics ---
5 packets transmitted, 5 received, 0% packet loss, time 99ms
rtt min/avg/max/mdev = 20.762/29.468/39.352/5.998 ms, pipe 3, ipg/ewma 24.858/34.077 ms
```

### **5. Проверка IS-IS интерфейсов на leaf-3**

```bash
leaf-3#show isis interface brief

IS-IS Instance: UNDERLAY VRF: default

Interface Level IPv4 Metric IPv6 Metric Type      Adjacency
--------- ----- ----------- ----------- --------- ---------
Ethernet1 L2             10          10 broadcast         1
Ethernet2 L2             10          10 broadcast         1
Loopback0 L2             10          10 loopback  (passive)
```

**Результат:** Все интерфейсы IS-IS находятся в рабочем состоянии, установлены соседства.

### **6. Проверка IS-IS базы данных на spine-1**

```bash
spine-1#show isis database

IS-IS Instance: UNDERLAY VRF: default
  IS-IS Level 2 Link State Database
    LSPID                   Seq Num  Cksum  Life Length IS Flags
    spine-1.00-00                11  34008 63315    147 L2 <>
    spine-1.0a-00                 1  20082 63250     51 L2 <>
    spine-2.00-00                 2  63073 63379    147 L2 <>
    spine-2.0a-00                 1  23140 63380     51 L2 <>
    leaf-1.00-00                  7  11564 63367    122 L2 <>
    leaf-1.08-00                  1  22124 62847     51 L2 <>
    leaf-1.09-00                  1  22632 63368     51 L2 <>
    leaf-2.00-00                  3  27877 63359    122 L2 <>
    leaf-3.00-00                  3  10777 63368    122 L2 <>
    leaf-3.0a-00                  1  23648 63327     51 L2 <>
    leaf-3.0b-00                  1  24156 63368     51 L2 <>
```

**Результат:** База данных IS-IS содержит информацию от всех пяти маршрутизаторов в топологии.

### **7. Детальная проверка ECMP**

#### **7.1 Проверка маршрута с ECMP**

```bash
leaf-1#show ip route 10.0.2.1

VRF: default
Codes: C - connected, S - static, K - kernel,
       O - OSPF, IA - OSPF inter area, E1 - OSPF external type 1,
       E2 - OSPF external type 2, N1 - OSPF NSSA external type 1,
       N2 - OSPF NSSA external type2, B - Other BGP Routes,
       B I - iBGP, B E - eBGP, R - RIP, I L1 - IS-IS level 1,
       I L2 - IS-IS level 2, O3 - OSPFv3, A B - BGP Aggregate,
       A O - OSPF Summary, NG - Nexthop Group Static Route,
       V - VXLAN Control Service, M - Martian,
       DH - DHCP client installed default route,
       DP - Dynamic Policy Route, L - VRF Leaked,
       G  - gRIBI, RC - Route Cache Route

 I L2     10.0.2.1/32 [115/30] via 10.1.1.0, Ethernet1
                               via 10.2.1.0, Ethernet2
```

**Результат:** Маршрут до leaf-2 имеет два равнозначных next-hop через оба spine коммутатора, что подтверждает работу ECMP.

#### **7.2 Трассировка маршрута**

```bash
leaf-1#traceroute 10.0.2.1 source 10.0.1.1
traceroute to 10.0.2.1 (10.0.2.1), 30 hops max, 60 byte packets
 1  10.1.1.0 (10.1.1.0)  45.430 ms  50.260 ms  62.335 ms
 2  10.0.2.1 (10.0.2.1)  73.341 ms  76.303 ms  86.759 ms
```

**Результат:** Traceroute показывает путь через spine-1. При реальной нагрузке трафик будет распределяться между обоими путями благодаря ECMP.

## **Практическое тестирование ECMP**

### **Тест: Проверка отказоустойчивости**

#### **Шаг 1: Отключение одного пути**
```bash
# На leaf-1 отключаем интерфейс к spine-1
leaf-1#configure terminal
leaf-1(config)#interface ethernet1
leaf-1(config-if-Et1)#shutdown
```

#### **Шаг 2: Проверка маршрутизации после отказа**
```bash
# Проверяем маршрут до leaf-2
leaf-1#show ip route 10.0.2.1
 I L2     10.0.2.1/32 [115/30] via 10.2.1.0, Ethernet2

# Проверяем связность
leaf-1#ping 10.0.2.1 source 10.0.1.1 count 5
```

#### **Шаг 3: Восстановление пути**
```bash
# Включаем интерфейс обратно
leaf-1(config-if-Et1)#no shutdown

# Проверяем восстановление ECMP
leaf-1#show ip route 10.0.2.1
 I L2     10.0.2.1/32 [115/30] via 10.1.1.0, Ethernet1
                               via 10.2.1.0, Ethernet2
```

## **Выводы по работе**

### **1. Результаты настройки IS-IS**

1. **Установлены IS-IS соседства:** Каждый spine коммутатор установил L2 соседство со всеми leaf коммутаторами
2. **Таблицы маршрутизации заполнены:** Все устройства узнали маршруты ко всем Loopback интерфейсам через IS-IS
3. **ECMP работает корректно:** Для трафика leaf-to-leaf наблюдаются два равнозначных пути через разные spine коммутаторы
4. **Связность обеспечена:** Успешная ping-проверка между всеми Loopback интерфейсами с 0% потерь пакетов

### **2. Особенности реализации ECMP в Clos-топологии**

#### **2.1 Распределение ECMP в текущей конфигурации:**
- **Leaf-to-leaf трафик (10.0.2.1, 10.0.3.1):** ✓ ECMP работает (два пути через spine-1 и spine-2)
- **Leaf-to-spine трафик (10.0.0.1, 10.0.0.2):** ✗ ECMP не применим (один прямой линк к каждому spine)

#### **2.2 Принцип работы:**
```
Leaf-1 → Leaf-2 пути:
  Путь A: Leaf-1 → Spine-1 → Leaf-2 (метрика 30)
  Путь B: Leaf-1 → Spine-2 → Leaf-2 (метрика 30)
  
Оба пути равнозначны → активирован ECMP
```

#### **2.3 Результаты проверки конкретного маршрута:**
```bash
leaf-1#show ip route 10.0.0.1
 I L2     10.0.0.1/32 [115/20] via 10.1.1.0, Ethernet1

leaf-1#show ip route 10.0.2.1
 I L2     10.0.2.1/32 [115/30] via 10.1.1.0, Ethernet1
                               via 10.2.1.0, Ethernet2
```

**Вывод:** ECMP работает только для меж-leaf маршрутов, что соответствует ожидаемому поведению в данной топологии.


### **3. Особенности реализации IS-IS в Clos-топологии**

#### **3.1 Конфигурация level-2 интерфейсов**
- Все spine-leaf интерфейсы настроены как `isis circuit-type level-2`
- Устраняет необходимость в L1/L2 смешивании для point-to-point линков
- Ускоряет установление соседства
- Упрощает диагностику и мониторинг

#### **3.2 Пассивные интерфейсы**
- Loopback интерфейсы настроены как `isis passive`
- Объявляют свои сети в IS-IS без установления соседств
- Уменьшают количество LSP в сети

#### **3.3 Single Level Design**
- Вся Underlay сеть находится в Level-2 домене
- Упрощает конфигурацию и диагностику
- Подходит для топологий среднего размера (до 50-100 узлов)

#### **3.4 System ID из NET адресов**
- Стабильные System ID, не зависящие от состояния физических интерфейсов
- Упрощают идентификацию устройств в выводах команд IS-IS
- Обеспечивают предсказуемость при перезагрузках

## **Типичные проблемы и их решение**

### **Проблема 1: IS-IS соседство не устанавливается**
**Причины:**
- Несовпадение уровня (L1/L2) на интерфейсах
- Несовпадение NET адреса (Area ID или System ID)
- ACL или firewall блокируют IS-IS пакеты
- Неправильная настройка типа интерфейса

**Решение:**
```bash
# Проверить конфигурацию интерфейсов
show isis interface detail

# Проверить соответствие NET адресов
show running-config | section router.isis

# Проверить сетевую связность
ping <neighbor-ip>
```

### **Проблема 2: Маршруты не появляются в таблице маршрутизации**
**Причины:**
- Интерфейс не включен в IS-IS (`isis enable` отсутствует)
- Пассивный интерфейс для нужной сети
- Проблемы с распространением LSP

**Решение:**
```bash
# Проверить IS-IS базу данных
show isis database

# Проверить конфигурацию IS-IS
show running-config | section isis

# Проверить, включен ли интерфейс в IS-IS
show running-config interfaces <interface-name>
```

### **Проблема 3: Нет балансировки нагрузки (ECMP)**
**Причины:**
- Разная стоимость интерфейсов
- Асимметричные пути
- Настройка maximum-paths

**Решение:**
```bash
# Проверить стоимость интерфейсов
show isis interface brief

# Проверить настройки ECMP
show running-config | include maximum-paths

# Проверить таблицу маршрутизации
show ip route <destination>
```

## **Заключение**

Настройка IS-IS в Underlay сети Clos-топологии успешно завершена. Достигнута полная IP-связанность между всеми устройствами с корректной работой ECMP для меж-leaf трафика.

**Ключевые достижения:**
- Все IS-IS соседства в состоянии UP
- Таблицы маршрутизации содержат оптимальные маршруты
- ECMP обеспечивает балансировку нагрузки для трафика между leaf коммутаторами

**Особенности реализации в Clos-топологии:**
- ECMP работает только для leaf-to-leaf трафика (что логично при одном прямом линке к каждому spine)
- Level-2 конфигурация интерфейсов упрощает управление и диагностику
- Single level design обеспечивает простоту и стабильность

**Готовность к следующему этапу:**
Underlay сеть с IS-IS успешно настроена и протестирована. Система готова к построению Overlay сетей и развертыванию сервисов поверх надежного Underlay. Следующим этапом может быть настройка BGP для overlay маршрутизации или реализация VXLAN для L2 расширения между leaf коммутаторами.

### **Дальнейшие шаги**
С настроенным IS-IS Underlay можно переходить к настройке:
1. **Overlay сетей** (например, VXLAN/EVPN)
2. **BGP для Overlay** (eBGP между spine и leaf)
3. **Сервисов доступа** (клиентские VLAN, VRF)
4. **Политик маршрутизации** между Overlay и Underlay
