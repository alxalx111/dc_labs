# Лабораторная работа: Настройка L3VNI для маршрутизации между VLAN в VXLAN EVPN

## **Цель работы**
Настроить L3 VNI (Layer 3 VXLAN Network Identifier) для обеспечения маршрутизации между клиентами в разных VLAN через VXLAN EVPN fabric с использованием Anycast Gateway.

### **Домашнее задание**
**Цель:** Настроить маршрутизацию в рамках Overlay между клиентами.

**Задачи:**
1) Настроить каждого клиента в своем VNI
2) Настроить маршрутизацию между клиентами
3) Зафиксировать в документации: план работы, адресное пространство, схему сети, конфигурацию устройств

## **Топология сети**
![Схема Clos-топологии с eBGP](lab.png)

### **Архитектура:**
- **2 spine коммутатора** (spine-1, spine-2) - AS 65500
- **3 leaf коммутатора:**
  - leaf-1 - AS 65501
  - leaf-2 - AS 65502  
  - leaf-3 - AS 65503
- **Клиенты:**
  - Client_1 - VLAN 100 (192.168.100.11/24) на leaf-1
  - Client_2 - VLAN 100 (192.168.100.12/24) на leaf-2
  - Client_3 - VLAN 100 (192.168.100.13/24) на leaf-3
  - Server_3 - VLAN 200 (192.168.200.13/24) на leaf-3

## **Схема IP-адресации**

### **Loopback интерфейсы (Router ID и VTEP):**
```
spine-1: 10.0.0.1/32        (AS 65500) - Router ID
spine-2: 10.0.0.2/32        (AS 65500) - Router ID

leaf-1: 10.0.1.1/32         (AS 65501) - Router ID
        10.0.1.11/32                   - VTEP Source

leaf-2: 10.0.2.1/32         (AS 65502) - Router ID  
        10.0.2.11/32                   - VTEP Source

leaf-3: 10.0.3.1/32         (AS 65503) - Router ID
        10.0.3.11/32                   - VTEP Source
```

### **Underlay сеть (Spine-Leaf линки):**
```
spine-1 <-> leaf-1: 10.1.1.0/31 (spine .0, leaf .1)
spine-1 <-> leaf-2: 10.1.2.0/31
spine-1 <-> leaf-3: 10.1.3.0/31

spine-2 <-> leaf-1: 10.2.1.0/31
spine-2 <-> leaf-2: 10.2.2.0/31
spine-2 <-> leaf-3: 10.2.3.0/31
```

### **Overlay сеть (Клиентские сети):**
```
VLAN 100 (CLIENTS): 192.168.100.0/24
  Anycast Gateway: 192.168.100.1/24 (на всех leaf в VRF TENANT_A)
  Клиенты:
    Client_1: 192.168.100.11/24 (на leaf-1)
    Client_2: 192.168.100.12/24 (на leaf-2)
    Client_3: 192.168.100.13/24 (на leaf-3)

VLAN 200 (SERVERS): 192.168.200.0/24
  Anycast Gateway: 192.168.200.1/24 (на leaf-1 и leaf-3 в VRF TENANT_A)
  Сервер:
    Server_3: 192.168.200.13/24 (на leaf-3)
```

### **VXLAN параметры:**
```
L2 VNI (MAC-VRF):
  VLAN 100 → VNI 10100
  VLAN 200 → VNI 10200

L3 VNI (IP-VRF):
  VRF TENANT_A → VNI 50001

VTEP адреса:
  leaf-1: 10.0.1.11
  leaf-2: 10.0.2.11  
  leaf-3: 10.0.3.11

Anycast Gateway MAC: 00:00:00:00:99:99
Flood VTEP списки (head-end replication):
  Все VTEP участвуют в BUM трафике
```

## **Параметры BGP EVPN**

### **Underlay BGP (IPv4):**
- **AS номера:** Spine: 65500, Leaf: 65501-65503
- **Тип:** eBGP
- **Назначение:** Связанность между Loopback адресами для Overlay

### **Overlay BGP (EVPN):**
- **Address Family:** l2vpn evpn
- **Тип:** eBGP мульти-хоп (через Loopback)
- **Назначение:** Обмен EVPN маршрутами для VXLAN
- **Типы маршрутов:** Type-2 (MAC/IP), Type-3 (Inclusive Multicast), Type-5 (IP Prefix)

