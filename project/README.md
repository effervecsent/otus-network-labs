
### Проектирование РЦОД на основе VxLAN EVPN и построение связности ЦОД и РЦОД


### Цель и архитектурная задача
Основная задача проекта — обеспечить бесшовную миграцию виртуальных машин и серверов из классической трехуровневой сети (левый ЦОД) в современную Clos-фабрику (правый ЦОД) с сохранением их исходных IP-адресов и сетевых настроек.


### Схема сети 

![mh.PNG](mh.PNG)

Для решения этой задачи реализована архитектура Stretched L2 VXLAN over BGP EVPN со следующими ключевыми принципами:
- Единые шлюзы по умолчанию (Default Gateways): На этапе миграции шлюзы для клиентских подсетей (10.10.10.1/24 для VLAN 10 и 20.20.20.1/24 для VLAN 20) терминируются строго в одном месте — на левом Firewall.
  
- 	Изоляция дублирующих шлюзов: На правых коммутаторах доступа (Leaf1–4) интерфейсы Anycast Gateway (Vlan10 и Vlan20) предварительно настроены, но находятся в административно выключенном состоянии (shutdown) до момента полного переезда всех хостов, чтобы исключить конфликт IP-адресов и асимметрию маршрутизации.
  
- Междоменная стыковка (VXLAN Stitching): Обмен EVPN-маршрутами (Type 2 MAC/IP и Type 3 IMET) между автономными системами двух ЦОДов осуществляется через выделенный транзитный узел Route Server (RS) по протоколу eBGP EVPN.


## 2. Адресный план и распределение ролей

### Таблица VLAN и VNI

| Клиентский сегмент | Подсеть | Шлюз по умолчанию | L2 VNI | Внутренний RT (правый ЦОД) | Внешний RT (DCI / RS) |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **VLAN 10** | `10.10.10.0/24` | `10.10.10.1` | `100010` | `10:100010` | `100:100010` |
| **VLAN 20** | `20.20.20.0/24` | `20.20.20.1` | `100020` | `20:100020` | `200:100020` |

### Роли сетевых устройств

| Узел | Роль в топологии | BGP AS | Loopback0 (Router-ID) | Loopback1 (VTEP IP) |
| :--- | :--- | :--- | :--- | :--- |
| **Firewall** | Терминация шлюзов VLAN 10/20, L2-мост и выход в интернет | — (OSPF) | — | — |
| **Sw-Edge-1 / 2** | Транзитный L2-проброс VLAN 10/20 наверх + OSPF/BGP в интернет | `65001` | — | — |
| **BGW1 / BGW2** | Пограничные VTEP-шлюзы левого ЦОДа (Anycast VTEP) | `65001` | `10.1.0.101` / `.102` | `100.100.100.100` |
| **RS** | Транзитный eBGP EVPN Route Server (стык ЦОДов) | `65000` | `10.255.255.23` | — |
| **BGW3 / BGW4** | Пограничные VTEP-шлюзы правого ЦОДа (VXLAN Stitching) | `65002` | `10.2.0.101` / `.102` | `200.200.200.200` |
| **Spine1 / 2** | Магистральные коммутаторы правой Clos-фабрики (Route Reflectors) | `65500` | `10.0.1.1` / `10.0.2.2` | — |
| **Leaf1 / 2** | Коммутаторы доступа правого ЦОДа для **VLAN 10** (MLAG) | `65501` | `10.0.0.1` / `.2` | `10.0.0.112` |
| **Leaf3 / 4** | Коммутаторы доступа правого ЦОДа для **VLAN 20** (EVPN ESI) | `65503` / `65504` | `10.0.0.3` / `.4` | `10.0.0.3` / `.4` |
| **Internet** | Эмулятор внешнего провайдера (`8.8.8.8`, `7.7.7.7`) | `12389` | `8.8.8.8` | `7.7.7.7` |


## 3. Детальное описание настроек по сегментам

### 3.1. Левый ЦОД (Классическая фабрика)

* **Firewall:**
  * Работает в смешанном режиме (SVI поверх L2-транков).
  * Порты вниз к уровню доступа (`Ethernet3`) и вверх к граничным коммутаторам (`Ethernet6`, `Ethernet7`) настроены в режиме `switchport mode trunk` и пропускают клиентские **VLAN 10** и **VLAN 20** транзитом на уровне Layer 2.
  * На `Firewall` подняты активные SVI-интерфейсы `Vlan10` (`10.10.10.1/24`) и `Vlan20` (`20.20.20.1/24`), выполняющие роль единых шлюзов.
  * Для защиты от L2-петель `Firewall` назначен корневым мостом Spanning Tree (`spanning-tree vlan-id 10,20 priority 4096`).
  * По отдельным транзитным SVI (`Vlan3` — `172.18.3.1/30` и `Vlan4` — `172.18.4.1/30`) настроен протокол **OSPF (Area 0)** в сторону `Sw-Edge-1/2` для получения маршрута по умолчанию (`0.0.0.0/0`) в интернет.
* **Sw-Edge-1 и Sw-Edge-2:**
  * На интерфейсах к `Firewall` (`Ethernet6` / `Ethernet7`) и на аплинках к пограничным шлюзам `BGW1/2` (`Ethernet5`) включён чистый L2-транк (`switchport trunk allowed vlan 10,20`) с поддержкой Jumbo Frames (`mtu 9214`).
  * На интерфейсах в сторону провайдера (`AS 12389`) настроены eBGP-сессии, откуда принимается маршрут по умолчанию и редистрибьютится в OSPF (`default-information originate`).
* **BGW1 и BGW2 (Пограничные VTEP левого ЦОДа):**
  * На интерфейсе `Vxlan1` выполнена привязка локальных VLAN к туннельным идентификаторам (`vlan 10 -> vni 100010`, `vlan 20 -> vni 100020`), источник туннелей — `Loopback1` (`100.100.100.100/32`).
  * В процессе `router bgp 65001` настроены MAC-VRF для `vlan 10` и `vlan 20` с экспортом изученных MAC-адресов (`redistribute learned`) и внешними тегами `route-target both 100:100010` и `200:100020`.
  * Настроены eBGP-сессии (в семействах `ipv4` и `evpn`) с Route Server (`192.168.1.2` и `192.168.2.2`), включён `allowas-in 3` и `mtu 9214` для прохождения инкапсулированных VXLAN-пакетов.

### 3.2. Междоменный стык (DCI / Route Server)

* **Route Server (`RS`, `AS 65000`):**
  * Связан прямыми L3-линками (`mtu 9214`) со всеми четырьмя пограничными шлюзами (`BGW1–4`).
  * В семействе `address-family evpn` настроена прозрачная пересылка маршрутов с сохранением оригинальных адресов VTEP (`neighbor ... next-hop-unchanged`) и передачей расширенных сообществ (`send-community extended`).
  * Параметр `next-hop-self` вынесен исключительно в семейство `address-family ipv4`, чтобы избежать отбрасывания маршрутов EVPN Type 2 и Type 3.
  * По протоколу `IPv4 Unicast` RS обеспечивает связность между VTEP-адресами левой и правой площадок (`100.100.100.100/32` и `200.200.200.200/32`).

### 3.3. Правый ЦОД (Clos-фабрика EVPN VXLAN)

* **BGW3 и BGW4 (Пограничные шлюзы переупаковки — VXLAN Stitching):**
  * Выполняют роль границы между внутренней Clos-фабрикой (`AS 65500`) и внешним DCI-доменом (`AS 65000`).
  * Для peer-group `DCI` (в сторону RS) в `address-family evpn` активирован режим `neighbor DCI domain remote`, а для `vlan 10` и `vlan 20` заданы отдельные идентификаторы `rd evpn domain remote 65002:100010` (и `100020`) и внешние таргеты `route-target import export evpn domain remote 100:100010` (и `200:100020`).
  * Это позволяет шлюзам снимать внутренний VXLAN-заголовок от правых Leaf и переупаковывать кадр с новым VTEP Next-Hop (`200.200.200.200`) при отправке в левый ЦОД (и наоборот).
* **Leaf1 и Leaf2 (Обслуживают VLAN 10):**
  * Объединены в отказоустойчивую пару **MLAG** (`Vlan4094`, `Port-Channel78`). Сервер `HOST-1` (`10.10.10.101`) подключён по LACP (`Port-Channel3`) в VLAN 10.
  * Настроен `Vxlan1` (`vlan 10 -> vni 100010`) и прописаны статические маршруты Underlay до пограничного Anycast VTEP `200.200.200.200/32` через оба Spine.
  * Интерфейс `Vlan10` (`ip address virtual 10.10.10.1/24`) переведён в режим `shutdown`.
