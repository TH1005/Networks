# 🌐 1. 고가용성을 위한 루프백으로 BGP Neighbor 맺기

<img width="931" height="322" alt="image" src="https://github.com/user-attachments/assets/c1691207-f9e8-44d1-9997-e7b90d637632" />


> **목표** <br>
> - B/B_1과 B/B_2는 중간의 ROUTER들을 통해서 루프백으로 BGP 세션 맺기<br>
> - 2개의 경로 중 1개를 끊어도 세션이 살아있는지 확인<br>
> - 각 B/B에 내부 라우터와 통신 테스트


<br>

---

## ⚙️ 2. 장비별 설정

### B/B_1 Config

```java
feature bgp

ip route 10.255.0.2/32 10.0.11.2
ip route 10.255.0.2/32 10.0.12.2

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
  timers bgp 5 15
  address-family ipv4 unicast
    network 20.1.0.0/30
  neighbor 10.255.0.2
    remote-as 65002
    update-source loopback0
    ebgp-multihop 2
    address-family ipv4 unicast
```

---

### B/B_2 config (AS65002)

```java
feature bgp

ip route 10.255.0.1/32 10.0.21.1
ip route 10.255.0.1/32 10.0.22.1

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
  timers bgp 5 15
  address-family ipv4 unicast
    network 20.2.0.0/30
  neighbor 10.255.0.1
    remote-as 65001
    update-source loopback0
    ebgp-multihop 2
    address-family ipv4 unicast
  ```

---

### Router_1 config (AS65003)

```java
interface Ethernet1
   no switchport
   ip address 10.0.11.2/30
!
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
interface Loopback0
   ip address 10.255.0.3/32
!
ip route 10.255.0.1/32 10.0.11.1
ip route 10.255.0.2/32 10.0.21.2
```

---

### Router_2 config (AS65004)

```java
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
interface Loopback0
   ip address 10.255.0.4/32
!
ip route 10.255.0.1/32 10.0.12.1
ip route 10.255.0.2/32 10.0.22.2

```

### 📝 설정 요약 
1. AS65001과 AS65002는 neighbor를 서로의 루프백으로 설정.
2. 세션을 맺기 위해서는 네트워크 통신이 가능해야하기 때문에 B/B과 라우터에 루프백으로 가는 정적 라우팅 경로를 추가.
3. 인접한 라우터가 아니기 때문에 멀티홉 수 설정 (BGP는 기본 TTL이 1이기 때문에 해당 설정을 넣어주지 않으면 중간 라우터에 도달한 후 사라짐).
4. 각 AS에서는 neighbor를 루프백으로 설정해놨기 때문에 update-source를 루프백으로 IP로 설정하도록 명령어 추가 (해당 명령어를 추가 안할 시 세션을 맺을 때 source ip를 인터페이스 ip로 달고 나가기 때문에 이웃으로 설정한 AS에 도착해도 설정된 이웃의 IP가 아니니 세션이 맺어지지가 않는다.)

---

## 📊 3. 설정 후 상태 확인

```java
BB-1# show ip bgp summary 
BGP summary information for VRF default, address family IPv4 Unicast
BGP router identifier 10.255.0.1, local AS number 65001
BGP table version is 66, IPv4 Unicast config peers 1, capable peers 1
2 network entries and 2 paths using 488 bytes of memory
BGP attribute entries [2/344], BGP AS path entries [1/6]
BGP community entries [0/0], BGP clusterlist entries [0/0]

Neighbor        V    AS MsgRcvd MsgSent   TblVer  InQ OutQ Up/Down  State/PfxRcd
10.255.0.2      4 65002     332     336       66    0    0 00:12:22 1

----------------------------------------------------------------------------------------

BB-2# show ip bgp summary 
BGP summary information for VRF default, address family IPv4 Unicast
BGP router identifier 10.255.0.2, local AS number 65002
BGP table version is 153, IPv4 Unicast config peers 1, capable peers 1
2 network entries and 2 paths using 488 bytes of memory
BGP attribute entries [2/344], BGP AS path entries [1/6]
BGP community entries [0/0], BGP clusterlist entries [0/0]

Neighbor        V    AS MsgRcvd MsgSent   TblVer  InQ OutQ Up/Down  State/PfxRcd
10.255.0.1      4 65001     332     329      153    0    0 00:12:34 1 


> AS끼리 정상적으로 BGP 세션을 맺은것을 확인할 수 있다.

## 📊 4. 광고 및 라우팅 테이블 확인

```java
BB-1# show ip bgp neighbors 10.0.11.2 advertised-routes 

Peer 10.0.11.2 routes for address family IPv4 Unicast:
BGP table version is 263, Local Router ID is 10.255.0.1
Status: s-suppressed, x-deleted, S-stale, d-dampened, h-history, *-valid, >-best
Path type: i-internal, e-external, c-confed, l-local, a-aggregate, r-redist, I-i
njected
Origin codes: i - IGP, e - EGP, ? - incomplete, | - multipath, & - backup, 2 - b
est2

   Network            Next Hop            Metric     LocPrf     Weight Path
