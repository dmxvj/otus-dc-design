## Домашнее задание №7

### 1. Схема сети и план нумерации для Underlay eBGP, MP-BGP L2VPN EVPN, L2/L3-GW Overlay VXLAN, ESI-LAG.

![](layout7-esi-lag.png)

### 2. План адресации.

#### 2.1 IP Loopback адреса и номера AS. 

| Hostname |   Loopback0  |   Loopback1   | ASN   |
| :------: | :-----------:|:-------------:|:------:
|  Spine1  | 10.0.0.1/32  | 10.0.0.101/32 | 65000 |
|  Spine2  | 10.0.0.2/32  | 10.0.0.102/32 | 65000 |
|  Leaf1   | 10.0.0.11/32 | 10.0.0.111/32 | 65001 |
|  Leaf2   | 10.0.0.22/32 | 10.0.0.122/32 | 65002 |
|  Leaf3   | 10.0.0.33/32 | 10.0.0.133/32 | 65003 |

#### 2.2 P2P подсети.

| Hostname |    Leaf1    |     Leaf2   |     Leaf3   |
| :------: | :----------:|:-----------:|:-----------:|
|  Spine1  | 10.1.1.0/31 | 10.1.1.2/31 | 10.1.1.4/31 |
|  Spine2  | 10.1.2.0/31 | 10.1.2.2/31 | 10.1.2.4/31 |

#### 2.3 Адреса хостов.

|  Hostname |        PC1        |        PC2        |        PC3        |        PC4        |
| :--------:| :----------------:|:-----------------:|:-----------------:|:-----------------:|
|  IP       |  192.168.1.1/24   |  192.168.2.2/24   |  192.168.1.3/24   |  192.168.2.4/24   |
|  Gateway  |  192.168.1.254/24 |  192.168.2.254/24 | 192.168.1.254/24  |  192.168.2.254/24 |
|   VLAN    |        101        |         102       |         101       |        102        |
|   MAC     | 00:50:79:66:68:06 | 00:50:79:66:68:07 | 00:50:79:66:68:08 | 00:50:79:66:68:09 |

### 3. План развёртывания ESI-LAG на Leaf на коммутаторах.

### Примечание: Underlay eBGP был развёрнут в ДЗ №4; L2 Overlay VXLAN в ДЗ №5, L3 VTEP в ДЗ №6. 
 
#### 3.1 Создаём агрегированный интерфейс Port-channel на двух коммутаторах, работающего под контролем протокола LACP.
 
    Leaf1#
        interface Ethernet5
            channel-group 1 mode active

    Leaf2#
        interface Ethernet5
            channel-group 1 mode active

#### 3.2 Переводим его в режим trunk и разрешаем прохождение виланов 101 и 102 на двух коммутаторах.

    Leaf1#
        interface Port-Channel1
            switchport trunk allowed vlan 101-102
            switchport mode trunk
 
    Leaf2#
        interface Port-Channel1
            switchport trunk allowed vlan 101-102
            switchport mode trunk

#### 3.3 Назначаем агрегированному интерфейсу уникальный системный идентификатор Id. 
 
    interface Port-Channel1
        lacp system-id 0000.0000.0001
 
#### 3.4 Создаём на агрегированном интерфейсе ESI - EVPN сегмент 10-байтный идентификатор необходимого для multihoming. 

    interface Port-Channel1
        evpn ethernet-segment
            identifier 0000:0000:0000:0000:0001
    
#### 3.5 Назначаем 6-байтный Route-Target на ESI для принятия EVPN апдейтов Type-4. 

    interface Port-Channel1
        evpn ethernet-segment
            route-target import 00:00:00:00:00:01

### 4. На стороне коммутатора CE создаём агрегированный Port-channel интерфейс.  

#### 4.1 С активацией протокола LACP.
 
    Switch1#
        interface GigabitEthernet0/0
        channel-protocol lacp
        channel-group 1 mode active

        interface GigabitEthernet1/0
        channel-protocol lacp
        channel-group 1 mode active

#### 4.2 Переводим порт в режим транка.

    Switch1#
        interface Port-channel1
        switchport trunk allowed vlan 101,102
        switchport trunk encapsulation dot1q
        switchport mode trunk

