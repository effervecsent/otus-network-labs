### Underlay.OSPF

Цель:
настроить OSPF для Underlay сети.

В этой самостоятельной работе:

- был настроен OSPF в Underlay сети для IP связанности между всеми сетевыми устройствами.
- выделено адресное пространство
- подготовлена схема сети 
- сконфигурированы 2 spine, 3 leaf коммутатора 
- проверена ip связность между всеми устройствами в OSPF-домене


### Схема 

![lab02-01.PNG](lab02-01.PNG)

Была выбрана следующая адресация: 

loopback-адреса 

Spine-01 10.0.1.1/32
Spine-02 10.0.2.2/32
Leaf-01 10.0.0.1/32
Leaf-02 10.0.0.2/32
Leaf-03 10.0.0.3/32

Топология p2p связей. В этой работе использовала /30 адреса для более легкого траблшутинга.

 ###Связи между SPINE-01 и LEAF-коммутаторами

 
 SPINE-01 ↔ LEAF-01
 Сеть: 172.16.1.0/30
 SPINE-01 (интерфейс Eth1): 172.16.1.2
 LEAF-01 (интерфейс Eth1): 172.16.1.1

 
 SPINE-01 ↔ LEAF-02
 Сеть: 172.16.2.0/30
 SPINE-01 (интерфейс Eth2): 172.16.2.2
 LEAF-02 (интерфейс Eth1): 172.16.2.1

 
 SPINE-01 ↔ LEAF-03
 Сеть: 172.16.3.0/30
 SPINE-01 (интерфейс Eth3): 172.16.3.2
 LEAF-03 (интерфейс Eth1): 172.16.3.1
 
 
 ###Связи между SPINE-02 и LEAF-коммутаторами
 SPINE-02 ↔ LEAF-01
 Сеть: 172.16.1.4/30
 SPINE-02 (интерфейс Eth1): 172.16.1.6
 LEAF-01 (интерфейс Eth2): 172.16.1.5

 
 SPINE-02 ↔ LEAF-02
 Сеть: 172.16.2.4/30
 SPINE-02 (интерфейс Eth2): 172.16.2.6
 LEAF-02 (интерфейс Eth2): 172.16.2.5

 
 SPINE-02 ↔ LEAF-03
 Сеть: 172.16.3.4/30
 SPINE-02 (интерфейс Eth3): 172.16.3.6
 LEAF-03 (интерфейс Eth2): 172.16.3.5


|Устройство|Интерфейс|IP-адрес и Маска|Тип подключения|Назначение 
|---|---|---|---|---|
SPINE-01|lo0|10.0.1.1/32|Loopback Локальный Router ID
SPINE-01|Eth1|172.16.1.2/30|P2P Линк|LEAF-01 (Eth1)
SPINE-01|Eth2|172.16.2.2/30|P2P Линк|LEAF-02 (Eth1)
SPINE-01|Eth3|172.16.3.2/30|P2P Линк|LEAF-03 (Eth1)|
SPINE-02|lo0|10.0.2.2/32|Loopback Локальный Router ID|
SPINE-02|Eth1|172.16.1.6/30|P2P Линк|LEAF-01 (Eth2)
SPINE-02|Eth2|172.16.2.6/30|P2P Линк|LEAF-02 (Eth2)|
SPINE-02|Eth3|172.16.3.6/30|P2P Линк|LEAF-03 (Eth2)
LEAF-01|lo0|10.0.0.1/32|Loopback Локальный Router ID
LEAF-01|Eth1|172.16.1.1/30|P2P Линк|SPINE-01 (Eth1)
LEAF-01|Eth2|172.16.1.5/30|P2P Линк|SPINE-02 (Eth1)
LEAF-02|lo0|10.0.0.2/32|Loopback Локальный Router ID
LEAF-02|Eth1|172.16.2.1/30|P2P Линк|SPINE-01 (Eth2)
LEAF-02|Eth2|172.16.2.5/30|P2P Линк|SPINE-02 (Eth2)
LEAF-03|lo0|10.0.0.3/32|Loopback Локальный Router ID
LEAF-03|Eth1|172.16.3.1/30|P2P Линк|SPINE-01 (Eth3)
LEAF-03|Eth2|172.16.3.5/30|P2P Линк|SPINE-02 (Eth3)