*>e10.0.12.0/30       10.0.12.2                                      0 65004 i # 외부피어에게 학습한 경로는 weight 0, i : IGP
*>e10.0.22.0/30       10.0.12.2                                      0 65004 i
*>l10.255.0.1/32      0.0.0.0                           100      32768 i       # 내가 만든 경로는 32786 weight 값을 가짐
*>e10.255.0.4/32      10.0.12.2                                      0 65004 i
*>l20.1.0.0/30        0.0.0.0                           100      32768 i

----------------------------------------------------------------------------------------

ROUTER-1#show ip route bgp 

VRF: default
Codes: C - connected, S - static, K - kernel, 
       O - OSPF, IA - OSPF inter area, E1 - OSPF external type 1,
       E2 - OSPF external type 2, N1 - OSPF NSSA external type 1,
       N2 - OSPF NSSA external type2, B - Other BGP Routes,
       B I - iBGP, B E - eBGP, R - RIP, I L1 - IS-IS level 1,
       I L2 - IS-IS level 2, O3 - OSPFv3, A B - BGP Aggregate,
       A O - OSPF Summary, NG - Nexthop Group Static Route,
       V - VXLAN Control Service, M - Martian,
       DH - DHCP client installed default route,
       DP - Dynamic Policy Route, L - VRF Leaked,
       G  - gRIBI, RC - Route Cache Route

 B E      10.0.12.0/30 [200/0] via 10.0.11.1, Ethernet1
 B E      10.0.22.0/30 [200/0] via 10.0.11.1, Ethernet1
 B E      10.255.0.1/32 [200/0] via 10.0.11.1, Ethernet1 # AS65001에게 직접 광고를 받음
 B E      10.255.0.2/32 [200/0] via 10.0.21.2, Ethernet2 
 B E      10.255.0.4/32 [200/0] via 10.0.11.1, Ethernet1
 B E      20.1.0.0/30 [200/0] via 10.0.11.1, Ethernet1 # AS65001에게 직접 광고를 받음
 B E      20.2.0.0/30 [200/0] via 10.0.21.2, Ethernet2
```

###  ✅ 광고 및 라우팅 테이블 분석

B/B_1이 피어링을 맺은 10.0.11.2에게 확인을 해보면
- *>l10.255.0.1/32      0.0.0.0                           100      32768 i
- *>l20.1.0.0/30        0.0.0.0                           100      32768 i
로컬 네트워크(l로 표기) 대역이 정상적으로 광고 되고 있음을 확인할 수 있다. LocalPref는 시스코의 기본 값인 100으로 설정 된다. 로컬 경로임에도 설정되는 이유는 BGP 표준에 따라 유효한 경로를 BGP 테이블에 올리기 위해서는 무조건 LocalPreference 값을 가져야한다.
로컬 네트워크가 weight 32768이 설정 되는 이유는 BGP는 weight 값을 가장 우선순위로 고려하는데 이때 로컬 네트워크로 통신하는 라우팅이 다른 AS에게 광고받은 경로보다 weight값이 낮게 되면 로컬 네트워크임에도 다른 AS를 우회할 수 있기 때문에 로컬 대역은 항상 최우선으로 잡는 것이다. 

ROUTER_1의 라우팅 테이블을 보면 10.255.0.1, 20.1.0.0/30에 대한 Next-Hop이 모두 B/B_1(AS65001) 방향이로 설정 되어있다.
별도의 정책을 설정하지 않았기 때문에 B/B_1에게 직접 광고 받고 라우팅 테이블을 갱신한것이다.

---

## 5. 라우팅 필터링 정책 추가
### B/B_1 Config 추가

```java
ip prefix-list BLOCK_To_65003 seq 10 permit 10.255.0.1/32

route-map MAP_TO_65003 deny 10
  match ip address prefix-list BLOCK_To_65003 
route-map MAP_TO_65003 permit 20

router bgp 65001
  address-family ipv4 unicast
    network 10.255.0.1/32
    network 20.1.0.0/30
  neighbor 10.0.11.2
    remote-as 65003
    address-family ipv4 unicast
      route-map MAP_TO_65003 out
  neighbor 10.0.12.2
    remote-as 65004
    address-family ipv4 unicast

