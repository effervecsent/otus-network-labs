
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
