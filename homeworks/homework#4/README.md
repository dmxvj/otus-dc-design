## Домашнее задание №4

### Схема сети и план нумерации для Underlay c eBGP.

![](layout4-ebgp.png)

### План адресации.

#### IP Loopbacks & ISO NET адреса c номером area 65501. 

| Hostname | Loopback0    | Loopback1     |              NET               |
| :------: | :-----------:|:-------------:|:-------------------------------:
|  Spine1  | 10.0.0.1/32  | 10.0.0.101/32 | 49.ffdd.0010.0000.0000.0001.00 |
|  Spine2  | 10.0.0.2/32  | 10.0.0.102/32 | 49.ffdd.0010.0000.0000.0002.00 |
|  Leaf1   | 10.0.0.11/32 | 10.0.0.111/32 | 49.ffdd.0010.0000.0000.0011.00 |
|  Leaf2   | 10.0.0.22/32 | 10.0.0.122/32 | 49.ffdd.0010.0000.0000.0022.00 |
|  Leaf3   | 10.0.0.33/32 | 10.0.0.133/32 | 49.ffdd.0010.0000.0000.0033.00 |

#### P2P подсети.

| Hostname |    Leaf1    |     Leaf2   |     Leaf3   |
| :------: | :----------:|:-----------:|:-----------:|
|  Spine1  | 10.1.1.0/31 | 10.1.1.2/31 | 10.1.1.4/31 |
|  Spine2  | 10.1.2.0/31 | 10.1.2.2/31 | 10.1.2.4/31 |

  
### План развёртывания протокола IS-IS для Underlay в домене на всех коммутаторах.
 
#### Запускаем именованный процесс ISIS с определением ISO и IP идентификатора узла в домене.
 
    router isis netcom 
        net 49.ffdd.0010.0000.0000.0001.00 
        router-id ipv4 10.0.0.1 

#### Разрешаем процесс ISIS на Core и Loopback интерфейсах.
 
    interface Ethernet 1-3 
        isis enable netcom 
 
    interface Loopback 0-1 
        isis enable netcom 
 
#### Назначаем имя коммутатора в процессе протокола IS-IS для возможности обмена именами узлов в домене IS-IS. 
 
    router isis netcom 
        is-hostname Spine1 
 
#### Определяем режим работы узла и его интерфейсов на 2-ом уровене. 

    router isis netcom 
        is-type level-2 

    interface Ethernet 1-3 
        isis circuit-type level-2 
    
#### Определяем возможность ECMP IPv4 на аплинках у Leaf и на даунлинках у Spine коммутаторов.

    router isis netcom 
        address-family ipv4 unicast 
            maximum-paths 8
 
#### Интерфейсы Loopback0 и Loopback1 будут пассивными без попыток определения соседства.

    interface Loopback 0-1
        isis passive 

#### Устанавливаем тип интерфейса p2p на Core интерфейсах для сокращения времени установления соседства между маршрутизаторами без выбора DIS.  

    interface Ethernet 1-3 
        isis network point-to-point 

#### Запускаем между core интерфейсами протокол BFD для ускорения определения разрыва соединений. 

    interface Ethernet 1-3 
        isis bfd 
        bfd interval 100 min-rx 100 multiplier 3 

#### Определяем аутентификацию между Core интерфейсами для усиления безопасности установления соседства. 
 
    interface Ethernet 1-3 
        isis authentication mode sha key-id 1 level-2 
        isis authentication key-id 1 algorithm sha-256 key 7 6iHxbIFmD0V3DZlY2vhNdQ== level-2 

### Итоговая конфигурация. 

    Spine1#show run | s isis 

    interface Ethernet1 
        isis enable netcom 
        isis bfd 
        isis circuit-type level-2 
        isis network point-to-point 
        isis authentication mode sha key-id 1 level-2 
        isis authentication key-id 1 algorithm sha-256 key 7 6iHxbIFmD0V3DZlY2vhNdQ== level-2  

    interface Ethernet2 
        isis enable netcom 
        isis bfd 
        isis circuit-type level-2 
        isis network point-to-point 
        isis authentication mode sha key-id 1 level-2 
        isis authentication key-id 1 algorithm sha-256 key 7 6iHxbIFmD0V3DZlY2vhNdQ== level-2 
 
    interface Ethernet3 
        isis enable netcom 
        isis bfd 
        isis circuit-type level-2 
        isis network point-to-point 
        isis authentication mode sha key-id 1 level-2 
        isis authentication key-id 1 algorithm sha-256 key 7 6iHxbIFmD0V3DZlY2vhNdQ== level-2 
    
    interface Loopback0 
        isis enable netcom 
        isis circuit-type level-2 
        isis passive 
    
    interface Loopback1 
        isis enable netcom 
        isis circuit-type level-2 
        isis passive 
    
    router isis netcom 
        net 49.ffdd.0010.0000.0000.0001.00 
        is-hostname Spine1 
        router-id ipv4 10.0.0.1 
        is-type level-2 
        log-adjacency-changes 
        set-overload-bit on-startup 180 
        ! 
        address-family ipv4 unicast 
            maximum-paths 8 
 
 
