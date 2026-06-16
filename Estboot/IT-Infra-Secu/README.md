# 📚 이스트캠프 보안프로그래머 과정 — 학습 노트

> 복습할 때는 분류별로 들어가서 해당 파일만 열면 됨.

---

## 📁 디렉터리 구조

```
Eastboot/
├── 01_OSI-Network/
│   └── OSI-7Layer.md          # OSI 7계층, TCP/UDP, DNS, VLAN, ACL
│
├── 02_Security-Solutions/
│   └── Security-Solutions.md  # 방화벽, IDS/IPS, UTM, EDR, DLP, DRM, PMS 등
│
├── 03_Infrastructure/
│   └── IT-Infrastructure.md   # 데이터센터, 인터넷 회선, OS, 망분리, 망연계, 백업
│
├── 04_Architecture/
│   └── Architecture.md        # 아키텍처 종류, 설계 6단계, 네트워크 장비 순서
│
├── 05_Linux-System/
│   └── Linux-Commands.md      # 리눅스 명령어, SSH, Samba, DNS, 계정 관리
│
├── 06_Attack-Defense/
│   └── Attack-Defense.md      # 공격 기법 (SQL Injection, XSS 등), 방어 기법
│
└── 07_Project-FIMS/
    └── FIMS-Project.md        # 과정1 FIMS 프로젝트 전체 회고 + 삽질 모음
```

---

## 🔗 주제별 빠른 찾기

| 주제 | 파일 |
|---|---|
| OSI 7계층 전체 흐름 | 01_OSI-Network/OSI-7Layer.md |
| TCP 3-way Handshake | 01_OSI-Network/OSI-7Layer.md |
| VLAN vs 세그멘테이션 | 01_OSI-Network/OSI-7Layer.md |
| ARP 동작 원리 / 스푸핑 | 01_OSI-Network/OSI-7Layer.md |
| L4 세션테이블 / Sticky / VIP | 01_OSI-Network/OSI-7Layer.md |
| 레이어 포함 관계 (L4⊃L3⊃L2) | 01_OSI-Network/OSI-7Layer.md |
| 방화벽 종류 및 UFW | 02_Security-Solutions/Security-Solutions.md |
| 방화벽 세대별 발전 (1·2·3세대 메커니즘) | 02_Security-Solutions/Security-Solutions.md |
| Stateful Inspection / 세션 추적 | 02_Security-Solutions/Security-Solutions.md |
| TP모드 (브릿지 방화벽) | 02_Security-Solutions/Security-Solutions.md |
| IDS/IPS 차이 | 02_Security-Solutions/Security-Solutions.md |
| 인라인 모드 / 하드웨어 바이패스 / TAP | 02_Security-Solutions/Security-Solutions.md |
| Snort IDS→IPS 전환 (Block Offenders / Legacy vs Inline / 체크섬 오프로딩) | 02_Security-Solutions/Security-Solutions.md |
| IDS/IPS 룰 튜닝 (Suppress / Pass List / SID 네임스페이스 / 해시·헥스 시그니처) | 02_Security-Solutions/Security-Solutions.md |
| 센서 분산·TAP 심화 / 음영지역 / 무선 NAT | 02_Security-Solutions/Security-Solutions.md |
| 방화벽 룰 처리 순서 (Top-Down) | 02_Security-Solutions/Security-Solutions.md |
| Anti-APT / 정적·동적 분석(샌드박스 VM) / WAF / 관제(SIEM) | 02_Security-Solutions/Security-Solutions.md |
| 웹 프록시 (Squid/SquidGuard / 아웃바운드 URL통제 / SSL Bump / SWG) | 02_Security-Solutions/Security-Solutions.md |
| Snort 룰셋(ET Open) 한계 / IDS≠WAF / threshold | 02_Security-Solutions/Security-Solutions.md |
| 탐지 패러다임 (시그니처 vs 애노멀리) / Threshold | 02_Security-Solutions/Security-Solutions.md |
| Darktrace (AI 자가학습 이상탐지 / NDR) | 02_Security-Solutions/Security-Solutions.md |
| VPN (IPsec vs SSL-VPN / Site-to-Site / 위험 수용·공격표면) | 02_Security-Solutions/Security-Solutions.md |
| VPN 심화 (키교환·OTP/MFA·SSL VPN 인증서·입구 SSL Inspection) | 02_Security-Solutions/Security-Solutions.md |
| EDR | 02_Security-Solutions/Security-Solutions.md |
| 망분리 / 망연계 | 03_Infrastructure/IT-Infrastructure.md |
| 데이터센터 구조 | 03_Infrastructure/IT-Infrastructure.md |
| 백업 (RTO/RPO) / LTO·PTL 테이프 | 03_Infrastructure/IT-Infrastructure.md |
| 실습 랩 (ESXi·pfSense·홈 네트워크 / 진행 로그) | ../Lab/ |
| 아키텍처 설계 6단계 | 04_Architecture/Architecture.md |
| 네트워크 장비 순서 (F/W→L3→LB) | 04_Architecture/Architecture.md |
| 보안 장비 배치 전체 흐름 (DDoS→…→IDS) | 04_Architecture/Architecture.md |
| 메일 서버 구성도 / DLP 복제라인 / 바이패스 이중화 | 04_Architecture/Architecture.md |
| 방화벽 이중화+탭/IDS "고민의 흔적" (2차 프로젝트 설계철학) | 04_Architecture/Architecture.md |
| 고가용성 HA (Active-Standby/Active-Active, 본딩·티밍) | 04_Architecture/Architecture.md |
| 방화벽 디자인 (구간별 A/B/C/D 코스 정책) | 04_Architecture/Architecture.md |
| NAT 설계 (DNAT/SNAT, LVS/ALB 논리 배치) | 04_Architecture/Architecture.md |
| 아키텍처 최적화 원칙 | 04_Architecture/Architecture.md |
| 리눅스 기본 명령어 | 05_Linux-System/Linux-Commands.md |
| SSH / Samba 실습 | 05_Linux-System/Linux-Commands.md |
| 웹 공격 기법 | 06_Attack-Defense/Attack-Defense.md |
| 브루트포스 / 크리덴셜 스터핑 | 06_Attack-Defense/Attack-Defense.md |
| RAT / 크랙 악성코드 / 키로깅 | 06_Attack-Defense/Attack-Defense.md |
| DDoS / 시그니처 우회 | 06_Attack-Defense/Attack-Defense.md |
| FIMS 전체 아키텍처 | 07_Project-FIMS/FIMS-Project.md |
| WireGuard / socat 삽질 | 07_Project-FIMS/FIMS-Project.md |

---

## ✅ 면접에서 주의할 표현

```
틀린 표현: "VLAN으로 망분리 했습니다"
맞는 표현: "VirtualBox 내부 네트워크로 세그멘테이션 했습니다.
            실제 VLAN은 물리 스위치가 필요하고 저희는 소프트웨어 레벨 격리였습니다."

틀린 표현: "HA구간에서 동작합니다"
맞는 표현: HA는 레이어 이름이 아니라 "고가용성(High Availability)" 설계 개념
```

---

## ❓ 추가 보충 필요한 부분

- ThroughPut 정확한 개념 (수업 중 잘 못 들음)
- PORT SECURITY 실습 방법
- VLAN ACL 실제 설정 (2부 UTM 활용편에서 다룰 예정)
- GNS3 환경 실습