### **EVPN Route-Target сообщества:**
```
VLAN 100: RT 65000:10100 (импорт/экспорт)
VLAN 200: RT 65000:10200 (импорт/экспорт)  
VRF TENANT_A: RT 65000:50001 (импорт/экспорт)
```

## **План настройки L3VNI**

### **1. Базовая настройка Underlay сети**
1. Настройка IP адресов на физических интерфейсах
2. Настройка Loopback интерфейсов для Router ID и VTEP
3. Настройка eBGP в address-family ipv4 для Underlay связности
4. Проверка IP связанности между всеми Loopback адресами

### **2. Настройка VXLAN с L3 VNI**
1. Создание интерфейса Vxlan1
2. Назначение source-interface (VTEP адреса)
3. Сопоставление VLAN к VNI (VLAN 100 → VNI 10100, VLAN 200 → VNI 10200)
4. Настройка L3 VNI для VRF (VNI 50001)
5. Конфигурация flood VTEP для BUM трафика

### **3. Настройка Anycast Gateway**
1. Настройка глобального виртуального MAC: `ip virtual-router mac-address 00:00:00:00:99:99`
2. Создание VRF TENANT_A с RD и VNI
3. Настройка SVI интерфейсов с Anycast Gateway IP (`ip address virtual`)
4. Активация IP маршрутизации в VRF

### **4. Настройка BGP EVPN для L3 маршрутизации**
1. Создание OVERLAY peer-group для eBGP мульти-хоп сессий
2. Активация address-family evpn
3. Настройка EVPN для каждого VLAN (L2 VNI) с Route Distinguisher и Route-Target
4. Настройка EVPN для VRF (L3 VNI) с redistribute connected
5. На spine: настройка route-reflector для EVPN маршрутов

### **5. Подключение клиентов и тестирование**
1. Настройка access портов в соответствующие VLAN
2. Проверка MAC learning и ARP таблицы в VRF
3. Тестирование L2 связанности внутри VLAN
4. Тестирование L3 маршрутизации между VLAN

## **Конфигурация устройств**

### **1. SPINE-1 конфигурация BGP EVPN**
```bash
! spine-1.cfg
router bgp 65500
   router-id 10.0.0.1
   neighbor 10.0.1.1 remote-as 65501
   neighbor 10.0.1.1 update-source Loopback0
   neighbor 10.0.1.1 allowas-in 1          # Разрешить свой AS в AS-PATH
   neighbor 10.0.1.1 ebgp-multihop 3       # eBGP через Loopback
   neighbor 10.0.1.1 route-reflector-client # RR для EVPN
   neighbor 10.0.1.1 send-community        # Отправлять community
   
   neighbor 10.0.2.1 remote-as 65502
   neighbor 10.0.2.1 update-source Loopback0
   neighbor 10.0.2.1 allowas-in 1
   neighbor 10.0.2.1 ebgp-multihop 3
   neighbor 10.0.2.1 route-reflector-client
   neighbor 10.0.2.1 send-community
   
   neighbor 10.0.3.1 remote-as 65503
   neighbor 10.0.3.1 update-source Loopback0
   neighbor 10.0.3.1 allowas-in 1
   neighbor 10.0.3.1 ebgp-multihop 3
   neighbor 10.0.3.1 route-reflector-client
   neighbor 10.0.3.1 send-community
   
   address-family evpn
      neighbor 10.0.1.1 activate
      neighbor 10.0.2.1 activate
      neighbor 10.0.3.1 activate
```