+++++++++++++++++++++++++++++++++++++++++  

    Leaf1#show run | s isis 
    interface Ethernet1 
        isis enable netcom  
        isis bfd    
        isis circuit-type level-2   
        isis network point-to-point 
        isis authentication mode sha key-id 1 level-2   
        isis authentication key-id 1 algorithm sha-256 key 7 6iHxbIFmD0V3DZlY2vhNdQ== level-2   

    interface Ethernet2 
        isis enable netcom  
        isis bfd    
        isis circuit-type level-2   
        isis network point-to-point 
        isis authentication mode sha key-id 1 level-2   
        isis authentication key-id 1 algorithm sha-256 key 7 6iHxbIFmD0V3DZlY2vhNdQ== level-2   

    interface Loopback0 
        isis enable netcom  
        isis circuit-type level-2   
        isis passive    

    interface Loopback1 
        isis enable netcom  
        isis circuit-type level-2   
        isis passive    

    router isis netcom  
        net 49.ffdd.0010.0000.0000.0011.00  
        is-hostname Leaf1   
        router-id ipv4 10.0.0.11    
        is-type level-2 
        log-adjacency-changes   
        set-overload-bit on-startup 180 
        !   
        address-family ipv4 unicast 
            maximum-paths 4 

###  Проверочная часть. 

#### Проверка работы протокола BFD. 

На примере показан пример проверки на одном коммутаторе. 

    Spine2#show bfd peers 
    VRF name: default 
    -----------------
    DstAddr       MyDisc    YourDisc  Interface/Transport    Type           LastUp  
    --------- ----------- ----------- -------------------- ------- ----------------
    10.1.2.1  2255156730  3039257150        Ethernet1(18)  normal   05/30/24 20:04 
    10.1.2.3  2537779188  4281995752        Ethernet2(19)  normal   05/30/24 20:04 
    10.1.2.5   161062417   787922893        Ethernet3(20)  normal   05/30/24 20:04 
 
    LastDown            LastDiag    State 
    -------------- ------------------- ----- 
         NA       No Diagnostic       Up 
         NA       No Diagnostic       Up 
         NA       No Diagnostic       Up 
 
#### Проверка соседства IS-IS узлов в домене. 

    Spine2#show isis neighbors  
    
    Instance  VRF      System Id        Type Interface          SNPA              State Hold time   Circuit Id           
    netcom    default  Leaf1            L2   Ethernet1          P2P               UP    28          10                  
    netcom    default  Leaf2            L2   Ethernet2          P2P               UP    29          10                   
    netcom    default  Leaf3            L2   Ethernet3          P2P               UP    30          10                   

#### Проверка таблицы маршрутизации, ECMP и IP связности на примере 3-его Leaf коммутатора. 

    Leaf3#show ip route isis 

    VRF: default
    Codes: C - connected, S - static, K - kernel, 
    ----------------------------------------------------------- 
        I L2 - IS-IS level 2, O3 - OSPFv3, A B - BGP Aggregate, 
    ----------------------------------------------------------- 
    I L2     10.0.0.1/32 [115/20] via 10.1.1.4, Ethernet1 
    I L2     10.0.0.2/32 [115/20] via 10.1.2.4, Ethernet2 
    I L2     10.0.0.11/32 [115/30] via 10.1.1.4, Ethernet1 
                                   via 10.1.2.4, Ethernet2 
    I L2     10.0.0.22/32 [115/30] via 10.1.1.4, Ethernet1 
                                   via 10.1.2.4, Ethernet2 
    I L2     10.0.0.101/32 [115/20] via 10.1.1.4, Ethernet1 
    I L2     10.0.0.102/32 [115/20] via 10.1.2.4, Ethernet2 
    I L2     10.0.0.111/32 [115/30] via 10.1.1.4, Ethernet1 
                                    via 10.1.2.4, Ethernet2 
    I L2     10.0.0.122/32 [115/30] via 10.1.1.4, Ethernet1 
                                    via 10.1.2.4, Ethernet2 
    I L2     10.1.1.0/31 [115/20] via 10.1.1.4, Ethernet1 
    I L2     10.1.1.2/31 [115/20] via 10.1.1.4, Ethernet1 
    I L2     10.1.2.0/31 [115/20] via 10.1.2.4, Ethernet2 
    I L2     10.1.2.2/31 [115/20] via 10.1.2.4, Ethernet2 

    Leaf3#ping 10.0.0.11 source 10.0.0.33
    PING 10.0.0.11 (10.0.0.11) from 10.0.0.33 : 72(100) bytes of data.
    80 bytes from 10.0.0.11: icmp_seq=1 ttl=63 time=9.81 ms
    80 bytes from 10.0.0.11: icmp_seq=2 ttl=63 time=6.23 ms
    80 bytes from 10.0.0.11: icmp_seq=3 ttl=63 time=6.24 ms
    80 bytes from 10.0.0.11: icmp_seq=4 ttl=63 time=6.33 ms
    80 bytes from 10.0.0.11: icmp_seq=5 ttl=63 time=7.70 ms

    --- 10.0.0.11 ping statistics ---
    5 packets transmitted, 5 received, 0% packet loss, time 37ms
    rtt min/avg/max/mdev = 6.231/7.264/9.810/1.390 ms, ipg/ewma 9.489/8.525 ms
    Leaf3# 
 
