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


> 중간 라우터를 통과해서 AS끼리 정상적으로 BGP 세션을 맺은 것을 확인할 수 있다.

## 📊 4. 라우팅 테이블 확인

```java
BB-1# show ip route bgp-65001 
IP Route Table for VRF "default"
'*' denotes best ucast next-hop
'**' denotes best mcast next-hop
'[x/y]' denotes [preference/metric]
'%<string>' in via output denotes VRF <string>

20.2.0.0/30, ubest/mbest: 1/0
    *via 10.255.0.2, [20/0], 00:56:56, bgp-65001, external, tag 65002

----------------------------------------------------------------------------------------

BB-2# show ip route bgp
IP Route Table for VRF "default"
'*' denotes best ucast next-hop
'**' denotes best mcast next-hop
'[x/y]' denotes [preference/metric]
'%<string>' in via output denotes VRF <string>

20.1.0.0/30, ubest/mbest: 1/0
    *via 10.255.0.1, [20/0], 00:57:32, bgp-65002, external, tag 65001

```

위의 라우팅 테이블을 보면 피어링 된 후에 각 AS의 내부 라우터에 대한 경로를 루프백 IP로 잡고있는 것을 확인할 수 있다. 
이렇게 인접하지 않은 AS와도 세션을 맺어서 라우팅 정보를 제공 받을 수 있다. 하지만 결국 10.255.0.2가 B/B_1(AS65001)에 있는 내부 라우터와 통신하기 위해서는 중간 라우터에게도 20.1.0.0 네트워크로가는 라우팅이 필요하기 때문에 static 라우팅을 사용한다면 일일히 하나씩 수정을 해줘야한다. 그래서 eBGP를 만 사용하지 않고 별도의 GRE, VLXLAN/EVPN터널링을 맺어서 트래픽이 루프백에 도달 하고 내부 라우터까지 도달 할 수 있도록 한다.

---


## 🔍 6. 통신 테스트 (traceroute)
```java
BB-1# ping 20.2.0.1 source 10.255.0.1
PING 20.2.0.1 (20.2.0.1) from 10.255.0.1: 56 data bytes
64 bytes from 20.2.0.1: icmp_seq=0 ttl=253 time=21.715 ms
64 bytes from 20.2.0.1: icmp_seq=1 ttl=253 time=14.179 ms
64 bytes from 20.2.0.1: icmp_seq=2 ttl=253 time=22.296 ms
64 bytes from 20.2.0.1: icmp_seq=3 ttl=253 time=16.944 ms
64 bytes from 20.2.0.1: icmp_seq=4 ttl=253 time=16.478 ms
----------------------------------------------------------------------------------------

BB-2# ping 20.1.0.2 source 10.255.0.2
PING 20.1.0.2 (20.1.0.2) from 10.255.0.2: 56 data bytes
64 bytes from 20.1.0.2: icmp_seq=0 ttl=252 time=18.415 ms
64 bytes from 20.1.0.2: icmp_seq=1 ttl=252 time=21.111 ms
64 bytes from 20.1.0.2: icmp_seq=2 ttl=252 time=17.372 ms
64 bytes from 20.1.0.2: icmp_seq=3 ttl=252 time=19.889 ms
64 bytes from 20.1.0.2: icmp_seq=4 ttl=252 time=24.069 ms
```
ICMP로 통신을 해보면 루프백 IP를 Source로 하면 정상적으로 통신이 되는 것을 확인 할 수 있다.

여기서 B/B_1은 20.2.0.0/30 네트워크로 가기 위해서 Next-Hop을 AS65002(10.255.0.2)로 잡고 있다. 하지만 AS65002는 직결된 라우터가 아니다. 패킷의 목적지는 20.2.0.1으로 설정이 되는데 B/B_1은 20.2.0.0/30의 정적 라우팅을 따로 추가해주지 않았다, 즉 20.2.0.0/30 으로 가기위해서 어떤 물리 인터페이스로 나가야 하는지 설정이 안되어있는 것이다. 그럼에도 통신이 되는 이유는 `재귀적 라우팅` 때문이다.

재귀적 라우팅은 특정 목적지로 가기위한 Next-Hop이 직결된 인터페이스가 아닐 때, Next-Hop IP를 목적지를 삼아 다시 라우팅 테이블을 조회한다. 이 과정을 직결된 인터페이스를 찾을 때 까지 계속 반복한다.
B/B_1에는 10.255.0.2로 가는 정적 라우팅이 잡혀있다. 결국 장비에서 10.255.0.2를 목적지 삼아서 직결된 Next-Hop을 찾아서 패킷을 포워딩 한 것이다. 

<img width="678" height="282" alt="image" src="https://github.com/user-attachments/assets/1493c6e6-491c-45fe-b560-57fa5228a943" />
<img width="493" height="79" alt="image" src="https://github.com/user-attachments/assets/18f963d8-f58d-4193-b8d5-b97c8901a91e" />

캡쳐된 패킷과 B/B_1에 직결된 ROUTER_2의 인터페이스 MAC을 보면 동일한 것을 볼 수 있다. 20.2.0.2의 목적지는 10.255.0.2지만 실제로 패킷은 10.255.0.2로 가는 Next-Hop중 하나인 ROUTER_2의 MAC으로 포워딩 되고 있다. 여기서 ROUTER_2는 20.2.0.0/30로 가는 경로를 직결 인터페이스로하여 정적 라우팅을 추가 해줬기 때문에 10.255.0.2를 가진 B/B_2에 도달할 수 있는 것이다. 패킷의 목적지는 20.2.0.2기 때문에 중간 라우터에 20.2.0.0/30으로 가는 라우팅을 빼고 10.255.0.2로 가는 라우팅을 넣으면 통신이 되지 않는다.

물론 20.2.0.2의 Next-Hop을 10.255.0.2로 설정하고 10.255.0.2의 Next-Hop을 직결 인터페이스로 하면 재귀적 라우팅을 조회하나 정적 라우팅을 그렇게 쓸 이유가 없다.





