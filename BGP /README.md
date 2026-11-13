# 🌐 BGP (Border Gateway Protocol)

## 📌 BGP란?

BGP는 **서로 다른 AS(Autonomous System)에 속한 라우터 간에 라우팅 정보를 교환하는 외부 게이트웨이 프로토콜(EGP)** 이다.

> 💡 **AS(Autonomous System)**
> 전 세계적으로 거대한 인터넷을 구성하는 하나의 독립된 네트워크 단위

### 📌 BGP를 사용하는 이유

| 이유 | 설명 |
|---|---|
| 📈 **확장성** | 수십만 개에 달하는 전 세계 인터넷 경로를 OSPF 같은 내부 라우팅 프로토콜(IGP)로 처리하기에는 한계가 있음 |
| 🎛️ **정책 기반 제어** | 통신사·기업 간의 **비용, 보안 정책**에 따라 경로를 유연하게 제어(Policy-Based Routing)해야 함 |

---

## 📌 BGP 동작 방식

- 🔌 **TCP 179번 포트**로 Peer 간 세션을 맺음
- 💓 주기적으로 **Keepalive**를 보내 세션 상태를 확인
- 🚨 세션이 끊기면 다른 Peer들에게도 **Update 패킷**을 전송해 최신 경로로 갱신

---

## 📌 경로벡터(Path Vector) 알고리즘

BGP는 **거리벡터(Distance Vector) 알고리즘을 뼈대**로 하는 **경로벡터(Path Vector) 알고리즘**을 사용한다.

### 📌 전체 경로 목록을 들고 다니는 이유: 라우팅 루프 방지

A, B, C 라우터가 있고 A가 **X 네트워크**를 광고한다고 가정하면,

1. 🅰️ → 🅱️ : X로 가려면 **A**를 거쳐가라고 광고
2. 🅱️ → 🅲 : X로 가려면 **B**를 거쳐가라고 광고
3. 🅲 → 🅰️ : X로 가려면 **C**를 거쳐가라고 광고

이 상태에서 A와 X 네트워크 간 **회선 장애나 메트릭 꼬임**이 발생하면,

```
C → B → A → C → B → A → ...  (TTL이 0이 될 때까지 무한 루프 🔄)
```

### 📌 BGP의 해결책

> 광고 패킷을 받았을 때 **AS Path에 내 AS 번호가 이미 있다면 그대로 Drop**
> → 루프가 원천 차단된다 ✅

## 📌 BGP는 "정책"으로 움직인다

| 구분 | 선택 기준 |
|---|---|
| 🏃 **IGP (내부)** | 가장 **가깝고 빠른** 경로 |
| 🏢 **BGP (외부)** | **관리자의 정책** (비즈니스 계약, 회선 비용, 보안) |

> **예시**
> A 통신사의 1번 회선이 물리적으로 1홉으로 짧지만 비용이 2배라면,
> 홉 수가 더 많더라도 **저렴한 2번 회선**을 선택해 라우팅할 수 있다.

BGP는 이러한 정책을 반영하기 위해 **인/아웃바운드 트래픽에 대한 경로 속성**을 활용한다.

---

## 📌 BGP Best Path 선택 순서 (전체)

> 💡 사전 조건: **Next-Hop에 도달 가능**해야 후보가 된다

| 순서 | 기준 | 선택 방향 |
|:---:|---|---|
| 1 | **Weight** (Cisco 전용) | 높은 값 |
| 2 | **Local Preference** | 높은 값 |
| 3 | **로컬에서 생성한 경로** (network, redistribute, aggregate) | 로컬 생성 우선 |
| 4 | **AS-Path 길이** | 짧은 경로 |
| 5 | **Origin** | IGP(i) < EGP(e) < Incomplete(?) |
| 6 | **MED** | 낮은 값 |
| 7 | **eBGP vs iBGP** | eBGP 우선 |
| 8 | **Next-Hop까지의 IGP Metric** | 낮은 값 |
| 9 | **가장 오래된 eBGP 경로** | 안정성 우선 |
| 10 | **Router ID** | 낮은 값 |
| 11 | **Cluster List 길이** | 짧은 경로 (Route Reflector 환경) |
| 12 | **Neighbor IP 주소** | 낮은 값 |


---

## 📌 Best Path 선택 순서 요약