### 5. Итоговые конфигурации Leaf коммутаторов. 

    Leaf1#shщц run 
    !
    service routing protocols model multi-agent
    !
    hostname Leaf1
    !
    spanning-tree mode mstp
    !
    vlan 101-102
    !
    interface Port-Channel1
        load-interval 30
        switchport trunk allowed vlan 101-102
        switchport mode trunk
    !
        evpn ethernet-segment
            identifier 0000:0000:0000:0000:0001
            route-target import 00:00:00:00:00:01
        lacp system-id 0000.0000.0001
    !
    interface Ethernet1
        description to-Spine1
        mtu 9000
        no switchport
        ip address 10.1.1.1/31
        bfd interval 100 min-rx 100 multiplier 3
    !
    interface Ethernet2
        description to-Spine2
        mtu 9000
        no switchport
        ip address 10.1.2.1/31
        bfd interval 100 min-rx 100 multiplier 3
    !
    interface Ethernet5
        channel-group 1 mode active
    !
    interface Loopback0
        ip address 10.0.0.11/32
    !
    interface Loopback1
        ip address 10.0.0.111/32
    !
    interface Vlan101
        description IRB VLAN101
        ip address virtual 192.168.1.254/24
    !
    interface Vlan102
        description IRB VLAN102
        ip address virtual 192.168.2.254/24
    !
    interface Vxlan1
        vxlan source-interface Loopback1
        vxlan udp-port 4789
        vxlan vlan 101 vni 10101
        vxlan vlan 102 vni 10102
    !
    ip virtual-router mac-address 00:1c:73:00:00:aa
    !
    ip routing
    !
    ip prefix-list connected-to-bgp
        seq 10 permit 10.0.0.0/24 ge 32
    !
    route-map REDIS_CONN permit 10
        match ip address prefix-list connected-to-bgp
        set origin igp
    !
    router bgp 65001
        router-id 10.0.0.11
        no bgp default ipv4-unicast
        timers bgp 1 3
        distance bgp 20 200 200
        maximum-paths 4 ecmp 64
        neighbor evpn-spines peer group
        neighbor evpn-spines remote-as 65000
        neighbor evpn-spines update-source Loopback0
        neighbor evpn-spines ebgp-multihop 3
        neighbor evpn-spines send-community extended
        neighbor spines peer group
        neighbor spines remote-as 65000
        neighbor spines out-delay 0
        neighbor spines bfd
        neighbor spines maximum-routes 10000 warning-only
        neighbor 10.0.0.1 peer group evpn-spines
        neighbor 10.0.0.2 peer group evpn-spines
        neighbor 10.1.1.0 peer group spines
        neighbor 10.1.1.0 password 7 B7rhB/vPbn0K7ECNtz1K5w==
        neighbor 10.1.2.0 peer group spines
        neighbor 10.1.2.0 password 7 qJkVzQI8BJZIFaQJU7/LYQ==
        !
        vlan 101
            rd 65001:10101
            route-target both 101:10101
            redistribute learned
        !
        vlan 102
            rd 65001:10102
            route-target both 102:10102
            redistribute learned
        !
        address-family evpn
            neighbor evpn-spines activate
        !
        address-family ipv4
            no neighbor evpn-spines activate
            neighbor spines activate
            redistribute connected route-map REDIS_CONN
    !
    end

