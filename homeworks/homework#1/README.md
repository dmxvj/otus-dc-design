## Домашнее задание №1

### Схема сети.

![](layout1.png)

### План адресации.

#### Loopbacks

| Hostname | Loopback0    | Loopback1     |
| :------: | :-----------:|:-------------:|
|  Spine1  | 10.0.0.1/32  | 10.0.0.101/32 |
|  Spine2  | 10.0.0.2/32  | 10.0.0.102/32 |
|  Leaf1   | 10.0.0.11/32 | 10.0.0.111/32 |
|  Leaf2   | 10.0.0.22/32 | 10.0.0.122/32 |
|  Leaf3   | 10.0.0.33/32 | 10.0.0.133/32 |

#### P2P

| Hostname |    Leaf1    |     Leaf2   |     Leaf3   |
| :------: | :----------:|:-----------:|:-----------:|
|  Spine1  | 10.1.1.0/31 | 10.1.1.2/31 | 10.1.1.4/31 |
|  Spine2  | 10.1.2.0/31 | 10.1.2.2/31 | 10.1.2.4/31 |

### Настройки нодов

Spine1#show run  
!  
hostname Spine1  
!  
interface Ethernet1  
   description to-Leaf1  
   mtu 9000   
   no switchport  
   ip address 10.1.1.0/31  
!  
interface Ethernet2  
   description to-Leaf2  
   mtu 9000  
   no switchport  
   ip address 10.1.1.2/31  
!  
interface Ethernet3  
   description to-Leaf3  
   mtu 9000  
   no switchport  
   ip address 10.1.1.4/31  
!  
interface Loopback0  
   ip address 10.0.0.1/32  
!  
interface Loopback1  
   ip address 10.0.0.101/32  
!  

++++++++++++++++++++++++++++++

Spine2#show run  
!  
hostname Spine2     
!  
interface Ethernet1  
   description to-Leaf1    
   mtu 9000  
   no switchport  
   ip address 10.1.2.0/31  
!  
interface Ethernet2  
   description to-Leaf2  
   mtu 9000  
   no switchport  
   ip address 10.1.2.2/31  
!  
interface Ethernet3  
   description to-Leaf3  
   mtu 9000  
   no switchport  
   ip address 10.1.2.4/31  
!  
interface Loopback0  
   ip address 10.0.0.2/32  
!  
interface Loopback1  
   ip address 10.0.0.102/32  
!  
+++++++++++++++++++++++++++++++

Leaf1#show run  
!  
hostname Leaf1  
!  
interface Ethernet1  
   description to-Spine1  
   mtu 9000  
   no switchport  
   ip address 10.1.1.1/31  
!  
interface Ethernet2  
   description to-Spine2  
   mtu 9000  
   no switchport  
   ip address 10.1.2.1/31  
!  
interface Loopback0  
   ip address 10.0.0.11/32  
!  
interface Loopback1  
   ip address 10.0.0.111/32  
!  
+++++++++++++++++++++++++++++++  

Leaf2#show run  
!  
hostname Leaf2  
!  
interface Ethernet1  
   description to-Spine1  
   mtu 9000  
   no switchport  
   ip address 10.1.1.3/31  
!  
interface Ethernet2  
   description to-Spine2  
   mtu 9000  
   no switchport  
   ip address 10.1.2.3/31  
!  
interface Loopback0  
   ip address 10.0.0.22/32  
!  
interface Loopback1  
   ip address 10.0.0.122/32  
!  
+++++++++++++++++++++++++++++++  

Leaf3#show run  
!  
hostname Leaf3  
!  
interface Ethernet1  
   description to-Spine1  
   mtu 9000  
   no switchport  
   ip address 10.1.1.5/31  
!  
interface Ethernet2  
   description to-Spine2  
   mtu 9000  
   no switchport  
   ip address 10.1.2.5/31  
!  
interface Loopback0  
   ip address 10.0.0.33/32  
