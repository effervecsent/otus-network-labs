### Проектирование адресного пространства

 ### Цель
  - собрать схему Clos и распределить адресное пространство

  ### Описание выполнения домашнего задания:

  В этой самостоятельной работе я собрала схему: 

  ![01.PNG](01.PNG)

  ### Таблица адресов для Underlay сети  

  |Device|Interface|IP Adress|Subnet Mask|Default Gateway
  |---|---|---|---|---|
  Spine-01|ETh1|10.0.0.2|255.255.255.0|10.0.0.1
  SPine-02|ETh1|10.0.0.3|255.255.255.0|10.0.0.1


this is my first lab
'''localhost#sh run
! Command: show running-config
! device: localhost (vEOS-lab, EOS-4.33.1.1F)
!
! boot system flash:/vEOS-lab.swi
!
no aaa root
!
switchport default mode routed
!
no service interface inactive port-id allocation disabled
!
transceiver qsfp default-mode 4x10G
!
service routing protocols model multi-agent
!
logging console informational
logging policy match match-list ztpFilter discard
!
logging level AAA errors
logging level ACCOUNTING errors
logging level ACL errors
logging level AGENT errors
logging level ALE errors
logging level ARP errors
logging level BFD errors