+++++++++++++++++++++++++++++++++++++++++  

    Leaf2#show run 
    !
    service routing protocols model multi-agent
    !
    hostname Leaf2
    !
    spanning-tree mode mstp
    !
    vlan 101-102
    !
    interface Port-Channel1
        switchport trunk allowed vlan 101-102
        switchport mode trunk
    !
        evpn ethernet-segment
            identifier 0000:0000:0000:0000:0001
            route-target import 00:00:00:00:00:01
        lacp system-id 0000.0000.0001
    !
    interface Port-Channel2
        switchport trunk allowed vlan 101-102
        switchport mode trunk
    !
        evpn ethernet-segment
            identifier 0000:0000:0000:0000:0002
            route-target import 00:00:00:00:00:02
        lacp system-id 0000.0000.0002
    !
    interface Ethernet1
        description to-Spine1
        mtu 9000
        no switchport
        ip address 10.1.1.3/31
        bfd interval 100 min-rx 100 multiplier 3
    !
    interface Ethernet2
        description to-Spine2
        mtu 9000
        no switchport
        ip address 10.1.2.3/31
            bfd interval 100 min-rx 100 multiplier 3
    !
    interface Ethernet5
        channel-group 1 mode active
    !
    interface Ethernet7
        channel-group 2 mode active
    !
    interface Loopback0
        ip address 10.0.0.22/32
    !
    interface Loopback1
        ip address 10.0.0.122/32
    !
    interface Vlan101
        description IRB VLAN102
        ip address virtual 192.168.1.254/24
    !
    interface Vlan102
        description IRB VLAN102
        ip address virtual 192.168.2.254/24
    !
    interface Vxlan1
        vxlan source-interface Loopback1
        vxlan udp-port 4789
        vxlan vlan 101 vni 10101
        vxlan vlan 102 vni 10102
    !
    ip virtual-router mac-address 00:1c:73:00:00:aa
    !
    ip routing
    !
    ip prefix-list connected-to-bgp
        seq 10 permit 10.0.0.0/24 ge 32
    !
    route-map REDIS_CONN permit 10
        match ip address prefix-list connected-to-bgp
        set origin igp
    !
    router bgp 65002
        router-id 10.0.0.22
        no bgp default ipv4-unicast
        timers bgp 1 3
        distance bgp 20 200 200
        maximum-paths 4 ecmp 64
        neighbor evpn peer group
        neighbor evpn remote-as 65000
        neighbor evpn update-source Loopback0
        neighbor evpn ebgp-multihop 3
        neighbor evpn send-community extended
        neighbor evpn-spines peer group
        neighbor evpn-spines remote-as 65000
        neighbor evpn-spines update-source Loopback0
        neighbor evpn-spines ebgp-multihop 3
        neighbor evpn-spines send-community extended
        neighbor spines peer group
        neighbor spines remote-as 65000
        neighbor spines out-delay 0
        neighbor spines bfd
        neighbor spines maximum-routes 10000 warning-only
        neighbor 10.0.0.1 peer group evpn-spines
        neighbor 10.0.0.2 peer group evpn-spines
        neighbor 10.1.1.2 peer group spines
        neighbor 10.1.1.2 password 7 6ZlbNVefGOoRTw2KYF4N2A==
        neighbor 10.1.2.2 peer group spines
        neighbor 10.1.2.2 password 7 k/sLtX4he3Tjv/dsbbHquA==
        !
        vlan 101
            rd 65002:10101
            route-target both 101:10101
            redistribute learned
        !
        vlan 102
            rd 65002:10102
            route-target both 102:10102
            redistribute learned
        !
        address-family evpn
            neighbor evpn-spines activate
        !
        address-family ipv4
            no neighbor evpn-spines activate
            neighbor spines activate
            redistribute connected route-map REDIS_CONN
    !
    end

++++++++++++++++++++++++++++++++++++++++++++++++++++++++++

    Leaf3#show run 
    !
    service routing protocols model multi-agent
    !
    hostname Leaf3
    !
    spanning-tree mode mstp
    !
    vlan 101-102
    !
    interface Port-Channel2
        switchport trunk allowed vlan 101-102
        switchport mode trunk
    !
        evpn ethernet-segment
            identifier 0000:0000:0000:0000:0002
            route-target import 00:00:00:00:00:02
        lacp system-id 0000.0000.0002
    !
    interface Ethernet1
        description to-Spine1
        mtu 9000
        no switchport
        ip address 10.1.1.5/31
        bfd interval 100 min-rx 100 multiplier 3
    !
    interface Ethernet2
        description to-Spine2
        mtu 9000
        no switchport
        ip address 10.1.2.5/31
        bfd interval 100 min-rx 100 multiplier 3
    !
    interface Ethernet7
        channel-group 2 mode active
    !
    interface Loopback0
        ip address 10.0.0.33/32
    !
    interface Loopback1
        ip address 10.0.0.133/32
    !
    interface Vlan101
        description IRB VLAN101
        ip address virtual 192.168.1.254/24
    !
    interface Vlan102
        description IRB VLAN102
        ip address virtual 192.168.2.254/24
    !
    interface Vxlan1
        vxlan source-interface Loopback1
        vxlan udp-port 4789
        vxlan vlan 101 vni 10101
        vxlan vlan 102 vni 10102
    !
    ip virtual-router mac-address 00:1c:73:00:00:aa
    !
    ip routing
    !
    ip prefix-list connected-to-bgp
        seq 10 permit 10.0.0.0/24 ge 32
    !
    route-map REDIS_CONN permit 10
        match ip address prefix-list connected-to-bgp
        set origin igp
    !
    router bgp 65003
        router-id 10.0.0.33
        no bgp default ipv4-unicast
        timers bgp 1 3
        distance bgp 20 200 200
        maximum-paths 4 ecmp 64
        neighbor evpn-spines peer group
        neighbor evpn-spines remote-as 65000
        neighbor evpn-spines update-source Loopback0
        neighbor evpn-spines ebgp-multihop 3
        neighbor evpn-spines send-community extended
        neighbor spines peer group
        neighbor spines remote-as 65000
        neighbor spines out-delay 0
        neighbor spines bfd
        neighbor spines maximum-routes 10000 warning-only
        neighbor 10.0.0.1 peer group evpn-spines
        neighbor 10.0.0.2 peer group evpn-spines
        neighbor 10.1.1.4 peer group spines
        neighbor 10.1.1.4 password 7 zWKcHc58qGjgbjmUvjsL3A==
        neighbor 10.1.2.4 peer group spines
        neighbor 10.1.2.4 password 7 qEWAlLTC4nfcCtaj0TBNoQ==
        !
        vlan 101
            rd 65003:10101
            route-target both 101:10101
            redistribute learned
        !
        vlan 102
            rd 65003:10102
            route-target both 102:10102
            redistribute learned
        !
        address-family evpn
            neighbor evpn-spines activate
        !
        address-family ipv4
            no neighbor evpn-spines activate
            neighbor spines activate
            redistribute connected route-map REDIS_CONN
    !
    end

