
# Проектная работа

## Построение фабрики VXLAN/EVPN для двух POD с использованием технологии Multisite

### Цель:
1. В разных населенных пунктах построить два POD (Site) по топологии Clos на основе VxLAN EVPN для L2 связанности между клиентами
2. Обеспечить сетевую связность между POD(Site) с использованием технологии Multisite

### План работ
1. Собрать схему Clos
2. Распределить адресное пространство
3. Настроить IS-IS в Underlay-сети в каждом POD
4. Настроить Overlay на основе iBGP в каждом POD
5. Настроить DCI между BGW каждого POD
6. Проверка результатов работы

### Краткое oписание объекта
У нас есть 2 POD разнесённые в разные города (город А и город Б). Каждый POD мы будем собирать по топологии Clos.

Перед нами стоит задача обеспечить связность двух POD по сети оператора связи (ISP).

### 1. Сборка схемы Clos

Схема по топологии Clos была собрана в EVE-NG. В качестве основных элементов схемы, Spine-, Leaf-, и BGW-оммутаторов, использовались виртуальные образы Arista vEOS-lab версии 4.29.2F.
В роли клиентов выступают виртуальные эмуляторы VPC. Именно будут являться источником трафика, именно их MAC-адреса будут транслироваться соседям по фабрике.
В роли маршрутизатора ISP выступает образ Cisco IOSv верси 15.7(3)M3.
![alt-текст](https://github.com/Gord-Gord/dc-network-design/blob/main/hometasks/dc-ht-5/scheme.png)

В начале работы на каждом коммутаторе активируем в EOS мультиагентную архитектуру для протоколов маршрутизации. Это необходимо для функционирования EVPN.
```
service routing protocols model multi-agent
```

### 2. Распределение адресного пространства

В качестве протокола, на котором будет строиться Underlay, был выбран IPv6. IS-IS-соседство будем строить от Link-Local адресов.

Таким образом, остаётся назначить IPv6-адреса только интерфейсам loopback 0 и loopback 1. Пусть IPv6-адреса, назначаемые интерфейсам loopback 0 на Spine-коммутаторах принадлежат сети fd12:dcN:1:0::/63, а на Leaf- и BGW-коммутаторах сети fd12:dcN:1:2::/63, где N - номер POD.

Также для каждого коммутатора необходимо определить NET (Network Entity Title) - уникальный идентификатор.
NET состоит из нескольких частей:
- AFI (Authority and Format Identifier) — идентификатор класса адреса. Всегда равно 49 и говорит, что это приватный адрес.
- Area ID — идентификатор зоны, к которой принадлежит узел. В нашем случае соответствует номерам POD, принимаем равными 0001 и 0002 соответственно.
- System ID — уникальный идентификатор самого устройства (см. ниже).
- NSEL (Network Selector) — селектор сети. Всегда должен быть равно 00. 

System-id будем формировать из router-id, которые мы ранее назначим коммутаторам, когда будем строить OverLay по протоколу BGP.  
Пребразуем router-id в System-id по следующей схеме:  
- если десятичная запись значения октета router-id имеет 1 или 2 разряда, то дополняем его впереди стоящими двумя или одним 0 соответственно, чтобы число разрядов в значении октета стало равным 3;
- если десятичная запись значения октета router-id имеет 3 разряда, то оставляем его без изменений;
- преобразуем полученную запись в System-id - XXXX.XXXX.XXXX.  
Пример: 10.1.1.1 -> 010.001.001.001 -> 0100.0100.1001

Сведём полученные данные в таблицу:

Device|loopback 0|loopback 1|router-id|System-id|NET
------|----------|----------|---------|---------|---
Spine1-1|fd12:dc1:1::1/128| нет |10.1.1.1|0100.0100.1001|49.0001.0100.0100.1001.00
Spine1-2|fd12:dc1:1::2/128| нет |10.1.1.2|0100.0100.1002|49.0001.0100.0100.1002.00
Leaf1-1|fd12:dc1:1::3/128| нет |10.1.1.3|0100.0100.1003|49.0001.0100.0100.1003.00
BGW1-1|fd12:dc1:1::4/128|fd12:dc1:100::100|10.1.1.4|0100.0100.1004|49.0001.0100.0100.1004.00
BGW1-2|fd12:dc1:1::5/128|fd12:dc1:100::100|10.1.1.5|0100.0100.1005|49.0001.0100.0100.1005.00
Spine2-1|fd12:dc2:1::1/128| нет |10.2.1.1|0100.0100.1001|49.0002.0100.0100.1001.00
Spine2-2|fd12:dc2:1::2/128| нет |10.2.1.2|0100.0100.1002|49.0002.0100.0100.1002.00
Leaf2-1|fd12:dc2:1::3/128| нет |10.2.1.3|0100.0100.1003|49.0002.0100.0100.1005.00
BGW2-1|fd12:dc2:1::4/128|fd12:dc2:100::100|10.2.1.4|0100.0100.1004|49.0002.0100.0100.1004.00
BGW2-2|fd12:dc2:1::5/128|fd12:dc2:100::100|10.2.1.5|0100.0100.1005|49.0002.0100.0100.1003.00

где:  
- loopback 0 3-ий и 4-ый октеты - Порядковый номер ЦОДа - dc1;  
- loopback 0 6-ой октет - Порядковый номер POD - 1;  
- loopback 0 16-ый октет Порядковый номер устройства в POD;  
- router-id 2-ой октет Порядковый номер ЦОДа - 1;  
- router-id 3-ой октет Порядковый номер POD - 1;  
- router-id 4-ой октет - Порядковый номер устройства в POD.  

У нас есть также виртуальные PC, играющие роль серверов. Они работают каждый в своём VLAN. Назначим каждому VPC IP-адрес, а каждому VLAN свой VNI:

Device|IP-адрес|VLAN ID|VNI
------|--------|-------|--
VPC1-1|10.10.1.3/24|10|1000010
VPC1-2|10.20.1.8/24|20|1000020
VPC2-1|10.10.1.14/24|10|1000010
VPC2-2|10.20.1.5/24|20|1000020

Примечание: Для краткости не будем приводить здесь команды назначения IPv6 адресов на интерфейсы loopback 0 и 1 каждого коммутатора. Они показаны в листингах конфигураций оборудования.

### 3. Настройка IS-IS (Underlay)
На примере коммутатора Spine1-1 покажем как выполнялась конфигурация коммутаторов в POD1:

- включаем на коммутаторе маршрутизацию IPv6:
```
Spine1-1(config)ipv6 unicast-routing vrf default
```

- далее, в vrf default запускаем процесс IS-IS и указываем NET:
```
Spine1-1(config)#router isis SITE1-UNDERLAY vrf default
Spine1-1(config-router-isis)#net 49.0001.0100.0100.1001.00
Spine1-1(config-router-isis)#address-family ipv6 unicast
```

- на каждом из физических интерфейсов устанавливаем mtu 9000, включаем IPv6, указываем, что интерфейс включен в инстанс SITE1-UNDERLAY процеса IS-IS, включаем BFD, задаём тип сети point-to-point, а также уровень отношений с соседями на этом интерфейсе: L1:
```
Spine1-1(config)#interface Ethernet1-3
Spine1-1(config-if-Et1-3)#mtu 9000
Spine1-1(config-if-Et1-3)#no switchport
Spine1-1(config-if-Et1-3)#ipv6 enable
Spine1-1(config-if-Et1-3)#isis enable SITE1-UNDERLAY
Spine1-1(config-if-Et1-3)#isis ipv6 bfd
Spine1-1(config-if-Et1-3)#isis circuit-type level-1
Spine1-1(config-if-Et1-3)#isis network point-to-point
```
- настраиваем аутентификацию, как дополнительный функционал протокола IS-IS.
    - вначале включаем аутентификации на интерфейсах. Это обеспечит аутентификацию только HELLO-пакетов.
    ```
   Spine1-1(config)#interface Ethernet1
   Spine1-1(config-if-Et1-3)#isis authentication mode sha key-id 33333
   Spine1-1(config-if-Et1-3)#isis authentication key-id 33333 algorithm sha-256 key 7 fQ2hST20Ozo7y2Zpdxveag==
   ```
   - теперь включаем аунтентификацию на уровне процесса. Это защитит LSDB от несанкционированной записи маршрутов:
   ```
   Spine1-1(config)#router isis SITE1-UNDERLAY
   Spine1-1(config-if-Et1-3)#authentication mode sha key-id 33333
   Spine1-1(config-if-Et1-3)#authentication key-id 33333 algorithm sha-256 key 7 fQ2hST20Ozo7y2Zpdxveag==
   ```
   - проверяем работу аутентификации на примере IS-IS соседства коммутаторов Spine1-1 и Leaf1-1, убедившись, что соседство между ними установлено и маршруты мы получаем: 
       ```
       Leaf1-1#show isis neighbors

       Instance  VRF      System Id        Type Interface          SNPA              State Hold time   Circuit Id
       SITE1-UNDERLAY  default  Spine1-1           L1   Ethernet1          P2P               UP    26          0B
       SITE1-UNDERLAY  default  Spine1-2           L1   Ethernet2          P2P               UP    30          0D

       Leaf1-1#show ipv6 route isis
       
       VRF: default
       Displaying 4 of 8 IPv6 routing table entries
       Codes: C - connected, S - static, K - kernel, O3 - OSPFv3,
              B - Other BGP Routes, A B - BGP Aggregate, R - RIP,
              I L1 - IS-IS level 1, I L2 - IS-IS level 2, DH - DHCP,
              NG - Nexthop Group Static Route, M - Martian,
              DP - Dynamic Policy Route, L - VRF Leaked,
              RC - Route Cache Route
       
         I L1     fd12:dc1:1::1/128 [115/20]
                   via fe80::5200:ff:fed7:ee0b, Ethernet1
         I L1     fd12:dc1:1::2/128 [115/20]
                   via fe80::5200:ff:fecb:38c2, Ethernet2
         I L1     fd12:dc1:1::4/128 [115/30]
                   via fe80::5200:ff:fed7:ee0b, Ethernet1
                   via fe80::5200:ff:fecb:38c2, Ethernet2
         I L1     fd12:dc1:1::5/128 [115/30]
                   via fe80::5200:ff:fed7:ee0b, Ethernet1
                   via fe80::5200:ff:fecb:38c2, Ethernet2
         I L1     fd12:dc1:100::100/128 [115/30]
                   via fe80::5200:ff:fed7:ee0b, Ethernet1
                   via fe80::5200:ff:fecb:38c2, Ethernet2
       ```
Выполняем аналогичные действия в POD2 согласно разработанному адресному плану. По окончании убеждаемся, что все устройства установили соседство по протоколу isis и обмениваются маршрутами.

### 4. Настройка EBGP (Overlay)

Процедура настройки EBGP на Spine- и Leaf-коммутаторах будут отличаться друг от друга.  
Опишем сначала процедуру настройки на Spine-коммутаторе:
- объявим peer-filter LEAF-AS-NUMBERS, включающий диапазон номеров приватных AS на Leaf 64521-64535;
```
peer-filter LEAF-AS-NUMBERS
   10 match as-range 64512-64535 result accept
```
- создаём на Spine-коммутаторах процесс BGP 64520, при этом указываем какую сеть IPv6 (Loopback 0) слушать, чтобы установить соседство с указанными в peer-filter номерами AS:
```
router bgp 64520
   router-id 10.1.1.1
   no bgp default ipv4-unicast
   maximum-paths 4 ecmp 4
   bgp listen range fd12:dc1:1:2::/64 peer-group LEAVES peer-filter LEAF-AS-NUMBERS
   neighbor LEAVES peer group
   neighbor LEAVES update-source Loopback0
   neighbor LEAVES ebgp-multihop 2
   neighbor LEAVES route-reflector-client
   neighbor LEAVES send-community extended
   neighbor LEAVES maximum-routes 0
   !
   address-family evpn
      neighbor LEAVES activate
   !
   address-family ipv4
      neighbor LEAVES activate
   !
   address-family ipv6
      neighbor LEAVES activate
```
Незабываем указать, что наши Spine будут являться клиентами и отражать маршруты, а также то, что максимальное число транзитных хопов для EBGP 2.

На Leaf-коммутаторах нет необходимости создавать peer-filter так, как ASN у наших Spine-коммутаторов один 64520. Кроме того, мы здесь должны будем указать IPv6 адреса, прописанные на Loopback 0 Spine-коммутаторов (пример для Leaf1):
```
router bgp 64521
   router-id 10.1.1.3
   no bgp default ipv4-unicast
   maximum-paths 4 ecmp 4
   neighbor SPINES peer group
   neighbor SPINES remote-as 64520
   neighbor SPINES update-source Loopback0
   neighbor SPINES ebgp-multihop 2
   neighbor SPINES send-community extended
   neighbor fd12:dc1:1::1 peer group SPINES
   neighbor fd12:dc1:1::2 peer group SPINES
   !
   address-family evpn
      neighbor SPINES activate
   !
   address-family ipv4
      neighbor SPINES activate
   !
   address-family ipv6
      neighbor SPINES activate
```
Смотрим что у нас получилось:
- Spine1:
```
Spine1#show ipv6 bgp summary
BGP summary information for VRF default
Router identifier 10.1.1.1, local AS number 64520
Neighbor Status Codes: m - Under maintenance
  Neighbor        V AS           MsgRcvd   MsgSent  InQ OutQ  Up/Down State   PfxRcd PfxAcc
  fd12:dc1:1:2::1 4 64521             63        58    0    0 00:43:28 Estab   0      0
  fd12:dc1:1:2::2 4 64522             64        62    0    0 00:43:13 Estab   0      0
  fd12:dc1:1:2::3 4 64523             62        61    0    0 00:43:36 Estab   0      0
```
- Leaf1:
```
Leaf1#show ipv6 bgp summary
BGP summary information for VRF default
Router identifier 10.1.1.3, local AS number 64521
Neighbor Status Codes: m - Under maintenance
  Neighbor      V AS           MsgRcvd   MsgSent  InQ OutQ  Up/Down State   PfxRcd PfxAcc
  fd12:dc1:1::1 4 64520             60        66    0    0 00:45:32 Estab   0      0
  fd12:dc1:1::2 4 64520             62        66    0    0 00:45:44 Estab   0      0
```
Итак, BGP-соседство установлено, переходим к следующему этапу, а именно к построению EVPN.

### 5. Настройка EVPN

На каждом Leaf-коммутаторе создаём интерфейсы VXLAN и каждому VLAN сопостовляем его VNI:
```
interface Vxlan1
   vxlan source-interface Loopback0
   vxlan udp-port 4789
   vxlan encapsulation ipv6
   vxlan vlan 10 vni 1000010
   vxlan vlan 20 vni 1000020
```
А в настройках процесса BGP указываем, что нужно дистрибьютить и включаем работу EVPN:
```
router bgp 64521
   vlan 10
      rd 1.1.1.3:10
      route-target both 64520:1000010
      redistribute learned
   !
   vlan 20
      rd 1.1.1.3:20
      route-target both 64520:1000020
      redistribute learned
   !
   address-family evpn
      neighbor SPINES activate
```
На Spine-коммутаторах этого делать не нужно, там достаточно включить EVPN:
```
router bgp 64520
 address-family evpn
    neighbor LEAVES activate
```


### Проверка результатов работы

- Убедимся, что каждый виртуальный PC видит своего соседа в том же VLAN:

![alt-text](https://github.com/Gord-Gord/dc-network-design/blob/main/hometasks/dc-ht-5/VPC_are_pinging_VPC.png)

Ок! Значит связность есть.

- Далее смотрим установилось ли у нас BGP-EVPN-соседство между Spine- и Leaf-коммутаторами. Об этом нам скажет статус Up в колонке State в строке с указанием  System-id IS-IS-соседа:

![alt-text](https://github.com/Gord-Gord/dc-network-design/blob/main/hometasks/dc-ht-5/spines-show-bgp-evpn-summary.png)
![alt-text](https://github.com/Gord-Gord/dc-network-design/blob/main/hometasks/dc-ht-5/leaves-show-bgp-evpn-summary.png)  
    
- Далее смотрим маршруты типа 2 и 3 на Spine-коммутаторах:

![alt-text](https://github.com/Gord-Gord/dc-network-design/blob/main/hometasks/dc-ht-5/spine-route-type2.png)
![alt-text](https://github.com/Gord-Gord/dc-network-design/blob/main/hometasks/dc-ht-5/spine-route-type3.png)

- Cмотрим маршруты типа 2 и 3 на Leaf-коммутаторах:

![alt-text](https://github.com/Gord-Gord/dc-network-design/blob/main/hometasks/dc-ht-5/leaves-route-type2.png)
![alt-text](https://github.com/Gord-Gord/dc-network-design/blob/main/hometasks/dc-ht-5/leaves-route-type3.png)

- Смотрим удалённые VTEP и типы тоннелей до них:

![alt-text](https://github.com/Gord-Gord/dc-network-design/blob/main/hometasks/dc-ht-5/leaves-vxlan-vtep.png)

- Таблицы MAC-адрессов каждого Leaf:

![alt-text](https://github.com/Gord-Gord/dc-network-design/blob/main/hometasks/dc-ht-5/leaves-mac-address-table.png)

Выводы:
1. Настроен BGP peering между Leaf и Spine в AF l2vpn evpn.
2. Связанность между клиентами обеспечена.

#### Листинги

[Spine1.cfg](./Spine1.cfg)  
[Spine2.cfg](./Spine2.cfg)  
[Leaf1.cfg](./Leaf1.cfg)  
[Leaf2.cfg](./Leaf2.cfg)  
[Leaf3.cfg](./Leaf3.cfg)