```

### 📝 설정 설명

| 순서 | 명령어 | 설명 |
|:----:|--------|------|
| 1 | `ip prefix-list BLOCK_To_65003 seq 10 permit 10.255.0.1/32` | 정책에서 매칭할 대상(`10.255.0.1/32`)을 정의하는 **prefix-list** 생성 |
| 2 | `route-map MAP_TO_65003 deny 10` | 광고 정책의 **첫 번째 규칙(seq 10)**. 조건에 매칭되는 경로를 **차단(deny)** |
| 3 | `match ip address prefix-list BLOCK_To_65003` | seq 10의 매칭 조건으로 위 prefix-list 지정. 즉 `10.255.0.1/32`만 차단 대상 |
| 4 | `route-map MAP_TO_65003 permit 20` | 매칭 조건이 없으므로 **나머지 모든 경로를 허용**. 없으면 route-map 끝의 **암묵적 deny** 때문에 나머지 경로도 전부 광고되지 않음 |
| 5 | `route-map MAP_TO_65003 out` | 10.0.11.2(AS65003) 이웃에게 **광고(out)할 때** 해당 route-map 적용 |


### 설정 적용 후 B/B_1 광고 내역
```java
BB-1# show ip bgp neighbors 10.0.11.2 advertised-routes 

Peer 10.0.11.2 routes for address family IPv4 Unicast:
BGP table version is 264, Local Router ID is 10.255.0.1
Status: s-suppressed, x-deleted, S-stale, d-dampened, h-history, *-valid, >-best
Path type: i-internal, e-external, c-confed, l-local, a-aggregate, r-redist, I-i
njected
Origin codes: i - IGP, e - EGP, ? - incomplete, | - multipath, & - backup, 2 - b
est2

   Network            Next Hop            Metric     LocPrf     Weight Path
*>e10.0.12.0/30       10.0.12.2                                      0 65004 i
*>e10.0.22.0/30       10.0.12.2                                      0 65004 i
*>e10.255.0.4/32      10.0.12.2                                      0 65004 i
*>l20.1.0.0/30        0.0.0.0                           100      32768 i
```
10.255.0.1을 광고하지 않는 정책을 설정 후 라우팅 광고 내역을 보면 10.255.0.1은 10.0.11.2(AS65003)에게 광고하지 않는 것으로 확인할 수 있다.

### 수정된 ROUTER_1 라우팅 테이블
```java
ROUTER-1#show ip route bgp

VRF: default
Codes: C - connected, S - static, K - kernel, 
       O - OSPF, IA - OSPF inter area, E1 - OSPF external type 1,
       E2 - OSPF external type 2, N1 - OSPF NSSA external type 1,
       N2 - OSPF NSSA external type2, B - Other BGP Routes,
       B I - iBGP, B E - eBGP, R - RIP, I L1 - IS-IS level 1,
       I L2 - IS-IS level 2, O3 - OSPFv3, A B - BGP Aggregate,
       A O - OSPF Summary, NG - Nexthop Group Static Route,
       V - VXLAN Control Service, M - Martian,
       DH - DHCP client installed default route,
       DP - Dynamic Policy Route, L - VRF Leaked,
       G  - gRIBI, RC - Route Cache Route

 B E      10.0.12.0/30 [200/0] via 10.0.11.1, Ethernet1
 B E      10.0.22.0/30 [200/0] via 10.0.11.1, Ethernet1
 B E      10.255.0.1/32 [200/0] via 10.0.21.2, Ethernet2 //AS65002 -> AS65004 -> AS65001로 경로가 잡힘
 B E      10.255.0.2/32 [200/0] via 10.0.21.2, Ethernet2
 B E      10.255.0.4/32 [200/0] via 10.0.11.1, Ethernet1
 B E      20.1.0.0/30 [200/0] via 10.0.11.1, Ethernet1
 B E      20.2.0.0/30 [200/0] via 10.0.21.2, Ethernet2
```
정책 적용 후 **ROUTER_1**의 라우팅 테이블을 확인해보면 10.255.0.1은 인접 AS임에도 AS65002 -> AS65004를 경우해서 들어가도록 설정이 되어있다.

---

## 🔍 6. 통신 테스트 (traceroute)
```java
ROUTER-1#traceroute 10.255.0.1 source 10.255.0.3
traceroute to 10.255.0.1 (10.255.0.1), 30 hops max, 60 byte packets
 1  10.0.21.2 (10.0.21.2)  12.740 ms  8.458 ms  16.785 ms
 2  10.0.22.1 (10.0.22.1)  24.672 ms  30.303 ms  33.501 ms
 3  10.255.0.1 (10.255.0.1)  41.069 ms  44.902 ms  48.791 ms

----------------------------------------------------------------------------------------

ROUTER-1#traceroute 20.1.0.2  source 10.255.0.3
traceroute to 20.1.0.2 (20.1.0.2), 30 hops max, 60 byte packets
 1  10.0.11.1 (10.0.11.1)  22.736 ms  27.573 ms  30.810 ms
 2  20.1.0.2 (20.1.0.2)  34.116 ms * *
```
traceroute로 통신 테스트를 해보면 20.1.0.2는 바로 AS65001로 가는 것을 볼 수 있고, 10.255.0.1은 AS65002 -> AS65004를 경우해서 들어가는 것을 확인할 수 있다.