!  
interface Loopback1  
   ip address 10.0.0.133/32  
!  

### Ping
  
Spine1#ping 10.1.1.1 size 9000 df rep 2  
PING 10.1.1.1 (10.1.1.1) 8972(9000) bytes of data.  
8980 bytes from 10.1.1.1: icmp_seq=1 ttl=64 time=5.77 ms  
8980 bytes from 10.1.1.1: icmp_seq=2 ttl=64 time=3.51 ms  
--- 10.1.1.1 ping statistics ---  
2 packets transmitted, 2 received, 0% packet loss, time 6ms  
rtt min/avg/max/mdev = 3.517/4.646/5.775/1.129 ms, ipg/ewma 6.652/5.492 ms  
Spine1#  
Spine1#ping 10.1.1.3 size 9000 df rep 2   
PING 10.1.1.3 (10.1.1.3) 8972(9000) bytes of data.  
8980 bytes from 10.1.1.3: icmp_seq=1 ttl=64 time=5.84 ms  
8980 bytes from 10.1.1.3: icmp_seq=2 ttl=64 time=3.44 ms  
--- 10.1.1.3 ping statistics ---  
2 packets transmitted, 2 received, 0% packet loss, time 6ms  
rtt min/avg/max/mdev = 3.448/4.645/5.843/1.199 ms, ipg/ewma 6.642/5.543 ms  
Spine1#  
Spine1#ping 10.1.1.5 size 9000 df rep 2  
PING 10.1.1.5 (10.1.1.5) 8972(9000) bytes of data.  
8980 bytes from 10.1.1.5: icmp_seq=1 ttl=64 time=5.88 ms  
8980 bytes from 10.1.1.5: icmp_seq=2 ttl=64 time=4.30 ms  
--- 10.1.1.5 ping statistics ---  
2 packets transmitted, 2 received, 0% packet loss, time 6ms  
rtt min/avg/max/mdev = 4.305/5.096/5.887/0.791 ms, ipg/ewma 6.779/5.689 ms  
Spine1#  
  
Spine2#ping 10.1.2.1 size 9000 df rep 2  
PING 10.1.2.1 (10.1.2.1) 8972(9000) bytes of data.  
8980 bytes from 10.1.2.1: icmp_seq=1 ttl=64 time=6.50 ms  
8980 bytes from 10.1.2.1: icmp_seq=2 ttl=64 time=3.87 ms  
--- 10.1.2.1 ping statistics ---  
2 packets transmitted, 2 received, 0% packet loss, time 8ms  
rtt min/avg/max/mdev = 3.877/5.190/6.503/1.313 ms, ipg/ewma 8.387/6.174 ms  
Spine2#  
Spine2#ping 10.1.2.3 size 9000 df rep 2  
PING 10.1.2.3 (10.1.2.3) 8972(9000) bytes of data.  
8980 bytes from 10.1.2.3: icmp_seq=1 ttl=64 time=17.6 ms  
8980 bytes from 10.1.2.3: icmp_seq=2 ttl=64 time=8.65 ms  
--- 10.1.2.3 ping statistics ---  
2 packets transmitted, 2 received, 0% packet loss, time 12ms  
rtt min/avg/max/mdev = 8.657/13.131/17.606/4.475 ms, pipe 2, ipg/ewma 12.343/16.487 ms  
Spine2#  
Spine2#ping 10.1.2.5 size 9000 df rep 2  
PING 10.1.2.5 (10.1.2.5) 8972(9000) bytes of data.  
8980 bytes from 10.1.2.5: icmp_seq=1 ttl=64 time=17.1 ms  
8980 bytes from 10.1.2.5: icmp_seq=2 ttl=64 time=7.74 ms  
--- 10.1.2.5 ping statistics ---  
2 packets transmitted, 2 received, 0% packet loss, time 11ms  
rtt min/avg/max/mdev = 7.741/12.454/17.167/4.713 ms, pipe 2, ipg/ewma 11.699/15.988 ms  
Spine2#  