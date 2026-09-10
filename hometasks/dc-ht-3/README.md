
## Построение Underlay сети (IS-IS)

### Цель:
Настроить IS-IS для Underlay сети.

### План работ
1. Собрать схему Clos;
2. Распределить адресное пространство;
3. Настроить IS-IS в Underlay-сети;
4. Настроить BFD;
5. Проверка результатов работы.

### Краткое oписание объекта
У нас есть ЦОД № 1, в который входит несколько PODов и в частности, POD № 1, схему которого мы и будем собирать по топологии Clos.

На самом верхнем уровне топологии Clos PODы объединяются коммутаторами Super-Spine, которые работают в area (10) IS-IS.

Интерфейсы коммутаторов POD № 1 будут принадлежать area 1.

### 1. Сборка схемы Clos

Схема по топологии Clos была собрана в EVE-NG. В качестве основных элементов схемы, Spine- и Leaf-коммутаторов, использовались виртуальные образы Arista vEOS-lab версии 4.29.2F.
![alt-текст](https://github.com/Gord-Gord/dc-network-design/blob/main/hometasks/dc-ht-2/scheme.png)

### 2. Распределение адресного пространства

В качестве протокола, на котором будет строиться Underlay, был выбран IPv6. IS-IS-соседство будем строить от Link-Local адресов.

Таким образом, остаётся назначить IPv6-адреса только интерфейсам loopback 0. Пусть IPv6-адреса, назначаемые интерфейсам loopback 0, принадлежат сети fd12:dc1:1:0::/64.

Также для каждого коммутатора необходимо определить NET (Network Entity Title) - уникальный идентификатор.
NET состоит из нескольких частей:
- AFI (Authority and Format Identifier) — идентификатор класса адреса. Всегда равно 49 и говорит, что это приватный адрес.
- Area ID — идентификатор зоны, к которой принадлежит узел. В нашем случае соответствует номеру POD, принимаем равным 0001.
- System ID — уникальный идентификатор самого устройства (см. ниже).
- NSEL (Network Selector) — селектор сети. Всегда должен быть равно 00. 

System-id будем формировать из router-id, которые мы ранее назначали коммутаторам, когда строили Underlay по протоколу OSPFv3.  
Пребразуем router-id в System-id по следующей схеме:  
- если десятичная запись значения октета router-id имеет 1 или 2 разряда, то дополняем его впереди стоящими двумя или одним 0 соответственно, чтобы число разрядов в значении октета стало равным 3;
- если десятичная запись значения октета router-id имеет 3 разряда, то оставляем его без изменений;
- преобразуем полученную запись в System-id - XXXX.XXXX.XXXX.  
Пример: 10.1.1.1 -> 010.001.001.001 -> 0100.0100.1001

Сведём полученные данные в таблицу:

Device|loopback 0|router-id|System-id|NET
------|----------|---------|---------|---
Spine1|fd12:dc1:1::1/128|10.1.1.1|0100.0100.1001|49.0001.0100.0100.1001.00
Spine2|fd12:dc1:1::2/128|10.1.1.2|0100.0100.1002|49.0001.0100.0100.1002.00
Leaf1|fd12:dc1:1::3/128|10.1.1.3|0100.0100.1003|49.0001.0100.0100.1003.00
Leaf2|fd12:dc1:1::4/128|10.1.1.4|0100.0100.1004|49.0001.0100.0100.1004.00
Leaf3|fd12:dc1:1::5/128|10.1.1.5|0100.0100.1005|49.0001.0100.0100.1005.00

где:  
- loopback 0 3-ий и 4-ый октеты - Порядковый номер ЦОДа - dc1;  
- loopback 0 6-ой октет - Порядковый номер POD - 1;  
- loopback 0 16-ый октет Порядковый номер устройства в POD;  
- router-id 2-ой октет Порядковый номер ЦОДа - 1;  
- router-id 3-ой октет Порядковый номер POD - 1;  
- router-id 4-ой октет - Порядковый номер устройства в POD.  

Примечание: Для краткости не будем приводить здесь команды назначения IPv6 адресов на интерфейсы loopback 0 каждого коммутатора. Они показаны в листингах конфигураций оборудования.

### Настройка IS-IS и включение BFD

На примере коммутатора Spine1 покажем как выполнялась конфигурация коммутаторов в POD:

- включаем на коммутаторе маршрутизацию IPv6:
```
Spine1(config)ipv6 unicast-routing vrf default
```

- далее, в vrf default запускаем процесс IS-IS и указываем NET:
```
Spine1(config)#router isis UNDERLAY vrf default
Spine1(config-router-isis)#net 49.0001.0100.0100.1001.00
Spine1(config-router-isis)#address-family ipv6 unicast
```

- на каждом из физических интерфейсов устанавливаем mtu 9214, включаем IPv6, указываем, что интерфейс включен в инстанс UNDERLAY процеса IS-IS, задаём тип сети point-to-point, а также уровень отношений с соседями на этом интерфейсе: L1:
```
Spine1(config)#interface Ethernet1-3
Spine1(config-if-Et1-3)#mtu 9214
Spine1(config-if-Et1-3)#no switchport
Spine1(config-if-Et1-3)#ipv6 enable
Spine1(config-if-Et1-3)#isis enable UNDERLAY
Spine1(config-if-Et1-3)#isis circuit-type level-1
Spine1(config-if-Et1-3)#isis network point-to-point
```

### Траблшутинг

После выполнения настроек на наших коммутаторах проверяем установилось ли соседство IS-IS. Поочередно на каждом коммутаторе командой "show isis neighbors" смотрим установленные коммутаторами по протоколу IS-IS отношения смежности.  
Вывод с коммутатора Spine1:
```
Spine1(config-router-isis)#show isis neighbors

Instance  VRF      System Id        Type Interface          SNPA              State Hold time   Circuit Id
UNDERLAY  default  Leaf1            L1   Ethernet1          P2P               UP    29          0B
UNDERLAY  default  Leaf2            L1   Ethernet2          P2P               INIT  24          0B
UNDERLAY  default  Leaf3            L1   Ethernet3          P2P               UP    29          0B
```
Как видим, коммутатор Spine1 установил соседство с Leaf1 и Leaf3, но с Leaf2 соседства нет.
Включаем перехват пакетов на Spine1 на интерфейсе eth2, на котором у нас линк с Leaf2. А затем на интерфейсе eth1 коммутатора Leaf2. Видим, что коммутаторы шлют пакеты с сообщениями IS-IS Hello. Изучаем содержимое этих сообщений. Видим что у обоих коммутаторов разные PDU Lenght.
Spine1:
![alt-text](https://github.com/Gord-Gord/dc-network-design/blob/main/hometasks/dc-ht-3/wireshark1.png)
Leaf2:
![alt-text](https://github.com/Gord-Gord/dc-network-design/blob/main/hometasks/dc-ht-3/wireshark2.png)
Значит допустили ошибку. Исправляем значения MTU на интерфейсах ethernet 1 и 2 Leaf2 и устанавливаем его 9124, как и у всех остальных.  
Также обращаем внимание, что Leaf-ы установили соседство только со Spine1, а со Spine2 соседства нет.
```
Leaf3#show isis neighbors

Instance  VRF      System Id        Type Interface          SNPA              State Hold time   Circuit Id
UNDERLAY  default  Spine1           L1   Ethernet1          P2P               UP    27          0D
```
Внимательно анализируем настройки протокола IS-IS на Spine2 и видим, что мы на его интерфейсах указали уровень отношений L2 тогда, как на всех интерфейсах всех Leaf-ов уровень отношений L1. Устраняем это расхождение.

### Проверка результатов работы

- В начале убедимся в установлении соседств bfd (статус Up в колонке State):

![alt-text](https://github.com/Gord-Gord/dc-network-design/blob/main/hometasks/dc-ht-2/spines_bfd_peers.png)
![alt-text](https://github.com/Gord-Gord/dc-network-design/blob/main/hometasks/dc-ht-2/leaves_bfd_peers.png)

- Далее смотрим установилось ли у нас IS-IS-соседство между Spine- и Leaf-коммутаторами. Об этом нам скажет словосочетание "state Full" в строке с указанием  router-id IS-IS-соседа:

![alt-text](https://github.com/Gord-Gord/dc-network-design/blob/main/hometasks/dc-ht-2/spines_ospf_neighbors.png)
![alt-text](https://github.com/Gord-Gord/dc-network-design/blob/main/hometasks/dc-ht-2/leaves_ospf_neighbors.png)  
    Здесь также стоит обратить внимание, что IS-IS работает в связке с BFD. Об этом говорит 
    строка "Bfd request is sent and the state is Up".

- Далее посмотрим какие маршруты получены по IS-IS и внесены в таблицу маршрутизации:

![alt-text](https://github.com/Gord-Gord/dc-network-design/blob/main/hometasks/dc-ht-2/spines_ospf_routes.png)
![alt-text](https://github.com/Gord-Gord/dc-network-design/blob/main/hometasks/dc-ht-2/leaves_ospf_routes.png)


- Проверим сетевую связность между интерфейсами loopback 0 разных коммутаторов:
    - Spine-коммутаторы пингуют loopback 0 Leaf-коммутаторов, при этом в качестве исходящего интерфейса обязательно указываем loopback 0 Spine-коммутатора:  
![Спайны пингуют лифы](https://github.com/Gord-Gord/dc-network-design/blob/main/hometasks/dc-ht-2/spines_are_pinging_leaves.png)  
    - Leaf-коммутаторы пингуют loopback 0 Spine-коммутаторов, при этом в качестве исходящего интерфейса обязательно указываем loopback 0 Leaf-коммутатора:
![Лифы пингуют спайны](https://github.com/Gord-Gord/dc-network-design/blob/main/hometasks/dc-ht-2/leaves_are_pinging_spines.png)  
    - Leaf-коммутаторы пингуют loopback 0 других Leaf-коммутаторов, при этом в качестве исходящего интерфейса обязательно указываем loopback 0 Leaf-коммутатора:
![Лифы пингуют Лифы](https://github.com/Gord-Gord/dc-network-design/blob/main/hometasks/dc-ht-2/leaves_are_pinging_leaves.png)

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
