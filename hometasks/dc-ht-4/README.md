
## Построение Underlay сети (BGP)

### Цель:
Настроить BGP для Underlay сети.

### План работ
1. Собрать схему Clos
2. Распределить адресное пространство
3. Настроить и отладка BGP в Underlay-сети
4. Проверка результатов работы

### Краткое oписание объекта
У нас есть ЦОД № 1, в который входит несколько PODов и в частности, POD № 1, схему которого мы и будем собирать по топологии Clos.

На самом верхнем уровне топологии Clos PODы объединяются коммутаторами Super-Spine, которые работают в area (10) BGP.

Интерфейсы коммутаторов POD № 1 будут принадлежать area 1.

### 1. Сборка схемы Clos

Схема по топологии Clos была собрана в EVE-NG. В качестве основных элементов схемы, Spine- и Leaf-коммутаторов, использовались виртуальные образы Arista vEOS-lab версии 4.29.2F.
![alt-текст](https://github.com/Gord-Gord/dc-network-design/blob/main/hometasks/dc-ht-4/scheme.png)

### 2. Распределение адресного пространства

В качестве протокола, на котором будет строиться Underlay, был выбран IPv6.
Всей сети POD1 выделено адресное пространство: fd12:dc1:1::/48, которое в свою очередь разбивается на следующие пулы IP-адресов:

Network|Назначение
-------|----------
fd12:dc1:1:0::/64|loopback 0
fd12:dc1:1:200::/55|p2p-link

Также необходимо определиться с номерами автомномных систем (AS). Они будут из диапазона приватных ASN.  
Чтобы трафик между Leaf-коммутаторами проходил ровно через один Spine-коммутатор, Spine1 и Spine2 будут иметь одинаковые номера AS. Пусть эти номера будет равными 64520. А Leaf-коммутаторам ASN будем назначать по-порядку: 64521, 64522, 64523 и т.д. 

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

Примечание: Для краткости не будем приводить здесь команды назначения IPv6 адресов на интерфейсы loopback 0 каждого коммутатора. Они показаны в листингах конфигураций оборудования. Отметим только то, что на каждом физическом интерфейсе устанавливаем mtu 9000, включаем IPv6.

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
   timers bgp 3 9
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
- далее необходимо объявить, что мы будем анонсировать нашим соседям. Для этого в настройки процесса BGP добавляем строку с ссылкой на карту маршрутизации:
```
router bgp 64520
   redistribute connected route-map rm-connected
   exit
```
- ну и создаём саму карту маршрутизации, которая указывает, что анонсировать нужно IPv6-адреса интерфейсов Loopback 0:
```
route-map rm-connected permit 10
   match interface Loopback0
```
Аналогичные действия выполняем на всех коммутаторах. По окончании мы видим, что соседства поднялись и получены маршруты:
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
Надо отметить, что, несмотря на то что коммутаторы установили соседство и успешно обменялись информацией о своих Loopback-интерфейсах, такой способ настройки BGP нельзя признать гибким и масштабируемым. При добавлении в сеть POD нового Leaf-коммутатора, нам каждый раз придётся прописывать в настройках BGP каждого Spine-коммутатора IPv6-адрес Link-local интерфейса добавляемого Leaf-коммутатора. И напротив: при добавлении в сеть POD нового Spine‑коммутатора мы будем вынуждены прописывать его в настройках всех подключаемых к нему Leaf‑коммутаторов.  
Чтобы уйти от такой немасштабируемой схемы:
- объявим peer-filter LEAF-AS-NUMBERS, включающий диапазон номеров приватных AS 64521-64535;
- выключаем на Spine-коммутаторах процесс BGP 64520 и создаем его заново:
- скажем новому процессу BGP какую сеть IPv6 слушать, чтобы установить соседство с указанными AS;
- ну и незабываем включить протокол BFD:
```
peer-filter LEAF-AS-NUMBERS
   10 match as-range 64512-64535 result accept

no router bgp 64520

router bgp 64520
   router-id 10.1.1.1
   timers bgp 3 9
   bgp listen range fd12:dc1:1:200::/55 peer-group LEAF-UNDERLAY peer-filter LEAF-AS-NUMBERS
   neighbor LEAF-UNDERLAY peer group
   neighbor LEAF-UNDERLAY bfd
   redistribute connected route-map rm-connected

   address-family ipv6
      neighbor LEAF-UNDERLAY activate
```
- на Leaf-коммутаторах действуем аналогично, но здесь нет необходимости создавать peer-filter так, как ASN у наших Spine-коммутаторов один 64520. Кроме того, мы здесь должны будем указать IPv6 адреса, прописанные на интерфейсах Spine-коммутаторов (пример для Leaf1):
```
no router bgp 64521

router bgp 64521
   router-id 10.1.1.3
   timers bgp 3 9
   neighbor SPINE-UNDERLAY peer group
   neighbor SPINE-UNDERLAY bfd
   neighbor fd12:dc1:1:200::1 peer group SPINE-UNDERLAY
   neighbor fd12:dc1:1:200::1 remote-as 64520
   neighbor fd12:dc1:1:203::1 peer group SPINE-UNDERLAY
   neighbor fd12:dc1:1:203::1 remote-as 64520
   redistribute connected route-map rm-connected
   !
   address-family ipv6
      neighbor SPINE-UNDERLAY activate
```
Смотрим что у нас получилось:
- Spine1:
```
Spine1#show ipv6 bgp summary
BGP summary information for VRF default
Router identifier 10.1.1.1, local AS number 64520
Neighbor Status Codes: m - Under maintenance
  Neighbor         V  AS           MsgRcvd   MsgSent  InQ OutQ  Up/Down State   PfxRcd PfxAcc
  fd12:dc1:1:200::2 4  64521            210       211    0    0 03:24:34 Estab   1      1
  fd12:dc1:1:201::2 4  64522            210       211    0    0 03:24:39 Estab   1      1
  fd12:dc1:1:202::2 4  64523            210       211    0    0 03:24:42 Estab   1      1
Spine1#show ipv6 route bgp

VRF: default
Displaying 3 of 14 IPv6 routing table entries
Codes: C - connected, S - static, K - kernel, O3 - OSPFv3,
       B - Other BGP Routes, A B - BGP Aggregate, R - RIP,
       I L1 - IS-IS level 1, I L2 - IS-IS level 2, DH - DHCP,
       NG - Nexthop Group Static Route, M - Martian,
       DP - Dynamic Policy Route, L - VRF Leaked,
       RC - Route Cache Route

 B E      fd12:dc1:1::3/128 [200/0]
           via fd12:dc1:1:200::2, Ethernet1
 B E      fd12:dc1:1::4/128 [200/0]
           via fd12:dc1:1:201::2, Ethernet2
 B E      fd12:dc1:1::5/128 [200/0]
           via fd12:dc1:1:202::2, Ethernet3
```
- Leaf1:
```
Leaf1#show ipv6 bgp summary
BGP summary information for VRF default
Router identifier 10.1.1.3, local AS number 64521
Neighbor Status Codes: m - Under maintenance
  Neighbor         V  AS           MsgRcvd   MsgSent  InQ OutQ  Up/Down State   PfxRcd PfxAcc
  fd12:dc1:1:200::1 4  64520            213       212    0    0 03:26:07 Estab   3      3
  fd12:dc1:1:203::1 4  64520            213       214    0    0 03:26:04 Estab   3      3
Leaf1#show ipv6 route bgp

VRF: default
Displaying 4 of 13 IPv6 routing table entries
Codes: C - connected, S - static, K - kernel, O3 - OSPFv3,
       B - Other BGP Routes, A B - BGP Aggregate, R - RIP,
       I L1 - IS-IS level 1, I L2 - IS-IS level 2, DH - DHCP,
       NG - Nexthop Group Static Route, M - Martian,
       DP - Dynamic Policy Route, L - VRF Leaked,
       RC - Route Cache Route

 B E      fd12:dc1:1::1/128 [200/0]
           via fd12:dc1:1:200::1, Ethernet1
 B E      fd12:dc1:1::2/128 [200/0]
           via fd12:dc1:1:203::1, Ethernet2
 B E      fd12:dc1:1::4/128 [200/0]
           via fd12:dc1:1:200::1, Ethernet1
 B E      fd12:dc1:1::5/128 [200/0]
           via fd12:dc1:1:200::1, Ethernet1
```
Соседства построены, маршруты получены, что и требовалось выполнить.
Такой подход, при котором на Spine‑коммутаторах заранее задают подсеть для BGP‑сессий и разрешённый диапазон ASN для установления соседства, представляется наиболее гибким и масштабируемым. Фактически подготовка Spine к подключению нового Leaf сводится к настройке IPv6‑адресов на интерфейсах Spine-коммутатора для подключения этого Leaf.
А на Leaf‑коммутаторе всё же требуется выполнить полный цикл настроек при его добавлении в IP‑фабрику.

Можем более детально посмотреть информацию на Spine1 о маршруте к Loopback 0 коммутатора Leaf1:
```
Spine1#show ipv6 bgp fd12:dc1:1::3/128
BGP routing table information for VRF default
Router identifier 10.1.1.1, local AS number 64520
BGP routing table entry for fd12:dc1:1::3/128
 Paths: 1 available
  64521
    fd12:dc1:1:200::2 from fd12:dc1:1:200::2 (10.1.1.3)
      Origin IGP, metric 0, localpref 100, IGP metric 1, weight 0, received 02:55:59 ago, valid, external, best
      Rx SAFI: Unicast
Spine1#
```
Мы видим, что происхождение (Origin) этого маршрута у нас стоит IGP - скорее всего это особенность Arista. А так как этот маршрут мы не получаем от протоколов семейства IGP, то более правильным будет установить Origin incomplete в route-map на всех коммутаторах:
```
route-map rm-connected permit 10
   set origin incomplete
```
Проверяем изменилась ли информация о получаемых маршрутах:
```
Spine1#show ipv6 bgp fd12:dc1:1::3/128
BGP routing table information for VRF default
Router identifier 10.1.1.1, local AS number 64520
BGP routing table entry for fd12:dc1:1::3/128
 Paths: 1 available
  64521
    fd12:dc1:1:200::2 from fd12:dc1:1:200::2 (10.1.1.3)
      Origin IGP, metric 0, localpref 100, IGP metric 1, weight 0, received 02:55:59 ago, valid, external, best
      Rx SAFI: Unicast
Spine1#
```
Видим что изменений нет. Оно и понятно, потому как BGP рассылает Update, когда что-то меняется в сети. Выключим на Leaf1 интерфейc, к которому подключен Spine1, а затем снова включим его. На Spine1 вновь проверяем изменилась ли информация:
```
Spine1#show ipv6 bgp fd12:dc1:1::3/128
BGP routing table information for VRF default
Router identifier 10.1.1.1, local AS number 64520
BGP routing table entry for fd12:dc1:1::3/128
 Paths: 1 available
  64521
    fd12:dc1:1:200::2 from fd12:dc1:1:200::2 (10.1.1.3)
      Origin INCOMPLETE, metric 0, localpref 100, IGP metric 1, weight 0, received 00:00:05 ago, valid, external, best
      Rx SAFI: Unicast
Spine1#
```
Да! Мы видим информация поменялась, теперь Origin INCOMPLETE.

### Проверка результатов работы

- В начале убедимся в установлении соседств bfd (статус Up в колонке State):

![alt-text](https://github.com/Gord-Gord/dc-network-design/blob/main/hometasks/dc-ht-4/spines_bfd_peers.png)
![alt-text](https://github.com/Gord-Gord/dc-network-design/blob/main/hometasks/dc-ht-4/leaves_bfd_peers.png)

- Далее смотрим установилось ли у нас BGP-соседство между Spine- и Leaf-коммутаторами. Об этом нам скажет статус Up в колонке State в строке с указанием  System-id BGP-соседа:

![alt-text](https://github.com/Gord-Gord/dc-network-design/blob/main/hometasks/dc-ht-4/spines_bgp_neighbors.png)
![alt-text](https://github.com/Gord-Gord/dc-network-design/blob/main/hometasks/dc-ht-4/leaves_bgp_neighbors.png)  
    
- Далее посмотрим какие маршруты получены по BGP и внесены в таблицу маршрутизации:

![alt-text](https://github.com/Gord-Gord/dc-network-design/blob/main/hometasks/dc-ht-4/spines_bgp_routes.png)
![alt-text](https://github.com/Gord-Gord/dc-network-design/blob/main/hometasks/dc-ht-4/leaves_bgp_routes.png)


- Проверим сетевую связность между интерфейсами loopback 0 разных коммутаторов:
    - Spine-коммутаторы пингуют loopback 0 Leaf-коммутаторов, при этом в качестве исходящего интерфейса обязательно указываем loopback 0 Spine-коммутатора:  
![Спайны пингуют лифы](https://github.com/Gord-Gord/dc-network-design/blob/main/hometasks/dc-ht-4/spines_are_pinging_leaves.png)  
    - Leaf-коммутаторы пингуют loopback 0 Spine-коммутаторов, при этом в качестве исходящего интерфейса обязательно указываем loopback 0 Leaf-коммутатора:
![Лифы пингуют спайны](https://github.com/Gord-Gord/dc-network-design/blob/main/hometasks/dc-ht-4/leaves_are_pinging_spines.png)  
    - Leaf-коммутаторы пингуют loopback 0 других Leaf-коммутаторов, при этом в качестве исходящего интерфейса обязательно указываем loopback 0 Leaf-коммутатора:
![Лифы пингуют Лифы](https://github.com/Gord-Gord/dc-network-design/blob/main/hometasks/dc-ht-4/leaves_are_pinging_leaves.png)

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
