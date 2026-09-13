### Underlay.BGP

Цель:
 - Настроbnm BGP в Underlay сети, для IP связанности между всеми сетевыми устройствами.
 - Зафиксировать в документации - план работы, адресное пространство, схему сети, конфигурацию устройств.
 - Убедиться в наличии IP связанности между устройствами в BGP домене

В этой самостоятельной работе:

- был настроен eBGP в Underlay сети для IP связанности между всеми сетевыми устройствами.
- выделено адресное пространство
- подготовлена схема сети 
- сконфигурированы 2 spine, 3 leaf коммутатора 
- проверена ip связность между всеми устройствами в BGP-домене


### Схема 

![bgp.PNG](bgp.PNG)

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


### Конфигурация BGP 

Был выбран eBGP, а не iBGP для простоты управления маршрутами: 
- не нужно строить full-mesh топологию и  route reflector-ы
- Используется стандартный механизм AS-Path для предотвращения петель.
Как только Leaf-01 видит свой номер AS в анонсе, он просто отбрасывает этот маршрут 

Также при использовании разных AS для каждого Leaf любые изменения, падения линков или сбои локализованы внутри конкретной автономной системы. Сбой на Leaf-01 не вызовет пересчета всей топологии iBGP/IGP на противоположной стороне фабрики.


Spine-коммутаторы находятся в одной автономной системе 65000. 
Настройка: 

```
SPINE-01#
router bgp 65000
   router-id 10.0.1.1
   address-family ipv4 
   maximum-paths 4 (Включаем режим ECMP  и разрешаем устанавливать до 4 параллельных маршрутов с одинаковой стоимостью). 
   neighbor 172.16.1.1 remote-as 65001
   neighbor 172.16.2.1 remote-as 65002
   neighbor 172.16.3.1 remote-as 65003
```

```
SPINE-02#
router bgp 65000
   router-id 10.0.2.2
   address-family ipv4 
   maximum-paths 4
   neighbor 172.16.1.5 remote-as 65001
   neighbor 172.16.2.5 remote-as 65002
   neighbor 172.16.3.5 remote-as 65003
```
Проверка соседей: 

```
SPINE-01#sh ip bgp summ
BGP summary information for VRF default
Router identifier 10.0.1.1, local AS number 65000
Neighbor Status Codes: m - Under maintenance
  Neighbor   V AS           MsgRcvd   MsgSent  InQ OutQ  Up/Down State   PfxRcd PfxAcc
  172.16.1.1 4 65001           1620      1628    0    0 04:55:26 Estab   1      1
  172.16.2.1 4 65002           1627      1620    0    0 04:54:29 Estab   1      1
  172.16.3.1 4 65003           1628      1625    0    0 04:55:16 Estab   0      0
SPINE-01#
```

```
SPINE-02#sh ip bgp summ
BGP summary information for VRF default
Router identifier 10.0.2.2, local AS number 65000
Neighbor Status Codes: m - Under maintenance
  Neighbor   V AS           MsgRcvd   MsgSent  InQ OutQ  Up/Down State   PfxRcd PfxAcc
  172.16.1.5 4 65001           1612      1608    0    0 04:53:46 Estab   1      1
  172.16.2.5 4 65002           1613      1609    0    0 04:53:12 Estab   1      1
  172.16.3.5 4 65003           1614      1611    0    0 04:53:46 Estab   0      0
SPINE-02#
```


Конфигурация на leaf-02: 
Cеть на loopback анонсировала через редистрибуцию и route-map. 
Использовала set origin incomplete, потому что на arista он сам не применяется. 
Включила community

```
!
route-map RM_red_conn permit 10
   match interface Loopback0
   set community 65001:100 65002:200 additive
   set origin incomplete
!
router bgp 65002
   router-id 10.0.0.2
   address-family ipv4
   maximum-paths 4
   neighbor 172.16.2.2 remote-as 65000
   neighbor 172.16.2.2 send-community standard extended
   neighbor 172.16.2.6 send-community standard extended
   redistribute connected route-map RM_red_conn
```


```
router bgp 65001
   router-id 10.0.0.1
   maximum-paths 4
   neighbor 172.16.1.2 remote-as 65000
   neighbor 172.16.1.2 send-community standard extended
   neighbor 172.16.1.6 remote-as 65000
   neighbor 172.16.1.6 send-community standard extended
   network 10.0.0.1/32
```