### **2. LEAF-1 конфигурация VXLAN EVPN с L3VNI**
```bash
! leaf-1.cfg - основные секции для L3VNI

! Настройка Anycast Gateway
ip virtual-router mac-address 00:00:00:00:99:99
ip routing vrf TENANT_A

! Настройка VRF с L3 VNI
vrf instance TENANT_A
   rd 10.0.1.11:50001

! Настройка SVI с Anycast Gateway
interface Vlan100
   description Anycast Gateway for VLAN 100
   vrf TENANT_A
   ip address virtual 192.168.100.1/24

interface Vlan200
   description Anycast Gateway for VLAN 200
   vrf TENANT_A
   ip address virtual 192.168.200.1/24

! Настройка VXLAN с L3 VNI
interface Vxlan1
   vxlan source-interface Loopback1
   vxlan udp-port 4789
   vxlan vlan 100 vni 10100
   vxlan vlan 200 vni 10200
   vxlan vrf TENANT_A vni 50001
   vxlan flood vtep 10.0.1.11 10.0.2.11 10.0.3.11

! Настройка BGP EVPN для L3 маршрутизации
router bgp 65501
   router-id 10.0.1.1
   
   ! Overlay peer-group для eBGP мульти-хоп
   neighbor OVERLAY peer group
   neighbor OVERLAY remote-as 65500
   neighbor OVERLAY update-source Loopback0
   neighbor OVERLAY ebgp-multihop 3
   neighbor OVERLAY send-community
   
   neighbor 10.0.0.1 peer group OVERLAY  # spine-1
   neighbor 10.0.0.2 peer group OVERLAY  # spine-2
   
   ! L2 EVPN сегменты (MAC-VRF)
   vlan 100
      rd 10.0.1.11:10100
      route-target import 65000:10100
      route-target export 65000:10100
      redistribute learned
   
   vlan 200
      rd 10.0.1.11:10200
      route-target import 65000:10200
      route-target export 65000:10200
      redistribute learned
   
   ! L3 EVPN сегмент (IP-VRF)
   vrf TENANT_A
      rd 10.0.1.11:50001
      route-target import evpn 65000:50001
      route-target export evpn 65000:50001
      redistribute connected
   
   ! Активация EVPN
   address-family evpn
      neighbor OVERLAY activate
```

### **3. LEAF-2 и LEAF-3 конфигурация**
Конфигурации leaf-2 и leaf-3 аналогичны leaf-1, но:
- Используют свои AS номера (65502, 65503)
- Используют свои VTEP адреса (10.0.2.11, 10.0.3.11)
- RD формируются на основе своих VTEP адресов
- На leaf-2 нет SVI для VLAN 200 (только VLAN 100)
- На leaf-3 есть SVI для обоих VLAN (100 и 200)

## **Архитектурные особенности реализации L3VNI**

### **1. Symmetric IRB (Integrated Routing and Bridging)**
В данной лабораторной работе реализована модель **Symmetric IRB**:
- Маршрутизация происходит как на ingress, так и на egress VTEP
- Используется единый L3 VNI (50001) для всех направлений трафика
- Все Leaf коммутаторы имеют полную L3 функциональность (Anycast Gateway)

### **2. Anycast Gateway преимущества:**
- **Единая точка входа/выхода:** Все клиенты используют один IP адрес шлюза
- **Оптимальность трафика:** Трафик идет напрямую внутри Leaf
- **Отказоустойчивость:** При выходе одного Leaf из строя, другие продолжают обслуживать шлюз

### **3. EVPN Type-5 маршруты:**
- Используются для распространения IP префиксов между VTEP
- Содержат информацию о L3 VNI (50001)
- Позволяют маршрутизировать трафик между разными подсетями через VXLAN
- Включают Router MAC для маршрутизации

### **4. Head-End Replication для BUM трафика:**
```
vxlan flood vtep 10.0.1.11 10.0.2.11 10.0.3.11
```
- Каждый VTEP реплицирует BUM трафик всем остальным VTEP
- Альтернатива: Underlay Multicast (не используется в данной lab)

### **5. Route Reflector на Spine:**
- Spine коммутаторы настроены как Route Reflector для EVPN маршрутов
- Упрощает BGP топологию (не нужен full-mesh между leaf)
- Стандартная практика в Clos-архитектуре


## **Диагностика и проверка L3VNI**

### **1. Проверка Anycast Gateway**

#### **На leaf-1:**
```bash
leaf-1#show running-config interface vlan100
interface Vlan100
   description Anycast Gateway for VLAN 100
   vrf TENANT_A
   ip address virtual 192.168.100.1/24

leaf-1#show running-config interface vlan200
interface Vlan200
   description Gateway for VLAN 200
   vrf TENANT_A
   ip address virtual 192.168.200.1/24
```

### **2. Проверка VRF и интерфейсов**

