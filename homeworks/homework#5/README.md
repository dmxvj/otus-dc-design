## Домашнее задание №5

### 1. Схема сети и план нумерации для Underlay eBGP, MP-BGP L2VPN EVPN, L2 VXLAN.

![](layout5-l2-vxlan.png)

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

  
### 3. План развёртывания протокола MP-eBGP на Spine на коммутаторах.
 
#### 3.1 Создаём отдельную bgp peer группу для Leaf коммутаторов с целью распространения в Pod AF L2VPN AFI 25 EVPN SAFI 70.
 
    router bgp 65000
        neighbor evpn-leaves peer group


#### 3.2 MP-eBGP сессии будут строятся на Loopback интерфейсах. Для Exterior MP-BPG указываем дополнительно параметр multihop, так как TTL в eBGP по умолчанию 1. 
 
        neighbor evpn-leaves update-source Loopback0
        neighbor evpn-leaves ebgp-multihop 3


#### 3.3 По умолчанию eBGP меняет параметр и адрес next-hop при получении анонсов, в нашем случае источники и конечные получатели апдейтов это только Leaf коммутаторы, поэтому оставляем адреса источников bgp апдейтов неизменными. 
 
        neighbor evpn-leaves next-hop-unchanged 
 
#### 3.4  Включаем возможность анонсов в MP-BGP расширенных bgp комьюнити с распространением route-distinguisher и route-target атрибутов. 

        neighbor evpn-leaves send-community extended

    
#### 3.5 Указываем какие соседи-лифы у нас будут входить в bgp peer группу evpn-leaves.

        neighbor 10.0.0.11 peer group evpn-leaves
        neighbor 10.0.0.11 remote-as 65001
        neighbor 10.0.0.22 peer group evpn-leaves
        neighbor 10.0.0.22 remote-as 65002
        neighbor 10.0.0.33 peer group evpn-leaves
        neighbor 10.0.0.33 remote-as 65003
 
#### 3.6 Активируем нашу peer группу и входящих в неё соседей в address-family evpn протокола MP-BGP. 

    router bgp 65000
        address-family evpn
            neighbor evpn-leaves activate


### 4. План развёртывания протокола MP-eBGP на Leaf на коммутаторах

#### 4.1 Создаём отдельную bgp peer группу для Spine коммутаторов с целью распространения в Pod MP-BGP AF L2VPN AFI 25 EVPN SAFI 70, оставляя по умолчанию поведение атрибута next-hop в eBGP.

    router bgp 65001 
        neighbor evpn-spines peer group
        neighbor evpn-spines remote-as 65000
        neighbor evpn-spines update-source Loopback0
        neighbor evpn-spines ebgp-multihop 3
        neighbor evpn-spines send-community extended

        neighbor 10.0.0.1 peer group evpn-spines
        neighbor 10.0.0.2 peer group evpn-spines

        address-family evpn
            neighbor evpn-spines activate


#### 4.2  
 
    router bgp 65000 
        neighbor 10.1.1.1 password 7 B7rhB/vPbn0K7ECNtz1K5w==
        neighbor 10.1.1.3 password 7 6ZlbNVefGOoRTw2KYF4N2A==
        neighbor 10.1.1.5 password 7 zWKcHc58qGjgbjmUvjsL3A==

#### Активируем соседей для начала маршрутизации IPv4 префиксов.

    router bgp 65000 
        address-family ipv4
            neighbor 10.1.1.1 activate
            neighbor 10.1.1.3 activate
            neighbor 10.1.1.5 activate

#### Делаем редистрибуцию локальных подсетей Loopback интерфейсов.

    router bgp 65000
        address-family ipv4
            network 10.0.0.1/32

Или через route-map на Leaf коммутаторах.

    ip prefix-list connected-to-bgp
    seq 10 permit 10.0.0.0/24 ge 32
    !
    route-map REDIS_CONN permit 10
    match ip address prefix-list connected-to-bgp
    set origin igp
    !

    router bgp 65000
        address-family ipv4
            redistribute connected route-map REDIS_CONN

### Итоговая конфигурация. 

    Spine1#show run | s bgp

    service routing protocols model multi-agent

    router bgp 65000
        router-id 10.0.0.1
        no bgp default ipv4-unicast
        timers bgp 1 3
        distance bgp 20 200 200
        maximum-paths 8 ecmp 64
        neighbor 10.1.1.1 remote-as 65001
        neighbor 10.1.1.1 out-delay 0
        neighbor 10.1.1.1 bfd
        neighbor 10.1.1.1 password 7 B7rhB/vPbn0K7ECNtz1K5w==
        neighbor 10.1.1.3 remote-as 65002
        neighbor 10.1.1.3 out-delay 0
        neighbor 10.1.1.3 bfd
        neighbor 10.1.1.3 password 7 6ZlbNVefGOoRTw2KYF4N2A==
        neighbor 10.1.1.5 remote-as 65003
        neighbor 10.1.1.5 out-delay 0
        neighbor 10.1.1.5 bfd
        neighbor 10.1.1.5 password 7 zWKcHc58qGjgbjmUvjsL3A==
        !
        address-family ipv4
            neighbor 10.1.1.1 activate
            neighbor 10.1.1.3 activate
            neighbor 10.1.1.5 activate
            network 10.0.0.1/32