###  6. Проверочная часть. 

#### 6.1 Сразу проверяем доступность хостов пингом. 

На примере показан пример проверки с хоста PC1. 

    PC1> ping 192.168.1.3 -c 2
    84 bytes from 192.168.1.3 icmp_seq=1 ttl=64 time=51.896 ms
    84 bytes from 192.168.1.3 icmp_seq=2 ttl=64 time=19.726 ms

    PC1> ping 192.168.2.2 -c 2
    84 bytes from 192.168.2.2 icmp_seq=1 ttl=63 time=141.124 ms
    84 bytes from 192.168.2.2 icmp_seq=2 ttl=63 time=22.423 ms

    PC1> ping 192.168.2.4 -c 2
    84 bytes from 192.168.2.4 icmp_seq=1 ttl=63 time=141.139 ms
    84 bytes from 192.168.2.4 icmp_seq=2 ttl=63 time=26.680 ms
 
#### 6.2 Проверяем наличие BGP апдейтов EVPN AD Type-1.

    Leaf2#show bgp evpn route-type ethernet-segment esi 0000:0000:0000:0000:0001 
    BGP routing table information for VRF default
    Router identifier 10.0.0.22, local AS number 65002
    Route status codes: * - valid, > - active, S - Stale, E - ECMP head, e - ECMP
                    c - Contributing to ECMP, % - Pending BGP convergence
    Origin codes: i - IGP, e - EGP, ? - incomplete
    AS Path Attributes: Or-ID - Originator ID, C-LST - Cluster List, LL Nexthop - Link Local Nexthop

          Network                Next Hop              Metric  LocPref Weight  Path
    * >Ec    RD: 10.0.0.111:1 ethernet-segment 0000:0000:0000:0000:0001 10.0.0.111
                                 10.0.0.111            -       100     0       65000 65001 i
    *  ec    RD: 10.0.0.111:1 ethernet-segment 0000:0000:0000:0000:0001 10.0.0.111
                                 10.0.0.111            -       100     0       65000 65001 i
    * >      RD: 10.0.0.122:1 ethernet-segment 0000:0000:0000:0000:0001 10.0.0.122
                                 -                     -       -       0       i

    Leaf2#show bgp evpn route-type ethernet-segment esi 0000:0000:0000:0000:0002
    BGP routing table information for VRF default
    Router identifier 10.0.0.22, local AS number 65002
    Route status codes: * - valid, > - active, S - Stale, E - ECMP head, e - ECMP
                    c - Contributing to ECMP, % - Pending BGP convergence
    Origin codes: i - IGP, e - EGP, ? - incomplete
    AS Path Attributes: Or-ID - Originator ID, C-LST - Cluster List, LL Nexthop - Link Local Nexthop

          Network                Next Hop              Metric  LocPref Weight  Path
    * >      RD: 10.0.0.122:1 ethernet-segment 0000:0000:0000:0000:0002 10.0.0.122
                                 -                     -       -       0       i
    * >Ec    RD: 10.0.0.133:1 ethernet-segment 0000:0000:0000:0000:0002 10.0.0.133
                                 10.0.0.133            -       100     0       65000 65003 i
    *  ec    RD: 10.0.0.133:1 ethernet-segment 0000:0000:0000:0000:0002 10.0.0.133
                                 10.0.0.133            -       100     0       65000 65003 i


