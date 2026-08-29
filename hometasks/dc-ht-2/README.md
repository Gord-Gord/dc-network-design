
## Проектирование адресного пространства

### Цель:
Настроить OSPF для Underlay сети.

### План работ
1. Собрать схему Clos;
2. Распределить адресное пространство;
3. Настроить OSPF в Underlay-сети;
4. Настроить BFD;
5. Проверка результатов работы.

### Краткое oписание объекта
У нас есть ЦОД № 1, в который входит несколько PODов и в частности, POD № 1, схему которого мы и будем собирать по топологии Clos.

На самом верхнем уровне топологии Clos PODы объединяются коммутаторами Super-Spine, которые работают в backbone area (0) OSPF.

А вот интерфейсы коммутаторов POD № 1 будут принадлежать stub area 1.
### 1. Сборка схемы Clos



Схема по топологии Clos была собрана в EVE-NG. В качестве основных элементов схемы, Spine- и Leaf-коммутаторов, использовались виртуальные образы Arista vEOS-lab версии 4.29.2F.
![alt-текст](https://github.com/Gord-Gord/dc-network-design/blob/main/hometasks/dc-ht-2/scheme.png)


### 2. Распределение адресного пространства

В качестве протокола, на котором будет строиться Underlay, был выбран IPv6.  
Так как OSPF-соседство при использовании IPv6 всегда строится от Link-Local адресов, то IPv6 на интерфейсы Ethernet Spine- и Leaf-коммутаторов было принято решение не назначать, дабы не загромождать конфигурацию.


Таким образом, остаётся назначить IPv6-адреса только интерфейсам loopback 0, а заодно и определить router-id для каждого коммутатора. Пусть адреса, назначаемые интерфейсам loopback 0, принадлежат сети fd12:dc1:1:0::/64.
Тогда адресное пространство сети ЦОДа 1 примет следующий вид:

Device|loopback 0|router-id|
------|----------|---------|
Spine1|fd12:dc1:1::1/128|10.1.1.1
Spine2|fd12:dc1:1::2/128|10.1.1.2
Leaf1|fd12:dc1:1::3/128|10.1.1.3
Leaf2|fd12:dc1:1::4/128|10.1.1.4
Leaf3|fd12:dc1:1::5/128|10.1.1.5

где:  
- loopback 0 3-ий и 4-ый октеты - Порядковый номер ЦОДа - dc1;  
- loopback 0 6-ой октет - Порядковый номер POD - 1;  
- loopback 0 16-ый октет Порядковый номер устройства в POD;  
- router-id 2-ой октет Порядковый номер ЦОДа - 1;  
- router-id 3-ой октет Порядковый номер POD - 1;  
- router-id 4-ой октет - Порядковый номер устройства в POD.  

Примечание: Для краткости не будем приводить здесь команды назначения IPv6 адресов на интерфейсы loopback 0 каждого коммутатора. Они показаны в листингах конфигураций оборудования.

### Настройка OSPF и включение BFD

На примере коммутатора Spine1 покажем как выполнялась конфигурация коммутаторов в POD:

- включаем на коммутаторе маршрутизацию IPv6:
```
Spine1(config)ipv6 unicast-routing vrf default
```

- далее, в vrf default запускаем процесс ospfv3 и присваиваем area 1 тип зоны stub:
```
Spine1(config)#ipv6 router ospf 1 vrf default
Spine1(config-router-ospf3)#router-id 10.1.1.1
Spine1(config-router-ospf3)#area 1 stub
```

- на каждом из физических интерфейсов устанавливаем mtu 9214, включаем IPv6, помещаем их в area 1, указываем тип сети point-to-point, включаем BFD:
```
interface Ethernet1
   mtu 9214
   ipv6 enable
   ipv6 ospf network point-to-point
   ipv6 ospf 1 area 1
   ipv6 ospf bfd
interface Ethernet2
   mtu 9214
   ipv6 enable
   ipv6 ospf network point-to-point
   ipv6 ospf 1 area 1
   ipv6 ospf bfd
interface Ethernet3
   mtu 9214
   ipv6 enable
   ipv6 ospf network point-to-point
   ipv6 ospf 1 area 1
   ipv6 ospf bfd
```

### Проверка результатов работы

- В начале убедимся, что у нас построилось OSPF-соседство между Spine- и Leaf-коммутаторами:

![alt-text](https://github.com/Gord-Gord/dc-network-design/blob/main/hometasks/dc-ht-2/spines_ospf_neighbors.png)
![alt-text](https://github.com/Gord-Gord/dc-network-design/blob/main/hometasks/dc-ht-2/leaves_ospf_neighbors.png)

- Далее посмотрим какие маршруты получены по OSPF и внесены в таблицу маршрутизации:

![alt-text](https://github.com/Gord-Gord/dc-network-design/blob/main/hometasks/dc-ht-2/spines_ospf_routes.png)
![alt-text](https://github.com/Gord-Gord/dc-network-design/blob/main/hometasks/dc-ht-2/leaves_ospf_routes.png)

- Убедимся в установлении соседства bfd:

![alt-text](https://github.com/Gord-Gord/dc-network-design/blob/main/hometasks/dc-ht-2/spines_bfd_peers.png)
![alt-text](https://github.com/Gord-Gord/dc-network-design/blob/main/hometasks/dc-ht-2/leaves_bfd_peers.png)

- Проверим сетевую связность между интерфейсами loopback 0 разных коммутаторов:
    - Spine-коммутаторы пингуют loopback 0 Leaf-коммутаторов, при этом в качестве исходящего интерфейса обязательно указываем loopback 0 Spine-коммутатора:

![Спайны пингуют лифы](https://github.com/Gord-Gord/dc-network-design/blob/main/hometasks/dc-ht-2/spines_are_pinging_leaves.png)

    - Leaf-коммутаторы пингуют loopback 0 Spine-коммутаторов, при этом в качестве исходящего интерфейса обязательно указываем loopback 0 Leaf-коммутатора:

![Лифы пингуют спайны](https://github.com/Gord-Gord/dc-network-design/blob/main/hometasks/dc-ht-2/leaves_are_pinging_spines.png)

#### Листинги

##### Spine-1
```
hostname Spine1
!
spanning-tree mode mstp
!
interface Ethernet1
   description DWL-Leaf1-Eth1
   mtu 9214
   no switchport
   ipv6 enable
   ipv6 ospf bfd
   ipv6 ospf network point-to-point
   ipv6 ospf 1 area 0.0.0.1
!
interface Ethernet2
   description DWL-Leaf2-Eth1
   mtu 9214
   no switchport
   ipv6 enable
   ipv6 ospf bfd
   ipv6 ospf network point-to-point
   ipv6 ospf 1 area 0.0.0.1
!
interface Ethernet3
   description DWL-Leaf3-Eth1
   mtu 9214
   no switchport
   ipv6 enable
   ipv6 ospf bfd
   ipv6 ospf network point-to-point
   ipv6 ospf 1 area 0.0.0.1
!
interface Ethernet4
!
interface Ethernet5
!
interface Loopback0
   ipv6 address fd12:dc1:1::1/128
   ipv6 ospf 1 area 0.0.0.1
!
interface Management1
!
ip routing
!
ipv6 unicast-routing
!
ipv6 router ospf 1
   router-id 10.1.1.1
   area 0.0.0.1 stub
!
```

##### Spine-2
```
hostname Spine2
!
spanning-tree mode mstp
!
interface Ethernet1
   description DWL-Leaf1-Eth2
   mtu 9214
   no switchport
   ipv6 enable
   ipv6 ospf bfd
   ipv6 ospf network point-to-point
   ipv6 ospf 1 area 0.0.0.1
!
interface Ethernet2
   description DWL-Leaf2-Eth2
   mtu 9214
   no switchport
   ipv6 enable
   ipv6 ospf bfd
   ipv6 ospf network point-to-point
   ipv6 ospf 1 area 0.0.0.1
!
interface Ethernet3
   description DWL-Leaf3-Eth2
   mtu 9214
   no switchport
   ipv6 enable
   ipv6 ospf bfd
   ipv6 ospf network point-to-point
   ipv6 ospf 1 area 0.0.0.1
!
interface Ethernet4
!
interface Ethernet5
!
interface Loopback0
   ipv6 address fd12:dc1:1::2/128
   ipv6 ospf 1 area 0.0.0.1
!
interface Management1
!
ip routing
!
ipv6 unicast-routing
!
ipv6 router ospf 1
   router-id 10.1.1.2
   area 0.0.0.1 stub
!
```

##### Leaf-1
```
hostname Leaf1
!
spanning-tree mode mstp
!
interface Ethernet1
   description UPL-Spine1-Eth1
   mtu 9214
   no switchport
   ipv6 enable
   ipv6 ospf bfd
   ipv6 ospf network point-to-point
   ipv6 ospf 1 area 0.0.0.1
!
interface Ethernet2
   description UPL-Spine2-Eth1
   mtu 9214
   no switchport
   ipv6 enable
   ipv6 ospf bfd
   ipv6 ospf network point-to-point
   ipv6 ospf 1 area 0.0.0.1
!
interface Ethernet3
   shutdown
!
interface Ethernet4
   shutdown
!
interface Ethernet5
   shutdown
!
interface Loopback0
   ipv6 address fd12:dc1:1::3/128
   ipv6 ospf 1 area 0.0.0.1
!
interface Management1
!
no ip routing
!
ipv6 unicast-routing
!
ipv6 router ospf 1
   router-id 10.1.1.3
   area 0.0.0.1 stub
!
```

##### Leaf-2
```
hostname Leaf2
!
spanning-tree mode mstp
!
interface Ethernet1
   description UPL-Spine1-Eth2
   mtu 9214
   no switchport
   ipv6 enable
   ipv6 ospf bfd
   ipv6 ospf network point-to-point
   ipv6 ospf 1 area 0.0.0.1
!
interface Ethernet2
   description UPL-Spine2-Eth2
   mtu 9214
   no switchport
   ipv6 enable
   ipv6 ospf bfd
   ipv6 ospf network point-to-point
   ipv6 ospf 1 area 0.0.0.1
!
interface Ethernet3
   shutdown
!
interface Ethernet4
   shutdown
!
interface Ethernet5
   shutdown
!
interface Loopback0
   ipv6 address fd12:dc1:1::4/128
   ipv6 ospf 1 area 0.0.0.1
!
interface Management1
!
no ip routing
!
ipv6 unicast-routing
!
ipv6 router ospf 1
   router-id 10.1.1.4
   area 0.0.0.1 stub
!
```

##### Leaf-3
```
hostname Leaf3
!
spanning-tree mode mstp
!
interface Ethernet1
   description UPL-Spine1-Eth3
   mtu 9214
   no switchport
   ipv6 enable
   ipv6 ospf bfd
   ipv6 ospf network point-to-point
   ipv6 ospf 1 area 0.0.0.1
!
interface Ethernet2
   description UPL-Spine2-Eth3
   mtu 9214
   no switchport
   ipv6 enable
   ipv6 ospf bfd
   ipv6 ospf network point-to-point
   ipv6 ospf 1 area 0.0.0.1
!
interface Ethernet3
   shutdown
!
interface Ethernet4
   shutdown
!
interface Ethernet5
   shutdown
!
interface Loopback0
   ipv6 address fd12:dc1:1::5/128
   ipv6 ospf 1 area 0.0.0.1
!
interface Management1
!
no ip routing
!
ipv6 unicast-routing
!
ipv6 router ospf 1
   router-id 10.1.1.5
   area 0.0.0.1 stub
!
```