| 순서 | 속성 | 적용 범위 | 우선 조건 | 주 용도 |
|:---:|---|---|---|---|
| 1 | **Weight** (Cisco 전용) | 🖥️ 해당 라우터만 | 값 높을수록 | 아웃바운드 제어 |
| 2 | **Local Preference** | 🏠 같은 AS 내부 | 값 높을수록 | 아웃바운드 제어 |
| 3 | **로컬 생성 경로** | 🖥️ 해당 라우터 | 로컬 생성 우선 | network / redistribute / aggregate로 만든 경로 우선 |
| 4 | **AS-Path 길이** | 🌍 외부 AS로 광고 | 경로 짧을수록 | 인바운드 제어 (Prepending) |
| 5 | **Origin** | 🌍 경로 생성 출처 | IGP(i) < EGP(e) < Incomplete(?) | 경로 출처 신뢰도 비교 |
| 6 | **MED** | 🤝 인접 AS | 값 낮을수록 | 인바운드 제어 |
| 7 | **eBGP vs iBGP** | 🔀 Peer 종류 | eBGP 우선 | AS 외부로 빠지는 경로 우선 |
| 8 | **Next-Hop IGP Metric** | 🏠 같은 AS 내부 | 값 낮을수록 | 가까운 출구로 내보내기 (Hot Potato) |
| 9 | **가장 오래된 eBGP 경로** | 🤝 eBGP Peer | 오래된 경로 우선 | 경로 안정성 (플래핑 방지) |
| 10 | **Router ID** | 🖥️ Peer 라우터 | 값 낮을수록 | Tie-breaker |
| 11 | **Cluster List 길이** | 🏠 Route Reflector 환경 | 짧을수록 | Tie-breaker (RR 루프 방지와 연관) |
| 12 | **Neighbor IP 주소** | 🖥️ Peer 라우터 | 값 낮을수록 | 최종 Tie-breaker |

---

### 📌 참고

- AS-Path Prepending은 속성 자체가 아니라 **AS-Path 길이를 늘리는 기법**이다. 실제 경로 선택 시에는 AS-Path 길이가 짧은 쪽이 우선된다.
- 위 순서(Weight → Local Pref → AS-Path → MED)는 Best Path 선택 순서와 대체로 일치하지만, 실제로는 사이에 `Origin`, 로컬 생성 경로 우선 등 다른 비교 단계도 존재한다.



## 📌 BGP 메시지 4종류

| 메시지 | 역할 |
|---|---|
| 🤝 **Open** | TCP 연결 후 세션 수립을 위해 가장 먼저 교환. AS 번호, Hold Time, Router ID, 지원 기능(Capability) 협상 |
| 📢 **Update** | 경로 광고(Advertise)와 경로 철회(Withdraw)를 전달. 경로 속성도 여기에 실려 감 |
| 💓 **Keepalive** | 세션 유지 확인용. 기본 60초 주기(Hold Time 180초, Cisco 기준) |
| ❌ **Notification** | 오류 발생 시 전송. 전송 후 세션이 즉시 종료됨 |

- Hold Time 동안 Keepalive/Update가 오지 않으면 Peer가 죽은 것으로 판단하고 세션을 내린다
- Open 메시지의 Hold Time은 양쪽 값 중 **작은 값**으로 협상된다

---

## 📌 BGP Peer 상태 (FSM)

```
Idle → Connect → Active → OpenSent → OpenConfirm → Established
```

| 상태 | 설명 |
|---|---|
| 💤 **Idle** | 초기 상태. 연결 시도 전이거나 오류 후 대기 중 |
| 🔌 **Connect** | TCP 3-way handshake 완료를 기다리는 중 |
| 🔁 **Active** | TCP 연결 재시도 중. 연결이 **실패**하고 있다는 의미 |
| 📤 **OpenSent** | TCP 연결 완료, Open 메시지를 보내고 상대 Open을 기다리는 중 |
| 📥 **OpenConfirm** | 상대 Open을 받았고 Keepalive를 기다리는 중 |
| ✅ **Established** | 세션 수립 완료. 이때부터 Update 교환 |

### 📌 트러블슈팅 포인트

- **Active에서 멈춤** (TCP 연결 자체가 안 됨)
  - Neighbor IP 설정 오류
  - Peer까지 라우팅 경로 없음
  - ACL/방화벽에서 **TCP 179** 차단
- **Idle에서 멈춤**
  - 설정 오류, 반복 실패로 인한 대기, 또는 neighbor가 shutdown 상태
- **OpenSent ↔ Idle 반복**
  - Open 메시지 내용 불일치 (**AS 번호 오류**, 인증 불일치 등)
- `show ip bgp summary`에서 State/PfxRcd 항목 확인
  - 숫자가 보이면 Established, `Active`/`Idle` 문자가 보이면 세션 미수립

---

## 📌 Next-Hop 속성

- 경로 광고 시 **"이 경로로 가려면 이 IP로 보내라"** 를 알려주는 속성
- **eBGP**로 광고할 때: Next-Hop이 **자기 자신의 IP**로 바뀜
- **iBGP**로 광고할 때: eBGP에서 받은 경로의 Next-Hop이 **바뀌지 않고 그대로 전달**됨

### 📌 문제 상황

