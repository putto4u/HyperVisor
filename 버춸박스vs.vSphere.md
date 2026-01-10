# VMware vSphere 네트워크 구성 및 VirtualBox 비교 분석

데스크톱 가상화 도구인 VirtualBox와 달리, 엔터프라이즈 베어메탈 하이퍼바이저인 **VMware vSphere (ESXi)**는 네트워크 구성 방식이 근본적으로 다릅니다. vSphere는 **vSwitch(가상 스위치)**와 **Port Group(포트 그룹)**, **Uplink(업링크)**의 조합으로 네트워크를 정의하며, NAT 기능을 하이퍼바이저 레벨에서 기본 제공하지 않는 것이 가장 큰 특징입니다.

---

### 1. VirtualBox vs vSphere 용어 및 기능 매핑

vSphere는 기본적으로 데이터센터의 물리적 스위치와 직접 연결되는 구조를 지향하므로, VirtualBox의 'NAT' 개념이 ESXi 표준 스위치에는 존재하지 않습니다.

| VirtualBox 모드 | vSphere (ESXi) 대응 개념 | 동작 방식 차이점 |
| --- | --- | --- |
| **Bridged (어댑터에 브리지)** | **VM Network (Standard Switch + Uplink)** | **vSphere의 표준.** 가상 머신이 물리 스위치에 직접 연결된 것처럼 동작합니다. IP는 사내 물리 공유기(Router)나 DHCP 서버에서 받아옵니다. |
| **NAT / NAT Network** | **지원 안 함 (기본)**<br>

<br>*(NSX 또는 라우터 VM 필요)* | ESXi 자체는 NAT를 수행하지 않습니다. 'Bridged'로 설정하고 상단의 **물리 방화벽/공유기**가 NAT를 수행하도록 구성합니다. |
| **Host-Only (호스트 전용)** | **Private vSwitch (No Uplink)** | **업링크(물리 랜선)가 연결되지 않은 vSwitch**를 생성하면, 해당 스위치 내부의 VM끼리만 통신 가능한 고립된 네트워크가 됩니다. |

---

### 2. vSphere 네트워크 핵심 구성 요소

vSphere 네트워킹을 이해하기 위해서는 다음 세 가지 요소의 관계를 파악해야 합니다.

#### A. vSwitch (Virtual Switch)

물리적 L2 스위치를 소프트웨어로 구현한 것입니다. VM들이 랜선을 꽂는 가상의 장비입니다.

* **Standard vSwitch (vSS):** 호스트(서버) 한 대마다 개별적으로 설정하는 스위치.
* **Distributed vSwitch (vDS):** vCenter가 관리하며 여러 호스트에 걸쳐 통합 설정되는 스위치.

#### B. Uplink (Physical NIC, vmnic)

가상 스위치와 실제 물리적 랜카드(NIC)를 연결하는 통로입니다.

* **Uplink 연결 O:** 외부 통신 가능 (인터넷, 사내망). VirtualBox의 **Bridged**와 유사.
* **Uplink 연결 X:** 외부 통신 불가, 내부 전용. VirtualBox의 **Host-Only**와 유사.

#### C. Port Group (포트 그룹)

가상 스위치 내에서 정책(VLAN ID, 보안 설정, 트래픽 쉐이핑)을 공유하는 포트들의 묶음입니다.

* 관리자는 "VM Network", "Management Network" 같은 라벨(이름)을 붙여 관리합니다.
* VM 생성 시 이 **Port Group의 이름**을 선택하여 네트워크를 연결합니다.

---

### 3. vSphere에서의 네트워크 격리 구현 (Host-Only 흉내내기)

vSphere에서 VirtualBox의 **호스트 전용(Host-Only)**과 같은 폐쇄망을 구축하려면 다음과 같이 설정합니다.

1. **새 vSwitch 생성:** ESXi 웹 클라이언트에서 `vSwitch1`을 새로 만듭니다.
2. **업링크 제거:** 스위치 생성 단계에서 **물리적 어댑터(vmnic)를 하나도 할당하지 않습니다.**
3. **포트 그룹 생성:** `Internal-Lab-Network`와 같은 이름으로 포트 그룹을 만들고 `vSwitch1`에 할당합니다.
4. **VM 연결:** VM의 네트워크 어댑터를 `Internal-Lab-Network`로 변경합니다.
* **결과:** 이 VM들은 외부 인터넷과 완전히 단절되며, 같은 vSwitch1에 연결된 VM끼리만 통신할 수 있습니다.



---

### 4. 요약: vSphere 네트워크 설계의 차이점

* **VirtualBox:** 사용자 편의를 위해 하이퍼바이저가 **가상 공유기(NAT)** 역할을 직접 수행해 줍니다.
* **vSphere:** 하이퍼바이저는 단순히 **L2 스위치** 역할만 수행합니다. 라우팅, NAT, DHCP 기능은 외부의 물리적 장비(공유기, L3 스위치)나 별도의 라우터 VM(예: pfSense)에 위임합니다.

Next Step: vSphere 환경에서 가상 스위치(vSwitch) 생성 및 포트 그룹 설정 실습 절차
