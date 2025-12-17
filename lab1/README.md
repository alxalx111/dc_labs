# Лабораторная работа: Настройка L3 Clos-топологии на Arista EOS

## **Часть 1: Проектирование адресного пространства и базовая настройка**

### **Цель работы**
Собрать схему CLOS топологии, распределить адресное пространство, выполнить базовую настройку интерфейсов и проверить локальную связность.

### **Топология сети**

![Схема Clos-топологии](lab.png)


### **Схема IP-адресации**

#### **Loopback интерфейсы (Router ID):**
- `spine-1`: 10.0.0.1/32
- `spine-2`: 10.0.0.2/32
- `leaf-1`: 10.0.1.1/32
- `leaf-2`: 10.0.2.1/32
- `leaf-3`: 10.0.3.1/32

#### **Spine-Leaf линки (/31):**
```
spine-1 <-> leaf-1: 10.1.1.0/31 (spine .0, leaf .1)
spine-1 <-> leaf-2: 10.1.2.0/31
spine-1 <-> leaf-3: 10.1.3.0/31

spine-2 <-> leaf-1: 10.2.1.0/31
spine-2 <-> leaf-2: 10.2.2.0/31
spine-2 <-> leaf-3: 10.2.3.0/31
```

#### **Клиентские сети (/24):**
- `leaf-1`: 192.168.1.0/24 (шлюз: 192.168.1.1)
- `leaf-2`: 192.168.2.0/24 (шлюз: 192.168.2.1)
- `leaf-3`: 192.168.3.0/24 (шлюз: 192.168.3.1)
- `leaf-3`: 192.168.4.0/24 (шлюз: 192.168.4.1)

#### **Управление (OOB):**
- `spine-1`: 10.255.0.1/24
- `spine-2`: 10.255.0.2/24
- `leaf-1`: 10.255.0.11/24
- `leaf-2`: 10.255.0.12/24
- `leaf-3`: 10.255.0.13/24

## **2. Конфигурация устройств**

### **2.1 SPINE-1 конфигурация**

```bash
! Полная конфигурация spine-1
hostname spine-1
no aaa root
service routing protocols model ribd
spanning-tree mode mstp

interface Ethernet1
   description to-leaf-1
   no switchport
   ip address 10.1.1.0/31

interface Ethernet2
   description to-leaf-2
   no switchport
   ip address 10.1.2.0/31

interface Ethernet3
   description to-leaf-3
   no switchport
   ip address 10.1.3.0/31

interface Loopback0
   ip address 10.0.0.1/32

interface Management1
   ip address 10.255.0.1/24

ip routing
```

### **2.2 SPINE-2 конфигурация**

```bash
! Полная конфигурация spine-2
hostname spine-2
no aaa root
service routing protocols model ribd
spanning-tree mode mstp

interface Ethernet1
   description to-leaf-1
   no switchport
   ip address 10.2.1.0/31

interface Ethernet2
   description to-leaf-2
   no switchport
   ip address 10.2.2.0/31

interface Ethernet3
   description to-leaf-3
   no switchport
   ip address 10.2.3.0/31

interface Loopback0
   ip address 10.0.0.2/32

interface Management1
   ip address 10.255.0.2/24

ip routing
```

### **2.3 LEAF-1 конфигурация**

```bash
! Полная конфигурация leaf-1
hostname leaf-1
no aaa root
service routing protocols model ribd
spanning-tree mode mstp

interface Ethernet1
   description to-spine-1
   no switchport
   ip address 10.1.1.1/31

interface Ethernet2
   description to-spine-2
   no switchport
   ip address 10.2.1.1/31

interface Ethernet3
   description to-client-1
   no switchport
   ip address 192.168.1.1/24

interface Loopback0
   ip address 10.0.1.1/32

interface Management1
   ip address 10.255.0.11/24

ip routing
```

### **2.4 LEAF-2 конфигурация**

```bash
! Полная конфигурация leaf-2
hostname leaf-2
no aaa root
service routing protocols model ribd
spanning-tree mode mstp

interface Ethernet1
   description to-spine-1
   no switchport
   ip address 10.1.2.1/31

interface Ethernet2
   description to-spine-2
   no switchport
   ip address 10.2.2.1/31

interface Ethernet3
   description to-client-2
   no switchport
   ip address 192.168.2.1/24

interface Loopback0
   ip address 10.0.2.1/32

interface Management1
   ip address 10.255.0.12/24

ip routing
```

### **2.5 LEAF-3 конфигурация**

