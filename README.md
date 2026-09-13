### Overlay на основе VxLAN EVPN для L2 связанности между клиентами.

Цель:
- Настроить BGP peering между Leaf и Spine в AF l2vpn evpn
- Настроить связанность между клиентами в первой зоне и убедитесь в её наличии
- Зафиксировать в документации - план работы, адресное пространство, схему сети, конфигурацию устройств

В этой самостоятельной работе:

- был настроен eBGP в Underlay сети для IP связанности между всеми сетевыми устройствами.
- выделено адресное пространство
- подготовлена схема сети 
- сконфигурированы 2 spine, 3 leaf коммутатора 
- проверена ip связность между всеми устройствами в BGP-домене
- Настроен BGP peering между Leaf и Spine в AF l2vpn evpn
- Настроить связанность между клиентами в первой зоне
- проверена связность 


### Схема 

![xxx.PNG](xxx.PNG)

Была выбрана следующая адресация для Underlay: 

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

configure terminal
!
router bgp 65000
   no bgp default ipv4-unicast
   !
   ! Активируем Underlay-соседей строго в IPv4
   address-family ipv4
      neighbor 172.16.1.1 activate
      neighbor 172.16.2.1 activate
      neighbor 172.16.3.1 activate
   !
   ! Создаем Overlay peer-group для EVPN
   neighbor EVPN-OVERLAY peer group
   neighbor EVPN-OVERLAY update-source Loopback0
   neighbor EVPN-OVERLAY ebgp-multihop 3
   neighbor EVPN-OVERLAY send-community extended
   !
   ! Привязываем реальные Loopback0-адреса Лифов к EVPN
   neighbor 10.0.0.1 peer group EVPN-OVERLAY
   neighbor 10.0.0.1 remote-as 65001
   neighbor 10.0.0.1 description to-LEAF-01-EVPN
   !
   neighbor 10.0.0.2 peer group EVPN-OVERLAY
   neighbor 10.0.0.2 remote-as 65002
   neighbor 10.0.0.2 description to-LEAF-02-EVPN
   !
   neighbor 10.0.0.3 peer group EVPN-OVERLAY
   neighbor 10.0.0.3 remote-as 65003
   neighbor 10.0.0.3 description to-LEAF-03-EVPN
   !
   ! Активируем семейство EVPN
   address-family evpn
      neighbor EVPN-OVERLAY activate
!
end


###Настройки Spine-02

configure terminal
!
router bgp 65000
   no bgp default ipv4-unicast
   !
   ! Активируем Underlay-соседей SPINE-02 строго в IPv4
   address-family ipv4
      neighbor 172.16.1.5 activate
      neighbor 172.16.2.5 activate
      neighbor 172.16.3.5 activate
   !
   ! Создаем Overlay peer-group для EVPN
   neighbor EVPN-OVERLAY peer group
   neighbor EVPN-OVERLAY update-source Loopback0
   neighbor EVPN-OVERLAY ebgp-multihop 3
   neighbor EVPN-OVERLAY send-community extended
   !
   ! Привязываем Loopback0-адреса Лифов к EVPN
   neighbor 10.0.0.1 peer group EVPN-OVERLAY
   neighbor 10.0.0.1 remote-as 65001
   neighbor 10.0.0.1 description to-LEAF-01-EVPN
   !
   neighbor 10.0.0.2 peer group EVPN-OVERLAY
   neighbor 10.0.0.2 remote-as 65002
   neighbor 10.0.0.2 description to-LEAF-02-EVPN
   !
   neighbor 10.0.0.3 peer group EVPN-OVERLAY
   neighbor 10.0.0.3 remote-as 65003
   neighbor 10.0.0.3 description to-LEAF-03-EVPN
   !
   ! Активируем семейство EVPN
   address-family evpn
      neighbor EVPN-OVERLAY activate
!
end


###Настройки leaf-01

configure terminal
!
! 1. Создаем интерфейс VXLAN и привязываем VLAN 10 к VNI 10010
interface Vxlan1
   vxlan source-interface Loopback0
   vxlan udp-port 4789
   vxlan vlan 10 vni 10010
