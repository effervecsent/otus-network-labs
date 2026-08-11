### Проектирование адресного пространства

 ### Цель
  - собрать схему Clos и распределить адресное пространство

  ### Описание выполнения домашнего задания:

  В этой самостоятельной работе я собрала схему: 

  ![01.PNG](01.PNG)

  ### Таблица адресов для Underlay сети  

  |Device|Interface|IP Adress|Subnet Mask
  |---|---|---|---|
  SPINE-01|Loopback0|10.255.0.1|255.255.255.255
  SPINE-01|Ethernet1|10.0.1.0|255.255.255.254
  SPINE-01|Ethernet2|10.0.1.2|255.255.255.254
  SPINE-01|Ethernet3|10.0.1.4|255.255.255.254
  SPINE-01|Ethernet4|10.0.0.0|255.255.255.254
  SPINE-02Loopback010.255.0.2255.255.255.255
  SPINE-02Ethernet110.0.2.0255.255.255.254
  SPINE-02Ethernet210.0.2.2255.255.255.254
  SPINE-02Ethernet310.0.2.4255.255.255.254
  SPINE-02Ethernet410.0.0.1255.255.255.254
  LEAF-01|Loopback0|10.255.0.11|255.255.255.255
  LEAF-01|Ethernet1|10.0.1.1|255.255.255.254
  LEAF-01|Ethernet2|10.0.2.1|255.255.255.254
  LEAF-02|Loopback0|10.255.0.12|255.255.255.255
  LEAF-02|Ethernet2|10.0.1.3|255.255.255.254
  LEAF-02|Ethernet3|10.0.2.3|255.255.255.254
  LEAF-03|Loopback0|10.255.0.13|255.255.255.255
  LEAF-03|Ethernet3|10.0.1.5|255.255.255.254
  LEAF-03|Ethernet4|10.0.2.5|255.255.255.254


###КОнфигурация SPINE-01: 

'''
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
'''
