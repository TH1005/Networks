
### 1. BGP 맺고 통신하기 인접 AS를 통해서 반대편 AS와 통신하기
<img width="1453" height="815" alt="image" src="https://github.com/user-attachments/assets/049e60ca-915c-455e-a0a7-6edca61c093e" />
AS65001 루프백 IP가 1번 또는 2번 경로를 통해서 루프백 IP를 인접 AS에게 광고하여 AS65002 라우터의 루프백과 통신을 시도

<br>
<br>
<br>

**B/B_1 config**

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
<br>

**B/B_2 config**
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
<br>

**Router_1 config**
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

<br>

**Router_2 config**
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

각 피어는 독립된 AS를 생성 후 물리적으로 연결된 인접 AS의 인터페이스 IP와 AS번호로 neighbor로 등록한다. 이웃을 등록 후 network 명령을 통해서 현재 라우터에서 광고할 네트워크 대역을 설정 한다.

<br>

**설정 후 상태**
```java
BB-1# show ip bgp summary 
BGP summary information for VRF default, address family IPv4 Unicast
BGP router identifier 10.255.0.1, local AS number 65001
BGP table version is 20, IPv4 Unicast config peers 2, capable peers 2
11 network entries and 14 paths using 3044 bytes of memory
BGP attribute entries [5/860], BGP AS path entries [4/32]
BGP community entries [0/0], BGP clusterlist entries [0/0]

Neighbor        V    AS MsgRcvd MsgSent   TblVer  InQ OutQ Up/Down  State/PfxRcd
10.0.11.2       4 65003      14      12       20    0    0 00:06:29 6         
10.0.12.2       4 65004      12      10       20    0    0 00:06:31 6

----------------------------------------------------------------------------------------

BB-2# show ip bgp summary 
BGP summary information for VRF default, address family IPv4 Unicast
BGP router identifier 10.255.0.2, local AS number 65002
BGP table version is 62, IPv4 Unicast config peers 2, capable peers 2
11 network entries and 20 paths using 3764 bytes of memory
BGP attribute entries [7/1204], BGP AS path entries [6/60]
BGP community entries [0/0], BGP clusterlist entries [0/0]

Neighbor        V    AS MsgRcvd MsgSent   TblVer  InQ OutQ Up/Down  State/PfxRcd
10.0.21.1       4 65003      23      14       62    0    0 00:06:19 9         
10.0.22.1       4 65004      15      10       62    0    0 00:06:22 9
```

<img width="913" height="170" alt="image" src="https://github.com/user-attachments/assets/27041158-f185-46e5-b301-e53b5c404e74" />
AS65001와 AS65002의 BGP summary를 보면 AS65003과 AS65004의 인터페이스와 BGP 세션을 정상적으로 맺고 있으며, 일정 주기마다 BGP Keppalive 패킷을 주고 받아서 세션을 유지하고 있다.

<br>

#### 라우팅 테이블 확인
```java
BB-1# show ip route bgp
IP Route Table for VRF "default"
'*' denotes best ucast next-hop
'**' denotes best mcast next-hop
'[x/y]' denotes [preference/metric]
'%<string>' in via output denotes VRF <string>

10.0.13.0/30, ubest/mbest: 1/0
    *via 10.0.11.2, [20/0], 00:00:47, bgp-65001, external, tag 65003
10.0.21.0/30, ubest/mbest: 1/0
    *via 10.0.11.2, [20/0], 00:04:24, bgp-65001, external, tag 65003
10.0.22.0/30, ubest/mbest: 1/0
    *via 10.0.12.2, [20/0], 00:00:18, bgp-65001, external, tag 65004
10.255.0.2/32, ubest/mbest: 1/0
    *via 10.0.11.2, [20/0], 00:00:47, bgp-65001, external, tag 65003
10.255.0.3/32, ubest/mbest: 1/0
    *via 10.0.11.2, [20/0], 00:04:24, bgp-65001, external, tag 65003
10.255.0.4/32, ubest/mbest: 1/0
    *via 10.0.12.2, [20/0], 00:00:18, bgp-65001, external, tag 65004
20.2.0.0/30, ubest/mbest: 1/0
    *via 10.0.11.2, [20/0], 00:00:47, bgp-65001, external, tag 65003

--------------------------------------------------------------------------

BB-2# show ip route bgp
IP Route Table for VRF "default"
'*' denotes best ucast next-hop
'**' denotes best mcast next-hop
'[x/y]' denotes [preference/metric]
'%<string>' in via output denotes VRF <string>

10.0.11.0/30, ubest/mbest: 1/0
    *via 10.0.21.1, [20/0], 00:23:34, bgp-65002, external, tag 65003
10.0.12.0/30, ubest/mbest: 1/0
    *via 10.0.22.1, [20/0], 00:23:36, bgp-65002, external, tag 65004
10.0.13.0/30, ubest/mbest: 1/0
    *via 10.0.22.1, [20/0], 00:23:36, bgp-65002, external, tag 65004
10.255.0.1/32, ubest/mbest: 1/0
    *via 10.0.21.1, [20/0], 00:01:25, bgp-65002, external, tag 65003
10.255.0.3/32, ubest/mbest: 1/0
    *via 10.0.21.1, [20/0], 00:23:34, bgp-65002, external, tag 65003
10.255.0.4/32, ubest/mbest: 1/0
    *via 10.0.22.1, [20/0], 00:23:36, bgp-65002, external, tag 65004
20.1.0.0/30, ubest/mbest: 1/0
    *via 10.0.21.1, [20/0], 00:01:25, bgp-65002, external, tag 65003
```

