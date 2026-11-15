# 🌐 1. 라우팅 필터링(Route Filtering) 정책 설정

<img width="1434" height="803" alt="image" src="https://github.com/user-attachments/assets/00ff561c-f7df-4c02-88f8-6eb6745796da" />

> **목표** <br>
> - B/B_1은 자신의 루프백(10.255.0.1)과 내부 라우터인 20.1.0.0/30 두개를 광고. <br>
> - 1번 경로로는 루프백을 광고하지 않는 정책을 설정 <br>
> - ROUTER_1에서 10.255.0.1와 20.1.0.2로 통신하는 경로가 어떻게 되는지 확인


<br>

---

## ⚙️ 2. 장비별 설정

### B/B_1 Config

```java
feature bgp

interface Ethernet1/1
  no switchport
  ip address 10.0.11.1/30
  no shutdown

interface Ethernet1/2
  no switchport
  ip address 10.0.12.1/30
  no shutdown

interface Ethernet1/3
  no switchport
  ip address 20.1.0.1/30
  no shutdown

interface loopback0
  ip address 10.255.0.1/32
icam monitor scale

...중략
router bgp 65001
  address-family ipv4 unicast
    network 10.255.0.1/32
    network 20.1.0.0/30
    maximum-paths 2
  neighbor 10.0.11.2
    remote-as 65003
    address-family ipv4 unicast
  neighbor 10.0.12.2
    remote-as 65004
    address-family ipv4 unicast
```

---

### B/B_2 config (AS65002)

```java
feature bgp


interface Ethernet1/1
  no switchport
  ip address 10.0.21.2/30
  no shutdown

interface Ethernet1/2
  no switchport
  ip address 10.0.22.2/30
  no shutdown

interface Ethernet1/3
  no switchport
  ip address 20.2.0.1/30
  no shutdown

interface loopback0
  ip address 10.255.0.2/32
icam monitor scale

...중략
router bgp 65002
  router-id 10.255.0.2
  address-family ipv4 unicast
    network 10.255.0.2/32
    network 20.2.0.0/30
  neighbor 10.0.21.1
    remote-as 65003
    address-family ipv4 unicast
  neighbor 10.0.22.1
    remote-as 65004
    address-family ipv4 unicast
  ```

---

### Router_1 config (AS65003)

```java
interface Ethernet1
   no switchport
   ip address 10.0.11.2/30
!
interface Ethernet1/1
!
interface Ethernet2
   no switchport
   ip address 10.0.21.1/30
!
interface Ethernet3
   no switchport
   ip address 20.3.0.1/30
!
interface Ethernet4
   no switchport
   ip address 10.0.13.1/30
!
interface Loopback0
   ip address 10.255.0.3/32
!
...중략
router bgp 65003
   router-id 10.255.0.3
   neighbor 10.0.11.1 remote-as 65001
   neighbor 10.0.13.2 remote-as 65004
   neighbor 10.0.21.2 remote-as 65002
   network 10.0.11.0/30
   network 10.0.13.0/30
   network 10.0.21.0/30
   network 10.255.0.3/32
```

---

### Router_2 config (AS65004)

```java
!
interface Ethernet1
   no switchport
   ip address 10.0.12.2/30
!
interface Ethernet2
   no switchport
   ip address 10.0.22.1/30
!
interface Ethernet3
   no switchport
   ip address 20.4.0.1/30
!
interface Ethernet4
   no switchport
   ip address 10.0.13.2/30
!
interface Loopback0
   ip address 10.255.0.4/32
!
...중략
router bgp 65004
   router-id 10.255.0.4
   neighbor 10.0.12.1 remote-as 65001
   neighbor 10.0.13.1 remote-as 65003
   neighbor 10.0.22.2 remote-as 65002
   network 10.0.12.0/30
   network 10.0.13.0/30
   network 10.0.22.0/30
   network 10.255.0.4/32
```

### 📝 중간 설정 요약 
1. 각 피어는 독립된 AS를 생성.
2. 물리적으로 연결된 인접 AS의 인터페이스 IP와 AS 번호를 neighbor로 등록.
3. 이웃 등록 후 network 명령으로 현재 라우터가 광고할 네트워크 대역을 설정.

---

## 📊 3. 설정 후 상태 확인

```java
BB-1# show ip bgp summary 
BGP summary information for VRF default, address family IPv4 Unicast
BGP router identifier 10.255.0.1, local AS number 65001
BGP table version is 263, IPv4 Unicast config peers 2, capable peers 2
11 network entries and 14 paths using 3044 bytes of memory
BGP attribute entries [5/860], BGP AS path entries [4/32]
BGP community entries [0/0], BGP clusterlist entries [0/0]

Neighbor        V    AS MsgRcvd MsgSent   TblVer  InQ OutQ Up/Down  State/PfxRcd
10.0.11.2       4 65003    5057    5018      263    0    0 00:36:01 6         
10.0.12.2       4 65004    8781    8753      263    0    0 00:22:06 6

----------------------------------------------------------------------------------------

BB-2# show ip bgp summary 
BGP summary information for VRF default, address family IPv4 Unicast
BGP router identifier 10.255.0.2, local AS number 65002
BGP table version is 172, IPv4 Unicast config peers 2, capable peers 2
11 network entries and 20 paths using 3764 bytes of memory
BGP attribute entries [7/1204], BGP AS path entries [6/60]
BGP community entries [0/0], BGP clusterlist entries [0/0]

Neighbor        V    AS MsgRcvd MsgSent   TblVer  InQ OutQ Up/Down  State/PfxRcd
10.0.21.1       4 65003   13167   13108      172    0    0 08:43:41 9         
10.0.22.1       4 65004   13164   13112      172    0    0 08:44:28 9

----------------------------------------------------------------------------------------

ROUTER-1#show ip bgp summary 
BGP summary information for VRF default
Router identifier 10.255.0.3, local AS number 65003
Neighbor Status Codes: m - Under maintenance
  Neighbor         V  AS           MsgRcvd   MsgSent  InQ OutQ  Up/Down State   PfxRcd PfxAcc
  10.0.11.1        4  65001           5071      5081    0    0 00:37:28 Estab   5      5
  10.0.13.2        4  65004              0         0    0    0    2d07h Active         
  10.0.21.2        4  65002          13167     13188    0    0 08:44:45 Estab   6      6


----------------------------------------------------------------------------------------

ROUTER-2#show ip bgp  summary 
BGP summary information for VRF default
Router identifier 10.255.0.4, local AS number 65004
Neighbor Status Codes: m - Under maintenance
  Neighbor         V  AS           MsgRcvd   MsgSent  InQ OutQ  Up/Down State   PfxRcd PfxAcc
  10.0.12.1        4  65001           8766      8807    0    0 00:22:57 Estab   8      8
  10.0.13.1        4  65003              0         0    0    0    2d07h Active         
  10.0.22.2        4  65002          13154     13174    0    0 08:44:56 Estab   7      7