### Конфигурация OSPF 

Все коммутаторы находятся в зоне Area 0.


Настройки на коммутаторах на примере Spine-01: 

```
SPINE-01(config)#router ospf 1

Инициализирует глобальный процесс маршрутизации OSPF.
Число 1 — это локальный идентификатор процесса (Process ID).
Он имеет значение только в рамках этого коммутатора и не обязательно должен совпадать на соседних устройствах.


SPINE-01(config-router-ospf)#router-id 10.0.1.1

Устанавливает уникальный идентификатор маршрутизатора (Router ID) в виде IPv4-адреса.
OSPF использует его для однозначного определения этого коммутатора в топологии (LSDB).
Назначен IP-адрес интерфейса Loopback 0.



SPINE-01(config)#interface eth 1-3
SPINE-01(config-if-Et1-3)#ip ospf area 0

Включает OSPF непосредственно на выбранных интерфейсах и относит их к магистральной зоне Area 0 (Backbone).
Все P2P-линки Spine-Leaf фабрики находятся в одной зоне ( Area 0) для обеспечения связности.

SPINE-01(config-if-Et1-3)#ip ospf network point-to-point
Принудительно меняет тип сети OSPF на Point-to-Point.
По умолчанию на Ethernet-интерфейсах OSPF пытается выбрать DR (Выделенный маршрутизатор) и BDR, что занимает до 40 секунд.
Режим P2P отключает этот механизм, ускоряя установление соседства и оптимизируя обмен маршрутами на прямых стыках.



SPINE-01(config)#inter loopback 0
SPINE-01(config-if-Lo0)#ip ospf area 0

Включает OSPF на интерфейсе Loopback 0 и помещает его в Area 0.


```

### Проверка связности 

```
SPINE-01#sh ip ospf ne
Neighbor ID     Instance VRF      Pri State                  Dead Time   Address         Interface
10.0.0.1        1        default  1   FULL                   00:00:34    172.16.1.1      Ethernet1
10.0.0.2        1        default  1   FULL                   00:00:31    172.16.2.1      Ethernet2
10.0.0.3        1        default  1   FULL                   00:00:33    172.16.3.1      Ethernet3

SPINE-01#sh ip route ospf 

 O        10.0.0.1/32 [110/20] via 172.16.1.1, Ethernet1
 O        10.0.0.2/32 [110/20] via 172.16.2.1, Ethernet2
 O        10.0.0.3/32 [110/20] via 172.16.3.1, Ethernet3
 O        10.0.2.2/32 [110/30] via 172.16.1.1, Ethernet1
                               via 172.16.2.1, Ethernet2
                               via 172.16.3.1, Ethernet3
 O        172.16.1.4/30 [110/20] via 172.16.1.1, Ethernet1
 O        172.16.2.4/30 [110/20] via 172.16.2.1, Ethernet2
 O        172.16.3.4/30 [110/20] via 172.16.3.1, Ethernet3

```

```
SPINE-02#sh ip ospf nei
Neighbor ID     Instance VRF      Pri State                  Dead Time   Address         Interface
10.0.0.1        1        default  1   FULL                   00:00:32    172.16.1.5      Ethernet1
10.0.0.2        1        default  1   FULL                   00:00:34    172.16.2.5      Ethernet2
10.0.0.3        1        default  1   FULL                   00:00:32    172.16.3.5      Ethernet3
SPINE-02#


SPINE-02#sh ip rou ospf

 O        10.0.0.2/32 [110/20] via 172.16.2.5, Ethernet2
 O        10.0.0.3/32 [110/20] via 172.16.3.5, Ethernet3
 O        10.0.1.1/32 [110/30] via 172.16.1.5, Ethernet1
                               via 172.16.2.5, Ethernet2
                               via 172.16.3.5, Ethernet3
 O        172.16.1.0/30 [110/20] via 172.16.1.5, Ethernet1
 O        172.16.2.0/30 [110/20] via 172.16.2.5, Ethernet2
 O        172.16.3.0/30 [110/20] via 172.16.3.5, Ethernet3

SPINE-02#
```