Community c leaf-02 видно на spine:
```
SPINE-01#
SPINE-01#
SPINE-01#sh ip bgp 10.0.0.2/32 
BGP routing table information for VRF default
Router identifier 10.0.1.1, local AS number 65000
BGP routing table entry for 10.0.0.2/32
 Paths: 1 available
  65002
    172.16.2.1 from 172.16.2.1 (10.0.0.2)
      Origin INCOMPLETE, metric 0, localpref 100, IGP metric 0, weight 0, tag 0
      Received 00:09:03 ago, valid, external, best
      Community: 65001:100 65002:200
      Rx SAFI: Unicast
```
```
SPINE-02#sh ip bgp 10.0.0.2/32 
BGP routing table information for VRF default
Router identifier 10.0.2.2, local AS number 65000
BGP routing table entry for 10.0.0.2/32
 Paths: 1 available
  65002
    172.16.2.5 from 172.16.2.5 (10.0.0.2)
      Origin INCOMPLETE, metric 0, localpref 100, IGP metric 0, weight 0, tag 0
      Received 00:10:44 ago, valid, external, best
      Community: 65001:100 65002:200
      Rx SAFI: Unicast
SPINE-02#
```

Создала community list и поменяла значение local pref для управления входящим трафиком:


```
route-map RM_CL_65002 permit 10
   match community CL_65002
   set local-preference 10
!
route-map RM_CL_65002 permit 20
!
router bgp 65003
   router-id 10.0.0.3
   maximum-paths 4
   neighbor 172.16.3.2 remote-as 65000
   neighbor 172.16.3.2 route-map RM_CL_65002 in
   neighbor 172.16.3.2 send-community standard extended
   neighbor 172.16.3.6 remote-as 65000
   neighbor 172.16.3.6 send-community standard extended
   network 10.0.0.3/32
!
end
```

Теперь  префикс со spine-01 с local pref всего 10:
```
LEAF-03#sh ip bgp 10.0.0.2/32
BGP routing table information for VRF default
Router identifier 10.0.0.3, local AS number 65003
BGP routing table entry for 10.0.0.2/32
 Paths: 2 available
  65000 65002
    172.16.3.6 from 172.16.3.6 (10.0.2.2)
      Origin INCOMPLETE, metric 0, localpref 100, IGP metric 0, weight 0, tag 0
      Received 00:10:44 ago, valid, external, best
      Rx SAFI: Unicast
  65000 65002
    172.16.3.2 from 172.16.3.2 (10.0.1.1)
      Origin INCOMPLETE, metric 0, localpref 10, IGP metric 0, weight 0, tag 0
      Received 00:02:34 ago, valid, external
      Community: 65001:100 65002:200
      Rx SAFI: Unicast
LEAF-03#
```

На leaf-01 leaf-03 loopback-сеть анонсирована через команду network:


```
LEAF-01#  
 router bgp 65001
   router-id 10.0.0.1
   maximum-paths 4
   neighbor 172.16.1.2 remote-as 65000
   neighbor 172.16.1.2 send-community standard extended
   neighbor 172.16.1.6 remote-as 65000
   neighbor 172.16.1.6 send-community standard extended
   network 10.0.0.1/32
```

```
LEAF-03#sh run sec bgp
logging level BGP errors
router bgp 65003
   router-id 10.0.0.3
   maximum-paths 4
   neighbor 172.16.3.2 remote-as 65000
   neighbor 172.16.3.2 send-community standard extended
   neighbor 172.16.3.6 remote-as 65000
   neighbor 172.16.3.6 send-community standard extended
   network 10.0.0.3/32
LEAF-03#

```

Проверка:

```
LEAF-01#sh ip bgp
BGP routing table information for VRF default
Router identifier 10.0.0.1, local AS number 65001
Route status codes: s - suppressed contributor, * - valid, > - active, E - ECMP head, e - ECMP
                    S - Stale, c - Contributing to ECMP, b - backup, L - labeled-unicast
                    % - Pending BGP convergence
Origin codes: i - IGP, e - EGP, ? - incomplete
RPKI Origin Validation codes: V - valid, I - invalid, U - unknown
AS Path Attributes: Or-ID - Originator ID, C-LST - Cluster List, LL Nexthop - Link Local Nexthop

          Network                Next Hop              Metric  AIGP       LocPref Weight  Path
 * >      10.0.0.1/32            -                     -       -          -       0       i
 * >Ec    10.0.0.2/32            172.16.1.2            0       -          100     0       65000 65002 ?
 *  ec    10.0.0.2/32            172.16.1.6            0       -          100     0       65000 65002 ?
 * >Ec    10.0.0.3/32            172.16.1.2            0       -          100     0       65000 65003 i
 *  ec    10.0.0.3/32            172.16.1.6            0       -          100     0       65000 65003 i
LEAF-01#
```