#### 6.3 Проверяем наличие BGP апдейтов EVPN Type-4.

    Leaf2#show bgp evpn route-type auto-discovery esi 0000:0000:0000:0000:0001 
    BGP routing table information for VRF default
    Router identifier 10.0.0.22, local AS number 65002
    Route status codes: * - valid, > - active, S - Stale, E - ECMP head, e - ECMP
                    c - Contributing to ECMP, % - Pending BGP convergence
    Origin codes: i - IGP, e - EGP, ? - incomplete
    AS Path Attributes: Or-ID - Originator ID, C-LST - Cluster List, LL Nexthop - Link Local Nexthop

          Network                Next Hop              Metric  LocPref Weight  Path
    * >Ec    RD: 65001:10101 auto-discovery 0 0000:0000:0000:0000:0001
                                 10.0.0.111            -       100     0       65000 65001 i
    *  ec    RD: 65001:10101 auto-discovery 0 0000:0000:0000:0000:0001
                                 10.0.0.111            -       100     0       65000 65001 i
    * >Ec    RD: 65001:10102 auto-discovery 0 0000:0000:0000:0000:0001
                                 10.0.0.111            -       100     0       65000 65001 i
    *  ec    RD: 65001:10102 auto-discovery 0 0000:0000:0000:0000:0001
                                 10.0.0.111            -       100     0       65000 65001 i
    * >      RD: 65002:10101 auto-discovery 0 0000:0000:0000:0000:0001
                                 -                     -       -       0       i
    * >      RD: 65002:10102 auto-discovery 0 0000:0000:0000:0000:0001
                                 -                     -       -       0       i
    * >Ec    RD: 10.0.0.111:1 auto-discovery 0000:0000:0000:0000:0001
                                 10.0.0.111            -       100     0       65000 65001 i
    *  ec    RD: 10.0.0.111:1 auto-discovery 0000:0000:0000:0000:0001
                                 10.0.0.111            -       100     0       65000 65001 i
    * >      RD: 10.0.0.122:1 auto-discovery 0000:0000:0000:0000:0001
                                 -                     -       -       0       i

    Leaf2#show bgp evpn route-type auto-discovery esi 0000:0000:0000:0000:0002
    BGP routing table information for VRF default
    Router identifier 10.0.0.22, local AS number 65002
    Route status codes: * - valid, > - active, S - Stale, E - ECMP head, e - ECMP
                    c - Contributing to ECMP, % - Pending BGP convergence
    Origin codes: i - IGP, e - EGP, ? - incomplete
    AS Path Attributes: Or-ID - Originator ID, C-LST - Cluster List, LL Nexthop - Link Local Nexthop

          Network                Next Hop              Metric  LocPref Weight  Path
    * >      RD: 65002:10101 auto-discovery 0 0000:0000:0000:0000:0002
                                 -                     -       -       0       i
    * >      RD: 65002:10102 auto-discovery 0 0000:0000:0000:0000:0002
                                 -                     -       -       0       i
    * >Ec    RD: 65003:10101 auto-discovery 0 0000:0000:0000:0000:0002
                                 10.0.0.133            -       100     0       65000 65003 i
    *  ec    RD: 65003:10101 auto-discovery 0 0000:0000:0000:0000:0002
                                 10.0.0.133            -       100     0       65000 65003 i
    * >Ec    RD: 65003:10102 auto-discovery 0 0000:0000:0000:0000:0002
                                 10.0.0.133            -       100     0       65000 65003 i
    *  ec    RD: 65003:10102 auto-discovery 0 0000:0000:0000:0000:0002
                                 10.0.0.133            -       100     0       65000 65003 i
    * >      RD: 10.0.0.122:1 auto-discovery 0000:0000:0000:0000:0002
                                 -                     -       -       0       i
    * >Ec    RD: 10.0.0.133:1 auto-discovery 0000:0000:0000:0000:0002
                                 10.0.0.133            -       100     0       65000 65003 i
    *  ec    RD: 10.0.0.133:1 auto-discovery 0000:0000:0000:0000:0002
                                 10.0.0.133            -       100     0       65000 65003 i
    Leaf2#