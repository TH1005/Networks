# 4.1 인바운드 경로 제어 (AS-Path Prepend & MED)

BGP는 같은 목적지에 대해 여러 경로를 광고받으면, 정해진 순서의 속성 비교를 통해 **best path** 하나를 선정한다. 
내가 이웃에게 아무 정책 없이 광고를 보내면 상대는 기본 규칙으로 경로를 고르므로, 나에게 들어오는 트래픽이 내가 의도한 경로로 오지 않을 수 있다.

이를 제어하기 위해 이웃에게 광고를 보낼 때(out) 해당 광고의 속성을 조작하여, 상대가 내 광고를 보고 선택하는 경로를 유도할 수 있다. 
route-map에서 **AS-Path Prepend**나 **MED** 값을 설정한 뒤 이웃에 적용하면 그 이웃에게 보내는 모든 광고에 적용되고, prefix-list로 대상을 지정하면 매칭되는 prefix의 광고에만 적용할 수 있다. 


# 1. Prepend 테스트
Prepend는 이웃에게 라우팅 광고를 할 때 의도적으로 AS Path의 길이를 늘려서 광고하여, 특정 경로의 우선 순위를 낮춰서 인바운드 트래픽을 제어하는 정책이다.

<br>

## 1.1. 구성도 
<img width="914" height="513" alt="image" src="https://github.com/user-attachments/assets/7b63f32d-953b-45d5-8b24-0efb49ec7568" />
4개의 외부 라우터가 있으며 각각의 AS를 가지고 인접 라우터와 피어링을 맺은 상태, `AS65001` 과 `AS65002` 는 내부 라우터 2개의 대역도 광고를 하고있다.

<br>

## 1.2. 테스트 시나리오

-  **회선 우선순위 설정**
AS65001과 AS65002의 주 회선을 정하고 백업 회선에 Prepend 설정으로 의도적으로 AS Path 경로를 늘려 주 회선이 **best pat**h가 되는지 확인. 


- **Prefix 별 우성순위 설정**
내부 라우터 2개가 각각 ROUTER_1과 ROUTER_2를 **best path**로 잡을 수 있도록 Prefix 설정 추가 후 확인

<br>

## 1.3. 테스트

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
```

---

### B/B_2 Config (AS65002)
```java
route-map MAP_Preend_65004 permit 10
  set as-path prepend 65002 65002 65002 
  
router bgp 65002
  router-id 10.255.0.2
  timers bgp 5 15
  address-family ipv4 unicast
    network 10.255.0.2/32
    network 20.2.0.0/30
    network 30.2.0.0/30
  neighbor 10.0.21.1
    remote-as 65003
    address-family ipv4 unicast
  neighbor 10.0.22.1
    remote-as 65004
    address-family ipv4 unicast
      route-map MAP_Preend_65004 out
```

`AS65001`와 `AS65002`는 각 `AS65004` 방향의 **AS path**를 의도적으로 늘려놓은 상태이다. 기존 **AS path** 에서 3홉을 더 늘려놨기 때문에 기존 경로에 AS path가 3개 더 추가 될 것이다. 예를들어 `AS65001`은 `AS65002`의 내부 라우터인 20.2.0.0/30으로 가기위해서는 65004 65002의 **AS path**를 가졌지만 **Prepend**가 적용되면서 65004 65002 65002 65002 65002로 **AS path**가 설정 되었을 것이다.