#### **На leaf-1:**
```bash
leaf-1#show vrf TENANT_A
   VRF            Protocols       State         Interfaces
-------------- --------------- ---------------- --------------------
   TENANT_A       IPv4            routing       Vl100, Vl200, Vl4094
   TENANT_A       IPv6            no routing    Vl4094
```

### **3. Проверка MAC таблицы с Anycast Gateway**

#### **На leaf-1:**
```bash
leaf-1#show mac address-table vlan 100
          Mac Address Table
------------------------------------------------------------------

Vlan    Mac Address       Type        Ports      Moves   Last Move
----    -----------       ----        -----      -----   ---------
 100    0000.0000.9999    STATIC      Cpu
 100    0050.7966.6806    DYNAMIC     Et3        1       0:02:19 ago
 100    0050.7966.6807    DYNAMIC     Vx1        1       0:00:40 ago
 100    0050.7966.6808    DYNAMIC     Vx1        1       0:00:48 ago
Total Mac Addresses for this criterion: 4

          Multicast Mac Address Table
------------------------------------------------------------------

Vlan    Mac Address       Type        Ports
----    -----------       ----        -----
Total Mac Addresses for this criterion: 0
```

```bash
leaf-1#show mac address-table vlan 200
          Mac Address Table
------------------------------------------------------------------

Vlan    Mac Address       Type        Ports      Moves   Last Move
----    -----------       ----        -----      -----   ---------
 200    0000.0000.9999    STATIC      Cpu
 200    0050.7966.6809    DYNAMIC     Vx1        1       0:02:02 ago
Total Mac Addresses for this criterion: 2

          Multicast Mac Address Table
------------------------------------------------------------------

Vlan    Mac Address       Type        Ports
----    -----------       ----        -----
Total Mac Addresses for this criterion: 0
```

**Вывод:** Anycast Gateway MAC `0000.0000.9999` присутствует в таблице MAC для обоих VLAN, клиенты видны локально и через VXLAN.

### **4. Проверка VXLAN VNI**

#### **На leaf-1:**
```bash
leaf-1#show vxlan vni
VNI to VLAN Mapping for Vxlan1
VNI         VLAN       Source       Interface       802.1Q Tag
----------- ---------- ------------ --------------- ----------
10100       100        static       Ethernet3       untagged
                                    Vxlan1          100
10200       200        static       Vxlan1          200

VNI to dynamic VLAN Mapping for Vxlan1
VNI         VLAN       VRF            Source
----------- ---------- -------------- ------------
50001       4094       TENANT_A       evpn
```

**Вывод:** Настроены L2 VNI (10100, 10200) и L3 VNI (50001) для VRF TENANT_A.

### **5. Проверка VXLAN туннелей**

#### **На leaf-1:**
```bash
leaf-1#show vxlan vtep
Remote VTEPS for Vxlan1:

VTEP            Tunnel Type(s)
--------------- --------------
10.0.1.11       flood
10.0.2.11       unicast, flood
10.0.3.11       unicast, flood

Total number of remote VTEPS:  3
```

**Вывод:** Все VTEP видят друг друга через flood и unicast туннели.

### **6. Проверка BGP EVPN сессий**

#### **На leaf-1:**
```bash
leaf-1#show bgp evpn summary
BGP summary information for VRF default
Router identifier 10.0.1.1, local AS number 65501
Neighbor Status Codes: m - Under maintenance
  Neighbor V AS           MsgRcvd   MsgSent  InQ OutQ  Up/Down State   PfxRcd PfxAcc
  10.0.0.1 4 65500         113983    114016    0    0 01:04:37 Estab   11     11
  10.0.0.2 4 65500         113995    114028    0    0 01:04:37 Estab   11     11
```

**Вывод:** BGP EVPN сессии установлены с обоими Spine, получено по 11 префиксов.

### **7. Проверка EVPN Type-5 маршрутов (IP Prefix)**

