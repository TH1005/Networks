# 4-1 인바운드 경로 제어 (AS-Path Prepend)

BGP는 같은 목적지에 대해 여러 경로를 광고받으면, 정해진 순서의 속성 비교를 통해 **best path** 하나를 선정한다. 
내가 이웃에게 아무 정책 없이 광고를 보내면 상대는 기본 규칙으로 경로를 고르므로, 나에게 들어오는 트래픽이 내가 의도한 경로로 오지 않을 수 있다.

이를 제어하기 위해 이웃에게 광고를 보낼 때(out) 해당 광고의 속성을 조작하여, 상대가 내 광고를 보고 선택하는 경로를 유도할 수 있다. 
route-map에서 **AS-Path Prepend**나 **MED** 값을 설정한 뒤 이웃에 적용하면 그 이웃에게 보내는 모든 광고에 적용되고, prefix-list로 대상을 지정하면 매칭되는 prefix의 광고에만 적용할 수 있다. 


<br>


## 1. 테스트 구성도 
<img width="914" height="513" alt="image" src="https://github.com/user-attachments/assets/7b63f32d-953b-45d5-8b24-0efb49ec7568" />

4개의 외부 라우터가 있으며 각각의 AS를 가지고 인접 라우터와 피어링을 맺은 상태, `AS65001`과 `AS65002`는 자신이 가진 2개의 네트워크 대역도 광고를 하고있다.

<br>

## 2. 테스트 시나리오

-  **회선 우선순위 설정**
`AS65001`의 주 회선을 정하고 백업 회선에 Prepend 설정으로 의도적으로 AS Path 경로를 늘려 주 회선이 **best path**가 되는지 확인. 


- **Prefix 별 우성순위 설정**
각 AS가 광고할 네트워크 2개가 각각 ROUTER_1과 ROUTER_2를 **best path**로 잡을 수 있도록 Prefix 설정 추가 후 확인

<br>

## 3. 테스트

### B/B_1 Config (AS65001)
```java
route-map MAP_Prepend_65004 permit 10
  set as-path prepend 65001 65001 65001

router bgp 65001
  router-id 10.255.0.1
  timers bgp 5 15
  address-family ipv4 unicast
    network 10.255.0.1/32
    network 20.1.0.0/30
    network 30.1.0.0/30
  neighbor 10.0.11.2
    remote-as 65003
    address-family ipv4 unicast
  neighbor 10.0.12.2
    remote-as 65004
    address-family ipv4 unicast
      route-map MAP_Prepend_65004 out
```

---


`AS65001`은 `AS65004`로 광고하는 경로에 **AS path**를 **Prepend**하여 의도적으로 길게 만들었다. 기존 **AS path**에 AS 번호 `65001`을 3번 더 추가하여, 실제로는 2개의 AS만 지나가지만 **AS path** 길이는 5로 인식된다. 예를 들어 `AS65002`는 `AS65001`가 광고하는 20.1.0.0/30으로 가기 위해서는 `65004 65001`의 **AS path**를 가졌지만 **Prepend**가 적용되면서 `65004 65001 65001 65001 65001`로 **AS path**가 설정 되었을 것이다.

### Best Path 확인 
```java
BB-2# show ip bgp 20.1.0.0
BGP routing table information for VRF default, address family IPv4 Unicast
BGP routing table entry for 20.1.0.0/30, version 476
Paths: (2 available, best #1)
Flags: (0x8008001a) (high32 00000000) on xmit-list, is in urib, is best urib rou
te, is in HW

  Advertised path-id 1
  Path type: external, path is valid, is best path, no labeled nexthop, in rib
  AS-Path: 65003 65001 , path sourced external to AS
    10.0.21.1 (metric 0) from 10.0.21.1 (10.255.0.3)
      Origin IGP, MED not set, localpref 100, weight 0

  Path type: external, path is valid, not best reason: AS Path, no labeled nexth
op
  AS-Path: 65004 65001 65001 65001 65001 , path sourced external to AS
    10.0.22.1 (metric 0) from 10.0.22.1 (10.255.0.4)
      Origin IGP, MED not set, localpref 100, weight 0

  Path-id 1 advertised to peers:
    10.0.22.1

-----------------------------------------------------------------------------------------

ROUTER-2#show ip bgp 20.1.0.0
BGP routing table information for VRF default
Router identifier 10.255.0.4, local AS number 65004
BGP routing table entry for 20.1.0.0/30
 Paths: 2 available
  65002 65003 65001
    10.0.22.2 from 10.0.22.2 (10.255.0.2)
      Origin IGP, metric 0, localpref 100, IGP metric 1, weight 0, received 01:45:33 ago, valid, external, best
      Rx SAFI: Unicast
  65001 65001 65001 65001
    10.0.12.1 from 10.0.12.1 (10.255.0.1)
      Origin IGP, metric 0, localpref 100, IGP metric 1, weight 0, received 05:55:38 ago, valid, external
      Rx SAFI: Unicast
```

---

위 출력에서 **localpref 100, weight 0**는 모두 기본값이다. 따라서 AS65002 입장에서 AS65001이 광고하는 20.1.0.0/30으로 가는 경로는 두 개중 AS Path가 더 짧은 10.0.21.1(AS65003 경유)이 **best path**로 선택되었다. AS-path 를 확인해보면 별도의 설정이 없는 `AS65003` 방향은 65003 65001로 설정이 되어있지만 **Prepend** 설정을 추가한 `AS65004` 방향은 `65004 65001 65001 65001 65001` 로 설정이 되어있다. <br>
`path is valid, not best reason: AS Path`로 출력이 되는 것으로 보아 경로는 유효하지만 **AS-Path** 우선순위에 밀려 **best path**로 선정되지 않은 것이다. 

또한 `AS65003`에서 직결된 라우터인 `AS65001`의 네트워크 20.1.0.0/30으로 가는 경로를 보면 직결임에도 **Prepend**
설정ㄴ 때문에 **AS path**가 `65001 65001 65001 65001`로 인식이 되고 있다. 이에 **AS Path**가 `65002 65003 65001`로 더 짧은 경로인 `AS65002`로 통신 하는 경로가 **best path**로 선정되있다. 