```
end
LEAF-02#sh ip bgp 
BGP routing table information for VRF default
Router identifier 10.0.0.2, local AS number 65002
Route status codes: s - suppressed contributor, * - valid, > - active, E - ECMP head, e - ECMP
                    S - Stale, c - Contributing to ECMP, b - backup, L - labeled-unicast
                    % - Pending BGP convergence
Origin codes: i - IGP, e - EGP, ? - incomplete
RPKI Origin Validation codes: V - valid, I - invalid, U - unknown
AS Path Attributes: Or-ID - Originator ID, C-LST - Cluster List, LL Nexthop - Link Local Nexthop

          Network                Next Hop              Metric  AIGP       LocPref Weight  Path
 * >Ec    10.0.0.1/32            172.16.2.2            0       -          100     0       65000 65001 i
 *  ec    10.0.0.1/32            172.16.2.6            0       -          100     0       65000 65001 i
 * >      10.0.0.2/32            -                     -       -          -       0       ?
 * >Ec    10.0.0.3/32            172.16.2.2            0       -          100     0       65000 65003 i
 *  ec    10.0.0.3/32            172.16.2.6            0       -          100     0       65000 65003 i
LEAF-02#

```


```
   network 10.0.0.3/32
LEAF-03#sh ip bgp
BGP routing table information for VRF default
Router identifier 10.0.0.3, local AS number 65003
Route status codes: s - suppressed contributor, * - valid, > - active, E - ECMP head, e - ECMP
                    S - Stale, c - Contributing to ECMP, b - backup, L - labeled-unicast
                    % - Pending BGP convergence
Origin codes: i - IGP, e - EGP, ? - incomplete
RPKI Origin Validation codes: V - valid, I - invalid, U - unknown
AS Path Attributes: Or-ID - Originator ID, C-LST - Cluster List, LL Nexthop - Link Local Nexthop

          Network                Next Hop              Metric  AIGP       LocPref Weight  Path
 * >Ec    10.0.0.1/32            172.16.3.2            0       -          100     0       65000 65001 i
 *  ec    10.0.0.1/32            172.16.3.6            0       -          100     0       65000 65001 i
 * >Ec    10.0.0.2/32            172.16.3.2            0       -          100     0       65000 65002 ?
 *  ec    10.0.0.2/32            172.16.3.6            0       -          100     0       65000 65002 ?
 * >      10.0.0.3/32            -                     -       -          -       0       i
LEAF-03#  

```

### Настройка bfd и его проверка на всех устройствах: 


```
router bgp 65001
   maximum-paths 4
   neighbor 172.16.1.1 bfd
   neighbor 172.16.1.1 route-map RM_CL_65002 in
   neighbor 172.16.1.2 remote-as 65000
   neighbor 172.16.1.2 bfd
   neighbor 172.16.1.2 send-community standard extended
   neighbor 172.16.1.6 remote-as 65000
   neighbor 172.16.1.6 bfd
   neighbor 172.16.1.6 send-community standard extended
   network 10.0.0.1/32
!
end
LEAF-01#show bfd peers
VRF name: default
-----------------
DstAddr        MyDisc    YourDisc  Interface/Transport    Type          LastUp 
---------- ----------- ----------- -------------------- ------- ---------------
172.16.1.2 3489880817   300166115        Ethernet1(14)  normal  09/02/26 20:58 
172.16.1.6 3786475106  3002683961        Ethernet2(15)  normal  09/02/26 20:58 

   LastDown            LastDiag    State
-------------- ------------------- -----
         NA       No Diagnostic       Up
         NA       No Diagnostic       Up

LEAF-01#

```

```

SPINE-01#sh run | section bgp
router bgp 65000
   router-id 10.0.1.1
   maximum-paths 4
   neighbor 172.16.1.1 remote-as 65001
   neighbor 172.16.1.1 bfd
   neighbor 172.16.2.1 remote-as 65002
   neighbor 172.16.2.1 bfd
   neighbor 172.16.3.1 remote-as 65003
   neighbor 172.16.3.1 bfd
   neighbor 172.16.3.1 send-community standard extended
SPINE-01#sh bfd peers
VRF name: default
-----------------
DstAddr        MyDisc    YourDisc  Interface/Transport    Type          LastUp 
---------- ----------- ----------- -------------------- ------- ---------------
172.16.1.1  300166115  3489880817        Ethernet1(14)  normal  09/02/26 20:58 
172.16.2.1 4079982080  4009147190        Ethernet2(15)  normal  09/03/26 19:53 
172.16.3.1 3876908631  3829982205        Ethernet3(16)  normal  09/02/26 21:00 

   LastDown            LastDiag    State
-------------- ------------------- -----
         NA       No Diagnostic       Up
         NA       No Diagnostic       Up
         NA       No Diagnostic       Up
```

