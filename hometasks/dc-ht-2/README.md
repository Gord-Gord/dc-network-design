
## Проектирование адресного пространства

### Цель:
Настроить OSPF для Underlay сети.сети

### План работ
1. Собрать схему Clos;
2. Распределить адресное пространство;
3. Настроить OSPF в Underlay-сети;
4. Настроить BFD;
5. Проверка результатов работы.

### 1. Сборка схемы Clos

Схема по топологии Clos была собрана в EVE-NG. В качестве основных элементов схемы, Spine- и Leaf-коммутаторов, использовались виртуальные образы Arista vEOS-lab версии 4.29.2F. В качестве клиентов Virtual PC.

![alt-текст](https://github.com/Gord-Gord/dc-network-design/blob/main/hometasks/dc-ht-2/scheme.png)

### 2. Распределение адресного пространства

В качестве протокола, на котором будет строиться Underlay, был выбран IPv6.
Так как OSPF-соседство при использовании IPv6 всегда строится от Link-Local адресов, то IPv6 на интерфейсы Ethernet Spine- и Leaf-коммутаторов принято решение не назначать, дабы не загромождать конфигурацию.
Таким образом, остаётся назначить IPv6-адреса только интерфейсам loopback 0 каждого коммутатора. Пусть адреса, назначаемые интерфейсам loopback 0, принадлежат сети fd12:dc:1:0::/64.
Тогда адресное пространство сети ЦОДа 1 примет следующий вид:

Device|loopback 0|loopback 1
------|----------|----------
Spine1|fd12:dc1:1::1/128
Spine2|fd12:dc1:1::2/128
Leaf1|fd12:dc1:1::3/128
Leaf2|fd12:dc1:1::4/128
Leaf3|fd12:dc1:1::5/128

где: 3-ий и 4-ый октет - Порядковый номер ЦОДа;
     6-ой октет 01 - Порядковый номер POD
     16-ой октет - Порядковый номер устройства в POD.

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

![alt-текст]( --- )

##### Leaf-коммутаторы пингуют Spine-коммутаторы

![alt-текст]( --- )

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