```
LEAF-01#sh ip rou ospf
 O        10.0.1.1/32 [110/20] via 172.16.1.2, Ethernet1
 O        10.0.2.2/32 [110/20] via 172.16.1.6, Ethernet2
 O        172.16.2.0/30 [110/20] via 172.16.1.2, Ethernet1
 O        172.16.2.4/30 [110/20] via 172.16.1.6, Ethernet2
 O        172.16.3.0/30 [110/20] via 172.16.1.2, Ethernet1
 O        172.16.3.4/30 [110/20] via 172.16.1.6, Ethernet2


LEAF-01#sh ip ospf nei
Neighbor ID     Instance VRF      Pri State                  Dead Time   Address         Interface
10.0.1.1        1        default  0   FULL                   00:00:37    172.16.1.2      Ethernet1
10.0.2.2        1        default  0   FULL                   00:00:30    172.16.1.6      Ethernet2
LEAF-01#

```

```
LEAF-02#sh ip rou ospf 

 O        10.0.0.1/32 [110/30] via 172.16.2.2, Ethernet1
                               via 172.16.2.6, Ethernet2
 O        10.0.0.3/32 [110/30] via 172.16.2.2, Ethernet1
                               via 172.16.2.6, Ethernet2
 O        10.0.1.1/32 [110/20] via 172.16.2.2, Ethernet1
 O        10.0.2.2/32 [110/20] via 172.16.2.6, Ethernet2
 O        172.16.1.0/30 [110/20] via 172.16.2.2, Ethernet1
 O        172.16.1.4/30 [110/20] via 172.16.2.6, Ethernet2
 O        172.16.3.0/30 [110/20] via 172.16.2.2, Ethernet1
 O        172.16.3.4/30 [110/20] via 172.16.2.6, Ethernet2

LEAF-02#sh ip ospf nei
Neighbor ID     Instance VRF      Pri State                  Dead Time   Address         Interface
10.0.1.1        1        default  0   FULL                   00:00:36    172.16.2.2      Ethernet1
10.0.2.2        1        default  0   FULL                   00:00:34    172.16.2.6      Ethernet2
```


```
LEAF-03#sh ip route ospf

 O        10.0.0.1/32 [110/30] via 172.16.3.2, Ethernet1
                               via 172.16.3.6, Ethernet2
 O        10.0.0.2/32 [110/30] via 172.16.3.2, Ethernet1
                               via 172.16.3.6, Ethernet2
 O        10.0.1.1/32 [110/20] via 172.16.3.2, Ethernet1
 O        10.0.2.2/32 [110/20] via 172.16.3.6, Ethernet2
 O        172.16.1.0/30 [110/20] via 172.16.3.2, Ethernet1
 O        172.16.1.4/30 [110/20] via 172.16.3.6, Ethernet2
 O        172.16.2.0/30 [110/20] via 172.16.3.2, Ethernet1
 O        172.16.2.4/30 [110/20] via 172.16.3.6, Ethernet2

LEAF-03#sh ip ospf nei
Neighbor ID     Instance VRF      Pri State                  Dead Time   Address         Interface
10.0.1.1        1        default  0   FULL                   00:00:35    172.16.3.2      Ethernet1
10.0.2.2        1        default  0   FULL                   00:00:29    172.16.3.6      Ethernet2
```


Ping удаленных loopback-интерфейсов:
```
LEAF-01#
LEAF-01#ping 10.0.0.3
PING 10.0.0.3 (10.0.0.3) 72(100) bytes of data.
80 bytes from 10.0.0.3: icmp_seq=1 ttl=63 time=29.9 ms
80 bytes from 10.0.0.3: icmp_seq=2 ttl=63 time=21.0 ms
80 bytes from 10.0.0.3: icmp_seq=3 ttl=63 time=19.8 ms
80 bytes from 10.0.0.3: icmp_seq=4 ttl=63 time=21.8 ms
80 bytes from 10.0.0.3: icmp_seq=5 ttl=63 time=19.6 ms

--- 10.0.0.3 ping statistics ---
5 packets transmitted, 5 received, 0% packet loss, time 97ms
rtt min/avg/max/mdev = 19.617/22.467/29.940/3.832 ms, pipe 2, ipg/ewma 24.257/26.058 ms
LEAF-01#
```