```
LEAF-02(config)#router bgp 65002
LEAF-02(config-router-bgp)#neighbor 172.16.2.2 bfd
LEAF-02(config-router-bgp)#neighbor 172.16.2.6 bfd
LEAF-02(config-router-bgp)#exit
LEAF-02(config)#
LEAF-02(config)#exit
LEAF-02#
LEAF-02#clear ip bgp
! Peerings for all neighbors were hard reset
LEAF-02#
LEAF-02#
LEAF-02#sh bfd peers
VRF name: default
-----------------
DstAddr        MyDisc    YourDisc  Interface/Transport    Type          LastUp 
---------- ----------- ----------- -------------------- ------- ---------------
172.16.2.2 4009147190  4079982080        Ethernet1(14)  normal  09/03/26 19:53 
172.16.2.6  556567343  1328777282        Ethernet2(15)  normal  09/03/26 19:53 

   LastDown            LastDiag    State
-------------- ------------------- -----
         NA       No Diagnostic       Up
         NA       No Diagnostic       Up

```

```
router bgp 65003
   router-id 10.0.0.3
   maximum-paths 4
   neighbor 172.16.2.2 bfd
   neighbor 172.16.3.2 remote-as 65000
   neighbor 172.16.3.2 bfd
   neighbor 172.16.3.2 route-map RM_CL_65002 in
   neighbor 172.16.3.2 send-community standard extended
   neighbor 172.16.3.6 remote-as 65000
   neighbor 172.16.3.6 bfd
   neighbor 172.16.3.6 send-community standard extended
   network 10.0.0.3/32
LEAF-03#sh bfd peers
VRF name: default
-----------------
DstAddr        MyDisc    YourDisc  Interface/Transport    Type          LastUp 
---------- ----------- ----------- -------------------- ------- ---------------
172.16.3.2 3829982205  3876908631        Ethernet1(15)  normal  09/02/26 21:00 
172.16.3.6 3577107039  3449176650        Ethernet2(16)  normal  09/02/26 21:00 

   LastDown            LastDiag    State
-------------- ------------------- -----
         NA       No Diagnostic       Up
         NA       No Diagnostic       Up

LEAF-03#

```


```
SPINE-02#sh run | sec bgp
router bgp 65000
   router-id 10.0.2.2
   maximum-paths 4
   neighbor 172.16.1.5 remote-as 65001
   neighbor 172.16.1.5 bfd
   neighbor 172.16.2.5 remote-as 65002
   neighbor 172.16.2.5 bfd
   neighbor 172.16.3.5 remote-as 65003
   neighbor 172.16.3.5 bfd
SPINE-02#sh bfd peers
VRF name: default
-----------------
DstAddr        MyDisc    YourDisc  Interface/Transport    Type          LastUp 
---------- ----------- ----------- -------------------- ------- ---------------
172.16.1.5 3002683961  3786475106        Ethernet1(14)  normal  09/02/26 20:58 
172.16.2.5 1328777282   556567343        Ethernet2(15)  normal  09/03/26 19:53 
172.16.3.5 3449176650  3577107039        Ethernet3(16)  normal  09/02/26 21:00 

   LastDown            LastDiag    State
-------------- ------------------- -----
         NA       No Diagnostic       Up
         NA       No Diagnostic       Up
         NA       No Diagnostic       Up

SPINE-02#

```


###Что было сделано после проверки дз: 

-анонсированы лупбеки на Spine01, Spine02 
-проверка пингов с лупбеков: 

```
SPINE-01#ping 10.0.0.3 source lo0
PING 10.0.0.3 (10.0.0.3) from 10.0.1.1 : 72(100) bytes of data.
80 bytes from 10.0.0.3: icmp_seq=1 ttl=64 time=12.6 ms
80 bytes from 10.0.0.3: icmp_seq=2 ttl=64 time=10.8 ms
80 bytes from 10.0.0.3: icmp_seq=3 ttl=64 time=10.6 ms
80 bytes from 10.0.0.3: icmp_seq=4 ttl=64 time=9.12 ms
80 bytes from 10.0.0.3: icmp_seq=5 ttl=64 time=6.88 ms

--- 10.0.0.3 ping statistics ---
5 packets transmitted, 5 received, 0% packet loss, time 56ms
rtt min/avg/max/mdev = 6.882/10.038/12.646/1.934 ms, ipg/ewma 14.040/11.202 ms
SPINE-01#ping 10.0.0.2 source lo0
PING 10.0.0.2 (10.0.0.2) from 10.0.1.1 : 72(100) bytes of data.
80 bytes from 10.0.0.2: icmp_seq=1 ttl=64 time=9.05 ms
80 bytes from 10.0.0.2: icmp_seq=2 ttl=64 time=8.50 ms
80 bytes from 10.0.0.2: icmp_seq=3 ttl=64 time=9.17 ms
80 bytes from 10.0.0.2: icmp_seq=4 ttl=64 time=13.7 ms
80 bytes from 10.0.0.2: icmp_seq=5 ttl=64 time=11.3 ms

--- 10.0.0.2 ping statistics ---
5 packets transmitted, 5 received, 0% packet loss, time 40ms
rtt min/avg/max/mdev = 8.506/10.369/13.797/1.961 ms, pipe 2, ipg/ewma 10.101/9.822 ms
SPINE-01#ping 10.0.0.1 source lo0
PING 10.0.0.1 (10.0.0.1) from 10.0.1.1 : 72(100) bytes of data.
80 bytes from 10.0.0.1: icmp_seq=1 ttl=64 time=8.31 ms
80 bytes from 10.0.0.1: icmp_seq=2 ttl=64 time=10.5 ms
80 bytes from 10.0.0.1: icmp_seq=3 ttl=64 time=6.34 ms
80 bytes from 10.0.0.1: icmp_seq=4 ttl=64 time=6.83 ms
80 bytes from 10.0.0.1: icmp_seq=5 ttl=64 time=7.77 ms

--- 10.0.0.1 ping statistics ---
5 packets transmitted, 5 received, 0% packet loss, time 41ms
rtt min/avg/max/mdev = 6.347/7.957/10.512/1.452 ms, ipg/ewma 10.335/8.081 ms
SPINE-01#
```


