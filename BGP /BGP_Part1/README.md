## BGP 맺고 통신하기

**목표**
1. 이웃 간 BGP 세션을 맺고 루프백을 광고하여 AS65003 또는 AS65004를 통해서 AS65001과 AS65002의 루프백끼리 통신.
2. AS65003을 우선순위로 설정하여 통신하도록 정책 설정
3. AS65002의 루프백은 AS65003 방향으로 AS65002의 내부 라우터(20.2.0.0/30)은 AS65004 방향으로 통신하도록정책 설정

**네트워크 구성도**
<img width="926" height="521" alt="image" src="https://github.com/user-attachments/assets/e6ff8264-2512-4e79-aa00-66dbf010d57a" />


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


