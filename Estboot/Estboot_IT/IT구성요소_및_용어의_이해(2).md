# 📚 인프라 & 시스템 기초

## 📝 서버 & 스토리지

- **RAID(Redundant Array of Independent Disks)**: 여러 하드디스크를 묶어 성능 향상 및 데이터 이중화(복제) 구현
  - RAID 0: 스트라이핑(성능↑, 안정성↓) / RAID 1: 미러링(복제) / RAID 5: 패리티 기반 분산 저장
- **스토리지**: 서버와 분리된 외장 저장 장치. NAS(네트워크), SAN(전용망) 형태로 구분

---

## 📝 데이터 센터 (IDC, Internet Data Center)

### 🔧 물리적 구조

```
인터넷 회선 → 방화벽(Firewall) → 백본 네트워크 → L3/L2 스위치 → 서버
```

> 위 구조는 **3계층(3-Tier) 네트워크 구조** (Core - Distribution - Access)의 전형적인 형태.  
> 각 계층이 역할을 분리함으로써 트래픽 제어, 보안, 확장성을 확보함.

### 💻 논리적 구조 (OS)

#### Windows
- 비즈니스용(Server)과 일반 사용자용(Client) OS 분리 운영
- 보안 패치, 버그 수정, 성능 업데이트를 시스템 차원에서 지원
- 내부 소스 코드 비공개 → **Closed Source (상용 라이선스)**
- 라이선스 지원 기간 존재 → **EOL(End of Life)** 이후 보안 업데이트 중단

> 💡 일반 사용자 친화적으로 설계되었으나, 내부 구조가 비공개여서 개발·보안 분석 환경에서는 제약이 많음