```

--- 10.0.0.3 ping statistics ---
5 packets transmitted, 5 received, 0% packet loss, time 2ms
rtt min/avg/max/mdev = 0.153/0.234/0.531/0.149 ms, ipg/ewma 0.524/0.378 ms
LEAF-03#ping 10.0.0.3 sou lo0
PING 10.0.0.3 (10.0.0.3) from 10.0.0.3 : 72(100) bytes of data.
80 bytes from 10.0.0.3: icmp_seq=1 ttl=64 time=0.847 ms
80 bytes from 10.0.0.3: icmp_seq=2 ttl=64 time=0.155 ms
80 bytes from 10.0.0.3: icmp_seq=3 ttl=64 time=0.206 ms
80 bytes from 10.0.0.3: icmp_seq=4 ttl=64 time=0.174 ms
80 bytes from 10.0.0.3: icmp_seq=5 ttl=64 time=0.171 ms

--- 10.0.0.3 ping statistics ---
5 packets transmitted, 5 received, 0% packet loss, time 11ms
rtt min/avg/max/mdev = 0.155/0.310/0.847/0.269 ms, ipg/ewma 2.958/0.569 ms
LEAF-03#ping 10.0.0.2 sou lo0
PING 10.0.0.2 (10.0.0.2) from 10.0.0.3 : 72(100) bytes of data.
80 bytes from 10.0.0.2: icmp_seq=1 ttl=63 time=23.5 ms
80 bytes from 10.0.0.2: icmp_seq=2 ttl=63 time=20.7 ms
80 bytes from 10.0.0.2: icmp_seq=3 ttl=63 time=12.5 ms
80 bytes from 10.0.0.2: icmp_seq=4 ttl=63 time=14.2 ms
80 bytes from 10.0.0.2: icmp_seq=5 ttl=63 time=16.9 ms

--- 10.0.0.2 ping statistics ---
5 packets transmitted, 5 received, 0% packet loss, time 81ms
rtt min/avg/max/mdev = 12.515/17.595/23.540/4.072 ms, pipe 2, ipg/ewma 20.423/20.408 ms
LEAF-03#ping 10.0.0.1 sou lo0
PING 10.0.0.1 (10.0.0.1) from 10.0.0.3 : 72(100) bytes of data.
80 bytes from 10.0.0.1: icmp_seq=1 ttl=64 time=9.98 ms
80 bytes from 10.0.0.1: icmp_seq=2 ttl=64 time=8.08 ms
80 bytes from 10.0.0.1: icmp_seq=3 ttl=64 time=9.08 ms
80 bytes from 10.0.0.1: icmp_seq=4 ttl=64 time=7.44 ms
80 bytes from 10.0.0.1: icmp_seq=5 ttl=64 time=9.04 ms

```