!
! 2. Настраиваем BGP для поддержки EVPN Overlay и Клиентской Зоны 1
router bgp 65001
   no bgp default ipv4-unicast
   !
   ! Активируем стыковых соседей-спайнов строго в семействе IPv4
   address-family ipv4
      neighbor 172.16.1.2 activate
      neighbor 172.16.1.6 activate
      network 10.0.0.1/32
   !
   ! Создаем Overlay-группу до Спайнов
   neighbor EVPN-SPINES peer group
   neighbor EVPN-SPINES remote-as 65000
   neighbor EVPN-SPINES update-source Loopback0
   neighbor EVPN-SPINES ebgp-multihop 3
   neighbor EVPN-SPINES send-community extended
   !
   ! Привязываем Loopback0-адреса Спайнов к EVPN группе
   neighbor 10.0.1.1 peer group EVPN-SPINES
   neighbor 10.0.1.1 description to-SPINE-01-EVPN
   neighbor 10.0.2.2 peer group EVPN-SPINES
   neighbor 10.0.2.2 description to-SPINE-02-EVPN
   !
   ! Активируем семейство EVPN для обмена MAC-адресами
   address-family evpn
      neighbor EVPN-SPINES activate
   !
   ! Включаем генерацию EVPN маршрутов для Клиентской Зоны 1 (VLAN 10)
   vlan 10
      rd 10.0.0.1:10010
      route-target both 10010:10010
      redistribute learned
!
end

###Настройки leaf-02
configure terminal
!
! 1. Создаем интерфейс VXLAN и сопоставляем VLAN 10 с VNI 10010 (общий VNI для Зоны 1)
interface Vxlan1
   vxlan source-interface Loopback0
   vxlan udp-port 4789
   vxlan vlan 10 vni 10010
!
! 2. Настраиваем BGP для поддержки EVPN Overlay и Клиентской Зоны 1
router bgp 65002
   no bgp default ipv4-unicast
   no redistribute connected route-map RM_red_conn
   !
   ! Активируем стыковых соседей-спайнов строго в семействе IPv4
   address-family ipv4
      neighbor 172.16.2.2 activate
      neighbor 172.16.2.6 activate
      network 10.0.0.2/32
   !
   ! Создаем Overlay-группу до Спайнов
   neighbor EVPN-SPINES peer group
   neighbor EVPN-SPINES remote-as 65000
   neighbor EVPN-SPINES update-source Loopback0
   neighbor EVPN-SPINES ebgp-multihop 3
   neighbor EVPN-SPINES send-community extended
   !
   ! Привязываем Loopback0-адреса Спайнов к EVPN группе
   neighbor 10.0.1.1 peer group EVPN-SPINES
   neighbor 10.0.1.1 description to-SPINE-01-EVPN
   neighbor 10.0.2.2 peer group EVPN-SPINES
   neighbor 10.0.2.2 description to-SPINE-02-EVPN
   !
   ! Активируем семейство EVPN для обмена MAC-адресами клиентов
   address-family evpn
      neighbor EVPN-SPINES activate
   !
   ! Включаем генерацию EVPN маршрутов для Клиентской Зоны 1 (VLAN 10)
   vlan 10
      rd 10.0.0.2:10010
      route-target both 10010:10010
      redistribute learned
!
end

###Настройки leaf-03



configure terminal
!
! 1. Создаем интерфейс VXLAN и привязываем VLAN 10 к единому VNI 10010 для Зоны 1
interface Vxlan1
   vxlan source-interface Loopback0
   vxlan udp-port 4789
   vxlan vlan 10 vni 10010