```bash
! Полная конфигурация leaf-3
hostname leaf-3
no aaa root
service routing protocols model ribd
spanning-tree mode mstp

interface Ethernet1
   description to-spine-1
   no switchport
   ip address 10.1.3.1/31

interface Ethernet2
   description to-spine-2
   no switchport
   ip address 10.2.3.1/31

interface Ethernet3
   description to-client-3
   no switchport
   ip address 192.168.3.1/24

interface Ethernet4
   description to-client-4
   no switchport
   ip address 192.168.4.1/24

interface Loopback0
   ip address 10.0.3.1/32

interface Management1
   ip address 10.255.0.13/24

ip routing
```

### **2.6 Конфигурация клиентских устройств**

#### **Client-1 (подключен к leaf-1, Eth3):**
```
IP адрес: 192.168.1.10/24
Шлюз: 192.168.1.1
```

#### **Client-2 (подключен к leaf-2, Eth3):**
```
IP адрес: 192.168.2.10/24
Шлюз: 192.168.2.1
```

#### **Client-3 (подключен к leaf-3, Eth3):**
```
IP адрес: 192.168.3.10/24
Шлюз: 192.168.3.1
```

#### **Client-4 (подключен к leaf-3, Eth4):**
```
IP адрес: 192.168.4.10/24
Шлюз: 192.168.4.1
```

## **3. Диагностика и проверка связности**

### **3.1 Проверка локальной связности leaf-1 с клиентом 1**

```bash
leaf-1#ping 192.168.1.10
PING 192.168.1.10 (192.168.1.10) 72(100) bytes of data.
80 bytes from 192.168.1.10: icmp_seq=1 ttl=64 time=11.8 ms
80 bytes from 192.168.1.10: icmp_seq=2 ttl=64 time=6.80 ms
80 bytes from 192.168.1.10: icmp_seq=3 ttl=64 time=5.48 ms
80 bytes from 192.168.1.10: icmp_seq=4 ttl=64 time=6.32 ms
80 bytes from 192.168.1.10: icmp_seq=5 ttl=64 time=4.70 ms

--- 192.168.1.10 ping statistics ---
5 packets transmitted, 5 received, 0% packet loss, time 51ms
rtt min/avg/max/mdev = 4.702/7.024/11.806/2.497 ms, ipg/ewma 12.874/9.294 ms
```

**Результат:** Успешная связность, среднее время отклика 7.024 мс.

### **3.2 Проверка локальной связности leaf-2 с клиентом 2**

```bash
leaf-2#ping 192.168.2.10
PING 192.168.2.10 (192.168.2.10) 72(100) bytes of data.
80 bytes from 192.168.2.10: icmp_seq=1 ttl=64 time=10.5 ms
80 bytes from 192.168.2.10: icmp_seq=2 ttl=64 time=5.97 ms
80 bytes from 192.168.2.10: icmp_seq=3 ttl=64 time=6.00 ms
80 bytes from 192.168.2.10: icmp_seq=4 ttl=64 time=13.1 ms
80 bytes from 192.168.2.10: icmp_seq=5 ttl=64 time=11.6 ms

--- 192.168.2.10 ping statistics ---
5 packets transmitted, 5 received, 0% packet loss, time 46ms
rtt min/avg/max/mdev = 5.976/9.468/13.162/2.961 ms, pipe 2, ipg/ewma 11.511/10.143 ms
```

**Результат:** Успешная связность, среднее время отклика 9.468 мс.

### **3.3 Проверка локальной связности leaf-3 с клиентом 3**

```bash
leaf-3#ping 192.168.3.10
PING 192.168.3.10 (192.168.3.10) 72(100) bytes of data.
80 bytes from 192.168.3.10: icmp_seq=1 ttl=64 time=8.74 ms
80 bytes from 192.168.3.10: icmp_seq=2 ttl=64 time=4.89 ms
80 bytes from 192.168.3.10: icmp_seq=3 ttl=64 time=4.29 ms
80 bytes from 192.168.3.10: icmp_seq=4 ttl=64 time=9.60 ms
80 bytes from 192.168.3.10: icmp_seq=5 ttl=64 time=5.46 ms

--- 192.168.3.10 ping statistics ---
5 packets transmitted, 5 received, 0% packet loss, time 46ms
rtt min/avg/max/mdev = 4.290/6.597/9.601/2.152 ms, ipg/ewma 11.648/7.676 ms
```

**Результат:** Успешная связность, среднее время отклика 6.597 мс.

### **3.4 Проверка локальной связности leaf-3 с клиентом 4**