* **Leaf3 и Leaf4 (Обслуживают VLAN 20):**
  * Используют бескластерный механизм **EVPN Multihoming (ESI `0000:0000:0000:0000:0001`)** без MLAG-линка. Сервер `HOST-2` (`20.20.20.3`) подключён по LACP (`Port-Channel1`) в VLAN 20.
  * Настроен `Vxlan1` (`vlan 20 -> vni 100020`) и маршруты до `200.200.200.200/32` через оба Spine.
  * Интерфейс `Vlan20` (`ip address virtual 20.20.20.1/24`) переведён в режим `shutdown`.

### 3.4. Конфигурация всех устройств: 

- Host11:
  
```
VPCS> sh ip 

NAME        : VPCS[1]
IP/MASK     : 10.10.10.11/24
GATEWAY     : 10.10.10.1
DNS         : 
MAC         : 00:50:79:66:68:0b
LPORT       : 20000
RHOST:PORT  : 127.0.0.1:30000
MTU         : 1500

VPCS> 
```

-Host 12: 
```
VPCS> sh ip 

NAME        : VPCS[1]
IP/MASK     : 20.20.20.12/24
GATEWAY     : 20.20.20.1
DNS         : 
MAC         : 00:50:79:66:68:0c
LPORT       : 20000
RHOST:PORT  : 127.0.0.1:30000
MTU         : 1500


```

- Stack-sw:

```
STACK-sw# sh run
! Command: show running-config
! device: STACK-sw (vEOS-lab, EOS-4.29.2F)
!
! boot system flash:/vEOS-lab.swi
!
no aaa root
!
transceiver qsfp default-mode 4x10G
!
service routing protocols model ribd
!
hostname STACK-sw
!
spanning-tree mode mstp
!
vlan 3-5,10,20
!
interface Ethernet1
   description to-Host11
   mtu 9214
   switchport access vlan 10
   spanning-tree portfast
!
interface Ethernet2
   description to-Host12
   mtu 9214
   switchport access vlan 20
   spanning-tree portfast
!
interface Ethernet3
   description to-Firewall-Trunk
   mtu 9214
   switchport trunk allowed vlan 3-4,10,20
   switchport mode trunk
!
interface Ethernet4
!
interface Ethernet5
!
interface Ethernet6
!
interface Ethernet7
!
interface Ethernet8
!
interface Management1
!
no ip routing
!
end
STACK-sw# 
```


-Firewall:

```

Firewall#ping 20.20.20.
ping: 20.20.20.: Name or service not known
Firewall#
Firewall#
Firewall#sh run
! Command: show running-config
! device: Firewall (vEOS-lab, EOS-4.29.2F)
!
! boot system flash:/vEOS-lab.swi
!
no aaa root
!
transceiver qsfp default-mode 4x10G
!
service routing protocols model multi-agent
!
hostname Firewall
!
spanning-tree mode mstp
spanning-tree vlan-id 10,20 priority 4096
!
vlan 3
   name OSPF-to-Sw-Edge-1
!
vlan 4
   name OSPF-to-Sw-Edge-2
!
vlan 10
   name Host-VLAN-10
!
vlan 20
   name Host-VLAN-20
!
interface Ethernet1
   shutdown
   ip ospf network point-to-point
!
interface Ethernet2
   shutdown
   ip ospf network point-to-point
!
interface Ethernet3
   description to-Stack-SW
   switchport trunk allowed vlan 3-4,10,20
   switchport mode trunk
!
interface Ethernet4
!
interface Ethernet5
!
interface Ethernet6
   description to sw-edge1
   switchport trunk allowed vlan 3,10,20
   switchport mode trunk
!
interface Ethernet7
   description to sw-edge2
   switchport trunk allowed vlan 4,10,20
   switchport mode trunk
!
interface Ethernet8
!
interface Management1
!
interface Vlan3
   description OSPF-SVI-to-Sw-Edge-1
   ip address 172.18.3.1/30
   ip ospf network point-to-point
   ip ospf area 0.0.0.0
!
interface Vlan4
   description OSPF-SVI-to-Sw-Edge-2
   ip address 172.18.4.1/30
   ip ospf network point-to-point
   ip ospf area 0.0.0.0
!
interface Vlan10
   description Gateway-for-Host-VLAN10
   ip address 10.10.10.1/24
!
interface Vlan20
   description Gateway-for-Host-VLAN20
   ip address 20.20.20.1/24
!
ip access-list ACL-FROM-EXTERNAL
   10 permit ospf any any
   20 permit icmp any any
   30 permit tcp any any eq bgp
   40 permit udp any any eq 4789
!
ip access-list ACL-FROM-INTERNAL
   10 permit ospf any any
   20 permit icmp any any
   30 permit tcp any any eq bgp
   40 permit ip 10.10.10.0/24 any
   50 permit ip 20.20.20.0/24 any
   60 permit udp any any eq 4789
!
ip routing
!
router ospf 1
   router-id 10.255.255.255
   passive-interface default
   passive-interface Ethernet1
   passive-interface Ethernet2
   passive-interface Ethernet3
   passive-interface Ethernet6
   passive-interface Ethernet7
   no passive-interface Vlan3
   no passive-interface Vlan4
   no passive-interface Vlan5
   no passive-interface Vlan10
   no passive-interface Vlan20
   network 10.10.10.0/24 area 0.0.0.0
   network 20.20.20.0/24 area 0.0.0.0
   max-lsa 12000
!
end
Firewall#
```


- SW-Edge1:

```
Sw-Edge-1#
Sw-Edge-1#sh run
! Command: show running-config
! device: Sw-Edge-1 (vEOS-lab, EOS-4.29.2F)
!
! boot system flash:/vEOS-lab.swi
!
no aaa root
!
transceiver qsfp default-mode 4x10G
!
service routing protocols model multi-agent
!
hostname Sw-Edge-1
!
spanning-tree mode mstp
!
vlan 3
   name OSPF-to-Firewall
!
vlan 10
   name Host-VLAN-10
!
vlan 20
   name Host-VLAN-20
!
interface Ethernet1
   no switchport
!
interface Ethernet2
   description L3-Cross-link-to-Sw-Edge-2
   no switchport
   ip address 192.168.10.1/30
   ip ospf area 0.0.0.0
!
interface Ethernet3
!
interface Ethernet4
   description to-Internet-Provider
   no switchport
   ip address 91.228.103.30/26
!
interface Ethernet5
   description to-BGW1-Trunk
   mtu 9214
   switchport trunk allowed vlan 10,20
   switchport mode trunk
!
interface Ethernet6
   description to-Firewall-Trunk
   switchport trunk allowed vlan 3,10,20
   switchport mode trunk
!
interface Ethernet7
   mtu 9214
!
interface Ethernet8
!
interface Management1
!
interface Vlan3
   description L3-Routing-to-Firewall
   ip address 172.18.3.2/30
   ip ospf network point-to-point
   ip ospf area 0.0.0.0
!
ip routing
!
router bgp 65001
   router-id 91.228.103.30
   no bgp default ipv4-unicast
   timers bgp 1 3
   neighbor 91.228.103.1 remote-as 12389
   neighbor 91.228.103.1 description to-Internet-Router
   !
   address-family ipv4
      neighbor 91.228.103.1 activate
      network 91.228.103.0/26
!
router ospf 1
   router-id 192.168.10.1
   passive-interface default
   no passive-interface Ethernet1
   no passive-interface Ethernet2
   passive-interface Ethernet5
   no passive-interface Ethernet5.100
   passive-interface Ethernet6
   no passive-interface Vlan3
   passive-interface Vlan4
   no passive-interface Vlan20
   no passive-interface Vlan100
   max-lsa 12000
   default-information originate
!
end
Sw-Edge-1#
```



- Sw-edge2:


```
Sw-Edge-2#sh run
! Command: show running-config
! device: Sw-Edge-2 (vEOS-lab, EOS-4.29.2F)
!
! boot system flash:/vEOS-lab.swi
!
no aaa root
!
transceiver qsfp default-mode 4x10G
!
service routing protocols model multi-agent
!
hostname Sw-Edge-2
!
spanning-tree mode mstp
!
vlan 4
   name OSPF-to-Firewall
!
vlan 10
   name Host-VLAN-10
!
vlan 20
   name Host-VLAN-20
!
interface Ethernet1
   no switchport
!
interface Ethernet2
   description L3-Cross-link-to-Sw-Edge-1
   no switchport
   ip address 192.168.10.2/30
   ip ospf area 0.0.0.0
!
interface Ethernet3
   description to-Internet-Provider
   no switchport
   ip address 91.228.103.130/26
!
interface Ethernet4
!
interface Ethernet5
   description to-BGW-2-Trunk
   mtu 9214
   switchport trunk allowed vlan 10,20
   switchport mode trunk
!
interface Ethernet6
!
interface Ethernet7
   description to-Firewall-Trunk
   mtu 9214
   switchport trunk allowed vlan 4,10,20
   switchport mode trunk
!
interface Ethernet8
!
interface Management1
!
interface Vlan4
   description L3-Routing-to-Firewall
   ip address 172.18.4.2/30
   ip ospf network point-to-point
   ip ospf area 0.0.0.0
!
ip routing
!
router bgp 65001
   router-id 91.228.103.130
   no bgp default ipv4-unicast
   timers bgp 1 3
   neighbor 91.228.103.129 remote-as 12389
   neighbor 91.228.103.129 description to-Internet-Router
   !
   address-family ipv4
      neighbor 91.228.103.129 activate
      network 91.228.103.128/26
!
router ospf 1
   router-id 192.168.10.2
   passive-interface default
   no passive-interface Ethernet1
   no passive-interface Ethernet2
   passive-interface Ethernet5
   no passive-interface Ethernet5.100
   passive-interface Ethernet7
   no passive-interface Vlan4
   passive-interface Vlan5
   no passive-interface Vlan20
   no passive-interface Vlan100
   max-lsa 12000
   default-information originate
!
end
 Sw-Edge-2#      

```

- BGW 1:

  
```

BGW1#sh run
! Command: show running-config
 ! device: BGW1 (vEOS-lab, EOS-4.29.2F)
!
! boot system flash:/vEOS-lab.swi
!
no aaa root
!
transceiver qsfp default-mode 4x10G
!
service routing protocols model multi-agent
!
hostname BGW1
!
spanning-tree mode mstp
!
vlan 10,20
!
interface Ethernet1
   description to-RS
   mtu 9214
   no switchport
   ip address 192.168.1.1/30
!
interface Ethernet2
!
interface Ethernet3
!
interface Ethernet4
!
interface Ethernet5
   description to_SW-EDge1
   mtu 9214
   switchport trunk allowed vlan 10,20
   switchport mode trunk
!
interface Ethernet6
!
interface Ethernet7
!
interface Ethernet8
!
interface Loopback0
   description BGP-EVPN
   ip address 10.1.0.101/32
!
interface Loopback1
   ip address 100.100.100.100/32
!
interface Management1
!
interface Vxlan1
   vxlan source-interface Loopback1
   vxlan udp-port 4789
   vxlan vlan 10 vni 100010
   vxlan vlan 20 vni 100020
   vxlan learn-restrict any
!
ip virtual-router mac-address 00:1c:73:00:00:10
!
ip routing
!
ip route 10.1.0.102/32 192.168.1.2
!
router bgp 65001
   router-id 10.1.0.101
   no bgp default ipv4-unicast
   distance bgp 20 200 200
   maximum-paths 2 ecmp 2
   neighbor DCI peer group
   neighbor DCI remote-as 65000
   neighbor DCI allowas-in 3
   neighbor DCI ebgp-multihop 3
   neighbor DCI send-community extended
   neighbor 10.1.0.102 remote-as 65001
   neighbor 10.1.0.102 update-source Loopback0
   neighbor 10.1.0.102 timers 60 180
   neighbor 10.1.0.102 send-community extended
   neighbor 192.168.1.2 peer group DCI
   neighbor 192.168.1.2 allowas-in 3
   neighbor 192.168.1.2 timers 60 180
   !
   vlan 10
      rd 65001:100010
      route-target both 100:100010
      redistribute learned
   !
   vlan 20
      rd 65001:100020
      route-target both 200:100020
      redistribute learned
   !
   address-family evpn
      neighbor DCI activate
      neighbor 10.1.0.102 activate
   !
   address-family ipv4
      neighbor DCI activate
      neighbor 10.1.0.102 activate
      network 10.1.0.101/32
      network 100.100.100.100/32
!
end
BGW1#      
```


- BGW 2:

  
```
BGW2#
BGW2#sh run
! Command: show running-config
! device: BGW2 (vEOS-lab, EOS-4.29.2F)
!
! boot system flash:/vEOS-lab.swi
!
no aaa root
!
transceiver qsfp default-mode 4x10G
!
service routing protocols model multi-agent
!
hostname BGW2
!
spanning-tree mode mstp
!
vlan 10,20
!
interface Ethernet1
   description to-RS
   mtu 9214
   no switchport
   ip address 192.168.2.1/30
!
interface Ethernet2
!
interface Ethernet3
!
interface Ethernet4
!
interface Ethernet5
   description to_SW-EDge2
   mtu 9214
   switchport trunk allowed vlan 10,20
   switchport mode trunk
!
interface Ethernet6
!
interface Ethernet7
!
interface Ethernet8
!
interface Loopback0
   description BGP-EVPN
   ip address 10.1.0.102/32
!
interface Loopback1
   ip address 100.100.100.100/32
!
interface Management1
!
interface Vxlan1
   vxlan source-interface Loopback1
   vxlan udp-port 4789
   vxlan vlan 10 vni 100010
   vxlan vlan 20 vni 100020
   vxlan learn-restrict any
!
ip virtual-router mac-address 00:1c:73:00:00:10
!
ip routing
!
ip route 10.1.0.101/32 192.168.2.2
!
router bgp 65001
   router-id 10.1.0.102
   no bgp default ipv4-unicast
   distance bgp 20 200 200
   maximum-paths 2 ecmp 2
   neighbor DCI peer group
   neighbor DCI remote-as 65000
   neighbor DCI allowas-in 3
   neighbor DCI ebgp-multihop 3
   neighbor DCI send-community extended
   neighbor 10.1.0.101 remote-as 65001
   neighbor 10.1.0.101 update-source Loopback0
   neighbor 10.1.0.101 timers 60 180
   neighbor 10.1.0.101 send-community extended
   neighbor 192.168.2.2 peer group DCI
   neighbor 192.168.2.2 allowas-in 3
   neighbor 192.168.2.2 timers 60 180
   !
   vlan 10
      rd 65001:100010
      route-target both 100:100010
      redistribute learned
   !
   vlan 20
      rd 65001:100020
      route-target both 200:100020
      redistribute learned
   !
   address-family evpn
      neighbor DCI activate
      neighbor 10.1.0.101 activate
   !
   address-family ipv4
      neighbor DCI activate
      neighbor 10.1.0.101 activate
      network 10.1.0.102/32
      network 100.100.100.100/32
!
end
BGW2# 
```


- RS:


```
RS#
RS#sh run
! Command: show running-config
! device: RS (vEOS-lab, EOS-4.29.2F)
!
! boot system flash:/vEOS-lab.swi
!
no aaa root
!
transceiver qsfp default-mode 4x10G
!
service routing protocols model multi-agent
!
hostname RS
!
spanning-tree mode mstp
!
interface Ethernet1
   description to-BGW1 (eBGP, AS 65001)
   mtu 9214
   no switchport
   ip address 192.168.1.2/30
!
interface Ethernet2
   description to-BGW2 (eBGP, AS 65001)
   mtu 9214
   no switchport
   ip address 192.168.2.2/30
!
interface Ethernet3
   description to-BGW3 (eBGP, AS 65002)
   mtu 9214
   no switchport
   ip address 192.168.3.2/30
!
interface Ethernet4
   description to-BGW4 (eBGP, AS 65002)
   mtu 9214
   no switchport
   ip address 192.168.4.2/30
!
interface Ethernet5
!
interface Ethernet6
!
interface Ethernet7
!
interface Ethernet8
!
interface Loopback0
   description RS-Router-ID
   ip address 10.255.255.23/32
!
interface Management1
!
ip routing
!
route-map PERMIT-ALL permit 10
!
router bgp 65000
   router-id 10.255.255.23
   no bgp default ipv4-unicast
   distance bgp 20 200 200
   maximum-paths 4 ecmp 4
   neighbor BGW-LEFT-eBGP peer group
   neighbor BGW-LEFT-eBGP remote-as 65001
   neighbor BGW-LEFT-eBGP send-community extended
   neighbor BGW-RIGHT-eBGP peer group
   neighbor BGW-RIGHT-eBGP remote-as 65002
   neighbor BGW-RIGHT-eBGP send-community extended
   neighbor 192.168.1.1 peer group BGW-LEFT-eBGP
   neighbor 192.168.2.1 peer group BGW-LEFT-eBGP
   neighbor 192.168.3.1 peer group BGW-RIGHT-eBGP
   neighbor 192.168.4.1 peer group BGW-RIGHT-eBGP
   !
   address-family evpn
      neighbor BGW-LEFT-eBGP activate
      neighbor BGW-LEFT-eBGP next-hop-unchanged
      neighbor BGW-RIGHT-eBGP activate
      neighbor BGW-RIGHT-eBGP next-hop-unchanged
   !
   address-family ipv4
      neighbor BGW-LEFT-eBGP activate
      neighbor BGW-LEFT-eBGP next-hop-self
      neighbor BGW-RIGHT-eBGP activate
      neighbor BGW-RIGHT-eBGP next-hop-self
!
end
RS# 
```