!
! 2. Настраиваем BGP для поддержки EVPN Overlay и Клиентской Зоны 1
router bgp 65003
   no bgp default ipv4-unicast
   !
   ! Активируем стыковых соседей-спайнов строго в семействе IPv4
   address-family ipv4
      neighbor 172.16.3.2 activate
      neighbor 172.16.3.6 activate
      network 10.0.0.3/32
   !
   ! Создаем Overlay-группу до Спайнов
   neighbor EVPN-SPINES peer group
   neighbor EVPN-SPINES remote-as 65000
   neighbor EVPN-SPINES update-source Loopback0
   neighbor EVPN-SPINES ebgp-multihop 3
   neighbor EVPN-SPINES send-community extended
   !
   ! Привязываем Loopback0-адреса Спайнов к EVPN группе
   neighbor 10.0.1.1 peer group EVPN-SPINES
   neighbor 10.0.1.1 description to-SPINE-01-EVPN
   neighbor 10.0.2.2 peer group EVPN-SPINES
   neighbor 10.0.2.2 description to-SPINE-02-EVPN
   !
   ! Активируем семейство EVPN для обмена MAC-адресами клиентов
   address-family evpn
      neighbor EVPN-SPINES activate
   !
   ! Включаем генерацию EVPN маршрутов для Клиентской Зоны 1 (VLAN 10)
   vlan 10
      rd 10.0.0.3:10010
      route-target both 10010:10010
      redistribute learned
!
end



###Настройка хостов

Client-01: 

VPCS> ip 10.1.11.101 255.255.255.0 10.1.11.1      
Checking for duplicate address...
VPCS : 10.1.11.101 255.255.255.0 gateway 10.1.11.1

VPCS> ping 10.1.11.1                              

84 bytes from 10.1.11.1 icmp_seq=1 ttl=64 time=14.280 ms
84 bytes from 10.1.11.1 icmp_seq=2 ttl=64 time=7.140 ms
84 bytes from 10.1.11.1 icmp_seq=3 ttl=64 time=15.274 ms
84 bytes from 10.1.11.1 icmp_seq=4 ttl=64 time=8.905 ms
84 bytes from 10.1.11.1 icmp_seq=5 ttl=64 time=7.395 ms

VPCS> 


VPCS> 

Client-02:

VPCS> ip 10.1.11.102 255.255.255.0 10.1.11.1
Checking for duplicate address...
VPCS : 10.1.11.102 255.255.255.0 gateway 10.1.11.1

VPCS> ping 10.1.11.1                        

84 bytes from 10.1.11.1 icmp_seq=1 ttl=64 time=12.134 ms
84 bytes from 10.1.11.1 icmp_seq=2 ttl=64 time=9.366 ms
84 bytes from 10.1.11.1 icmp_seq=3 ttl=64 time=10.777 ms
84 bytes from 10.1.11.1 icmp_seq=4 ttl=64 time=9.823 ms
84 bytes from 10.1.11.1 icmp_seq=5 ttl=64 time=7.691 ms



Client -03: 

VPCS> ip 10.1.11.103 255.255.255.0 10.1.11.1
Checking for duplicate address...
VPCS : 10.1.11.103 255.255.255.0 gateway 10.1.11.1

VPCS> ping 10.1.11.1                        

84 bytes from 10.1.11.1 icmp_seq=1 ttl=64 time=6.517 ms
84 bytes from 10.1.11.1 icmp_seq=2 ttl=64 time=7.333 ms
84 bytes from 10.1.11.1 icmp_seq=3 ttl=64 time=8.338 ms
84 bytes from 10.1.11.1 icmp_seq=4 ttl=64 time=7.783 ms
84 bytes from 10.1.11.1 icmp_seq=5 ttl=64 time=8.502 ms

VPCS> 


Client -04: 

VPCS> ip 10.1.11.104 255.255.255.0 10.1.11.1
Checking for duplicate address...
VPCS : 10.1.11.104 255.255.255.0 gateway 10.1.11.1

VPCS> ping 10.1.11.1                        

84 bytes from 10.1.11.1 icmp_seq=1 ttl=64 time=11.438 ms
84 bytes from 10.1.11.1 icmp_seq=2 ttl=64 time=7.675 ms
84 bytes from 10.1.11.1 icmp_seq=3 ttl=64 time=7.447 ms
84 bytes from 10.1.11.1 icmp_seq=4 ttl=64 time=7.706 ms
84 bytes from 10.1.11.1 icmp_seq=5 ttl=64 time=8.798 ms