```
LEAF-03#
LEAF-03#ping 10.0.0.1
PING 10.0.0.1 (10.0.0.1) 72(100) bytes of data.
80 bytes from 10.0.0.1: icmp_seq=1 ttl=63 time=143 ms
80 bytes from 10.0.0.1: icmp_seq=2 ttl=63 time=138 ms
80 bytes from 10.0.0.1: icmp_seq=3 ttl=63 time=136 ms
80 bytes from 10.0.0.1: icmp_seq=4 ttl=63 time=132 ms
80 bytes from 10.0.0.1: icmp_seq=5 ttl=63 time=127 ms

--- 10.0.0.1 ping statistics ---
5 packets transmitted, 5 received, 0% packet loss, time 52ms
rtt min/avg/max/mdev = 127.135/135.442/143.030/5.410 ms, pipe 5, ipg/ewma 13.209/138.842 ms
LEAF-03#ping 10.0.0.2
PING 10.0.0.2 (10.0.0.2) 72(100) bytes of data.
80 bytes from 10.0.0.2: icmp_seq=1 ttl=63 time=45.6 ms
80 bytes from 10.0.0.2: icmp_seq=2 ttl=63 time=41.8 ms
80 bytes from 10.0.0.2: icmp_seq=3 ttl=63 time=37.5 ms
80 bytes from 10.0.0.2: icmp_seq=4 ttl=63 time=31.9 ms
80 bytes from 10.0.0.2: icmp_seq=5 ttl=63 time=17.7 ms

--- 10.0.0.2 ping statistics ---
5 packets transmitted, 5 received, 0% packet loss, time 80ms
rtt min/avg/max/mdev = 17.793/34.967/45.662/9.721 ms, pipe 4, ipg/ewma 20.005/39.586 ms
```


Cостояние LSDB на всех коммутаторах: 

```
LEAF-03#

LEAF-02#
LEAF-02#sh ip ospf data

            OSPF Router with ID(10.0.0.2) (Instance ID 1) (VRF default)


                 Router Link States (Area 0.0.0.0)

Link ID         ADV Router      Age         Seq#         Checksum Link count
10.0.1.1        10.0.1.1        1059        0x80000034   0xd9c7   7
10.0.2.2        10.0.2.2        1085        0x80000035   0x4d35   7
10.0.0.1        10.0.0.1        1843        0x8000002d   0x78e9   5
10.0.0.2        10.0.0.2        1105        0x80000031   0xc98d   5
10.0.0.3        10.0.0.3        1059        0x80000031   0x68e7   5
LEAF-02#



LEAF-03#sh ip ospf data 

            OSPF Router with ID(10.0.0.3) (Instance ID 1) (VRF default)


                 Router Link States (Area 0.0.0.0)

Link ID         ADV Router      Age         Seq#         Checksum Link count
10.0.1.1        10.0.1.1        921         0x80000034   0xd9c7   7
10.0.2.2        10.0.2.2        947         0x80000035   0x4d35   7
10.0.0.2        10.0.0.2        969         0x80000031   0xc98d   5
10.0.0.1        10.0.0.1        1706        0x8000002d   0x78e9   5
10.0.0.3        10.0.0.3        920         0x80000031   0x68e7   5
LEAF-03#

```