AS65001과 AS65002를 보면 서로의 루프백으로 위한 라우팅을 AS65003 방향으로 잡고 있다. BGP 같은 경우는 아무런 정책이 없을 때는 AS-Path경로가 짧은 쪽을 우선으로 선정하게 되어있다. 만약 경로 비용이 같다면 아래 4가지를 통해서 Best Path를 선정한다.
1. Origin Code : IGP (i) > EGP (e) > Incomplete (?) 순으로 우선순위를 가짐.
2. 가장 낮은 MED (Metric): 인접 AS에서 넘겨준 메트릭 값이 낮은 쪽을 선택.
3. Next-Hop까지의 IGP 메트릭: 경로의 Next-Hop IP까지 도달하는 데 걸리는 내부 라우팅(OSPF, ISIS 등) 비용이 가장 낮은 경로를 선택.
4. 가장 오래된 경로 (Oldest Path / 먼저 연결된 링크): 위 1~4번까지 모두 똑같고, 두 경로 모두 eBGP로 받은 경우에만 가장 먼저 수립.

현재 설정에는 별도의 Metric값이나 IGP가 없기 때문에 세션을 맺은지 가장 오래된 경로를 Best Path로 선정한다.

#### Best Path 확인
```java
BB-1# show ip bgp 10.255.0.2
BGP routing table information for VRF default, address family IPv4 Unicast
BGP routing table entry for 10.255.0.2/32, version 89
Paths: (2 available, best #1)
Flags: (0x8008001a) (high32 00000000) on xmit-list, is in urib, is best urib rou
te, is in HW

  Advertised path-id 1
  Path type: external, path is valid, is best path, no labeled nexthop, in rib
  AS-Path: 65003 65002 , path sourced external to AS
    10.0.11.2 (metric 0) from 10.0.11.2 (10.255.0.3)
      Origin IGP, MED not set, localpref 100, weight 0

  Path type: external, path is valid, not best reason: newer EBGP path, no label
ed nexthop
  AS-Path: 65004 65002 , path sourced external to AS
    10.0.12.2 (metric 0) from 10.0.12.2 (10.255.0.4)
      Origin IGP, MED not set, localpref 100, weight 0

  Path-id 1 advertised to peers:
    10.0.12.2

--------------------------------------------------------------------------------------------------

BB-2# show ip bgp 10.255.0.1
BGP routing table information for VRF default, address family IPv4 Unicast
BGP routing table entry for 10.255.0.1/32, version 76
Paths: (2 available, best #2)
Flags: (0x8008001a) (high32 00000000) on xmit-list, is in urib, is best urib rou
te, is in HW

  Path type: external, path is valid, not best reason: newer EBGP path, no label
ed nexthop
  AS-Path: 65004 65001 , path sourced external to AS
    10.0.22.1 (metric 0) from 10.0.22.1 (10.255.0.4)
      Origin IGP, MED not set, localpref 100, weight 0

  Advertised path-id 1
  Path type: external, path is valid, is best path, no labeled nexthop, in rib
  AS-Path: 65003 65001 , path sourced external to AS
    10.0.21.1 (metric 0) from 10.0.21.1 (10.255.0.3)
      Origin IGP, MED not set, localpref 100, weight 0

  Path-id 1 advertised to peers:
    10.0.22.1      
```
두 장비 모두 Best Path를 AS65003 방향으로 잡고 있다. AS65004가 Best Path가 아닌 이유를 보면  `newer EBGP path`로 새로 학습된 EBGP 경로이기 때문이다.


#### 통신 시도하기
```java
BB-1# traceroute 10.255.0.2
traceroute to 10.255.0.2 (10.255.0.2), 30 hops max, 40 byte packets
 1  10.0.11.2 (10.0.11.2) (AS 65003)  10.036 ms  8.199 ms  7.82 ms
 2  10.255.0.2 (10.255.0.2) (AS 65002)  35.027 ms  16.403 ms  21.479 ms
```
B/B_1에서 traceroute로 AS65002의 루프백과 통신을 시도하면 라우팅 테이블에 학습된 대로 10.0.11.2(AS65003) 방향으로 통신이되는 것을 확인 할 수 있다.



#### 경로 업데이트
BGP는 라우팅 경로에 이상이 감지되면 변경된 정보만을 담은 UPDATE 패킷을 인접 피어(Peer)에게 즉시 송신하여 경로를 갱신(철회 또는 우회 경로 재광고)한다.

<img width="1332" height="552" alt="image" src="https://github.com/user-attachments/assets/2423cbeb-9731-411c-a315-bac49b90d3c2" />
현재 AS65003 방향으로가는 링크를 끊어버린 후 AS65002의 루프백 IP로 가는 경로를 업데이트 메세지로 AS65004 방향으로 다시 업데이트 받았다.

```java
BB-1# traceroute 10.255.0.2
traceroute to 10.255.0.2 (10.255.0.2), 30 hops max, 40 byte packets
 1  10.0.12.2 (10.0.12.2) (AS 65004)  12.257 ms  7.793 ms  9.096 ms
 2  10.255.0.2 (10.255.0.2) (AS 65002)  14.877 ms  14.562 ms  13.054 ms
```
다시 통신을 시도해보면 이번에는 10.0.12.2(AS65004) 방향으로 트래픽이 흐른것을 볼 수 있다.