- BGW 3:


```
BGW3#sh run
! Command: show running-config
! device: BGW3 (vEOS-lab, EOS-4.29.2F)
!
! boot system flash:/vEOS-lab.swi
!
no aaa root
!
transceiver qsfp default-mode 4x10G
!
service routing protocols model multi-agent
!
hostname BGW3
!
spanning-tree mode mstp
!
vlan 10,20
!
interface Ethernet1
   description to-RS
   mtu 9214
   no switchport
   ip address 192.168.3.1/30
!
interface Ethernet2
!
interface Ethernet3
!
interface Ethernet4
!
interface Ethernet5
   description to-SPINE-01
   mtu 9214
   no switchport
   ip address 172.16.5.1/30
!
interface Ethernet6
   description to-SPINE-02
   mtu 9214
   no switchport
   ip address 172.16.7.1/30
!
interface Ethernet7
!
interface Ethernet8
!
interface Loopback0
   description BGP-EVPN-Router-ID
   ip address 10.2.0.101/32
!
interface Loopback1
   description Anycast-VTEP
   ip address 200.200.200.200/32
!
interface Management1
!
interface Vxlan1
   vxlan source-interface Loopback1
   vxlan udp-port 4789
   vxlan vlan 10 vni 100010
   vxlan vlan 20 vni 100020
   vxlan learn-restrict any
!
ip routing
!
router bgp 65002
   router-id 10.2.0.101
   no bgp default ipv4-unicast
   distance bgp 20 200 200
   maximum-paths 2 ecmp 2
   neighbor DCI peer group
   neighbor DCI remote-as 65000
   neighbor DCI send-community extended
   neighbor EVPN peer group
   neighbor EVPN remote-as 65500
   neighbor EVPN update-source Loopback0
   neighbor EVPN ebgp-multihop 3
   neighbor EVPN send-community extended
   neighbor UNDERLAY peer group
   neighbor UNDERLAY remote-as 65500
   neighbor UNDERLAY allowas-in 1
   neighbor 10.0.1.1 peer group EVPN
   neighbor 10.0.2.2 peer group EVPN
   neighbor 10.2.0.102 remote-as 65002
   neighbor 10.2.0.102 update-source Loopback0
   neighbor 10.2.0.102 description iBGP-to-BGW4-via-Spines
   neighbor 172.16.5.2 peer group UNDERLAY
   neighbor 172.16.7.2 peer group UNDERLAY
   neighbor 192.168.3.2 peer group DCI
   !
   vlan 10
      rd 65002:10
      rd evpn domain remote 65002:100010
      route-target both 10:100010
      route-target import export evpn domain remote 100:100010
      redistribute learned
   !
   vlan 20
      rd 65002:20
      rd evpn domain remote 65002:100020
      route-target both 20:100020
      route-target import export evpn domain remote 200:100020
      redistribute learned
   !
   address-family evpn
      neighbor DCI activate
      neighbor DCI domain remote
      neighbor EVPN activate
      neighbor 10.0.1.1 activate
      neighbor 10.0.2.2 activate
      neighbor 10.2.0.102 activate
   !
   address-family ipv4
      neighbor DCI activate
      neighbor UNDERLAY activate
      neighbor UNDERLAY next-hop-self
      neighbor 10.2.0.102 activate
      network 10.2.0.101/32
      network 200.200.200.200/32
!
end
BGW3#
```

- BGW 4:
```
BGW4#sh run
! Command: show running-config
! device: BGW4 (vEOS-lab, EOS-4.29.2F)
!
! boot system flash:/vEOS-lab.swi
!
no aaa root
!
transceiver qsfp default-mode 4x10G
!
service routing protocols model multi-agent
!
hostname BGW4
!
spanning-tree mode mstp
!
vlan 10,20
!
interface Ethernet1
   description to-RS
   mtu 9214
   no switchport
   ip address 192.168.4.1/30
!
interface Ethernet2
!
interface Ethernet3
!
interface Ethernet4
!
interface Ethernet5
   description to-SPINE-02
   mtu 9214
   no switchport
   ip address 172.16.8.1/30
!
interface Ethernet6
   description to-SPINE-01
   mtu 9214
   no switchport
   ip address 172.16.6.1/30
!
interface Ethernet7
!
interface Ethernet8
!
interface Loopback0
   description BGP-EVPN-Router-ID
   ip address 10.2.0.102/32
!
interface Loopback1
   description Anycast-VTEP ( BGW3)
   ip address 200.200.200.200/32
!
interface Management1
!
interface Vxlan1
   vxlan source-interface Loopback1
   vxlan udp-port 4789
   vxlan vlan 10 vni 100010
   vxlan vlan 20 vni 100020
   vxlan learn-restrict any
!
ip routing
!
router bgp 65002
   router-id 10.2.0.102
   no bgp default ipv4-unicast
   distance bgp 20 200 200
   maximum-paths 2 ecmp 2
   neighbor DCI peer group
   neighbor DCI remote-as 65000
   neighbor DCI send-community extended
   neighbor EVPN peer group
   neighbor EVPN remote-as 65500
   neighbor EVPN update-source Loopback0
   neighbor EVPN ebgp-multihop 3
   neighbor EVPN send-community extended
   neighbor UNDERLAY peer group
   neighbor UNDERLAY remote-as 65500
   neighbor UNDERLAY allowas-in 1
   neighbor 10.0.1.1 peer group EVPN
   neighbor 10.0.2.2 peer group EVPN
   neighbor 10.2.0.101 remote-as 65002
   neighbor 10.2.0.101 update-source Loopback0
   neighbor 10.2.0.101 description iBGP-to-BGW3-via-Spines
   neighbor 172.16.6.2 peer group UNDERLAY
   neighbor 172.16.8.2 peer group UNDERLAY
   neighbor 192.168.4.2 peer group DCI
   !
   vlan 10
      rd 65002:10
      rd evpn domain remote 65002:100010
      route-target both 10:100010
      route-target import export evpn domain remote 100:100010
      redistribute learned
   !
   vlan 20
      rd 65002:20
      rd evpn domain remote 65002:100020
      route-target both 20:100020
      route-target import export evpn domain remote 200:100020
      redistribute learned
   !
   address-family evpn
      neighbor DCI activate
      neighbor DCI domain remote
      neighbor EVPN activate
      neighbor 10.0.1.1 activate
      neighbor 10.0.2.2 activate
      neighbor 10.2.0.101 activate
   !
   address-family ipv4
      neighbor DCI activate
      neighbor UNDERLAY activate
      neighbor UNDERLAY next-hop-self
      neighbor 10.2.0.101 activate
      network 10.2.0.102/32
      network 200.200.200.200/32
!
end
```

- Spine 1:


