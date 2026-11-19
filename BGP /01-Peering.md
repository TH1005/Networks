
# 🌐 1. BGP 세션 구성 및 통신 테스트
<img width="1453" height="815" alt="image" src="https://github.com/user-attachments/assets/049e60ca-915c-455e-a0a7-6edca61c093e" />

> **목표** : AS65001의 루프백 IP를 1번 또는 2번 경로를 통해 인접 AS에 광고하고, AS65002 라우터의 루프백과 통신을 시도한다.

<br>

---

## ⚙️ 2. 장비별 설정

### B/B_1 config (AS65001)

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
<br>

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

<br>

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

### 📝 설정 요약

1. 각 피어는 **독립된 AS**를 생성.
2. 물리적으로 연결된 인접 AS의 **인터페이스 IP**와 **AS 번호**를 `neighbor`로 등록.
3. 이웃 등록 후 `network` 명령으로 **현재 라우터가 광고할 네트워크 대역**을 설정.

<br>

---

## 📊 3. 설정 후 상태 확인

### ⚙️ BGP 세션 (show ip bgp summary)
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

> AS65001과 AS65002의 BGP summary를 보면 AS65003, AS65004와 BGP 세션이 **정상적으로 수립**되어 있다.
> 일정 주기마다 **Keepalive 패킷**을 주고받으며 세션을 유지한다.

<br>

---

## 📊  4. 라우팅 테이블 확인
```java
BB-1# show ip route 10.255.0.2 
IP Route Table for VRF "default"
'*' denotes best ucast next-hop
'**' denotes best mcast next-hop
'[x/y]' denotes [preference/metric]
'%<string>' in via output denotes VRF <string>

10.255.0.2/32, ubest/mbest: 1/0
    *via 10.0.11.2, [20/0], 00:02:06, bgp-65001, external, tag 65003

--------------------------------------------------------------------------

BB-2# show ip route 10.255.0.1
IP Route Table for VRF "default"
'*' denotes best ucast next-hop
'**' denotes best mcast next-hop
'[x/y]' denotes [preference/metric]
'%<string>' in via output denotes VRF <string>

10.255.0.1/32, ubest/mbest: 1/0
    *via 10.0.21.1, [20/0], 00:04:16, bgp-65002, external, tag 65003
```
### ✅ 라우팅 테이블 항목 해석
| 항목 | 설명 |
|------|------|
| `10.255.0.1/32` | 목적지 네트워크 주소 |
| `ubest/mbest` | 유니캐스트/멀티캐스트 최적 경로 수 (현재 유니캐스트 최적 경로 1개) |
| `via` | Next-Hop |
| `[20/0]` | `[AD/Metric]` 값. eBGP로 생성된 경로의 기본 AD (OSPF: 110, iBGP: 200 등) |
| `00:04:16` | 라우팅 테이블에 등록된 후 경과 시간 |
| `bgp-65002` | 현재 로컬 AS 번호 |
| `external` | 외부 AS와 BGP(eBGP)를 맺고 있다는 의미 |
| `tag 65003` | 상대 AS 번호 기록 (루핑 방지 목적) |
 
<br>

---

## 📌 5. Best Path 선정
  
AS65001과 AS65002 모두 서로의 루프백으로 가는 경로를 **AS65003 방향**으로 잡고 있다.
BGP는 별도 정책이 없을 때 **AS-Path가 짧은 경로**를 우선 선택한다. AS-Path 길이까지 같다면 아래 순서로 Best Path를 결정한다.
| 순서 | 기준 | 설명 |
|:----:|------|------|
| 1 | **Origin Code** | `IGP (i)` > `EGP (e)` > `Incomplete (?)` 순으로 우선 |
| 2 | **가장 낮은 MED** | 인접 AS가 전달한 메트릭 값이 낮은 쪽 선택 |
| 3 | **Next-Hop까지의 IGP 메트릭** | Next-Hop IP까지의 내부 라우팅(OSPF, IS-IS 등) 비용이 가장 낮은 경로 선택 |
| 4 | **가장 오래된 경로** | 위 항목이 모두 같고 두 경로 모두 eBGP로 받은 경우, 먼저 수립된 경로 선택 |