+++++++++++++++++++++++++++++++++++++++++  

    Leaf1#show run | s bgp

    service routing protocols model multi-agent

    ip prefix-list connected-to-bgp
        seq 10 permit 10.0.0.0/24 ge 32

    route-map REDIS_CONN permit 10
        match ip address prefix-list connected-to-bgp

    router bgp 65001
        router-id 10.0.0.11
        no bgp default ipv4-unicast
        timers bgp 1 3
        distance bgp 20 200 200
        maximum-paths 4 ecmp 64
        neighbor spines peer group
        neighbor spines remote-as 65000
        neighbor spines out-delay 0
        neighbor spines bfd
        neighbor spines maximum-routes 10000 warning-only
        neighbor 10.1.1.0 peer group spines
        neighbor 10.1.1.0 password 7 B7rhB/vPbn0K7ECNtz1K5w==
        neighbor 10.1.2.0 peer group spines
        neighbor 10.1.2.0 password 7 qJkVzQI8BJZIFaQJU7/LYQ==
        !
        address-family ipv4
            neighbor spines activate
            redistribute connected route-map REDIS_CONN

###  Проверочная часть. 

#### Проверка работы протокола BFD. 

На примере показан пример проверки на одном коммутаторе. 

    Spine2#show bfd peers
    VRF name: default
    -----------------
    DstAddr       MyDisc    YourDisc  Interface/Transport    Type           LastUp 
    --------- ----------- ----------- -------------------- ------- ----------------
    10.1.2.1  1478589633  2354517210        Ethernet1(18)  normal   05/31/24 20:52 
    10.1.2.3  1154948017  2355561545        Ethernet2(19)  normal   05/31/24 20:54 
    10.1.2.5   320984180  2824604348        Ethernet3(20)  normal   05/31/24 20:56 

         LastDown            LastDiag    State
    -------------------- ------------------- -----
        05/31/24 20:50       No Diagnostic       Up
               NA            No Diagnostic       Up
               NA            No Diagnostic       Up
 
#### Проверка установления соседства BGP пиров. 

    Spine2#show ip bgp summary 
    BGP summary information for VRF default
    Router identifier 10.0.0.2, local AS number 65000
    Neighbor Status Codes: m - Under maintenance
        Neighbor V AS           MsgRcvd   MsgSent  InQ OutQ  Up/Down State   PfxRcd PfxAcc
        10.1.2.1 4 65001         449319    449209    0    0    4d02h Estab   2      2
        10.1.2.3 4 65002         420143    420041    0    0    4d03h Estab   2      2
        10.1.2.5 4 65003         419768    419772    0    0    4d03h Estab   2      2

#### Проверка таблицы маршрутизации, ECMP и IP связности на примере 3-его Leaf коммутатора. 

    Leaf3#show ip route 

    VRF: default
    Codes: C - connected, S - static, K - kernel, 
           B I - iBGP, B E - eBGP, R - RIP, I L1 - IS-IS level 1,
       
    Gateway of last resort is not set

    B E      10.0.0.1/32 [20/0] via 10.1.1.4, Ethernet1
    B E      10.0.0.2/32 [20/0] via 10.1.2.4, Ethernet2
    B E      10.0.0.11/32 [20/0] via 10.1.1.4, Ethernet1
                                 via 10.1.2.4, Ethernet2
    B E      10.0.0.22/32 [20/0] via 10.1.1.4, Ethernet1
                                 via 10.1.2.4, Ethernet2
    C        10.0.0.33/32 is directly connected, Loopback0
    B E      10.0.0.111/32 [20/0] via 10.1.1.4, Ethernet1
                                  via 10.1.2.4, Ethernet2
    B E      10.0.0.122/32 [20/0] via 10.1.1.4, Ethernet1
                                  via 10.1.2.4, Ethernet2
    C        10.0.0.133/32 is directly connected, Loopback1
    C        10.1.1.4/31 is directly connected, Ethernet1
    C        10.1.2.4/31 is directly connected, Ethernet2


    Leaf3#ping 10.0.0.11 source 10.0.0.33 size 9000 df-bit repeat 4
    PING 10.0.0.11 (10.0.0.11) from 10.0.0.33 : 8972(9000) bytes of data.
    8980 bytes from 10.0.0.11: icmp_seq=1 ttl=63 time=8.46 ms
    8980 bytes from 10.0.0.11: icmp_seq=2 ttl=63 time=9.29 ms
    8980 bytes from 10.0.0.11: icmp_seq=3 ttl=63 time=9.81 ms
    8980 bytes from 10.0.0.11: icmp_seq=4 ttl=63 time=9.16 ms

    --- 10.0.0.11 ping statistics ---
    4 packets transmitted, 4 received, 0% packet loss, time 30ms
    rtt min/avg/max/mdev = 8.466/9.185/9.817/0.486 ms, ipg/ewma 10.150/8.780 ms