```
Spine1#sh run
! Command: show running-config
! device: Spine1 (vEOS-lab, EOS-4.29.2F)
!
! boot system flash:/vEOS-lab.swi
!
no aaa root
!
transceiver qsfp default-mode 4x10G
!
service routing protocols model multi-agent
!
hostname Spine1
!
spanning-tree mode mstp
!
interface Ethernet1
   description connected-to-Leaf1-Ethernet1
   mtu 9214
   no switchport
   ip address 172.16.1.2/30
!
interface Ethernet2
   description connected-to-Leaf2-Ethernet1
   mtu 9214
   no switchport
   ip address 172.16.2.2/30
!
interface Ethernet3
   description connected-to-Leaf3-Ethernet1
   mtu 9214
   no switchport
   ip address 172.16.3.2/30
!
interface Ethernet4
   description connected-to-Leaf4-Ethernet1
   mtu 9214
   no switchport
   ip address 172.16.4.2/30
!
interface Ethernet5
   description connected-to-BGW3-Ethernet5
   mtu 9214
   no switchport
   ip address 172.16.5.2/30
!
interface Ethernet6
   description connected-to-BGW4-Ethernet6
   mtu 9214
   no switchport
   ip address 172.16.6.2/30
!
interface Ethernet7
!
interface Ethernet8
!
interface Loopback0
   ip address 10.0.1.1/32
!
interface Management1
!
ip routing
!
peer-filter LEAVES_ASN
   10 match as-range 65501-65535 result accept
!
router bgp 65500
   router-id 10.0.1.1
   no bgp default ipv4-unicast
   distance bgp 20 200 200
   bgp listen range 10.0.0.0/16 peer-group EVPN peer-filter LEAVES_ASN
   bgp listen range 172.16.0.0/16 peer-group UNDERLAY peer-filter LEAVES_ASN
   neighbor EVPN peer group
   neighbor EVPN next-hop-unchanged
   neighbor EVPN update-source Loopback0
   neighbor EVPN ebgp-multihop 3
   neighbor EVPN send-community extended
   neighbor UNDERLAY peer group
   neighbor 10.0.0.3 peer group EVPN
   neighbor 10.0.0.3 remote-as 65503
   neighbor 10.0.0.3 description Static-EVPN-to-Leaf3
   neighbor 10.0.0.4 peer group EVPN
   neighbor 10.0.0.4 remote-as 65504
   neighbor 10.0.0.4 description Static-EVPN-to-Leaf4
   neighbor 10.2.0.101 peer group EVPN
   neighbor 10.2.0.101 remote-as 65002
   neighbor 10.2.0.101 description Static-EVPN-to-BGW3
   neighbor 10.2.0.102 peer group EVPN
   neighbor 10.2.0.102 remote-as 65002
   neighbor 10.2.0.102 description Static-EVPN-to-BGW4
   neighbor 172.16.5.1 remote-as 65002
   neighbor 172.16.5.1 description Static-Underlay-to-BGW3
   neighbor 172.16.6.1 remote-as 65002
   neighbor 172.16.6.1 description Static-Underlay-to-BGW4
   !
   address-family evpn
      neighbor EVPN activate
      neighbor EVPN next-hop-unchanged
      neighbor 10.0.0.3 activate
      neighbor 10.0.0.4 activate
      neighbor 10.2.0.101 activate
      neighbor 10.2.0.102 activate
   !
   address-family ipv4
      neighbor UNDERLAY activate
      neighbor UNDERLAY next-hop-unchanged
      neighbor 172.16.5.1 activate
      neighbor 172.16.6.1 activate
      network 10.0.1.1/32
      network 172.16.5.0/30
      network 172.16.6.0/30
!
end
Spine1#  
```

- Spine 2:


```
Spine2#wr mem 
Copy completed successfully.
Spine2#sh run
! Command: show running-config
! device: Spine2 (vEOS-lab, EOS-4.29.2F)
!
! boot system flash:/vEOS-lab.swi
!
no aaa root
!
transceiver qsfp default-mode 4x10G
!
service routing protocols model multi-agent
!
hostname Spine2
!
spanning-tree mode mstp
!
interface Ethernet1
   description connected-to-Leaf1-Ethernet2
   mtu 9214
   no switchport
   ip address 172.16.1.6/30
!
interface Ethernet2
   description connected-to-Leaf2-Ethernet2
   mtu 9214
   no switchport
   ip address 172.16.2.6/30
!
interface Ethernet3
   description connected-to-Leaf3-Ethernet2
   mtu 9214
   no switchport
   ip address 172.16.3.6/30
!
interface Ethernet4
   description connected-to-Leaf4-Ethernet2
   mtu 9214
   no switchport
   ip address 172.16.4.6/30
!
interface Ethernet5
   description connected-to-BGW4-Ethernet5
   mtu 9214
   no switchport
   ip address 172.16.8.2/30
!
interface Ethernet6
   description connected-to-BGW3-Ethernet6
   mtu 9214
   no switchport
   ip address 172.16.7.2/30
!
interface Ethernet7
!
interface Ethernet8
!
interface Loopback0
   ip address 10.0.2.2/32
!
interface Management1
!
ip routing
!
peer-filter LEAVES_ASN
   10 match as-range 65501-65535 result accept
!
router bgp 65500
   router-id 10.0.2.2
   no bgp default ipv4-unicast
   distance bgp 20 200 200
   bgp listen range 10.0.0.0/16 peer-group EVPN peer-filter LEAVES_ASN
   bgp listen range 172.16.0.0/16 peer-group UNDERLAY peer-filter LEAVES_ASN
   neighbor EVPN peer group
   neighbor EVPN next-hop-unchanged
   neighbor EVPN update-source Loopback0
   neighbor EVPN ebgp-multihop 3
   neighbor EVPN send-community extended
   neighbor UNDERLAY peer group
   neighbor 10.0.0.3 peer group EVPN
   neighbor 10.0.0.3 remote-as 65503
   neighbor 10.0.0.3 description Static-EVPN-to-Leaf3
   neighbor 10.0.0.4 peer group EVPN
   neighbor 10.0.0.4 remote-as 65504
   neighbor 10.0.0.4 description Static-EVPN-to-Leaf4
   neighbor 10.2.0.101 peer group EVPN
   neighbor 10.2.0.101 remote-as 65002
   neighbor 10.2.0.101 description Static-EVPN-to-BGW3
   neighbor 10.2.0.102 peer group EVPN
   neighbor 10.2.0.102 remote-as 65002
   neighbor 10.2.0.102 description Static-EVPN-to-BGW4
   neighbor 172.16.7.1 remote-as 65002
   neighbor 172.16.7.1 description Static-Underlay-to-BGW3
   neighbor 172.16.8.1 remote-as 65002
   neighbor 172.16.8.1 description Static-Underlay-to-BGW4
   !
   address-family evpn
      neighbor EVPN activate
      neighbor EVPN next-hop-unchanged
      neighbor 10.0.0.3 activate
      neighbor 10.0.0.4 activate
      neighbor 10.2.0.101 activate
      neighbor 10.2.0.102 activate
   !
   address-family ipv4
      neighbor UNDERLAY activate
      neighbor UNDERLAY next-hop-unchanged
      neighbor 172.16.7.1 activate
      neighbor 172.16.8.1 activate
      network 10.0.2.2/32
      network 172.16.7.0/30
      network 172.16.8.0/30
!
end
Spine2#
```

- Leaf 1:


```
Leaf1#write memory
Copy completed successfully.
Leaf1#sh run
! Command: show running-config
! device: Leaf1 (vEOS-lab, EOS-4.29.2F)
!
! boot system flash:/vEOS-lab.swi
!
no aaa root
!
transceiver qsfp default-mode 4x10G
!
service routing protocols model multi-agent
!
no logging console
!
hostname Leaf1
!
spanning-tree mode mstp
no spanning-tree vlan-id 4094
!
vlan 10
   name VL10
!
vlan 4094
   name MLAG-PEERLINK
   trunk group MLAG-PEERLINK
!
vrf instance MGMT
!
vrf instance VRF1
!
interface Port-Channel3
   description ---Link-to-Server1---
   switchport trunk allowed vlan 10
   switchport mode trunk
   mlag 1
!
interface Port-Channel78
   description MLAG-PEERLINK
   switchport mode trunk
   switchport trunk group MLAG-PEERLINK
   spanning-tree link-type point-to-point
!
interface Ethernet1
   description connected-to-Spine1-Ethernet1
   mtu 9214
   no switchport
   ip address 172.16.1.1/30
!
interface Ethernet2
   description connected-to-Spine2-Ethernet1
   mtu 9214
   no switchport
   ip address 172.16.1.5/30
!
interface Ethernet3
   description ---Link-to-Server1---
   switchport mode trunk
   channel-group 3 mode active
!
interface Ethernet4
!
interface Ethernet5
!
interface Ethernet6
!
interface Ethernet7
   description MLAG-PEERLINK
   channel-group 78 mode active
!
interface Ethernet8
   description MLAG-PEERLINK
   channel-group 78 mode active
!
interface Loopback0
   ip address 10.0.0.1/32
!
interface Loopback1
   description VXLAN-VTEP
   ip address 10.0.0.112/32
!
interface Management1
   vrf MGMT
   ip address 192.168.0.1/30
!
interface Vlan10
   shutdown
   vrf VRF1
   ip address virtual 10.10.10.1/24
!
interface Vlan4094
   ip address 172.16.101.1/30
!
interface Vxlan1
   vxlan source-interface Loopback1
   vxlan udp-port 4789
   vxlan vlan 10 vni 100010
   vxlan vrf VRF1 vni 111111
   vxlan learn-restrict any
!
ip virtual-router mac-address 00:1c:73:00:00:10
!
ip routing
ip routing vrf MGMT
ip routing vrf VRF1
!
mlag configuration
   domain-id LEAVES-1-2
   local-interface Vlan4094
   peer-address 172.16.101.2
   peer-address heartbeat 192.168.0.2 vrf MGMT
   peer-link Port-Channel78
   dual-primary detection delay 1 action errdisable all-interfaces
!
ip route 10.0.0.3/32 172.16.1.2
ip route 10.0.0.3/32 172.16.1.6
ip route 10.0.0.4/32 172.16.1.2
ip route 10.0.0.4/32 172.16.1.6
ip route 200.200.200.200/32 172.16.1.2
ip route 200.200.200.200/32 172.16.1.6
!
router bgp 65501
   router-id 10.0.0.1
   no bgp default ipv4-unicast
   timers bgp 10 30
   distance bgp 20 200 200
   maximum-paths 2 ecmp 2
   neighbor EVPN peer group
   neighbor EVPN remote-as 65500
   neighbor EVPN update-source Loopback0
   neighbor EVPN ebgp-multihop 3
   neighbor EVPN send-community extended
   neighbor UNDERLAY peer group
   neighbor UNDERLAY remote-as 65500
   neighbor UNDERLAY-MLAG peer group
   neighbor UNDERLAY-MLAG remote-as 65501
   neighbor UNDERLAY-MLAG next-hop-self
   neighbor 10.0.1.1 peer group EVPN
   neighbor 10.0.2.2 peer group EVPN
   neighbor 172.16.1.2 peer group UNDERLAY
   neighbor 172.16.1.6 peer group UNDERLAY
   neighbor 172.16.101.2 peer group UNDERLAY-MLAG
   !
   vlan 10
      rd auto
      route-target both 10:100010
      redistribute learned
   !
   address-family evpn
      neighbor EVPN activate
   !
   address-family ipv4
      neighbor UNDERLAY activate
      neighbor UNDERLAY-MLAG activate
      network 10.0.0.1/32
      network 10.0.0.112/32
   !
   vrf VRF1
      rd 65501:111111
      route-target import evpn 1:111111
      route-target export evpn 1:111111
      redistribute connected
!
end
```

- Leaf 2:
```
Leaf1#write memory
Copy completed successfully.
Leaf1#sh run
! Command: show running-config
! device: Leaf1 (vEOS-lab, EOS-4.29.2F)
!
! boot system flash:/vEOS-lab.swi
!
no aaa root
!
transceiver qsfp default-mode 4x10G
!
service routing protocols model multi-agent
!
no logging console
!
hostname Leaf1
!
spanning-tree mode mstp
no spanning-tree vlan-id 4094
!
vlan 10
   name VL10
!
vlan 4094
   name MLAG-PEERLINK
   trunk group MLAG-PEERLINK
!
vrf instance MGMT
!
vrf instance VRF1
!
interface Port-Channel3
   description ---Link-to-Server1---
   switchport trunk allowed vlan 10
   switchport mode trunk
   mlag 1
!
interface Port-Channel78
   description MLAG-PEERLINK
   switchport mode trunk
   switchport trunk group MLAG-PEERLINK
   spanning-tree link-type point-to-point
!
interface Ethernet1
   description connected-to-Spine1-Ethernet1
   mtu 9214
   no switchport
   ip address 172.16.1.1/30
!
interface Ethernet2
   description connected-to-Spine2-Ethernet1
   mtu 9214
   no switchport
   ip address 172.16.1.5/30
!
interface Ethernet3
   description ---Link-to-Server1---
   switchport mode trunk
   channel-group 3 mode active
!
interface Ethernet4
!
interface Ethernet5
!
interface Ethernet6
!
interface Ethernet7
   description MLAG-PEERLINK
   channel-group 78 mode active
!
interface Ethernet8
   description MLAG-PEERLINK
   channel-group 78 mode active
!
interface Loopback0
   ip address 10.0.0.1/32
!
interface Loopback1
   description VXLAN-VTEP
   ip address 10.0.0.112/32
!
interface Management1
   vrf MGMT
   ip address 192.168.0.1/30
!
interface Vlan10
   shutdown
   vrf VRF1
   ip address virtual 10.10.10.1/24
!
interface Vlan4094
   ip address 172.16.101.1/30
!
interface Vxlan1
   vxlan source-interface Loopback1
   vxlan udp-port 4789
   vxlan vlan 10 vni 100010
   vxlan vrf VRF1 vni 111111
   vxlan learn-restrict any
!
ip virtual-router mac-address 00:1c:73:00:00:10
!
ip routing
ip routing vrf MGMT
ip routing vrf VRF1
!
mlag configuration
   domain-id LEAVES-1-2
   local-interface Vlan4094
   peer-address 172.16.101.2
   peer-address heartbeat 192.168.0.2 vrf MGMT
   peer-link Port-Channel78
   dual-primary detection delay 1 action errdisable all-interfaces
!
ip route 10.0.0.3/32 172.16.1.2
ip route 10.0.0.3/32 172.16.1.6
ip route 10.0.0.4/32 172.16.1.2
ip route 10.0.0.4/32 172.16.1.6
ip route 200.200.200.200/32 172.16.1.2
ip route 200.200.200.200/32 172.16.1.6
!
router bgp 65501
   router-id 10.0.0.1
   no bgp default ipv4-unicast
   timers bgp 10 30
   distance bgp 20 200 200
   maximum-paths 2 ecmp 2
   neighbor EVPN peer group
   neighbor EVPN remote-as 65500
   neighbor EVPN update-source Loopback0
   neighbor EVPN ebgp-multihop 3
   neighbor EVPN send-community extended
   neighbor UNDERLAY peer group
   neighbor UNDERLAY remote-as 65500
   neighbor UNDERLAY-MLAG peer group
   neighbor UNDERLAY-MLAG remote-as 65501
   neighbor UNDERLAY-MLAG next-hop-self
   neighbor 10.0.1.1 peer group EVPN
   neighbor 10.0.2.2 peer group EVPN
   neighbor 172.16.1.2 peer group UNDERLAY
   neighbor 172.16.1.6 peer group UNDERLAY
   neighbor 172.16.101.2 peer group UNDERLAY-MLAG
   !
   vlan 10
      rd auto
      route-target both 10:100010
      redistribute learned
   !
   address-family evpn
      neighbor EVPN activate
   !
   address-family ipv4
      neighbor UNDERLAY activate
      neighbor UNDERLAY-MLAG activate
      network 10.0.0.1/32
      network 10.0.0.112/32
   !
   vrf VRF1
      rd 65501:111111
      route-target import evpn 1:111111
      route-target export evpn 1:111111
      redistribute connected
!
end
```

- Leaf 3:


```
Leaf3#sh run
! Command: show running-config
! device: Leaf3 (vEOS-lab, EOS-4.29.2F)
!
! boot system flash:/vEOS-lab.swi
!
no aaa root
!
transceiver qsfp default-mode 4x10G
!
service routing protocols model multi-agent
!
link tracking group CORE-TRACKING
   recovery delay 1
!
hostname Leaf3
!
spanning-tree mode mstp
!
vlan 20
   name VL20
!
vrf instance VRF1
!
interface Port-Channel1
   description Link_to_host3
   switchport trunk allowed vlan 20
   switchport mode trunk
   !
   evpn ethernet-segment
      identifier 0000:0000:0000:0000:0001
      designated-forwarder election algorithm preference 20
      route-target import 00:00:00:00:00:01
   lacp system-id 1111.2222.3333
!
interface Ethernet1
   description connected-to-Spine1-Ethernet3
   mtu 9214
   no switchport
   ip address 172.16.3.1/30
   link tracking group CORE-TRACKING upstream
!
interface Ethernet2
   description connected-to-Spine2-Ethernet3
   mtu 9214
   no switchport
   ip address 172.16.3.5/30
   link tracking group CORE-TRACKING upstream
!
interface Ethernet3
   description ---Link-to-Server3---
   switchport mode trunk
   channel-group 1 mode active
   link tracking group CORE-TRACKING downstream
!
interface Ethernet4
!
interface Ethernet5
!
interface Ethernet6
!
interface Ethernet7
!
interface Ethernet8
!
interface Loopback0
   ip address 10.0.0.3/32
!
interface Management1
!
interface Vlan20
   shutdown
   vrf VRF1
   ip address virtual 20.20.20.1/24
!
interface Vxlan1
   vxlan source-interface Loopback0
   vxlan udp-port 4789
   vxlan vlan 20 vni 100020
   vxlan vrf VRF1 vni 111111
   vxlan learn-restrict any
!
ip virtual-router mac-address 00:1c:73:00:00:10
!
ip routing
ip routing vrf VRF1
!
ip route 10.0.0.112/32 172.16.3.2
ip route 10.0.0.112/32 172.16.3.6
ip route 10.0.1.1/32 172.16.3.2
ip route 10.0.2.2/32 172.16.3.6
ip route 200.200.200.200/32 172.16.3.2
ip route 200.200.200.200/32 172.16.3.6
!
router bgp 65503
   router-id 10.0.0.3
   no bgp default ipv4-unicast
   distance bgp 20 200 200
   maximum-paths 2 ecmp 2
   neighbor EVPN peer group
   neighbor EVPN remote-as 65500
   neighbor EVPN update-source Loopback0
   neighbor EVPN ebgp-multihop 3
   neighbor EVPN send-community extended
   neighbor UNDERLAY peer group
   neighbor UNDERLAY remote-as 65500
   neighbor 10.0.1.1 peer group EVPN
   neighbor 10.0.2.2 peer group EVPN
   neighbor 172.16.3.2 peer group UNDERLAY
   neighbor 172.16.3.6 peer group UNDERLAY
   !
   vlan 20
      rd auto
      route-target both 20:100020
      redistribute learned
   !
   address-family evpn
      neighbor EVPN activate
   !
   address-family ipv4
      neighbor UNDERLAY activate
      network 10.0.0.3/32
   !
   vrf VRF1
      rd 65503:111111
      route-target import evpn 1:111111
      route-target export evpn 1:111111
      redistribute connected
!
end
Leaf3#
```

- Leaf 4:


```
Leaf4#
Leaf4#sh run
! Command: show running-config
! device: Leaf4 (vEOS-lab, EOS-4.29.2F)
!
! boot system flash:/vEOS-lab.swi
!
no aaa root
!
transceiver qsfp default-mode 4x10G
!
service routing protocols model multi-agent
!
link tracking group CORE-TRACKING
   recovery delay 1
!
hostname Leaf4
!
spanning-tree mode mstp
!
vlan 20
   name VL20
!
vrf instance VRF1
!
interface Port-Channel1
   description Link_to_host3
   switchport trunk allowed vlan 20
   switchport mode trunk
   !
   evpn ethernet-segment
      identifier 0000:0000:0000:0000:0001
      designated-forwarder election algorithm preference 50
      route-target import 00:00:00:00:00:01
   lacp system-id 1111.2222.3333
!
interface Ethernet1
   description connected-to-Spine1-Ethernet3
   mtu 9214
   no switchport
   ip address 172.16.4.1/30
   link tracking group CORE-TRACKING upstream
!
interface Ethernet2
   description connected-to-Spine2-Ethernet3
   mtu 9214
   no switchport
   ip address 172.16.4.5/30
   link tracking group CORE-TRACKING upstream
!
interface Ethernet3
   description ---Link-to-Server3---
   switchport mode trunk
   channel-group 1 mode active
   link tracking group CORE-TRACKING downstream
!
interface Ethernet4
!
interface Ethernet5
!
interface Ethernet6
!
interface Ethernet7
!
interface Ethernet8
!
interface Loopback0
   ip address 10.0.0.4/32
!
interface Management1
!
interface Vlan20
   shutdown
   vrf VRF1
   ip address virtual 20.20.20.1/24
!
interface Vxlan1
   vxlan source-interface Loopback0
   vxlan udp-port 4789
   vxlan vlan 20 vni 100020
   vxlan vrf VRF1 vni 111111
   vxlan learn-restrict any
!
ip virtual-router mac-address 00:1c:73:00:00:10
!
ip routing
ip routing vrf VRF1
!
ip route 10.0.0.112/32 172.16.4.2
ip route 10.0.0.112/32 172.16.4.6
ip route 10.0.1.1/32 172.16.4.2
ip route 10.0.2.2/32 172.16.4.6
ip route 200.200.200.200/32 172.16.4.2
ip route 200.200.200.200/32 172.16.4.6
!
router bgp 65504
   router-id 10.0.0.4
   no bgp default ipv4-unicast
   distance bgp 20 200 200
   maximum-paths 2 ecmp 2
   neighbor EVPN peer group
   neighbor EVPN remote-as 65500
   neighbor EVPN update-source Loopback0
   neighbor EVPN ebgp-multihop 3
   neighbor EVPN send-community extended
   neighbor UNDERLAY peer group
   neighbor UNDERLAY remote-as 65500
   neighbor 10.0.1.1 peer group EVPN
   neighbor 10.0.2.2 peer group EVPN
   neighbor 172.16.4.2 peer group UNDERLAY
   neighbor 172.16.4.6 peer group UNDERLAY
   !
   vlan 20
      rd auto
      route-target both 20:100020
      redistribute learned
   !
   address-family evpn
      neighbor EVPN activate
   !
   address-family ipv4
      neighbor UNDERLAY activate
      network 10.0.0.4/32
   !
   vrf VRF1
      rd 65504:111111
      route-target import evpn 1:111111
      route-target export evpn 1:111111
      redistribute connected
!
end
Leaf4#
```
- Host 1:


```
HOST-1#sh run
! Command: show running-config
! device: HOST-1 (vEOS-lab, EOS-4.29.2F)
!
! boot system flash:/vEOS-lab.swi
!
no aaa root
!
transceiver qsfp default-mode 4x10G
!
service routing protocols model multi-agent
!
hostname HOST-1
!
spanning-tree mode mstp
!
vlan 10
!
interface Port-Channel1
   description link-to-leaves-1-2
   switchport mode trunk
!
interface Ethernet1
   description Link-to-Leaf-Fabric
   switchport trunk allowed vlan 10
   switchport mode trunk
   channel-group 1 mode active
!
interface Ethernet2
   channel-group 1 mode active
!
interface Ethernet3
!
interface Ethernet4
!
interface Ethernet5
!
interface Ethernet6
!
interface Ethernet7
!
interface Ethernet8
!
interface Management1
!
interface Vlan10
   description IP-Address-for-Host1
   ip address 10.10.10.101/24
!
ip routing
!
ip route 0.0.0.0/0 10.10.10.1
!
end
```
- Host 2:


```
HOST-2#
HOST-2#sh run
! Command: show running-config
! device: HOST-2 (vEOS-lab, EOS-4.29.2F)
!
! boot system flash:/vEOS-lab.swi
!
no aaa root
!
transceiver qsfp default-mode 4x10G
!
service routing protocols model multi-agent
!
hostname HOST-2
!
spanning-tree mode mstp
!
vlan 20
!
interface Port-Channel1
   switchport mode trunk
!
interface Ethernet1
   description description ---Link-to-Leaf3---
   switchport mode trunk
   channel-group 1 mode active
!
interface Ethernet2
   description ---Link-to-Leaf4---
   switchport mode trunk
   channel-group 1 mode active
!
interface Ethernet3
!
interface Ethernet4
!
interface Ethernet5
!
interface Ethernet6
!
interface Ethernet7
!
interface Ethernet8
!
interface Management1
!
interface Vlan20
   ip address 20.20.20.3/24
!
ip routing
!
ip route 0.0.0.0/0 20.20.20.1
!
end
```

### 4. Переключение шлюзов с ЦОД на РЦОД 