#### **На leaf-1:**
```bash
leaf-1#show bgp evpn route-type ip-prefix ipv4
BGP routing table information for VRF default
Router identifier 10.0.1.1, local AS number 65501

          Network                Next Hop              Metric  LocPref Weight  Path
 * >      RD: 10.0.1.11:50001 ip-prefix 192.168.100.0/24
                                 -                     -       -       0       i
 * >Ec    RD: 10.0.2.11:50001 ip-prefix 192.168.100.0/24
                                 10.0.2.11             -       100     0       65500 65502 i
 *  ec    RD: 10.0.2.11:50001 ip-prefix 192.168.100.0/24
                                 10.0.2.11             -       100     0       65500 65502 i
 * >Ec    RD: 10.0.3.11:50001 ip-prefix 192.168.100.0/24
                                 10.0.3.11             -       100     0       65500 65503 i
 *  ec    RD: 10.0.3.11:50001 ip-prefix 192.168.100.0/24
                                 10.0.3.11             -       100     0       65500 65503 i
 * >      RD: 10.0.1.11:50001 ip-prefix 192.168.200.0/24
                                 -                     -       -       0       i
 * >Ec    RD: 10.0.3.11:50001 ip-prefix 192.168.200.0/24
                                 10.0.3.11             -       100     0       65500 65503 i
 *  ec    RD: 10.0.3.11:50001 ip-prefix 192.168.200.0/24
                                 10.0.3.11             -       100     0       65500 65503 i
```

**Вывод:** Получены Type-5 маршруты для всех подсетей от всех Leaf с правильными next-hop (VTEP адреса).

### **8. Проверка маршрутизации в VRF**

#### **На leaf-1:**
```bash
leaf-1#show ip route vrf TENANT_A

VRF: TENANT_A
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

Gateway of last resort is not set

 B E      192.168.100.12/32 [20/0] via VTEP 10.0.2.11 VNI 50001 router-mac 50:00:00:03:37:66 local-interface Vxlan1
 B E      192.168.100.13/32 [20/0] via VTEP 10.0.3.11 VNI 50001 router-mac 50:00:00:d5:5d:c0 local-interface Vxlan1
 C        192.168.100.0/24 is directly connected, Vlan100
 B E      192.168.200.13/32 [20/0] via VTEP 10.0.3.11 VNI 50001 router-mac 50:00:00:d5:5d:c0 local-interface Vxlan1
 C        192.168.200.0/24 is directly connected, Vlan200

```

**Вывод:** VRF маршрутизация работает, видны BGP маршруты к удаленным хостам через VXLAN туннели.

### **9. Тестирование связанности клиентами**

#### **Внутри VLAN связность (L2 через VNI 10100):**
```bash
Client_1> ping 192.168.100.12
84 bytes from 192.168.100.12 icmp_seq=1 ttl=64 time=103.338 ms
84 bytes from 192.168.100.12 icmp_seq=2 ttl=64 time=61.873 ms

Client_1> ping 192.168.100.13  
84 bytes from 192.168.100.13 icmp_seq=1 ttl=64 time=84.921 ms
84 bytes from 192.168.100.13 icmp_seq=2 ttl=64 time=57.703 ms

Client_2> ping 192.168.100.13
84 bytes from 192.168.100.13 icmp_seq=1 ttl=64 time=104.366 ms
84 bytes from 192.168.100.13 icmp_seq=2 ttl=64 time=67.592 ms
```

**Вывод:** TTL=64 указывает на прямое L2 соединение через VNI 10100 без маршрутизации.

#### **Меж-VLAN маршрутизация (L3 через VNI 50001):**
```bash
Client_1> ping 192.168.200.13
84 bytes from 192.168.200.13 icmp_seq=1 ttl=62 time=84.175 ms
84 bytes from 192.168.200.13 icmp_seq=2 ttl=62 time=111.991 ms

Client_2> ping 192.168.200.13
84 bytes from 192.168.200.13 icmp_seq=1 ttl=62 time=85.146 ms
84 bytes from 192.168.200.13 icmp_seq=2 ttl=62 time=90.619 ms
```

**Вывод:** TTL=62 указывает на маршрутизацию через 2 хопа (Leaf → Leaf), что подтверждает работу L3 VNI 50001.

#### **Проверка Anycast Gateway на клиентах:**
```bash
Client_1> sh arp
00:00:00:00:99:99  192.168.100.1 expires in 116 seconds
```

**Вывод:** Клиент видит Anycast Gateway с правильным виртуальным MAC адресом.


### **10. Механизм L3 маршрутизации через VXLAN**

#### **Сценарий маршрутизации Client_1 → Server_3:**

