
## Построение Overlay на основе VxLAN EVPN L2

### Цель:
Настроить Overlay на основе VxLAN EVPN для L2 связанности между клиентами

### План работ
1. Собрать схему Clos
2. Распределить адресное пространство
3. Настроить IS-IS в Underlay-сети
4. Настроить Overlay на основе eBGP
5. Траблшутинг
6. Проверка результатов работы

### Краткое oписание объекта
У нас есть ЦОД № 1, в который входит несколько PODов и в частности, POD № 1, схему которого мы и будем собирать по топологии Clos.

На самом верхнем уровне топологии Clos PODы объединяются коммутаторами Super-Spine, которые работают в area (10) IS-IS.

Интерфейсы коммутаторов POD № 1 будут принадлежать area 1.

### 1. Сборка схемы Clos

Схема по топологии Clos была собрана в EVE-NG. В качестве основных элементов схемы, Spine- и Leaf-коммутаторов, использовались виртуальные образы Arista vEOS-lab версии 4.29.2F.
В роли клиентов выступают виртуальные эмуляторы VPC. Именно будут являться источником трафика, именно их MAC-адреса будут транслироваться соседям по фабрике.
![alt-текст](https://github.com/Gord-Gord/dc-network-design/blob/main/hometasks/dc-ht-5/scheme.png)

### 2. Распределение адресного пространства

В качестве протокола, на котором будет строиться Underlay, был выбран IPv6. IS-IS-соседство будем строить от Link-Local адресов.

Таким образом, остаётся назначить IPv6-адреса только интерфейсам loopback 0. Пусть IPv6-адреса, назначаемые интерфейсам loopback 0 на Spine-коммутаторах принадлежат сети fd12:dc1:1:0::/63, а на Leaf-коммутаторах сети fd12:dc1:1:2::/63

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
Leaf1|fd12:dc1:1:2::3/128|10.1.1.3|0100.0100.1003|49.0001.0100.0100.1003.00
Leaf2|fd12:dc1:1:2::4/128|10.1.1.4|0100.0100.1004|49.0001.0100.0100.1004.00
Leaf3|fd12:dc1:1:2::5/128|10.1.1.5|0100.0100.1005|49.0001.0100.0100.1005.00

где:  
- loopback 0 3-ий и 4-ый октеты - Порядковый номер ЦОДа - dc1;  
- loopback 0 6-ой октет - Порядковый номер POD - 1;  
- loopback 0 16-ый октет Порядковый номер устройства в POD;  
- router-id 2-ой октет Порядковый номер ЦОДа - 1;  
- router-id 3-ой октет Порядковый номер POD - 1;  
- router-id 4-ой октет - Порядковый номер устройства в POD.  

У нас есть также виртуальные PC, играющие роль серверов. У них будет следующая IP-адресация:

Device|VLAN ID|IP-адрес
------|-------|--------
VPC1|10|10.10.1.3/24
VPC2|20|10.20.1.18/24
VPC3|10|10.10.1.14/24
VPC4|20|10.20.1.5/24

Примечание: Для краткости не будем приводить здесь команды назначения IPv6 адресов на интерфейсы loopback 0 каждого коммутатора. Они показаны в листингах конфигураций оборудования.

### Настройка IS-IS
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

- на каждом из физических интерфейсов устанавливаем mtu 9000, включаем IPv6, указываем, что интерфейс включен в инстанс UNDERLAY процеса IS-IS, включаем BFD, задаём тип сети point-to-point, а также уровень отношений с соседями на этом интерфейсе: L1:
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
- в качестве эксперимента попробуем аутентификацию, как дополнительный функционал протокола IS-IS.
    - вначале включаем аутентификации на интерфейсах. Это обеспечит аутентификацию только HELLO-пакетов.
    ```
   Spine1(config)#interface Ethernet1
   Spine1(config-if-Et1-3)#isis authentication mode sha key-id 33333
   Spine1(config-if-Et1-3)#isis authentication key-id 33333 algorithm sha-256 key 7 a8n1JXStqfBfR+URg/kKog==
   ```
   - теперь включаем аунтентификацию на уровне процесса. Это защитит LSDB от несанкционированной записи маршрутов:
   ```
   Spine1(config)#router isis UNDERLAY
   Spine1(config-if-Et1-3)#authentication mode sha key-id 33333
   Spine1(config-if-Et1-3)#authentication key-id 33333 algorithm sha-256 key 7 a8n1JXStqfBfR+URg/kKog== level-1
   ```
   - проверяем работу аутентификации на примере IS-IS соседства коммутаторов Spine1 и Leaf3:
       - вначале убеждаемся, что соседство между ними на данный момент установлено и маршруты мы получаем: 
       ```
       Leaf3#show isis neighbors

       Instance  VRF      System Id        Type Interface          SNPA              State Hold time   Circuit Id
       UNDERLAY  default  Spine1           L1   Ethernet1          P2P               UP    26          0B
       UNDERLAY  default  Spine2           L1   Ethernet2          P2P               UP    30          0D
       Leaf3#show ipv6 route isis
       
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
        I L1     fd12:dc1:1:2::1/128 [115/30]
                  via fe80::5200:ff:fed7:ee0b, Ethernet1
                  via fe80::5200:ff:fecb:38c2, Ethernet2
        I L1     fd12:dc1:1:2::2/128 [115/30]
                  via fe80::5200:ff:fed7:ee0b, Ethernet1
                  via fe80::5200:ff:fecb:38c2, Ethernet2
       ```
       - выключаем авторизацию на уровне процесса isis на коммутаторе Leaf3:
       ```
       Leaf3(config-router-isis)#no authentication key-id 33333 algorithm sha-256 key 7 a8n1JXStqfBfR+URg/kKog== level-1
       Leaf3(config-router-isis)#no authentication mode sha key-id 33333
       ```
       - очищаем instance UNDERLAY процесса isis:
           ```
           Leaf3(config-router-isis)#clear isis UNDERLAY instance

           IS-IS instance UNDERLAY cleared.
           ```
       - и теперь вновь смотрим соседство и маршруты:
           ```
           Leaf3(config-router-isis)#show isis neighbors

           Instance  VRF      System Id        Type Interface          SNPA              State Hold time   Circuit Id
           UNDERLAY  default  Spine1           L1   Ethernet1          P2P               UP    29          0B
           UNDERLAY  default  Spine2           L1   Ethernet2          P2P               UP    29          0D
    
           Leaf3(config)#show ipv6 route isis
           
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
            I L1     fd12:dc1:1:2::1/128 [115/30]
                      via fe80::5200:ff:fed7:ee0b, Ethernet1
                      via fe80::5200:ff:fecb:38c2, Ethernet2
            I L1     fd12:dc1:1:2::2/128 [115/30]
                      via fe80::5200:ff:fed7:ee0b, Ethernet1
                      via fe80::5200:ff:fecb:38c2, Ethernet2

           Spine1#show isis neighbors
           
           Instance  VRF      System Id        Type Interface          SNPA              State Hold time   Circuit Id
           UNDERLAY  default  Leaf1            L1   Ethernet1          P2P               UP    25          09
           UNDERLAY  default  Leaf2            L1   Ethernet2          P2P               UP    27          0B
           UNDERLAY  default  0100.0100.1005   L1   Ethernet3          P2P               UP    25          09
           Spine1#show ipv6 route isis
           
           VRF: default
           Displaying 3 of 7 IPv6 routing table entries
           Codes: C - connected, S - static, K - kernel, O3 - OSPFv3,
                  B - Other BGP Routes, A B - BGP Aggregate, R - RIP,
                  I L1 - IS-IS level 1, I L2 - IS-IS level 2, DH - DHCP,
                  NG - Nexthop Group Static Route, M - Martian,
                  DP - Dynamic Policy Route, L - VRF Leaked,
                  RC - Route Cache Route
           
            I L1     fd12:dc1:1::2/128 [115/30]
                      via fe80::5200:ff:fed5:5dc0, Ethernet1
                      via fe80::5200:ff:fe03:3766, Ethernet2
            I L1     fd12:dc1:1:2::1/128 [115/20]
                      via fe80::5200:ff:fed5:5dc0, Ethernet1
            I L1     fd12:dc1:1:2::2/128 [115/20]
                      via fe80::5200:ff:fe03:3766, Ethernet2
           ```
       Как видим, соседство между коммутаторами по-прежнему установлено. При этом, несмотря на выключенную аутентификацию, в GRT присутствует маршрутная информация, полученная по протоколу IS‑IS.  
Это объясняется тем, что аутентификация в настройках процесса IS‑IS на Leaf3 отключена — соответственно, отключена и проверка поступающих пакетов LSP. Поэтому маршрутная информация продолжает поступать на Leaf3 и попадает в его GRT.  

В то же время LSP от Leaf3 к коммутатору Spine1 идут без ключа и, соответственно, не проходят аутентификацию на Spine1: он продолжает их проверять, так как аутентификация на нём включена. В результате информация из этих пакетов не попадает в GRT Spine1 — в частности, у него уже отсутствует информация о Loopback‑интерфейсе 0 коммутатора Leaf3.

       - далее выключаем аутентификацию на интерфейсе Ethernet 2, сбрасываем соседство в instance UNDERLAY и смотрим результат:
       ```
       Leaf3(config)#interface ethernet 2
       Leaf3(config-if-Et2)#no isis authentication key-id 33333 algorithm sha-256 key 7 a8n1JXStqfBfR+URg/kKog==
       Leaf3(config-if-Et2)#no isis authentication mode sha key-id 33333
       Leaf3(config-if-Et2)#clear isis UNDERLAY neighbor all
       
       Leaf3(config-if-Et2)#show isis neighbor
       
       Instance  VRF      System Id        Type Interface          SNPA              State Hold time   Circuit Id
       UNDERLAY  default  Spine1           L1   Ethernet1          P2P               UP    29          0B
       UNDERLAY  default  Spine2           L1   Ethernet2          P2P               INIT  25          0D
       Leaf3#show ipv6 route isis

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
        I L1     fd12:dc1:1::2/128 [115/40]
                  via fe80::5200:ff:fed7:ee0b, Ethernet1
        I L1     fd12:dc1:1:2::1/128 [115/30]
                  via fe80::5200:ff:fed7:ee0b, Ethernet1
        I L1     fd12:dc1:1:2::2/128 [115/30]
                  via fe80::5200:ff:fed7:ee0b, Ethernet1
        ```
        Здесь мы видим, что соседство со Spine2, ранее установленое по Ethernet2, нарушено, а все маршруты к Loopback-интерфейсам других коммутаторов построены через Ethernet1.  
        Вывод: аутентификация устройство по протоколу IS-IS работает.

### Настроука EBGP (Overlay)

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

### Настройка EVPN

На каждом Leaf-коммутаторе создаём интерфейсы VXLAN и каждому VLAN сопостовляем его VNI:
```
interface Vxlan1
   vxlan source-interface Loopback0
   vxlan udp-port 4789
   vxlan encapsulation ipv6
   vxlan vlan 10 vni 1000010
   vxlan vlan 20 vni 1000020
```
А в настройка процесс BGP указываем, что нужно дистрибьютить и включаем работу EVPN:
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

- В начале убедимся в установлении соседств bfd (статус Up в колонке State):

![alt-text](https://github.com/Gord-Gord/dc-network-design/blob/main/hometasks/dc-ht-3/spines_bfd_peers.png)
![alt-text](https://github.com/Gord-Gord/dc-network-design/blob/main/hometasks/dc-ht-3/leaves_bfd_peers.png)

- Далее смотрим установилось ли у нас IS-IS-соседство между Spine- и Leaf-коммутаторами. Об этом нам скажет статус Up в колонке State в строке с указанием  System-id IS-IS-соседа:

![alt-text](https://github.com/Gord-Gord/dc-network-design/blob/main/hometasks/dc-ht-3/spines_isis_neighbors.png)
![alt-text](https://github.com/Gord-Gord/dc-network-design/blob/main/hometasks/dc-ht-3/leaves_isis_neighbors.png)  
    
- Далее посмотрим какие маршруты получены по IS-IS и внесены в таблицу маршрутизации:

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
   isis enable UNDERLAY
   no isis bfd
   isis ipv6 bfd
   isis circuit-type level-1
   isis network point-to-point
!
interface Ethernet2
   mtu 9000
   no switchport
   ipv6 enable
   isis enable UNDERLAY
   no isis bfd
   isis ipv6 bfd
   isis circuit-type level-1
   isis network point-to-point
!
interface Ethernet3
   mtu 9000
   no switchport
   ipv6 enable
   isis enable UNDERLAY
   no isis bfd
   isis ipv6 bfd
   isis circuit-type level-1
   isis network point-to-point
!
interface Ethernet4
!
interface Ethernet5
!
interface Loopback0
   ipv6 address fd12:dc1:1::1/128
   isis enable UNDERLAY
   isis circuit-type level-1
!
interface Management1
!
no ip routing
!
ipv6 unicast-routing
!
router isis UNDERLAY
   net 49.0001.0100.0100.1001.00
   !
   address-family ipv6 unicast
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
   isis enable UNDERLAY
   no isis bfd
   isis ipv6 bfd
   isis circuit-type level-1
   isis network point-to-point
!
interface Ethernet2
   mtu 9000
   no switchport
   ipv6 enable
   isis enable UNDERLAY
   no isis bfd
   isis ipv6 bfd
   isis circuit-type level-1
   isis network point-to-point
!
interface Ethernet3
   mtu 9000
   no switchport
   ipv6 enable
   isis enable UNDERLAY
   no isis bfd
   isis ipv6 bfd
   isis circuit-type level-1
   isis network point-to-point
!
interface Ethernet4
!
interface Ethernet5
!
interface Loopback0
   ipv6 address fd12:dc1:1::2/128
   isis enable UNDERLAY
   isis circuit-type level-1
!
interface Management1
!
no ip routing
!
ipv6 unicast-routing
!
router isis UNDERLAY
   net 49.0001.0100.0100.1002.00
   !
   address-family ipv6 unicast
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
   isis enable UNDERLAY
   no isis bfd
   isis ipv6 bfd
   isis circuit-type level-1
   isis network point-to-point
!
interface Ethernet2
   mtu 9000
   no switchport
   ipv6 enable
   isis enable UNDERLAY
   no isis bfd
   isis ipv6 bfd
   isis circuit-type level-1
   isis network point-to-point
!
interface Ethernet3
!
interface Ethernet4
!
interface Ethernet5
!
interface Loopback0
   ipv6 address fd12:dc1:1::3/128
   isis enable UNDERLAY
!
interface Management1
!
no ip routing
!
ipv6 unicast-routing
!
router isis UNDERLAY
   net 49.0001.0100.0100.1003.00
   !
   address-family ipv6 unicast
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
   isis enable UNDERLAY
   no isis bfd
   isis ipv6 bfd
   isis circuit-type level-1
   isis network point-to-point
!
interface Ethernet2
   mtu 9000
   no switchport
   ipv6 enable
   isis enable UNDERLAY
   no isis bfd
   isis ipv6 bfd
   isis circuit-type level-1
   isis network point-to-point
!
interface Ethernet3
!
interface Ethernet4
!
interface Ethernet5
!
interface Loopback0
   ipv6 address fd12:dc1:1::4/128
   isis enable UNDERLAY
!
interface Management1
!
no ip routing
!
ipv6 unicast-routing
!
router isis UNDERLAY
   net 49.0001.0100.0100.1004.00
   !
   address-family ipv6 unicast
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
   isis enable UNDERLAY
   no isis bfd
   isis ipv6 bfd
   isis circuit-type level-1
   isis network point-to-point
!
interface Ethernet2
   mtu 9000
   no switchport
   ipv6 enable
   isis enable UNDERLAY
   no isis bfd
   isis ipv6 bfd
   isis circuit-type level-1
   isis network point-to-point
!
interface Ethernet3
!
interface Ethernet4
!
interface Ethernet5
!
interface Loopback0
   ipv6 address fd12:dc1:1::5/128
   isis enable UNDERLAY
!
interface Management1
!
no ip routing
!
ipv6 unicast-routing
!
router isis UNDERLAY
   net 49.0001.0100.0100.1005.00
   !
   address-family ipv6 unicast
!
end
```