Проверяем, что в arp таблице хостов видим mac-адрес шлюзов на Firewall: 
```
Total Mac Addresses for this criterion: 0
Firewall#sh inter vlan 10
Vlan10 is up, line protocol is up (connected)
  Hardware is Vlan, address is 5000.00d8.ac19 (bia 5000.00d8.ac19)
  Description: Gateway-for-Host-VLAN10
  Internet address is 10.10.10.1/24
  Broadcast address is 255.255.255.255
  IP MTU 1500 bytes (default)
  Up 13 minutes, 28 seconds
Firewall#sh inter vlan 20
Vlan20 is up, line protocol is up (connected)
  Hardware is Vlan, address is 5000.00d8.ac19 (bia 5000.00d8.ac19)
  Description: Gateway-for-Host-VLAN20
  Internet address is 20.20.20.1/24
  Broadcast address is 255.255.255.255
  IP MTU 1500 bytes (default)
  Up 15 minutes, 14 seconds
Firewall#

```
```
HOST-1>show arp
Address         Age (sec)  Hardware Addr   Interface
10.10.10.1        0:10:29  5000.00d8.ac19  Vlan10, Port-Channel1
HOST-1>
```


```
HOST-2>show ip arp
Address         Age (sec)  Hardware Addr   Interface
20.20.20.1        0:13:38  5000.00d8.ac19  Vlan20, Port-Channel1
HOST-2>
```

Выключаем шлюзы на Firewall и поднимаем их в правой фабрике на leaf-коммутаторах. 

На Firewall:
```
configure terminal
interface Vlan10
   shutdown
!
interface Vlan20
   shutdown
end
```


На Leaf1 и Leaf2 (для VLAN 10):

```
configure terminal
interface Vlan10
   no shutdown
end

```

На Leaf3 и Leaf4 (для VLAN 20):

```
configure terminal
interface Vlan20
   no shutdown
end
```

Чистим arp-cash и проверяем доступность: 
```
HOST-1#clear arp-cache
HOST-1#
HOST-1#
HOST-1#
HOST-1#ping 10.10.10.1
PING 10.10.10.1 (10.10.10.1) 72(100) bytes of data.
80 bytes from 10.10.10.1: icmp_seq=1 ttl=64 time=25.5 ms
80 bytes from 10.10.10.1: icmp_seq=2 ttl=64 time=46.4 ms
80 bytes from 10.10.10.1: icmp_seq=3 ttl=64 time=27.9 ms
80 bytes from 10.10.10.1: icmp_seq=4 ttl=64 time=21.0 ms
80 bytes from 10.10.10.1: icmp_seq=5 ttl=64 time=15.0 ms

--- 10.10.10.1 ping statistics ---
5 packets transmitted, 5 received, 0% packet loss, time 101ms
rtt min/avg/max/mdev = 15.074/27.217/46.429/10.567 ms, pipe 2, ipg/ewma 25.395/25.735 ms
HOST-1#ping 10.10.10.11
PING 10.10.10.11 (10.10.10.11) 72(100) bytes of data.

--- 10.10.10.11 ping statistics ---
5 packets transmitted, 0 received, 100% packet loss, time 47ms


HOST-1#
HOST-1#ping 10.10.10.11
PING 10.10.10.11 (10.10.10.11) 72(100) bytes of data.

--- 10.10.10.11 ping statistics ---
5 packets transmitted, 0 received, 100% packet loss, time 51ms

HOST-1#ping 10.10.10.11
PING 10.10.10.11 (10.10.10.11) 72(100) bytes of data.
80 bytes from 10.10.10.11: icmp_seq=1 ttl=64 time=192 ms
80 bytes from 10.10.10.11: icmp_seq=2 ttl=64 time=185 ms
80 bytes from 10.10.10.11: icmp_seq=3 ttl=64 time=186 ms
80 bytes from 10.10.10.11: icmp_seq=4 ttl=64 time=191 ms
80 bytes from 10.10.10.11: icmp_seq=5 ttl=64 time=194 ms

--- 10.10.10.11 ping statistics ---
5 packets transmitted, 5 received, 0% packet loss, time 48ms
rtt min/avg/max/mdev = 185.867/190.119/194.509/3.505 ms, pipe 5, ipg/ewma 12.016/191.600 ms
HOST-1#ping 20.20.20.1
PING 20.20.20.1 (20.20.20.1) 72(100) bytes of data.
80 bytes from 20.20.20.1: icmp_seq=1 ttl=63 time=46.6 ms
80 bytes from 20.20.20.1: icmp_seq=2 ttl=63 time=48.9 ms
80 bytes from 20.20.20.1: icmp_seq=3 ttl=63 time=86.0 ms
80 bytes from 20.20.20.1: icmp_seq=4 ttl=63 time=81.7 ms
80 bytes from 20.20.20.1: icmp_seq=5 ttl=63 time=51.7 ms

--- 20.20.20.1 ping statistics ---
5 packets transmitted, 5 received, 0% packet loss, time 88ms
rtt min/avg/max/mdev = 46.633/63.018/86.036/17.187 ms, pipe 4, ipg/ewma 22.048/55.075 ms
HOST-1#ping 20.20.20.3
PING 20.20.20.3 (20.20.20.3) 72(100) bytes of data.
80 bytes from 20.20.20.3: icmp_seq=1 ttl=62 time=117 ms
80 bytes from 20.20.20.3: icmp_seq=2 ttl=62 time=106 ms
80 bytes from 20.20.20.3: icmp_seq=3 ttl=62 time=119 ms
80 bytes from 20.20.20.3: icmp_seq=4 ttl=62 time=117 ms
80 bytes from 20.20.20.3: icmp_seq=5 ttl=62 time=119 ms

--- 20.20.20.3 ping statistics ---
5 packets transmitted, 5 received, 0% packet loss, time 50ms
rtt min/avg/max/mdev = 106.512/116.048/119.423/4.851 ms, pipe 5, ipg/ewma 12.633/116.986 ms
HOST-1#

```

## 5. Архитектура отказоустойчивости и автоматизации переключения 

При проектировании автоматического переключения шлюзов по умолчанию (Failover) в растянутом L2-домене ключевым риском является ситуация **Split-Brain** — одновременная работа двух шлюзов с одинаковыми IP-адресами (`10.10.10.1` и `20.20.20.1`) в левом и правом ЦОДах при обрыве межцодового канала связи (DCI).

Для исключения этого риска предложена автономная схема автоматизации на базе механизма **Fencing (принудительной самоизоляции площадки)**:

### 5.1. Алгоритм автоматического переключения (Failover)
Контроллер автоматизации (или встроенный обработчик событий Arista EOS Event-Handler), размещённый внутри правого ЦОДа, непрерывно отслеживает состояние внешних DCI-линков и BGP-сессий на пограничных шлюзах `BGW3` и `BGW4`:

1. **Детектирование аварии с выдержкой времени:** Если оба внешних канала (на `BGW3` и `BGW4` в сторону Route Server) недоступны непрерывно более **15 минут** (защита от кратковременных сбоев и флапов BGP), инцидент классифицируется как потеря связности с левым ЦОДом.
2. **Принудительная изоляция (Fencing):** Скрипт переводит внешние интерфейсы `Ethernet1` (в сторону `RS`) на `BGW3` и `BGW4` в состояние административного отключения (**`shutdown`**). Правый ЦОД намеренно изолируется от левой площадки на уровне Layer 2.
3. **Активация локальных шлюзов:** После подтверждения изоляции DCI-стыков скрипт активирует распределённые шлюзы Anycast Gateway (**`no shutdown`**) на интерфейсах `Vlan10` (`Leaf1/2`) и `Vlan20` (`Leaf3/4`), а также рассылает Gratuitous ARP для обновления кэша серверов.

### 5.2. Преимущества подхода и порядок восстановления (Failback)
* **Полная защита от Split-Brain:** Даже если причиной сбоя был обрыв оптики, а левый ЦОД продолжал работать (или перезагрузился и восстановился автоматически), при восстановлении физического канала L2-мост не склеится бесконтрольно, так как порты на `BGW3/4` заблокированы административно.
* **Автономность:** Механизм не зависит от внешних облачных арбитров и третьих площадок — внутренние сервисы правого ЦОДа продолжают штатно взаимодействовать между VLAN 10 и VLAN 20 через локальные Leaf-коммутаторы.
* **Безопасный ручной возврат (Manual Failback):** После устранения аварии и проверки стабильности левого ЦОДа инженер выполняет возврат в контролируемом окне:
  1. Переводит интерфейсы `Vlan10` и `Vlan20` на правых `Leaf1–4` в состояние **`shutdown`**.
  2. Включает внешние DCI-интерфейсы на `BGW3` и `BGW4` командой **`no shutdown`**, восстанавливая единый растянутый L2-домен.