1. **Client_1 отправляет пакет:** 
   - Назначение: `192.168.200.13` (Server_3)
   - Шлюз по умолчанию: `192.168.100.1` (Anycast Gateway)
   - DST MAC: `00:00:00:00:99:99` (виртуальный MAC Anycast Gateway)

2. **Leaf-1 получает пакет:**
   - Определяет что трафик нужно маршрутизировать из VLAN 100 в VLAN 200
   - Ищет маршрут в VRF TENANT_A: `192.168.200.13/32 via VTEP 10.0.3.11 VNI 50001`

3. **Leaf-1 инкапсулирует пакет в VXLAN:**
   ```
   Внешний заголовок (Underlay):
     SRC IP: 10.0.1.11 (VTEP leaf-1)
     DST IP: 10.0.3.11 (VTEP leaf-3)
     Протокол: VXLAN (порт 4789)
     VNI: 50001 (L3 VNI для VRF TENANT_A)
   
   Внутренний кадр (Overlay):
     SRC MAC: 00:00:00:00:99:99 (Anycast Gateway MAC)
     DST MAC: 50:00:00:d5:5d:c0 (Router MAC leaf-3)
     SRC IP: 192.168.100.11 (Client_1)
     DST IP: 192.168.200.13 (Server_3)
   ```

4. **Leaf-3 получает и обрабатывает пакет:**
   - Видит что DST MAC = его Router MAC (`50:00:00:d5:5d:c0`)
   - Деинкапсулирует VXLAN заголовок
   - Маршрутизирует пакет в VLAN 200
   - Отправляет пакет Server_3 с DST MAC = `00:50:79:66:68:09`

5. **Ответный трафик (Server_3 → Client_1):**
   - Server_3 отправляет ответ на `192.168.100.11`
   - Шлюз: `192.168.200.1` (Anycast Gateway)
   - Аналогичный процесс в обратном направлении через VNI 50001

#### **Ключевые компоненты:**
- **Anycast Gateway MAC (`00:00:00:00:99:99`):** Единый виртуальный MAC для всех клиентов
- **Router MAC (например `50:00:00:d5:5d:c0`):** Физический MAC VTEP интерфейса для L3 маршрутизации
- **L3 VNI 50001:** Используется исключительно для меж-VLAN трафика
- **Симметричная IRB:** Маршрутизация происходит на ingress и egress VTEP


## **Итоги диагностики**

1. **Anycast Gateway настроен:** ✅ Единый виртуальный MAC `0000.0000.9999` на всех Leaf
2. **L3 VNI работает:** ✅ VNI 50001 настроен для VRF TENANT_A
3. **VXLAN туннели установлены:** ✅ Все VTEP видят друг друга через unicast и flood туннели
4. **BGP EVPN работает:** ✅ Сессии установлены, Type-5 маршруты получены
5. **VRF маршрутизация работает:** ✅ Маршруты к удаленным сетям присутствуют в VRF
6. **L2 связность работает:** ✅ Клиенты внутри VLAN общаются с TTL=64
7. **L3 маршрутизация работает:** ✅ Клиенты между VLAN общаются с TTL=62 (маршрутизация)

## **Выводы лабораторной работы**

### **Достигнутые результаты:**
1. ✅ **Настроен Anycast Gateway** на всех Leaf коммутаторах с единым виртуальным MAC
2. ✅ **Реализован L3 VNI 50001** для маршрутизации между VLAN через VRF TENANT_A
3. ✅ **Настроена симметричная IRB модель** с оптимальной маршрутизацией трафика
4. ✅ **Обеспечена полная связность:**
   - Внутри VLAN (через L2 VNI 10100)
   - Между VLAN (через L3 VNI 50001)
5. ✅ **Достигнута отказоустойчивость** через Anycast Gateway

### **Архитектура успешно реализована:**
- **Underlay:** eBGP между Leaf и Spine для базовой связности
- **Overlay:** VXLAN с EVPN для L2/L3 виртуализации
- **L2 VNI:** 10100 для VLAN 100, 10200 для VLAN 200
- **L3 VNI:** 50001 для VRF TENANT_A (меж-VLAN маршрутизации)
- **Anycast Gateway:** Единые IP/MAC шлюзы на всех Leaf