```

PINE-02#ping 10.0.0.1 sou lo0
PING 10.0.0.1 (10.0.0.1) from 10.0.2.2 : 72(100) bytes of data.
80 bytes from 10.0.0.1: icmp_seq=1 ttl=64 time=0.875 ms
80 bytes from 10.0.0.1: icmp_seq=2 ttl=64 time=0.179 ms
80 bytes from 10.0.0.1: icmp_seq=3 ttl=64 time=0.170 ms
80 bytes from 10.0.0.1: icmp_seq=4 ttl=64 time=0.154 ms
80 bytes from 10.0.0.1: icmp_seq=5 ttl=64 time=0.191 ms

--- 10.0.0.1 ping statistics ---
5 packets transmitted, 5 received, 0% packet loss, time 11ms
rtt min/avg/max/mdev = 0.154/0.313/0.875/0.281 ms, ipg/ewma 2.781/0.585 ms
SPINE-02#ping 10.0.0.2 sou lo0
PING 10.0.0.2 (10.0.0.2) from 10.0.2.2 : 72(100) bytes of data.
80 bytes from 10.0.0.2: icmp_seq=1 ttl=64 time=8.85 ms
80 bytes from 10.0.0.2: icmp_seq=2 ttl=64 time=9.89 ms
80 bytes from 10.0.0.2: icmp_seq=3 ttl=64 time=6.04 ms
80 bytes from 10.0.0.2: icmp_seq=4 ttl=64 time=6.53 ms
80 bytes from 10.0.0.2: icmp_seq=5 ttl=64 time=9.50 ms

--- 10.0.0.2 ping statistics ---
5 packets transmitted, 5 received, 0% packet loss, time 39ms
rtt min/avg/max/mdev = 6.042/8.165/9.897/1.576 ms, ipg/ewma 9.911/8.497 ms
SPINE-02#ping 10.0.0.3 sou lo0
PING 10.0.0.3 (10.0.0.3) from 10.0.2.2 : 72(100) bytes of data.
80 bytes from 10.0.0.3: icmp_seq=1 ttl=64 time=10.9 ms
80 bytes from 10.0.0.3: icmp_seq=2 ttl=64 time=9.31 ms
80 bytes from 10.0.0.3: icmp_seq=3 ttl=64 time=6.52 ms
80 bytes from 10.0.0.3: icmp_seq=4 ttl=64 time=6.94 ms
80 bytes from 10.0.0.3: icmp_seq=5 ttl=64 time=8.90 ms

--- 10.0.0.3 ping statistics ---
5 packets transmitted, 5 received, 0% packet loss, time 46ms
rtt min/avg/max/mdev = 6.527/8.518/10.911/1.613 ms, ipg/ewma 11.656/9.672 ms
SPINE-02#


```


```

LEAF-02#ping 10.0.0.1 sou lo0
PING 10.0.0.1 (10.0.0.1) from 10.0.0.2 : 72(100) bytes of data.
80 bytes from 10.0.0.1: icmp_seq=1 ttl=64 time=13.5 ms
80 bytes from 10.0.0.1: icmp_seq=2 ttl=64 time=7.44 ms
80 bytes from 10.0.0.1: icmp_seq=3 ttl=64 time=6.43 ms
80 bytes from 10.0.0.1: icmp_seq=4 ttl=64 time=7.50 ms
80 bytes from 10.0.0.1: icmp_seq=5 ttl=64 time=7.15 ms

--- 10.0.0.1 ping statistics ---
5 packets transmitted, 5 received, 0% packet loss, time 53ms
rtt min/avg/max/mdev = 6.433/8.418/13.562/2.602 ms, ipg/ewma 13.263/10.903 ms
LEAF-02#ping 10.0.0.2 sou lo0
PING 10.0.0.2 (10.0.0.2) from 10.0.0.2 : 72(100) bytes of data.
80 bytes from 10.0.0.2: icmp_seq=1 ttl=64 time=0.828 ms
80 bytes from 10.0.0.2: icmp_seq=2 ttl=64 time=0.184 ms
80 bytes from 10.0.0.2: icmp_seq=3 ttl=64 time=0.193 ms
80 bytes from 10.0.0.2: icmp_seq=4 ttl=64 time=0.208 ms
80 bytes from 10.0.0.2: icmp_seq=5 ttl=64 time=0.178 ms

--- 10.0.0.2 ping statistics ---
5 packets transmitted, 5 received, 0% packet loss, time 6ms
rtt min/avg/max/mdev = 0.178/0.318/0.828/0.255 ms, ipg/ewma 1.646/0.564 ms
LEAF-02#ping 10.0.0.3 sou lo0
PING 10.0.0.3 (10.0.0.3) from 10.0.0.2 : 72(100) bytes of data.
80 bytes from 10.0.0.3: icmp_seq=1 ttl=63 time=20.9 ms
80 bytes from 10.0.0.3: icmp_seq=2 ttl=63 time=22.1 ms
80 bytes from 10.0.0.3: icmp_seq=3 ttl=63 time=19.3 ms
80 bytes from 10.0.0.3: icmp_seq=4 ttl=63 time=17.3 ms
80 bytes from 10.0.0.3: icmp_seq=5 ttl=63 time=14.6 ms

--- 10.0.0.3 ping statistics ---
5 packets transmitted, 5 received, 0% packet loss, time 78ms
rtt min/avg/max/mdev = 14.690/18.921/22.195/2.658 ms, pipe 2, ipg/ewma 19.652/19.735 ms
LEAF-02#ping 10.0.1.1 sou lo0
PING 10.0.1.1 (10.0.1.1) from 10.0.0.2 : 72(100) bytes of data.
80 bytes from 10.0.1.1: icmp_seq=1 ttl=64 time=8.77 ms
80 bytes from 10.0.1.1: icmp_seq=2 ttl=64 time=8.23 ms
80 bytes from 10.0.1.1: icmp_seq=3 ttl=64 time=7.62 ms
80 bytes from 10.0.1.1: icmp_seq=4 ttl=64 time=11.0 ms
80 bytes from 10.0.1.1: icmp_seq=5 ttl=64 time=12.6 ms

--- 10.0.1.1 ping statistics ---
5 packets transmitted, 5 received, 0% packet loss, time 49ms
rtt min/avg/max/mdev = 7.621/9.677/12.688/1.903 ms, pipe 2, ipg/ewma 12.339/9.358 ms

--- 10.0.1.2 ping statistics ---
5 packets transmitted, 0 received, 100% packet loss, time 49ms

LEAF-02#ping 10.0.2.2 sou lo0
PING 10.0.2.2 (10.0.2.2) from 10.0.0.2 : 72(100) bytes of data.
80 bytes from 10.0.2.2: icmp_seq=1 ttl=64 time=12.5 ms
80 bytes from 10.0.2.2: icmp_seq=2 ttl=64 time=10.7 ms
80 bytes from 10.0.2.2: icmp_seq=3 ttl=64 time=7.72 ms
80 bytes from 10.0.2.2: icmp_seq=4 ttl=64 time=10.8 ms
80 bytes from 10.0.2.2: icmp_seq=5 ttl=64 time=6.95 ms

--- 10.0.2.2 ping statistics ---
5 packets transmitted, 5 received, 0% packet loss, time 55ms
rtt min/avg/max/mdev = 6.954/9.772/12.589/2.110 ms, ipg/ewma 13.788/11.073 ms
LEAF-02#

```