```bash
leaf-3#ping 192.168.4.10
PING 192.168.4.10 (192.168.4.10) 72(100) bytes of data.
80 bytes from 192.168.4.10: icmp_seq=1 ttl=64 time=6.16 ms
80 bytes from 192.168.4.10: icmp_seq=2 ttl=64 time=4.27 ms
80 bytes from 192.168.4.10: icmp_seq=3 ttl=64 time=8.97 ms
80 bytes from 192.168.4.10: icmp_seq=4 ttl=64 time=9.79 ms
80 bytes from 192.168.4.10: icmp_seq=5 ttl=64 time=9.26 ms

--- 192.168.4.10 ping statistics ---
5 packets transmitted, 5 received, 0% packet loss, time 36ms
rtt min/avg/max/mdev = 4.271/7.693/9.790/2.124 ms, ipg/ewma 9.222/7.058 ms
```

**Результат:** Успешная связность, среднее время отклика 7.693 мс.

## **4. Выводы по части 1**

### **4.1 Результаты диагностики**
1. **Все клиентские устройства доступны** со своих leaf коммутаторов:
   - Client-1 (192.168.1.10) доступен с leaf-1
   - Client-2 (192.168.2.10) доступен с leaf-2  
   - Client-3 (192.168.3.10) доступен с leaf-3
   - Client-4 (192.168.4.10) доступен с leaf-3

2. **Качество связи:** Все проверки ping показывают 0% потерь пакетов
3. **Время отклика:** В пределах нормы (4-13 мс) для лабораторной среды

### **4.2 Состояние конфигураций**
1. **IP маршрутизация включена** на всех устройствах
2. **Loopback интерфейсы** настроены с правильными адресами
3. **Интерфейсы spine-leaf** настроены с сетями /31
4. **Клиентские интерфейсы** настроены с сетями /24
5. **Сеть управления** настроена и доступна

### **4.3 Готовность к следующему этапу**
Топология успешно собрана и настроена. Все локальные соединения работоспособны. Система готова к настройке динамической маршрутизации OSPF для обеспечения связности между всеми сегментами сети.

## **5. Особенности реализации топологии**

1. **Зачем нужны Loopback интерфейсы в данной конфигурации?**
   - Обеспечивают стабильный адрес для управления устройством независимо от состояния физических интерфейсов
   - Используются как Router ID для протоколов динамической маршрутизации (в частности, для OSPF)
   - Упрощают диагностику и мониторинг сети через стабильные точки доступа

2. **Каковы преимущества использования сетей /31 для spine-leaf линков?**
   - Минимизация потерь IP-адресного пространства (используется только 2 адреса на каждый point-to-point линк)
   - Соответствие стандарту RFC 3021, который разрешает использование масок /31 для сетей с двумя узлами
   - Упрощение конфигурации и устранение неиспользуемых broadcast адресов
   - Улучшение безопасности за счет исключения широковещательных адресов

3. **Почему выбрана именно Clos-топология?**
   - Обеспечивает высокую степень отказоустойчивости за счет множественных путей
   - Позволяет горизонтальное масштабирование (добавление новых spine и leaf коммутаторов)
   - Предсказуемая и постоянная задержка между любыми узлами сети
   - Упрощает управление и диагностику благодаря регулярной структуре

4. **Какие особенности реализации L3 на leaf коммутаторах?**
   - Каждый leaf работает как маршрутизатор для своих клиентских сетей
   - Клиентские сети сегментированы на уровне L3, что повышает безопасность и управляемость
   - Все межсегментные коммуникации проходят через spine коммутаторы
   - Обеспечивается изоляция отказов на уровне отдельных leaf

5. **Почему используется OOB (Out-of-Band) сеть управления?**
   - Обеспечивает доступ к устройствам даже при проблемах в рабочей (in-band) сети
   - Позволяет разделить трафик управления и пользовательский трафик
   - Упрощает устранение неисправностей и проведение работ по обслуживанию
   - Повышает безопасность за счет физического разделения сетей

6. **Как обеспечивается отказоустойчивость в данной архитектуре?**
   - Каждый leaf подключен к обоим spine коммутаторам
   - Использование ECMP (Equal-Cost Multi-Path) для балансировки нагрузки
   - Автоматическое перераспределение трафика при отказе одного из путей
   - Отсутствие единых точек отказа на уровне spine-уровня

## **6. Следующий этап**

В следующей части лабораторной работы будет настроен протокол динамической маршрутизации OSPF для обеспечения:
1. Связности между всеми leaf коммутаторами через spine
2. Маршрутизации между клиентскими сетями
3. Отказоустойчивости через ECMP (Equal-Cost Multi-Path)
4. Оптимальной маршрутизации в Clos