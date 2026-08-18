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
SPINE-01|lo0|10.0.0.1/32|Loopback Локальный Router ID
SPINE-01|Eth1|172.16.1.2/30|P2P Линк|LEAF-01 (Eth1)
SPINE-01|Eth2|172.16.2.2/30|P2P Линк|LEAF-02 (Eth1)
SPINE-01|Eth3|172.16.3.2/30|P2P Линк|LEAF-03 (Eth1)|
SPINE-02|lo0|10.0.0.2/32|Loopback Локальный Router ID|
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

Все коммутаторы находятся в зоне Area 0 
Настройки на коммутаторах на примере Spine-01: 

```
SPINE-01(config)#router ospf 1
SPINE-01(config-router-ospf)#router-id 10.0.1.1
SPINE-01(config-router-ospf)#exi
SPINE-01(config)#interface eth 1-3
SPINE-01(config-if-Et1-3)#ip ospf area 0
SPINE-01(config-if-Et1-3)#ip ospf network point-to-point 
SPINE-01(config-if-Et1-3)#exi
SPINE-01(config)#inter loopback 0
SPINE-01(config-if-Lo0)#ip ospf area 0
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

