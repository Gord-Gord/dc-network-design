
## Проектирование адресного пространства

### Цель:
1. Собрать схему Clos;
2. Распределить адресное пространство.

### Сборка схемы Clos

Схема по топологии Clos был собрана в EVE-NG. В качестве основных элементов схемы, Spine- и Leaf-коммутаторов, использовались виртуальные образы Arista vEOS-lab версии 4.29.2F. В качестве клиентов Virtual PC.

![alt-текст](https://github.com/Gord-Gord/dc-network-design/blob/main/hometasks/dc-ht-1/scheme.png)

### Распределение адресного пространства

В качестве протокола, на котором будет строиться Underlay, был выбран IPv6.
Всей сети ЦОДа выделено адресное пространство: fd12:dc:1::/48, которое в свою очередь разбивается на следующие пулы IP-адресов:

Network|Назначение
-------|----------
fd12:dc:1:0::/64|loopback 0
fd12:dc:1:1::/64|loopback 1
fd12:dc:1:000::/55|p2p-link
fd12:dc:1:200::/55|client and services

Таким образом устройствам в сети ЦОДа назначены следующие IPv6-адреса:

Device|eth1|eth2|eth3|loopback 0|loopback 1
------|----|----|----|----------|----------
Spine-1|fd12:dc:1:100::1/64|fd12:dc:1:101::1/64|fd12:dc:1:102::1/64|fd12:dc:1::1/128|fd12:dc:2::1/128
Spine-2|fd12:dc:1:103::1/64|fd12:dc:1:104::1/64|fd12:dc:1:105::1/64|fd12:dc:1::2/128|fd12:dc:2::2/128
Leaf-1|fd12:dc:1:100::2/64|fd12:dc:1:103::2/64|нет|fd12:dc:1::3/128|fd12:dc:1:1::3/128
Leaf-2|fd12:dc:1:101::2/64|fd12:dc:1:104::2/64|нет|fd12:dc:1::4/128|fd12:dc:1:1::4/128
Leaf-3|fd12:dc:1:102::2/64|fd12:dc:1:105::2/64|нет|fd12:dc:1::5/128|fd12:dc:1:1::5/128
Client-1|fd12:dc:1:300::1/64
Client-2|fd12:dc:1:300::2/64
Client-3|fd12:dc:1:300::3/64
Client-4|fd12:dc:1:300::4/64

#### Результаты вывода команды show ipv6 interface brief:
##### Spine-1:

```
spine-1#show ipv6 interface brief
Interface  Status    MTU   IPv6 Address                 Addr State  Addr Source
Et1        up       1500   fe80::5200:ff:fed7:ee0b/64   up          link local
                           fd12:dc:1:100::1/64          up          config
Et2        up       1500   fe80::5200:ff:fed7:ee0b/64   up          link local
                           fd12:dc:1:101::1/64          up          config
Et3        up       1500   fe80::5200:ff:fed7:ee0b/64   up          link local
                           fd12:dc:1:102::1/64          up          config
Lo0        up      65535   fe80::ff:fe00:0/64           up          link local
                           fd12:dc:1::1/128             up          config
Lo1        up      65535   fe80::ff:fe00:0/64           up          link local
                           fd12:dc:2::1/128             up          config
```

##### Spine-2:
```
spine-2#show ipv6 interface brief
Interface  Status    MTU   IPv6 Address                 Addr State  Addr Source
Et1        up       1500   fe80::5200:ff:fecb:38c2/64   up          link local
                           fd12:dc:1:103::1/64          up          config
Et2        up       1500   fe80::5200:ff:fecb:38c2/64   up          link local
                           fd12:dc:1:104::1/64          up          config
Et3        up       1500   fe80::5200:ff:fecb:38c2/64   up          link local
                           fd12:dc:1:105::1/64          up          config
Lo0        up      65535   fe80::ff:fe00:0/64           up          link local
                           fd12:dc:1::2/128             up          config
Lo1        up      65535   fe80::ff:fe00:0/64           up          link local
                           fd12:dc:2::2/128             up          config
```

##### Leaf-1:
```
leaf-1#show ipv6 interface brief
Interface  Status    MTU   IPv6 Address                 Addr State  Addr Source
Et1        up       1500   fe80::5200:ff:fed5:5dc0/64   up          link local
                           fd12:dc:1:100::2/64          up          config
Et2        up       1500   fe80::5200:ff:fed5:5dc0/64   up          link local
                           fd12:dc:1:103::2/64          up          config
Lo0        up      65535   fe80::ff:fe00:0/64           up          link local
                           fd12:dc:1::3/128             up          config
Lo1        up      65535   fe80::ff:fe00:0/64           up          link local
                           fd12:dc:1:1::3/128           up          config
```

##### Leaf-2
```
leaf-2#show ipv6 interface brief
Interface  Status    MTU   IPv6 Address                 Addr State  Addr Source
Et1        up       1500   fe80::5200:ff:fe03:3766/64   up          link local
                           fd12:dc:1:101::2/64          up          config
Et2        up       1500   fe80::5200:ff:fe03:3766/64   up          link local
                           fd12:dc:1:104::2/64          up          config
Lo0        up      65535   fe80::ff:fe00:0/64           up          link local
                           fd12:dc:1::5/128             up          config
Lo1        up      65535   fe80::ff:fe00:0/64           up          link local
                           fd12:dc:1:1::5/128           up          config
```

##### Leaf-3
```
leaf-3#show ipv6 interface brief
Interface  Status    MTU   IPv6 Address                 Addr State  Addr Source
Et1        up       1500   fe80::5200:ff:fe15:f4e8/64   up          link local
                           fd12:dc:1:102::2/64          up          config
Et2        up       1500   fe80::5200:ff:fe15:f4e8/64   up          link local
                           fd12:dc:1:105::2/64          up          config
Lo0        up      65535   fe80::ff:fe00:0/64           up          link local
                           fd12:dc:1::4/128             up          config
Lo1        up      65535   fe80::ff:fe00:0/64           up          link local
                           fd12:dc:1:1::4/128           up          config
```

#### Скриншоты

##### Spine-коммутаторы пингуют Leaf-коммутаторы

![alt-текст](https://github.com/Gord-Gord/dc-network-design/blob/main/hometasks/dc-ht-1/spines_are_pinging_leaves.png)

##### Leaf-коммутаторы пингуют Spine-коммутаторы

![alt-текст](https://github.com/Gord-Gord/dc-network-design/blob/main/hometasks/dc-ht-1/leaves_are_pinging_spines.png)

#### Листинги

##### Spine-1
```
hostname spine-1
!
spanning-tree mode mstp
!
interface Ethernet1
   no switchport
   ipv6 enable
   ipv6 address fd12:dc:1:100::1/64
!
interface Ethernet2
   no switchport
   ipv6 enable
   ipv6 address fd12:dc:1:101::1/64
!
interface Ethernet3
   no switchport
   ipv6 enable
   ipv6 address fd12:dc:1:102::1/64
!
interface Ethernet4
!
interface Ethernet5
!
interface Loopback0
   ipv6 address fd12:dc:1::1/128
!
interface Loopback1
   ipv6 address fd12:dc:2::1/128
!
```

##### Spine-2
```
hostname spine-2
!
spanning-tree mode mstp
!
interface Ethernet1
   no switchport
   ipv6 enable
   ipv6 address fd12:dc:1:103::1/64
!
interface Ethernet2
   no switchport
   ipv6 enable
   ipv6 address fd12:dc:1:104::1/64
!
interface Ethernet3
   no switchport
   ipv6 enable
   ipv6 address fd12:dc:1:105::1/64
!
interface Ethernet4
!
interface Ethernet5
!
interface Loopback0
   ipv6 address fd12:dc:1::2/128
!
interface Loopback1
   ipv6 address fd12:dc:2::2/128
```

##### Leaf-1
```
hostname leaf-1
!
spanning-tree mode mstp
!
interface Ethernet1
   no switchport
   ipv6 enable
   ipv6 address fd12:dc:1:100::2/64
!
interface Ethernet2
   no switchport
   ipv6 enable
   ipv6 address fd12:dc:1:103::2/64
!
interface Ethernet3
!
interface Ethernet4
!
interface Ethernet5
!
interface Loopback0
   ipv6 address fd12:dc:1::3/128
!
interface Loopback1
   ipv6 address fd12:dc:1:1::3/128
```

##### Leaf-2
```
hostname leaf-2
!
interface Ethernet1
   no switchport
   ipv6 enable
   ipv6 address fd12:dc:1:101::2/64
!
interface Ethernet2
   no switchport
   ipv6 enable
   ipv6 address fd12:dc:1:104::2/64
!
interface Ethernet3
!
interface Ethernet4
!
interface Ethernet5
!
interface Loopback0
   ipv6 address fd12:dc:1::4/128
!
interface Loopback1
   ipv6 address fd12:dc:1:1::4/128
!
```

##### Leaf-3
```
hostname leaf-3
!
interface Ethernet1
   no switchport
   ipv6 enable
   ipv6 address fd12:dc:1:102::2/64
!
interface Ethernet2
   no switchport
   ipv6 enable
   ipv6 address fd12:dc:1:105::2/64
!
interface Ethernet3
!
interface Ethernet4
!
interface Ethernet5
!
interface Loopback0
   ipv6 address fd12:dc:1::5/128
!
interface Loopback1
   ipv6 address fd12:dc:1:1::5/128
!
```