#### Linux
- **GNU(GNU's Not Unix)** 철학 기반 → 자유 소프트웨어 운동에서 출발
- 소스 코드 전체 공개 → **Open Source**
- 배포판(Distro)에 따라 다양한 버전 존재 (Ubuntu, CentOS, Kali 등)
- 대부분의 서버·개발·보안 환경의 표준 OS
- 특히 **Embedded Systems**(임베디드 장비)에서 광범위하게 사용
- 개발·관리·보안 설정을 사용자가 직접 수행

> 💡 보안 실습 환경(LAB)으로 최적. Kali Linux 등 수많은 오픈소스 보안 도구 활용 가능

#### Unix
- 높은 안정성과 신뢰성으로 금융·은행권에서 오랜 기간 사용
- Unix 계열마다 지원 CPU가 다름
  - Solaris → SPARC / AIX → POWER / **HP-UX → IA-64(Itanium)**
- 상용 라이선스 + EOL 지원 기간 존재

> 💡 최근 **U to L(Unix to Linux)** 전환 추세 → 한국거래소 등 주요 기관도 Linux로 이전.  
> 비용, 유연성, 오픈소스 생태계 측면에서 Unix의 경쟁력 하락 중

---

## 📝 소프트웨어 계층

### 웹 서비스 구조

```
사용자(Client)
    ↕ HTTP Request/Response
Web Server (정적)     ← Apache, Nginx, IIS
    ↕
WAS (동적)            ← Tomcat, JBoss, WebLogic
    ↕
DB Server             ← MySQL, Oracle, PostgreSQL
```

#### Web Server (정적 영역)
- HTML, CSS, 이미지 등 정적 콘텐츠를 저장·제공
- 대표 솔루션: **Apache**, **Nginx**, **IIS(Microsoft)**

#### WAS - Web Application Server (동적 영역)
- 비즈니스 로직 처리 + DB 연동 → 동적 결과물 생성 및 응답
- 대표 솔루션: Tomcat, JBoss, WebLogic

#### 그룹웨어 (Groupware)
- 기업·조직 내 협업을 위한 통합 소프트웨어 (이메일, 결재, 메신저 등)
- 사내 계정·권한 정보가 집중되어 있어 **공격자(침투 테스터, 레드팀)의 핵심 타깃**

---

## 📝 가상화 (Virtualization)

### Hypervisor
하나의 물리 머신 위에서 여러 OS를 동시에 실행할 수 있도록 자원을 분리·할당하는 기술

| 구분 | 설명 | 예시 |
|---|---|---|
| **Type 1 (Bare-metal)** | 하드웨어 위에 직접 설치. 성능↑, 보안↑ | VMware ESXi, Hyper-V, KVM |
| **Type 2 (Hosted)** | 기존 OS 위에 설치. 설치 편의성↑, 오버헤드↑ | VirtualBox, VMware Workstation |

### HCI (Hyper-Converged Infrastructure)
- Hypervisor로 가상화된 여러 서버를 **클러스터(Cluster)** 기술로 묶어 하나의 통합 인프라처럼 운영하는 방식
- 스토리지·네트워크·컴퓨팅을 소프트웨어 정의 방식으로 통합 관리

### Container (컨테이너)
- VM과 달리 **OS 커널을 공유**하면서 프로세스 단위로 격리하는 경량 가상화 기술
- 대표 기술: **Docker**, **Kubernetes(K8s)**

| | VM (가상머신) | Container |
|---|---|---|
| 격리 단위 | OS 전체 | 프로세스 |
| 부팅 시간 | 분 단위 | 초 단위 |
| 용량 | 수 GB | 수십~수백 MB |
| 보안 격리 수준 | 높음 | 상대적으로 낮음 |

> 💡 현대 클라우드·DevOps 환경에서는 VM + Container를 함께 사용하는 것이 일반적

---

## 💭 오늘의 회고

### 배운 점
- 데이터센터 하드웨어 구성을 넘어 소프트웨어 계층의 구성 이해
- 내가 사용하던 OS들이 어떤 성질과 철학으로 설계되었는지 파악
- 광케이블 선 정리 포함, 네트워크 아키텍처는 처음 설계가 매우 중요함

### 어려운 점 / 개선할 점
- 네트워크 흐름과 소프트웨어 계층이 실제로 어떻게 연동되는지 더 깊이 이해 필요
- 이해한 것이 전부라는 생각 배제 → 추가 자료 찾아 검증하는 습관

### 액션 플랜
- [ ] 가상화 심화: Hypervisor Type 1 vs 2 실습 (VirtualBox로 직접 구성)
- [ ] OS 내부 차이 탐구: Linux 부팅 과정(커널 → init → systemd)
- [ ] Docker 기초 개념 및 실습 환경 구성

---

## 📖 IT 용어 정리

| 용어 | 설명 |
|---|---|
| **RAID** | Redundant Array of Independent Disks. 디스크 묶음을 통한 성능·안정성 확보 |
| **망분리** | 인터넷망과 내부(업무)망을 물리적 또는 논리적으로 분리하는 보안 정책 |
| **RAT** | Remote Access Trojan. 원격 접속 기능을 가진 악성코드 |
| **Cool / Hot Zone** | 서버 전면부(흡기, Cool) / 후면부(배기, Hot). 데이터센터 냉각 설계 기준 |
| **TPS** | 기계실(Technical Power Supply Room) / 또는 Transaction Per Second (맥락에 따라 다름) |
| **DBMS** | DataBase Management System. 데이터베이스를 관리하는 소프트웨어 (MySQL, Oracle 등) |
| **SQL** | Structured Query Language. 데이터베이스에서 데이터를 조회·삽입·수정·삭제하기 위한 **질의 언어** |
| **Dump** | 프로그램 실행 중 메모리·로그 등의 데이터를 파일로 출력하는 행위 |
| **X Window System** | Unix/Linux 계열에서 **GUI 환경을 제공하는 디스플레이 프레임워크**. OS 자체가 아님 |
| **리버스 엔지니어링** | 완성된 제품·프로그램을 분석해 내부 구조를 파악하는 역설계 기법 |
| **R&D** | Research & Development. 연구 및 개발 |
| **GNU** | *GNU's Not Unix*의 재귀 약자. 리처드 스톨먼이 시작한 자유 소프트웨어 프로젝트. Linux 커널과 결합해 GNU/Linux가 됨 |
| **Embedded System** | 특정 기능 수행을 위해 기기 내부에 내장된 전용 컴퓨터 시스템 |
| **U to L** | Unix에서 Linux 체제로 전환하는 마이그레이션 작업 |
| **EOL** | End of Life. 소프트웨어·OS의 공식 지원 종료 시점 |
| **HCI** | Hyper-Converged Infrastructure. 컴퓨팅·스토리지·네트워크를 소프트웨어로 통합한 인프라 |
| **Cluster** | 여러 서버를 하나의 시스템처럼 묶어 고가용성·성능을 확보하는 구성 |