```
SPINE-01#sh ip ospf data

            OSPF Router with ID(10.0.1.1) (Instance ID 1) (VRF default)


                 Router Link States (Area 0.0.0.0)

Link ID         ADV Router      Age         Seq#         Checksum Link count
10.0.0.1        10.0.0.1        1736        0x8000002d   0x78e9   5
10.0.2.2        10.0.2.2        979         0x80000035   0x4d35   7
10.0.0.2        10.0.0.2        1000        0x80000031   0xc98d   5
10.0.0.3        10.0.0.3        952         0x80000031   0x68e7   5
10.0.1.1        10.0.1.1        951         0x80000034   0xd9c7   7
SPINE-01#

SPINE-02#sh ip ospf data

            OSPF Router with ID(10.0.2.2) (Instance ID 1) (VRF default)


                 Router Link States (Area 0.0.0.0)

Link ID         ADV Router      Age         Seq#         Checksum Link count
10.0.1.1        10.0.1.1        975         0x80000034   0xd9c7   7
10.0.0.1        10.0.0.1        1757        0x8000002d   0x78e9   5
10.0.0.2        10.0.0.2        1021        0x80000031   0xc98d   5
10.0.0.3        10.0.0.3        974         0x80000031   0x68e7   5
10.0.2.2        10.0.2.2        999         0x80000035   0x4d35   7
SPINE-02#

LEAF-01#sh ip ospf data

            OSPF Router with ID(10.0.0.1) (Instance ID 1) (VRF default)


                 Router Link States (Area 0.0.0.0)

Link ID         ADV Router      Age         Seq#         Checksum Link count
10.0.1.1        10.0.1.1        1031        0x80000034   0xd9c7   7
10.0.2.2        10.0.2.2        1057        0x80000035   0x4d35   7
10.0.0.2        10.0.0.2        1079        0x80000031   0xc98d   5
10.0.0.3        10.0.0.3        1031        0x80000031   0x68e7   5
10.0.0.1        10.0.0.1        1813        0x8000002d   0x78e9   5
LEAF-01#

LEAF-02#
LEAF-02#sh ip ospf data

            OSPF Router with ID(10.0.0.2) (Instance ID 1) (VRF default)


                 Router Link States (Area 0.0.0.0)

Link ID         ADV Router      Age         Seq#         Checksum Link count
10.0.1.1        10.0.1.1        1059        0x80000034   0xd9c7   7
10.0.2.2        10.0.2.2        1085        0x80000035   0x4d35   7
10.0.0.1        10.0.0.1        1843        0x8000002d   0x78e9   5
10.0.0.2        10.0.0.2        1105        0x80000031   0xc98d   5
10.0.0.3        10.0.0.3        1059        0x80000031   0x68e7   5
LEAF-02#
```


### Настройка bfd

Между Spine-01 eth1 и Leaf-01 eth1 настроен BFD:
```
SPINE-01(config)#inter eth1 
SPINE-01(config-if-Et1)#bfd interval 200 min-rx 200 multiplier 3
SPINE-01(config-if-Et1)#ip ospf neighbor bfd
SPINE-01(config-if-Et1)#exit


LEAF-01(config)#inter eth1 
LEAF-01(config-if-Et1)#bfd interval 200  min-rx 200 multiplier 3
LEAF-01(config-if-Et1)#ip ospf neighbor bfd
```

Проверка:

```
SPINE-01#show bfd peers
VRF name: default
-----------------
DstAddr        MyDisc    YourDisc  Interface/Transport    Type          LastUp 
---------- ----------- ----------- -------------------- ------- ---------------
172.16.1.1 4267533140  2397352423        Ethernet1(14)  normal  08/19/26 08:46 

   LastDown            LastDiag    State
-------------- ------------------- -----
         NA       No Diagnostic       Up



SPINE-01#show bfd peers det
VRF name: default
-----------------
Peer Addr 172.16.1.1, Intf Ethernet1, Type normal, Role active, State Up
VRF default, LAddr 172.16.1.2, LD/RD 3021997832/369354112
Session state is Up and not using echo function
Hardware Acceleration: Async Off, Echo Off
Last Up 08/19/26 08:53:35.728
Last Down NA
Last Diag: No Diagnostic
Authentication mode: None
Shared-secret profile: None
TxInt: 200 ms, RxInt: 200 ms, Multiplier: 3
Received RxInt: 200 ms, Received Multiplier: 3
Rx Count: 1022, Rx Interval (ms) min/max/avg: 69/256/175 last: 27 ms ago
Tx Count: 1028, Tx Interval (ms) min/max/avg: 150/200/174 last: 80 ms ago
Detect Time: 600 ms
Sched Delay: 1*TxInt: 1007, 2*TxInt: 20, 3*TxInt: 0, GT 3*TxInt: 0
Registered protocols: ospf
Uptime: 02:59.17
Last packet:  Version: 1             - Diagnostic: 0          
              State bit: Up          - Demand bit: 0          
              Poll bit: 0            - Final bit: 0           
              Multiplier: 3          - Length: 24             
              My Discr.: 369354112   - Your Discr.: 3021997832
              Min tx interval: 200   - Min rx interval: 200   
              Min Echo interval: 200                          


LEAF-01#sh bfd peers
VRF name: default
-----------------
DstAddr        MyDisc    YourDisc  Interface/Transport    Type          LastUp 
----------- ---------- ----------- -------------------- ------- ---------------
172.16.1.2  369354112  3021997832        Ethernet1(14)  normal  08/19/26 08:53 

   LastDown            LastDiag    State
-------------- ------------------- -----
         NA       No Diagnostic       Up

LEAF-01#sh bfd peers det
VRF name: default
-----------------
Peer Addr 172.16.1.2, Intf Ethernet1, Type normal, Role active, State Up
VRF default, LAddr 172.16.1.1, LD/RD 369354112/3021997832
Session state is Up and not using echo function
Hardware Acceleration: Async Off, Echo Off
Last Up 08/19/26 08:53:34.649
Last Down NA
Last Diag: No Diagnostic
Authentication mode: None
Shared-secret profile: None
TxInt: 200 ms, RxInt: 200 ms, Multiplier: 3
Received RxInt: 200 ms, Received Multiplier: 3
Rx Count: 1194, Rx Interval (ms) min/max/avg: 60/272/174 last: 129 ms ago
Tx Count: 1189, Tx Interval (ms) min/max/avg: 151/200/175 last: 147 ms ago
Detect Time: 600 ms
Sched Delay: 1*TxInt: 1161, 2*TxInt: 27, 3*TxInt: 0, GT 3*TxInt: 0
Registered protocols: ospf
Uptime: 03:28.33
Last packet:  Version: 1             - Diagnostic: 0         
              State bit: Up          - Demand bit: 0         
              Poll bit: 0            - Final bit: 0          
              Multiplier: 3          - Length: 24            
              My Discr.: 3021997832  - Your Discr.: 369354112
              Min tx interval: 200   - Min rx interval: 200  
              Min Echo interval: 200                         

```


