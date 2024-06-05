## Домашнее задание №4

### Схема сети и план нумерации для Underlay c eBGP.

![](layout4-ebgp.png)

### План адресации.

#### IP Loopback адреса и номера AS. 

| Hostname |   Loopback0  |   Loopback1   | ASN   |
| :------: | :-----------:|:-------------:|:------:
|  Spine1  | 10.0.0.1/32  | 10.0.0.101/32 | 65000 |
|  Spine2  | 10.0.0.2/32  | 10.0.0.102/32 | 65000 |
|  Leaf1   | 10.0.0.11/32 | 10.0.0.111/32 | 65001 |
|  Leaf2   | 10.0.0.22/32 | 10.0.0.122/32 | 65002 |
|  Leaf3   | 10.0.0.33/32 | 10.0.0.133/32 | 65003 |

#### P2P подсети.

| Hostname |    Leaf1    |     Leaf2   |     Leaf3   |
| :------: | :----------:|:-----------:|:-----------:|
|  Spine1  | 10.1.1.0/31 | 10.1.1.2/31 | 10.1.1.4/31 |
|  Spine2  | 10.1.2.0/31 | 10.1.2.2/31 | 10.1.2.4/31 |

  
### План развёртывания протокола eBGP для Underlay в Pod на всех коммутаторах.
 
#### Меняем режим работы коммутатора с ribd на multi-agent для полного функционала протокола BGP.
 
    service routing protocols model multi-agent 


#### Запускаем процесс eBGP и назначаем Id коммутатору.
 
    router bgp 65000 
        router-id 10.0.0.1 

 
#### Ставим таймеры Keepalive и Hold на минимальные значения 1 и 3 сек. 
 
    router bgp 65000
        timers bgp 1 3 
 
#### Устанавливаем административную дистанцию для маршрутов полученных по eBGP 20, а для iBGP маршрутов и локальных подсетей 200. 

    router bgp 65000 
        distance bgp 20 200 200 

    
#### Определяем возможность ECMP для IPv4 префиксов на аплинках у Leaf и на даунлинках у Spine коммутаторов.

    router bgp 65000 
        maximum-paths 8 ecmp 64
 
#### Подключаем соседей.

    router bgp 65000
        neighbor 10.1.1.1 remote-as 65001
        neighbor 10.1.1.3 remote-as 65002
        neighbor 10.1.1.5 remote-as 65003 

#### Таймер MRAI протокола eBGP для немедленного анонса изменений в сторону соседей устанавливаем на 0.  

    router bgp 65000 
        neighbor 10.1.1.1 out-delay 0
        neighbor 10.1.1.3 out-delay 0
        neighbor 10.1.1.5 out-delay 0 

#### Активируем протокол BFD для соседей и на Core интерфейсах. 

    router bgp 65000 
        neighbor 10.1.1.1 bfd 
        neighbor 10.1.1.3 bfd
        neighbor 10.1.1.5 bfd

    interface Ethernet1-3
        bfd interval 100 min-rx 100 multiplier 3 

#### Определяем аутентификацию для соседей. 
 
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

Или через route-map 

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
               NA       No Diagnostic       Up
               NA       No Diagnostic       Up
 
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
