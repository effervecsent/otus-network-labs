### Проектирование адресного пространства

 ### Цель
  - собрать схему Clos и распределить адресное пространство

  ### Описание выполнения домашнего задания:

  В этой самостоятельной работе я собрала схему: 

  ![02.PNG](01.PNG)

Была выбрана следующая адресация для loopback и p2p:

Loopback-адреса

Spine-1: 10.255.0.1/32

Spine-2: 10.255.0.2/32

Leaf-1: 10.255.0.11/32

Leaf-2: 10.255.0.12/32

Leaf-3: 10.255.0.13/32


Выбор маски /32 для Loopback-интерфейсов:

- Изоляция и экономия: Loopback-интерфейс — это виртуальный порт, который не подключен к физическому проводу. Ему физически необходим всего один уникальный IP-адрес для идентификации устройства. Маска /32 (255.255.255.255) выделяет ровно один адрес, исключая нерациональный расход емкости сети.

- Стабильность протоколов (Router ID): IP-адрес на Loopback никогда не переходит в состояние Down (в отличие от физических портов, где может повредиться кабель). Этот адрес используется протоколом BGP в качестве стабильного Router ID и для установления Overlay-сессий (iBGP/EVPN).

 - Логическое разделение пулов: Использование диапазона 10.255.0.0/24 для Loopback-адресов позволяет сетевому инженеру с первого взгляда отличать служебный трафик маршрутизаторов от транзитного трафика линков или клиентских данных.



Топология P2P-связей

Для P2P-линков используются подсети /31.

Spine-1 ↔ Leaf-1: 10.0.1.0/31 (Spine: .0, Leaf: .1)

Spine-1 ↔ Leaf-2: 10.0.1.2/31 (Spine: .2, Leaf: .3)

Spine-1 ↔ Leaf-3: 10.0.1.4/31 (Spine: .4, Leaf: .5)

Spine-2 ↔ Leaf-1: 10.0.2.0/31 (Spine: .0, Leaf: .1)

Spine-2 ↔ Leaf-2: 10.0.2.2/31 (Spine: .2, Leaf: .3)

Spine-2 ↔ Leaf-3: 10.0.2.4/31 (Spine: .4, Leaf: .5)

Выбор маски /31 для Point-to-Point (P2P) линков:

- Экономия адресного пространства на 50%: Согласно стандарту RFC 3021, современные коммутаторы (включая Arista EOS) на P2P-линках не требуют адреса сети и broadcast-адреса. Маска /31 занимает всего 2 IP-адреса, что экономит в масштабах дата-центра тысячи сетевых адресов.

- Защита от сетевых атак: Отсутствие направленного широковещательного адреса (Directed Broadcast) в подсетях /31 исключает возможность проведения Smurf-атак или создания широковещательного шторма внутри транзитных каналов фабрики.


### Структурированная логика IPAM

Выделение адресов выполнено по определенной логике, что упрощает траблшутинг и позволяет использовать суммаризацию маршрутов:

- Разделение по Spine-коммутаторам:Третий октет адреса указывает, к какому Spine-коммутатору ведет линк.

Подсеть 10.0.**1**.X/31 жестко закреплена за интерфейсами SPINE-01.

Подсеть 10.0.**2**.X/31 жестко закреплена за интерфейсами SPINE-02.Четные и нечетные хосты:Все четные IP-адреса (заканчивающиеся на .0, .2, .4) всегда назначаются на стороне Spine-коммутаторов.