### Настройка аутентификации OSPF 

Между интерфейсами Spine-02 ETh3 - Leaf-03 ETh 2 настроена аутентификация: 

```
SPINE-02(config)#int eth3
SPINE-02(config-if-Et3)#ip ospf message-digest-key 1 md5 otus
SPINE-02(config-if-Et3)#ip ospf authentication message-digest 
SPINE-02(config-if-Et3)#



LEAF-03(config)#inter eth 2
LEAF-03(config-if-Et2)#ip ospf message-digest-key 1 md5 otus
LEAF-03(config-if-Et2)#ip ospf authentication message-digest


```

Проверка: 

```

LEAF-03#sh ip ospf ne
Neighbor ID     Instance VRF      Pri State                  Dead Time   Address         Interface
10.0.1.1        1        default  0   FULL                   00:00:32    172.16.3.2      Ethernet1
10.0.2.2        1        default  0   EXCH START             00:00:36    172.16.3.6      Ethernet2
LEAF-03#sh ip ospf ne
Neighbor ID     Instance VRF      Pri State                  Dead Time   Address         Interface
10.0.1.1        1        default  0   FULL                   00:00:29    172.16.3.2      Ethernet1
10.0.2.2        1        default  0   FULL                   00:00:38    172.16.3.6      Ethernet2
LEAF-03#
LEAF-03#sh ip ospf ne 10.0.2.2 deta
Neighbor 10.0.2.2, instance 1, VRF default, interface address 172.16.3.6
  In area 0.0.0.0 interface Ethernet2
  Neighbor priority is 0, State is FULL, 6 state changes
  Adjacency was established 00:00:23 ago
  Current state was established 00:00:23 ago
  DR IP Address 0.0.0.0 BDR IP Address 0.0.0.0
  Options is E
  Dead timer is due in 00:00:30
  Inactivity timer deferred 0 times
  LSAs retransmitted 0 times to this neighbor
  Graceful-restart-helper mode is Inactive
  Graceful-restart attempts: 0
LEAF-03#sh ip ospf inter eth 2
Ethernet2 is up
  Interface Address 172.16.3.5/30, instance 1, VRF default, Area 0.0.0.0
  Network Type Point-To-Point, Cost: 10
  Transmit Delay is 1 sec, State P2P
  Interface Speed: 1000 mbps
  No Designated Router on this network
  No Backup Designated Router on this network
  Timer intervals configured, Hello 10, Dead 40, Retransmit 5
  Neighbor Count is 1
  Message-digest authentication, using key id 1
  Traffic engineering is disabled
```