### Проерка связности 


SPINE-01#sh bgp evpn summ
BGP summary information for VRF default
Router identifier 10.0.1.1, local AS number 65000
Neighbor Status Codes: m - Under maintenance
  Description              Neighbor V AS           MsgRcvd   MsgSent  InQ OutQ  Up/Down State   PfxRcd PfxAcc
  to-LEAF-01-EVPN          10.0.0.1 4 65001             51        52    0    0 00:00:45 Estab   1      1
  to-LEAF-02-EVPN          10.0.0.2 4 65002             43        45    0    0 00:00:44 Estab   1      1
  to-LEAF-03-EVPN          10.0.0.3 4 65003             39        40    0    0 00:00:44 Estab   1      1
SPINE-01#
SPINE-01#


SPINE-02#sh bgp evpn summ
BGP summary information for VRF default
Router identifier 10.0.2.2, local AS number 65000
Neighbor Status Codes: m - Under maintenance
  Description              Neighbor V AS           MsgRcvd   MsgSent  InQ OutQ  Up/Down State   PfxRcd PfxAcc
  to-LEAF-01-EVPN          10.0.0.1 4 65001             39        36    0    0 00:01:33 Estab   1      1
  to-LEAF-02-EVPN          10.0.0.2 4 65002             49        40    0    0 00:02:21 Estab   1      1
  to-LEAF-03-EVPN          10.0.0.3 4 65003             42        34    0    0 00:02:24 Estab   1      1
SPINE-02#
SPINE-02#
SPINE-02#

LEAF-01#sh bgp evpn summ
BGP summary information for VRF default
Router identifier 10.0.0.1, local AS number 65001
Neighbor Status Codes: m - Under maintenance
  Description              Neighbor V AS           MsgRcvd   MsgSent  InQ OutQ  Up/Down State   PfxRcd PfxAcc
  to-SPINE-01-EVPN         10.0.1.1 4 65000             52        50    0    0 00:00:13 Estab   2      2
  to-SPINE-02-EVPN         10.0.2.2 4 65000              7        10    0    0 00:01:11 Estab   2      2
LEAF-01#
LEAF-01#
LEAF-01#
LEAF-01#

LEAF-02#sh bgp evpn summ
BGP summary information for VRF default
Router identifier 10.0.0.2, local AS number 65002
Neighbor Status Codes: m - Under maintenance
  Description              Neighbor V AS           MsgRcvd   MsgSent  InQ OutQ  Up/Down State   PfxRcd PfxAcc
  to-SPINE-01-EVPN         10.0.1.1 4 65000             49        47    0    0 00:03:37 Estab   2      2
  to-SPINE-02-EVPN         10.0.2.2 4 65000             44        53    0    0 00:05:24 Estab   2      2
LEAF-02#

LEAF-03#
LEAF-03#sh bgp evpn summ
BGP summary information for VRF default
Router identifier 10.0.0.3, local AS number 65003
Neighbor Status Codes: m - Under maintenance
  Description              Neighbor V AS           MsgRcvd   MsgSent  InQ OutQ  Up/Down State   PfxRcd PfxAcc
  to-SPINE-01-EVPN         10.0.1.1 4 65000             40        41    0    0 00:00:57 Estab   2      2
  to-SPINE-02-EVPN         10.0.2.2 4 65000             35        43    0    0 00:02:47 Estab   2      2
LEAF-03#
LEAF-03#


проверка интерфейсов vxlan 

LEAF-01#sh inter vxlan1
Vxlan1 is up, line protocol is up (connected)
  Hardware is Vxlan
  Source interface is Loopback0 and is active with 10.0.0.1
  Listening on UDP port 4789
  Replication/Flood Mode is headend with Flood List Source: EVPN
  Remote MAC learning via EVPN
  VNI mapping to VLANs
  Static VLAN to VNI mapping is 
    [10, 10010]      
  Note: All Dynamic VLANs used by VCS are internal VLANs.
        Use 'show vxlan vni' for details.
  Static VRF to VNI mapping is not configured
  Headend replication flood vtep list is:
    10 10.0.0.3        10.0.0.2       
  Shared Router MAC is 0000.0000.0000