Все нечетные IP-адреса (заканчивающиеся на .1, .3, .5) всегда назначаются на стороне Leaf-коммутаторов.Пример: Видя в логах адрес 10.0.2.4, инженер сразу понимает: это интерфейс на SPINE-02, ведущий к третьему лифу.


  ### Таблица адресов для Underlay сети  

  |Device|Interface|IP Adress|Subnet Mask
  |---|---|---|---|
  SPINE-01|Loopback0|10.255.0.1|255.255.255.255
  SPINE-01|Ethernet1|10.0.1.0|255.255.255.254
  SPINE-01|Ethernet2|10.0.1.2|255.255.255.254
  SPINE-01|Ethernet3|10.0.1.4|255.255.255.254
  SPINE-01|Ethernet4|10.0.0.0|255.255.255.254
  SPINE-02|Loopback0|10.255.0.2|255.255.255.255
  SPINE-02|Ethernet1|10.0.2.0|255.255.255.254
  SPINE-02|Ethernet2|10.0.2.2|255.255.255.254
  SPINE-02|Ethernet3|10.0.2.4|255.255.255.254
  SPINE-02|Ethernet4|10.0.0.1|255.255.255.254
  LEAF-01|Loopback0|10.255.0.11|255.255.255.255
  LEAF-01|Ethernet1|10.0.1.1|255.255.255.254
  LEAF-01|Ethernet2|10.0.2.1|255.255.255.254
  LEAF-02|Loopback0|10.255.0.12|255.255.255.255
  LEAF-02|Ethernet2|10.0.1.3|255.255.255.254
  LEAF-02|Ethernet3|10.0.2.3|255.255.255.254
  LEAF-03|Loopback0|10.255.0.13|255.255.255.255
  LEAF-03|Ethernet3|10.0.1.5|255.255.255.254
  LEAF-03|Ethernet4|10.0.2.5|255.255.255.254


### Кoнфигурация SPINE-01: 

```
SPINE-01#sh run
! Command: show running-config
! device: SPINE-01 (vEOS-lab, EOS-4.29.2F)
!
! boot system flash:/vEOS-lab.swi
!
no aaa root
!
transceiver qsfp default-mode 4x10G
!
service routing protocols model ribd
!
hostname SPINE-01
!
spanning-tree mode mstp
!
interface Ethernet1
   description tot-LEAFtot-LEAF-01-eth1
   no switchport
   ip address 10.0.1.0/31
!
interface Ethernet2
   description to LEAF-02-eth2
   no switchport
   ip address 10.0.1.2/31
!
interface Ethernet3
   description to-LEAF-03-eth3
   no switchport
   ip address 10.0.1.4/31
!
interface Ethernet4
   description to-SPINE-02-eth4
   no switchport
   ip address 10.0.0.0/31
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
   ip address 10.255.0.1/32
!
interface Management1
!
ip routing
!
end 
```

### Конфигурация SPINE-02

```

SPINE-02#sh run
! Command: show running-config
! device: SPINE-02 (vEOS-lab, EOS-4.29.2F)
!
! boot system flash:/vEOS-lab.swi
!
no aaa root
!
transceiver qsfp default-mode 4x10G
!
service routing protocols model ribd
!
hostname SPINE-02
!
spanning-tree mode mstp
!
interface Ethernet1
   description to-LEAF-01_eth2
   no switchport
   ip address 10.0.2.0/31
!
interface Ethernet2
   description to-LEAF02-eth3
   no switchport
   ip address 10.0.2.2/31
!
interface Ethernet3
   description to-LEAF-03-eth4
   no switchport
   ip address 10.0.2.4/31
!
interface Ethernet4
   description to SPINE-01-eth4
   no switchport
   ip address 10.0.0.1/31
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
   ip address 10.255.0.2/32
!
interface Management1
!
ip routing
```

### Кoнфигурация leaf-01
```
hostname LEAF-01
ip routing

vlan 10
   name CLIENT_NETWORK

interface Loopback0
   ip address 10.255.0.11/32

interface Ethernet1
   description TO_SPINE-01_Eth1
   no switchport
   ip address 10.0.1.1/31

interface Ethernet2
   description TO_SPINE-02_Eth1
   no switchport
   ip address 10.0.2.1/31

interface Ethernet5
   description TO_Client-01
   switchport mode access
   switchport access vlan 10
   spanning-tree portfast

interface Ethernet6
   description UNUSED_PORT
   shutdown

interface Vlan10
   description Gateway_for_LEAF-01_Clients
   ip address 10.1.11.1/24

```

### Конфигурация LEAF-02

```
hostname LEAF-02
ip routing

vlan 10
   name CLIENT_NETWORK

interface Loopback0
   ip address 10.255.0.12/32

interface Ethernet2
   description TO_SPINE-01_Eth2
   no switchport
   ip address 10.0.1.3/31

interface Ethernet3
   description TO_SPINE-02_Eth2
   no switchport
   ip address 10.0.2.3/31

interface Ethernet5
   description TO_Client-02
   switchport mode access
   switchport access vlan 10
   spanning-tree portfast

interface Ethernet6
   description UNUSED_PORT
   shutdown

interface Vlan10
   description Gateway_for_LEAF-02_Clients
   ip address 10.1.12.1/24
```

