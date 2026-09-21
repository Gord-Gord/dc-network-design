
## Построение Underlay сети (BGP)

### Цель:
Настроить BGP для Underlay сети.

### План работ
1. Собрать схему Clos
2. Распределить адресное пространство
3. Настроить BGP в Underlay-сети
4. Траблшутинг
5. Проверка результатов работы

### Краткое oписание объекта
У нас есть ЦОД № 1, в который входит несколько PODов и в частности, POD № 1, схему которого мы и будем собирать по топологии Clos.

На самом верхнем уровне топологии Clos PODы объединяются коммутаторами Super-Spine, которые работают в area (10) BGP.

Интерфейсы коммутаторов POD № 1 будут принадлежать area 1.

### 1. Сборка схемы Clos

Схема по топологии Clos была собрана в EVE-NG. В качестве основных элементов схемы, Spine- и Leaf-коммутаторов, использовались виртуальные образы Arista vEOS-lab версии 4.29.2F.
![alt-текст](https://github.com/Gord-Gord/dc-network-design/blob/main/hometasks/dc-ht-3/scheme.png)

### 2. Распределение адресного пространства

В качестве протокола, на котором будет строиться Underlay, был выбран IPv6.
Всей сети POD1 выделено адресное пространство: fd12:dc1:1::/48, которое в свою очередь разбивается на следующие пулы IP-адресов:

Network|Назначение
-------|----------
fd12:dc1:1:0::/64|loopback 0
fd12:dc1:1:200::/55|p2p-link

Также необходимо определиться с номерами автомномных систем (AS). Они будут из диапазона приватных ASN.  
Чтобы трафик между Leaf-коммутаторами проходил ровно через один Spine-коммутаторв, Spine1 и Spine2 будут иметь одинаковые номера AS. Пусть этот номера будет равным 64520. А Leaf-коммутаторам ASN будем назначать по-порядку: 64521, 64522, 64523 и т.д. 

Сведём полученные данные в таблицу:

Device|eth1|eth2|eth3|loopback 0|router-id|ASN
------|----|----|----|----------|---------|---
Spine1|fd12:dc1:1:200::1/64|fd12:dc1:1:201::1/64|fd12:dc1:1:202::1/64|fd12:dc1:1::1/128|10.1.1.1|64520
Spine2|fd12:dc1:1:203::1/64|fd12:dc1:1:204::1/64|fd12:dc1:1:205::1/64|fd12:dc1:1::2/128|10.1.1.2|64520
Leaf1|fd12:dc1:1:200::2/64|fd12:dc1:1:203::2/64|нет|fd12:dc1:1::3/128|10.1.1.3|64521
Leaf2|fd12:dc1:1:201::2/64|fd12:dc1:1:204::2/64|нет|fd12:dc1:1::4/128|10.1.1.4|64522
Leaf3|fd12:dc1:1:202::2/64|fd12:dc1:1:205::2/64|нет|fd12:dc1:1::5/128|10.1.1.5|64523

где:  
- 3-ий и 4-ый октеты каждого IPv6 адреса - Порядковый номер ЦОДа - dc1;  
- 6-ой октет каждого IPv6 адреса - Порядковый номер POD - 1;  
- IPv6 адрес loopback 0 16-ый октет Порядковый номер устройства в POD;  
- router-id 2-ой октет Порядковый номер ЦОДа - 1;  
- router-id 3-ой октет Порядковый номер POD - 1;  
- router-id 4-ой октет - Порядковый номер устройства в POD.  

Примечание: Для краткости не будем приводить здесь команды назначения IPv6 адресов на интерфейсы loopback 0 каждого коммутатора. Они показаны в листингах конфигураций оборудования.

### Настройка BGP

Вначале попробуем построить BGP-соседство от Link-Local адресов.

На коммутаторе Spine1 выполняем следующие действия:
- включаем на коммутаторе маршрутизацию IPv6:
```
Spine1(config)ipv6 unicast-routing vrf default
```
- далее, в vrf default запускаем процесс BGP и указываем Link-Local адреса интерфейсов Leaf-коммутаторов:
```
router bgp 64520
   router-id 10.1.1.1
   neighbor fe80::5200:ff:fe03:3766%Et2 remote-as 64522
   neighbor fe80::5200:ff:fe15:f4e8%Et3 remote-as 64523
   neighbor fe80::5200:ff:fed5:5dc0%Et1 remote-as 64521
```
- для того, чтобы построилось соседство необходимо в настройках семейства протокола IPv6 отдельно активировать сессии BGP с IPv6-адресами интерфейсов Leaf-коммутаторов:
```
router bgp 64520
   address-family ipv6
      neighbor fe80::5200:ff:fe03:3766%Et2 activate
      neighbor fe80::5200:ff:fe15:f4e8%Et3 activate
      neighbor fe80::5200:ff:fed5:5dc0%Et1 activate
```
- далее необходимо объявить, что мы будем анонсировать нащим соседям. Для этого в настройки процесса BGЗ добавляем строку:
```
router bgp 64520
   redistribute connected route-map rm-connected
   exit
```
- ну и создаём саму карту маршрутизации, которая указывает, что анонсировать нужно IPv6-адреса интерфейсов LoopBack:
```
route-map rm-connected permit 10
   match interface Loopback0
```
Аналогичные действия выполняем на всех коммутаторах. По окончании мы видим, что соседство поднялось и получены маршруты:
```
Spine1#show ipv6 bgp summary
BGP summary information for VRF default
Router identifier 10.1.1.1, local AS number 64520
Neighbor Status Codes: m - Under maintenance
  Neighbor         V  AS           MsgRcvd   MsgSent  InQ OutQ  Up/Down State   PfxRcd PfxAcc
  fe80::5200:ff:fe03:3766%Et2 4  64522             42        43    0    0 00:36:28 Estab   1      1
  fe80::5200:ff:fe15:f4e8%Et3 4  64523             42        43    0    0 00:36:27 Estab   1      1
  fe80::5200:ff:fed5:5dc0%Et1 4  64521             42        43    0    0 00:36:24 Estab   1      1
Spine1#show ipv6 route bgp

VRF: default
Displaying 3 of 8 IPv6 routing table entries
Codes: C - connected, S - static, K - kernel, O3 - OSPFv3,
       B - Other BGP Routes, A B - BGP Aggregate, R - RIP,
       I L1 - IS-IS level 1, I L2 - IS-IS level 2, DH - DHCP,
       NG - Nexthop Group Static Route, M - Martian,
       DP - Dynamic Policy Route, L - VRF Leaked,
       RC - Route Cache Route

 B E      fd12:dc1:1::3/128 [200/0]
           via fe80::5200:ff:fed5:5dc0, Ethernet1
 B E      fd12:dc1:1::4/128 [200/0]
           via fe80::5200:ff:fe03:3766, Ethernet2
 B E      fd12:dc1:1::5/128 [200/0]
           via fe80::5200:ff:fe15:f4e8, Ethernet3
```
Надо отметить, что, несмотря на то что коммутаторы установили соседство и успешно обменялись информацией о своих Loopback-интерфейсах, такой способ настройки BGP нельзя признать гибким и масштабируемым. При добавлении в сеть POD нового Leaf-коммутатора, нам каждый раз придется придётся прописывать в настройках BGP каждого Spine-коммутатора IPv6-адрес Link-local интерфейса добавляемого Leaf-коммутатора. И напротив: при добавлении в сеть POD нового Spine‑коммутатора мы будем вынуждены прописывать его в настройках всех подключаемых к нему Leaf‑коммутаторов.
Чтобы 
- на каждом из физических интерфейсов устанавливаем mtu 9000, включаем IPv6, указываем, что интерфейс включен в инстанс UNDERLAY процеса BGP, включаем BFD, задаём тип сети point-to-point, а также уровень отношений с соседями на этом интерфейсе: L1:
```
Spine1(config)#interface Ethernet1-3
Spine1(config-if-Et1-3)#mtu 9000
Spine1(config-if-Et1-3)#no switchport
Spine1(config-if-Et1-3)#ipv6 enable
Spine1(config-if-Et1-3)#isis enable UNDERLAY
Spine1(config-if-Et1-3)#isis ipv6 bfd
Spine1(config-if-Et1-3)#isis circuit-type level-1
Spine1(config-if-Et1-3)#isis network point-to-point
```

### Траблшутинг

После выполнения настроек на наших коммутаторах проверяем установилось ли соседство BGP. Поочередно на каждом коммутаторе командой "show isis neighbors" смотрим установленные коммутаторами отношения смежности по протоколу BGP.  
Вывод с коммутатора Spine1:
```
Spine1(config-router-isis)#show isis neighbors

Instance  VRF      System Id        Type Interface          SNPA              State Hold time   Circuit Id
UNDERLAY  default  Leaf1            L1   Ethernet1          P2P               UP    29          0B
UNDERLAY  default  Leaf2            L1   Ethernet2          P2P               INIT  24          0B
UNDERLAY  default  Leaf3            L1   Ethernet3          P2P               UP    29          0B
```
Как видим, коммутатор Spine1 установил соседство с Leaf1 и Leaf3, но с Leaf2 соседства нет.
Включаем перехват пакетов на Spine1 на интерфейсе eth2, на котором у нас линк с Leaf2. А затем на интерфейсе eth1 коммутатора Leaf2. Видим, что коммутаторы шлют пакеты с сообщениями BGP Hello. Изучаем содержимое этих сообщений. Видим что у обоих коммутаторов разные PDU Lenght.
Spine1:
![alt-text](https://github.com/Gord-Gord/dc-network-design/blob/main/hometasks/dc-ht-3/wireshark1.png)
Leaf2:
![alt-text](https://github.com/Gord-Gord/dc-network-design/blob/main/hometasks/dc-ht-3/wireshark2.png)
Значит допустили ошибку. Исправляем значения MTU на интерфейсах ethernet 1 и 2 Leaf2 и устанавливаем его 9000, как и у всех остальных.  
Также обращаем внимание, что Leaf-ы установили соседство только со Spine1, а со Spine2 соседства нет.
```
Leaf3#show isis neighbors

Instance  VRF      System Id        Type Interface          SNPA              State Hold time   Circuit Id
UNDERLAY  default  Spine1           L1   Ethernet1          P2P               UP    27          0D
```
Внимательно анализируем настройки протокола BGP на Spine2 и видим, что мы на его интерфейсах указали уровень отношений L2 тогда, как на всех интерфейсах всех Leaf-ов уровень отношений L1. Устраняем это расхождение.

### Проверка результатов работы

- В начале убедимся в установлении соседств bfd (статус Up в колонке State):

![alt-text](https://github.com/Gord-Gord/dc-network-design/blob/main/hometasks/dc-ht-3/spines_bfd_peers.png)
![alt-text](https://github.com/Gord-Gord/dc-network-design/blob/main/hometasks/dc-ht-3/leaves_bfd_peers.png)

- Далее смотрим установилось ли у нас BGP-соседство между Spine- и Leaf-коммутаторами. Об этом нам скажет статус Up в колонке State в строке с указанием  System-id BGP-соседа:

![alt-text](https://github.com/Gord-Gord/dc-network-design/blob/main/hometasks/dc-ht-3/spines_isis_neighbors.png)
![alt-text](https://github.com/Gord-Gord/dc-network-design/blob/main/hometasks/dc-ht-3/leaves_isis_neighbors.png)  
    
- Далее посмотрим какие маршруты получены по BGP и внесены в таблицу маршрутизации:

![alt-text](https://github.com/Gord-Gord/dc-network-design/blob/main/hometasks/dc-ht-3/spines_isis_routes.png)
![alt-text](https://github.com/Gord-Gord/dc-network-design/blob/main/hometasks/dc-ht-3/leaves_isis_routes.png)


- Проверим сетевую связность между интерфейсами loopback 0 разных коммутаторов:
    - Spine-коммутаторы пингуют loopback 0 Leaf-коммутаторов, при этом в качестве исходящего интерфейса обязательно указываем loopback 0 Spine-коммутатора:  
![Спайны пингуют лифы](https://github.com/Gord-Gord/dc-network-design/blob/main/hometasks/dc-ht-3/spines_are_pinging_leaves.png)  
    - Leaf-коммутаторы пингуют loopback 0 Spine-коммутаторов, при этом в качестве исходящего интерфейса обязательно указываем loopback 0 Leaf-коммутатора:
![Лифы пингуют спайны](https://github.com/Gord-Gord/dc-network-design/blob/main/hometasks/dc-ht-3/leaves_are_pinging_spines.png)  
    - Leaf-коммутаторы пингуют loopback 0 других Leaf-коммутаторов, при этом в качестве исходящего интерфейса обязательно указываем loopback 0 Leaf-коммутатора:
![Лифы пингуют Лифы](https://github.com/Gord-Gord/dc-network-design/blob/main/hometasks/dc-ht-3/leaves_are_pinging_leaves.png)

#### Листинги

##### Spine-1
```
hostname Spine1
!
spanning-tree mode mstp
!
interface Ethernet1
   mtu 9000
   no switchport
   ipv6 enable
!
interface Ethernet2
   mtu 9000
   no switchport
   ipv6 enable
!
interface Ethernet3
   mtu 9000
   no switchport
   ipv6 enable
!
interface Ethernet4
!
interface Ethernet5
!
interface Loopback0
   ipv6 address fd12:dc1:1::1/128
!
interface Management1
!
no ip routing
!
ipv6 unicast-routing
!
end
```

##### Spine-2
```
hostname Spine2
!
spanning-tree mode mstp
!
interface Ethernet1
   mtu 9000
   no switchport
   ipv6 enable
!
interface Ethernet2
   mtu 9000
   no switchport
   ipv6 enable
!
interface Ethernet3
   mtu 9000
   no switchport
   ipv6 enable
!
interface Ethernet4
!
interface Ethernet5
!
interface Loopback0
   ipv6 address fd12:dc1:1::2/128
!
interface Management1
!
no ip routing
!
ipv6 unicast-routing
!
end
```

##### Leaf-1
```
hostname Leaf1
!
spanning-tree mode mstp
!
interface Ethernet1
   mtu 9000
   no switchport
   ipv6 enable
!
interface Ethernet2
   mtu 9000
   no switchport
   ipv6 enable
!
interface Ethernet3
!
interface Ethernet4
!
interface Ethernet5
!
interface Loopback0
   ipv6 address fd12:dc1:1::3/128
!
interface Management1
!
no ip routing
!
ipv6 unicast-routing
!
end
```

##### Leaf-2
```
hostname Leaf2
!
spanning-tree mode mstp
!
interface Ethernet1
   mtu 9000
   no switchport
   ipv6 enable
!
interface Ethernet2
   mtu 9000
   no switchport
   ipv6 enable
!
interface Ethernet3
!
interface Ethernet4
!
interface Ethernet5
!
interface Loopback0
   ipv6 address fd12:dc1:1::4/128
!
interface Management1
!
no ip routing
!
ipv6 unicast-routing
!
end
```

##### Leaf-3
```
hostname Leaf3
!
spanning-tree mode mstp
!
interface Ethernet1
   mtu 9000
   no switchport
   ipv6 enable
!
interface Ethernet2
   mtu 9000
   no switchport
   ipv6 enable
!
interface Ethernet3
!
interface Ethernet4
!
interface Ethernet5
!
interface Loopback0
   ipv6 address fd12:dc1:1::5/128
!
interface Management1
!
no ip routing
!
ipv6 unicast-routing
!
end
```