```
LEAF-01#
LEAF-01#
LEAF-01#ping 10.0.2.2 sour lo0
PING 10.0.2.2 (10.0.2.2) from 10.0.0.1 : 72(100) bytes of data.
80 bytes from 10.0.2.2: icmp_seq=1 ttl=64 time=12.7 ms
80 bytes from 10.0.2.2: icmp_seq=2 ttl=64 time=7.84 ms
80 bytes from 10.0.2.2: icmp_seq=3 ttl=64 time=10.8 ms
80 bytes from 10.0.2.2: icmp_seq=4 ttl=64 time=9.01 ms
80 bytes from 10.0.2.2: icmp_seq=5 ttl=64 time=7.54 ms

--- 10.0.2.2 ping statistics ---
5 packets transmitted, 5 received, 0% packet loss, time 55ms
rtt min/avg/max/mdev = 7.543/9.596/12.747/1.957 ms, ipg/ewma 13.764/11.094 ms
LEAF-01#ping 10.0.1.1 sour lo0
PING 10.0.1.1 (10.0.1.1) from 10.0.0.1 : 72(100) bytes of data.
80 bytes from 10.0.1.1: icmp_seq=1 ttl=64 time=10.6 ms
80 bytes from 10.0.1.1: icmp_seq=2 ttl=64 time=11.9 ms
80 bytes from 10.0.1.1: icmp_seq=3 ttl=64 time=7.89 ms
80 bytes from 10.0.1.1: icmp_seq=4 ttl=64 time=7.13 ms
80 bytes from 10.0.1.1: icmp_seq=5 ttl=64 time=9.21 ms

--- 10.0.1.1 ping statistics ---
5 packets transmitted, 5 received, 0% packet loss, time 51ms
rtt min/avg/max/mdev = 7.130/9.362/11.966/1.759 ms, ipg/ewma 12.791/9.904 ms
LEAF-01#ping 10.0.0.1 sour lo0
PING 10.0.0.1 (10.0.0.1) from 10.0.0.1 : 72(100) bytes of data.
80 bytes from 10.0.0.1: icmp_seq=1 ttl=64 time=0.555 ms
80 bytes from 10.0.0.1: icmp_seq=2 ttl=64 time=0.151 ms
80 bytes from 10.0.0.1: icmp_seq=3 ttl=64 time=0.153 ms
80 bytes from 10.0.0.1: icmp_seq=4 ttl=64 time=0.151 ms
80 bytes from 10.0.0.1: icmp_seq=5 ttl=64 time=0.150 ms

--- 10.0.0.1 ping statistics ---
5 packets transmitted, 5 received, 0% packet loss, time 3ms
rtt min/avg/max/mdev = 0.150/0.232/0.555/0.161 ms, ipg/ewma 0.867/0.388 ms
LEAF-01#ping 10.0.0.2 sour lo0
PING 10.0.0.2 (10.0.0.2) from 10.0.0.1 : 72(100) bytes of data.
80 bytes from 10.0.0.2: icmp_seq=1 ttl=63 time=21.3 ms
80 bytes from 10.0.0.2: icmp_seq=2 ttl=63 time=21.0 ms
80 bytes from 10.0.0.2: icmp_seq=3 ttl=63 time=13.3 ms
80 bytes from 10.0.0.2: icmp_seq=4 ttl=63 time=14.4 ms
80 bytes from 10.0.0.2: icmp_seq=5 ttl=63 time=16.4 ms

--- 10.0.0.2 ping statistics ---
5 packets transmitted, 5 received, 0% packet loss, time 76ms
rtt min/avg/max/mdev = 13.347/17.341/21.331/3.310 ms, pipe 2, ipg/ewma 19.247/19.189 ms
LEAF-01#ping 10.0.0.3 sour lo0
PING 10.0.0.3 (10.0.0.3) from 10.0.0.1 : 72(100) bytes of data.
80 bytes from 10.0.0.3: icmp_seq=1 ttl=63 time=21.5 ms
80 bytes from 10.0.0.3: icmp_seq=2 ttl=63 time=19.7 ms
80 bytes from 10.0.0.3: icmp_seq=3 ttl=63 time=18.8 ms
80 bytes from 10.0.0.3: icmp_seq=4 ttl=63 time=19.0 ms
80 bytes from 10.0.0.3: icmp_seq=5 ttl=63 time=16.2 ms

--- 10.0.0.3 ping statistics ---
5 packets transmitted, 5 received, 0% packet loss, time 84ms
rtt min/avg/max/mdev = 16.235/19.091/21.524/1.713 ms, pipe 2, ipg/ewma 21.248/20.192 ms
LEAF-01#

```