LEAF-01#

LEAF-02#sh inter vxlan 1
Vxlan1 is up, line protocol is up (connected)
  Hardware is Vxlan
  Source interface is Loopback0 and is active with 10.0.0.2
  Listening on UDP port 4789
  Replication/Flood Mode is headend with Flood List Source: EVPN
  Remote MAC learning via EVPN
  VNI mapping to VLANs
  Static VLAN to VNI mapping is 
    [10, 10010]      
  Note: All Dynamic VLANs used by VCS are internal VLANs.
        Use 'show vxlan vni' for details.
  Static VRF to VNI mapping is not configured
  Headend replication flood vtep list is:
    10 10.0.0.1        10.0.0.3       
  Shared Router MAC is 0000.0000.0000
LEAF-02#

LEAF-03#
LEAF-03#
LEAF-03#sh inter vxlan 1
Vxlan1 is up, line protocol is up (connected)
  Hardware is Vxlan
  Source interface is Loopback0 and is active with 10.0.0.3
  Listening on UDP port 4789
  Replication/Flood Mode is headend with Flood List Source: EVPN
  Remote MAC learning via EVPN
  VNI mapping to VLANs
  Static VLAN to VNI mapping is 
    [10, 10010]      
  Note: All Dynamic VLANs used by VCS are internal VLANs.
        Use 'show vxlan vni' for details.
  Static VRF to VNI mapping is not configured
  Headend replication flood vtep list is:
    10 10.0.0.1        10.0.0.2       
  Shared Router MAC is 0000.0000.0000
LEAF-03#


### Проверка BGP EVPN Routing Table




SPINE-01#
SPINE-01#
SPINE-01#sh bgp evpn route-type mac-ip
BGP routing table information for VRF default
Router identifier 10.0.1.1, local AS number 65000
Route status codes: * - valid, > - active, S - Stale, E - ECMP head, e - ECMP
                    c - Contributing to ECMP, % - Pending BGP convergence
Origin codes: i - IGP, e - EGP, ? - incomplete
AS Path Attributes: Or-ID - Originator ID, C-LST - Cluster List, LL Nexthop - Link Local Nexthop

          Network                Next Hop              Metric  LocPref Weight  Path
 * >      RD: 10.0.0.1:10010 mac-ip 0050.7966.6806
                                 10.0.0.1              -       100     0       65001 i
 * >      RD: 10.0.0.1:10010 mac-ip 0050.7966.6806 10.1.11.10
                                 10.0.0.1              -       100     0       65001 i
 * >      RD: 10.0.0.1:10010 mac-ip 0050.7966.6806 10.1.11.100
                                 10.0.0.1              -       100     0       65001 i
 * >      RD: 10.0.0.1:10010 mac-ip 0050.7966.6806 10.1.11.101
                                 10.0.0.1              -       100     0       65001 i
 * >      RD: 10.0.0.2:10010 mac-ip 0050.7966.6807
                                 10.0.0.2              -       100     0       65002 i
 * >      RD: 10.0.0.2:10010 mac-ip 0050.7966.6807 10.1.11.102
                                 10.0.0.2              -       100     0       65002 i
 * >      RD: 10.0.0.2:10010 mac-ip 0050.7966.6807 10.1.11.200
                                 10.0.0.2              -       100     0       65002 i
 * >      RD: 10.0.0.3:10010 mac-ip 0050.7966.6808
                                 10.0.0.3              -       100     0       65003 i
 * >      RD: 10.0.0.3:10010 mac-ip 0050.7966.6808 10.1.11.103
                                 10.0.0.3              -       100     0       65003 i
 * >      RD: 10.0.0.3:10010 mac-ip 0050.7966.6809
                                 10.0.0.3              -       100     0       65003 i
 * >      RD: 10.0.0.3:10010 mac-ip 0050.7966.6809 10.1.11.104
                                 10.0.0.3              -       100     0       65003 i
SPINE-01#