```
[외부 AS] --eBGP-- [R1] --iBGP-- [R2]
  Next-Hop: 외부 AS의 IP (R1은 알지만 R2는 모를 수 있음)
```

- R2가 외부 AS의 Next-Hop IP까지 가는 경로를 모르면 해당 BGP 경로는 **사용 불가(Inaccessible)** 상태가 된다
- 테이블에 경로는 있는데 `>` (best) 표시가 안 붙는 경우 이 문제를 의심한다

### 🔧 해결책

- R1에서 iBGP Peer에게 **`next-hop-self`** 설정
  → iBGP로 광고할 때 Next-Hop을 R1 자신의 IP로 바꿔서 전달

```
router bgp 65001
 neighbor 10.0.0.2 next-hop-self
```

- 또는 외부 연결 구간 네트워크를 IGP에 광고하는 방법도 있지만, 보통 `next-hop-self`를 쓴다

---

## 🏷️ Community 속성

- 경로에 **태그(꼬리표)** 를 붙이는 속성. 형식은 `AS번호:값` (예: `65001:100`)
- 태그를 기준으로 Route-map에서 Local Pref, MED, 필터링 등 정책을 일괄 적용할 수 있다
- 경로가 많을 때 Prefix를 하나하나 지정하지 않고 **그룹 단위로 정책 관리**가 가능하다
- Cisco에서는 기본적으로 Peer에게 전달되지 않으므로 **`send-community`** 설정이 필요하다

### Well-known Community

| 값 | 의미 |
|---|---|
| `no-export` | eBGP Peer(다른 AS)에게 광고하지 않음 |
| `no-advertise` | 어떤 Peer에게도 광고하지 않음 |
| `local-AS` | 로컬 AS 밖으로(Confederation 포함) 광고하지 않음 |
| `internet` | 제한 없이 광고 |

> 💡 ISP들이 "이 Community 붙이면 Local Pref 올려줌", "이 Community 붙이면 Prepend 해줌" 같은 방식으로 고객에게 정책 제어권을 열어주기도 한다

---

## 🚧 Route Filtering

- 어떤 경로를 **받을지(In) / 보낼지(Out)** 걸러내는 것
- BGP는 정책 기반이므로 필터링이 사실상 **필수**이다

| 도구 | 용도 |
|---|---|
| **Prefix-list** | IP 대역 + 서브넷 길이 기준 필터링 (가장 많이 사용) |
| **AS-Path ACL** | AS-Path 정규표현식 기준 필터링 |
| **Route-map** | 조건(match)에 따라 속성 변경(set) 또는 허용/거부. 가장 강력함 |
| **Distribute-list** | ACL 기반 필터링 (구형 방식) |

### 📌 예시

```
ip prefix-list ALLOW-ONLY seq 10 permit 203.0.113.0/24

route-map FILTER-OUT permit 10
 match ip address prefix-list ALLOW-ONLY
 set local-preference 200

router bgp 65001
 neighbor 192.0.2.1 route-map FILTER-OUT out
```

### ⚠️ 필터링을 안 하면?

- 고객/Peer가 잘못 광고한 경로를 그대로 받아서 다른 곳에 퍼뜨릴 수 있다 (**Route Leak**)
- 사설 IP 대역, 기본 경로(0.0.0.0/0) 등이 의도치 않게 전파될 수 있다
- 흔히 쓰는 안전장치: **`maximum-prefix`** 로 Peer가 보내는 경로 수 제한

---

## 🔐 BGP 보안 이슈

BGP는 설계 당시 **광고된 경로를 그대로 신뢰**하는 구조라 보안에 취약하다.

### 🏴‍☠️ BGP Hijacking

- 다른 AS가 소유한 IP 대역을 **자기 것인 것처럼 광고**해서 트래픽을 가로채는 공격이다
- 유형
  - **Origin 위조**: 남의 Prefix를 자기 AS 시작으로 광고
  - **More Specific 광고**: 원래 `/22`인 대역을 `/24`로 쪼개 광고 → **Longest Prefix Match** 때문에 더 구체적인 경로가 우선됨
- 사례: 2008년 파키스탄 ISP가 YouTube 대역을 잘못 광고해 전 세계 YouTube 접속이 장애를 겪음

### 🛡️ 대응 방법

| 방법 | 설명 |
|---|---|
| **Prefix Filtering** | 고객/Peer가 광고할 수 있는 Prefix만 허용 |
| **RPKI / ROA** | Prefix의 정당한 소유 AS를 암호학적으로 검증 (Route Origin Validation) |
| **MD5 / TCP-AO 인증** | Peer 세션 자체를 인증해 세션 탈취 방지 |
| **maximum-prefix** | 비정상적으로 많은 경로 유입 차단 |
| **GTSM (TTL Security)** | 직접 연결된 Peer에서 온 패킷만 허용 (TTL 255 검사) |
