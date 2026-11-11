## BGP 맺기 

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