###Выводы bgp summary со всех устройств: 


```
LEAF-01#sh bgp summ
BGP summary information for VRF default
Router identifier 10.0.0.1, local AS number 65001
Neighbor            AS Session State AFI/SAFI                AFI/SAFI State   NLRI Rcd   NLRI Acc
---------- ----------- ------------- ----------------------- -------------- ---------- ----------
172.16.1.2       65000 Established   IPv4 Unicast            Negotiated              3          3
172.16.1.6       65000 Established   IPv4 Unicast            Negotiated              3          3
LEAF-01#
LEAF-01#


```



```
LEAF-02#sh bgp summ
BGP summary information for VRF default
Router identifier 10.0.0.2, local AS number 65002
Neighbor            AS Session State AFI/SAFI                AFI/SAFI State   NLRI Rcd   NLRI Acc
---------- ----------- ------------- ----------------------- -------------- ---------- ----------
172.16.2.2       65000 Established   IPv4 Unicast            Negotiated              3          3
172.16.2.6       65000 Established   IPv4 Unicast            Negotiated              3          3
LEAF-02#



```

```
LEAF-03#sh bgp summ
BGP summary information for VRF default
Router identifier 10.0.0.3, local AS number 65003
Neighbor            AS Session State AFI/SAFI                AFI/SAFI State   NLRI Rcd   NLRI Acc
---------- ----------- ------------- ----------------------- -------------- ---------- ----------
172.16.3.2       65000 Established   IPv4 Unicast            Negotiated              3          3
172.16.3.6       65000 Established   IPv4 Unicast            Negotiated              3          3
LEAF-03#

```

```
SPINE-02#sh bgp summ
BGP summary information for VRF default
Router identifier 10.0.2.2, local AS number 65000
Neighbor            AS Session State AFI/SAFI                AFI/SAFI State   NLRI Rcd   NLRI Acc
---------- ----------- ------------- ----------------------- -------------- ---------- ----------
172.16.1.5       65001 Established   IPv4 Unicast            Negotiated              1          1
172.16.2.5       65002 Established   IPv4 Unicast            Negotiated              1          1
172.16.3.5       65003 Established   IPv4 Unicast            Negotiated              1          1

```

```
SPINE-01#sh bgp summ
BGP summary information for VRF default
Router identifier 10.0.1.1, local AS number 65000
Neighbor            AS Session State AFI/SAFI                AFI/SAFI State   NLRI Rcd   NLRI Acc
---------- ----------- ------------- ----------------------- -------------- ---------- ----------
172.16.1.1       65001 Established   IPv4 Unicast            Negotiated              1          1
172.16.2.1       65002 Established   IPv4 Unicast            Negotiated              1          1
172.16.3.1       65003 Established   IPv4 Unicast            Negotiated              1          1
SPINE-01#
SPINE-01#
SPINE-01#
```