### Конфигурация LEAF-03
```
hostname LEAF-03
ip routing

vlan 10
   name CLIENT_NETWORK

interface Loopback0
   ip address 10.255.0.13/32

interface Ethernet3
   description TO_SPINE-01_Eth3
   no switchport
   ip address 10.0.1.5/31

interface Ethernet4
   description TO_SPINE-02_Eth3
   no switchport
   ip address 10.0.2.5/31

interface Ethernet5
   description TO_Client-03
   switchport mode access
   switchport access vlan 10
   spanning-tree portfast

interface Ethernet6
   description TO_Client-04
   switchport mode access
   switchport access vlan 10
   spanning-tree portfast

interface Vlan10
   description Gateway_for_LEAF-03_Clients
   ip address 10.1.13.1/24
```

### Проверка доступности: 

- Ping c LEAF-01 до SPINE-01

```
LEAF-01#ping 10.0.1.0
PING 10.0.1.0 (10.0.1.0) 72(100) bytes of data.
80 bytes from 10.0.1.0: icmp_seq=1 ttl=64 time=17.1 ms
80 bytes from 10.0.1.0: icmp_seq=2 ttl=64 time=16.5 ms
80 bytes from 10.0.1.0: icmp_seq=3 ttl=64 time=8.72 ms
80 bytes from 10.0.1.0: icmp_seq=4 ttl=64 time=6.98 ms
80 bytes from 10.0.1.0: icmp_seq=5 ttl=64 time=10.0 ms

--- 10.0.1.0 ping statistics ---
5 packets transmitted, 5 received, 0% packet loss, time 66ms
rtt min/avg/max/mdev = 6.982/11.879/17.105/4.148 ms, pipe 2, ipg/ewma 16.594/14.267 ms
LEAF-01#
```

- Ping c LEAF-02 до SPINE-01
```
LEAF-02#ping 10.0.1.2
PING 10.0.1.2 (10.0.1.2) 72(100) bytes of data.
80 bytes from 10.0.1.2: icmp_seq=1 ttl=64 time=61.5 ms
80 bytes from 10.0.1.2: icmp_seq=2 ttl=64 time=53.7 ms
80 bytes from 10.0.1.2: icmp_seq=3 ttl=64 time=33.7 ms
80 bytes from 10.0.1.2: icmp_seq=4 ttl=64 time=25.7 ms
80 bytes from 10.0.1.2: icmp_seq=5 ttl=64 time=8.96 ms

--- 10.0.1.2 ping statistics ---
5 packets transmitted, 5 received, 0% packet loss, time 104ms
rtt min/avg/max/mdev = 8.960/36.741/61.529/19.001 ms, pipe 4, ipg/ewma 26.122/47.731 ms

LEAF-02#
```

- Ping c LEAF-01 до SPINE-02
```
LEAF-01#ping 10.0.2.0
PING 10.0.2.0 (10.0.2.0) 72(100) bytes of data.
80 bytes from 10.0.2.0: icmp_seq=1 ttl=64 time=61.2 ms
80 bytes from 10.0.2.0: icmp_seq=2 ttl=64 time=53.4 ms
80 bytes from 10.0.2.0: icmp_seq=3 ttl=64 time=68.0 ms
80 bytes from 10.0.2.0: icmp_seq=4 ttl=64 time=65.1 ms
80 bytes from 10.0.2.0: icmp_seq=5 ttl=64 time=64.2 ms

--- 10.0.2.0 ping statistics ---
5 packets transmitted, 5 received, 0% packet loss, time 46ms
rtt min/avg/max/mdev = 53.482/62.441/68.029/4.970 ms, pipe 5, ipg/ewma 11.573/62.071 ms
LEAF-01#
```
- Ping c client-01 до LEAF-01


VPCS> ping 10.1.11.1
```
84 bytes from 10.1.11.1 icmp_seq=1 ttl=64 time=8.115 ms
84 bytes from 10.1.11.1 icmp_seq=2 ttl=64 time=7.942 ms
84 bytes from 10.1.11.1 icmp_seq=3 ttl=64 time=8.083 ms
84 bytes from 10.1.11.1 icmp_seq=4 ttl=64 time=11.968 ms
84 bytes from 10.1.11.1 icmp_seq=5 ttl=64 time=10.387 ms
```