> 현재 구성에는 별도의 Metric 값이나 IGP가 없으므로, **세션을 먼저 맺은(가장 오래된) 경로**가 Best Path로 선정된다.

### 🔎 Best Path 확인 (show ip bgp)
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
> 두 장비 모두 Best Path를 **AS65003 방향**으로 잡고 있다.
> AS65004 경로가 Best Path가 아닌 이유는 `newer EBGP path`, 즉 **나중에 학습된 eBGP 경로**이기 때문이다.

<br>

---


## 🔍 6. 통신 테스트 (traceroute)
```java
BB-1# trace 10.255.0.2 source 10.255.0.1
traceroute to 10.255.0.2 (10.255.0.2) from 10.255.0.1 (10.255.0.1), 30 hops max, 40 byte packets
 1  10.0.11.2 (10.0.11.2) (AS 65003)  12.223 ms  10.831 ms  7.793 ms
 2  10.255.0.2 (10.255.0.2) (AS 65002)  18.831 ms  18.753 ms  18.235 ms
```
B/B_1에서 traceroute로 AS65002의 루프백과 통신을 시도하면 라우팅 테이블에 학습된 대로 10.0.11.2(AS65003) 방향으로 통신이되는 것을 확인 할 수 있다.

<br>

---

## 🔄 7. 백업 경로 테스트
BGP는 Best Path가 다운되면 **보유하고 있던 백업 경로를 즉시 Best Path로 승격**하고 라우팅 테이블을 수정한다.
이 승격은 여러 출구 중 하나를 선택한 것이므로 **외부 AS 라우터에 별도로 광고하지 않는다.**

### 🛠️ AS65003 방향 링크 단절 후 Best Path

```
BB-1# show ip bgp 10.255.0.2
BGP routing table information for VRF default, address family IPv4 Unicast
BGP routing table entry for 10.255.0.2/32, version 191
Paths: (1 available, best #1)
Flags: (0x8008001a) (high32 00000000) on xmit-list, is in urib, is best urib rou
te, is in HW

  Advertised path-id 1
  Path type: external, path is valid, is best path, no labeled nexthop, in rib
  AS-Path: 65004 65002 , path sourced external to AS
    10.0.12.2 (metric 0) from 10.0.12.2 (10.255.0.4)
      Origin IGP, MED not set, localpref 100, weight 0

  Path-id 1 not advertised to any peer
```

### 🔍 단절 후 traceroute

```java
BB-1# traceroute 10.255.0.2 source 10.255.0.1
traceroute to 10.255.0.2 (10.255.0.2), 30 hops max, 40 byte packets
 1  10.0.12.2 (10.0.12.2) (AS 65004)  12.257 ms  7.793 ms  9.096 ms
 2  10.255.0.2 (10.255.0.2) (AS 65002)  14.877 ms  14.562 ms  13.054 ms
```

> AS65003 방향 링크를 끊은 뒤 Best Path가 **AS65004 방향**으로 바뀌었고, AS65003 방향 경로는 사라졌다.
> traceroute 결과에서도 트래픽이 **10.0.12.2 (AS65004)** 쪽으로 흐르는 것을 확인할 수 있다.

<br>

### 🔍 BGP Update 패킷 분석

<img width="810" height="595" alt="image" src="https://github.com/user-attachments/assets/50bf7407-1c9c-4920-97b9-9bb95f0ded43" />

- ROUTER_1은 B/B_1과의 세션이 끊겼으므로 `10.255.0.1 via 10.0.11.1` 라우팅을 **삭제**한다.
- 라우팅 테이블이 바뀌었으므로, B/B_2에게 "10.255.0.1로 가기 위해 AS65003을 경유하는 경로를 **철회**하라"는 Update 패킷을 보낸다.

---

<img width="799" height="686" alt="image" src="https://github.com/user-attachments/assets/a8f4b8c6-ebd6-4d87-962f-4fc0acda517c" />

- ROUTER_1이 보낸 Update에 따라 B/B_2도 경로를 수정하고, **새로 찾은 경로를 다시 ROUTER_1에게 광고**한다.
- 경로 속성은 `65002 65004 65001`을 경유하는 경로이다.
- 대상 네트워크는 `10.255.0.1/32`, `20.1.0.0/30`으로, B/B_1의 네트워크로 가기 위한 경로이다.
