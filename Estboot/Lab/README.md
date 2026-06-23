# 🧪 실습 랩 (과정2) — ESXi · pfSense 가상 인프라

이론(`../IT-Infra-Secu/`)에서 배운 보안 장비 배치·탭·DLP·가상화를 **내 PC 위에 직접 세워보는** 실습 기록.
이 디렉터리는 **실습이 진행되면서 계속 업데이트**된다 (환경이 바뀌면 아래 "현재 환경 상태"를 갱신하고, 그날 한 일은 날짜별 로그로 추가).

---

## 📁 구성

```
Lab/
├── README.md                  # 이 파일 — 랩 개요 + 현재 환경 상태(살아있는 문서) + 진행 로그 인덱스
├── 2026-06-09-환경구축.md      # 6/9 — Workstation Pro·ESXi 설치, 목표 랩 구성, 홈 네트워크 설계
├── 2026-06-10-pfSense구축.md   # 6/10 — pfSense WAN/LAN 분리, 대역충돌 해결, xubuntu LAN 검증
├── 2026-06-11-VLAN구축.md      # 6/11 — VLAN으로 Web/Web_Dev 분리(ESXi VGT 4095 트렁크·pfSense VLAN 라우팅)
├── 2026-06-12-삼바·DNS·NAT.md  # 6/12 — 삼바 파일서버(VLAN30)·내부DNS(VLAN40)·NAT 포트포워딩·Alias
├── 2026-06-15-웹프록시·Snort.md # 6/15 — Squid/SquidGuard 아웃바운드 URL 통제 + Snort IDS 탐지실험·SSL Bump 보류
├── 2026-06-16-Snort-IPS전환·Kali·VPN.md # 6/16 — Snort IDS→IPS 인라인 전환·룰 튜닝(Suppress/Pass) + Kali 공격기 투입 + VPN 흐름
├── 2026-06-17-OpenVPN·SecurityOnion.md  # 6/17 — pfSense OpenVPN 구축 + Security Onion 미러 센서 도입
├── 2026-06-18-SecurityOnion-룰추가.md   # 6/18 — Security Onion Detections 커스텀 룰 추가·동기화(~15분)·PCAP 확인 (이론: NGIPS 요건·룰 옵션 분석)
├── 2026-06-19-Kali포트스캔·Hydra·패킷분석.md # 6/19 — Kali nmap 포트스캔·Hydra 사전공격(SSH)·IPS 인라인+전체룰에 nmap→알람폭주→ESM 필요성·와이어샤크 IP헤더
└── 2026-06-22-와이어샤크-전송계층·nmap스캔비교.md # 6/22 — 와이어샤크로 L4 관찰(ping -l로 단편화·TCP 3way/헤더/윈도우·UDP+ICMP unreachable) + nmap -sT vs -sS(스텔스) 대조
```

---

## 🖥️ 현재 환경 상태 (최신: 2026-06-22)

### 하드웨어 / 소프트웨어 스택

```
하드웨어: PC 1대 (서브컴 = LAB 전용 머신)

소프트웨어 (위 → 아래로 쌓임)
 ├ VMware Workstation Pro   ← Broadcom 배포본          [설치 완료]
 ├ ESXi 8.0                 ← Workstation 위 타입1 HV   [설치 완료]
 ├ pfSense (UTM)            ← 가상 방화벽(FW)            [구성 완료] WAN/LAN + VLAN 10~40 라우팅 + NAT 포워딩 + Alias
 │  ├ Squid + SquidGuard    ← 포워드/캐싱 프록시 + URL필터 [구성] 투명프록시·HTTP 차단 / SSL Bump 보류
 │  ├ Snort (IPS·인라인)    ← ET Open 룰셋              [전환] 6/16 IDS→IPS(Block Offenders·Inline) + Suppress/Pass 튜닝
 │  └ OpenVPN (원격접속)     ← 내장 패키지               [구성] 6/17 CA·서버인증서·터널 /24·redirect GW·client-export
 ├ Kali Linux (공격기)      ← 본컴 VirtualBox·WAN쪽       [구성] 6/16 192.168.1.250 static · IPS/IDS 검증용(필요시만 가동)
 ├ Security Onion (NSM센서) ← 미러 기반 IDS/관제          [도입중] 6/17 SPAN 미러 수신(Suricata+Zeek+Elastic+SOC 3.1) — 실설치 재시도 예정
 ├ xubuntu (관리 데스크탑)   ← LAN 전용 게스트            [설치 완료] pfSense 관리용
 ├ Web 서버 (Ubuntu Server) ← VLAN 10(운영)             [구성] G/W 10.10.10.1 · 아파치+PHP+MariaDB+webhack · 프록시 화이트리스트(ubuntu.com만)
 ├ Web_Dev 서버 (Ubuntu)    ← VLAN 20(개발)             [구성] G/W 10.10.20.1 · 삼바 마운트 · xubuntu-core+firefox(프록시 테스트) · 블랙리스트(games)
 ├ Samba 서버 (Ubuntu)      ← VLAN 30(파일공유)          [설치 완료] G/W 10.10.30.1 · 포트 445 · 아웃바운드 80/443 닫음
 ├ DNS 서버 (BIND9)         ← VLAN 40(내부 이름풀이)      [설치 완료] G/W 10.10.40.1 · 포트 53 · 아웃바운드 80/443 닫음
 └ Kubuntu Desktop          ← 데스크탑용 게스트          [예정]
```

> 🦑 6/15 추가: pfSense에 **Squid/SquidGuard**로 아웃바운드 URL 통제(web=화이트리스트 / webdev=블랙리스트, HTTP만 — HTTPS는 SSL Bump 보류). **Snort(IDS)**를 WAN·VLAN10에 가동해 C2·Nikto·디렉터리 트래버설·커스텀 SQLi 4종 탐지 확인. samba/dns의 안 쓰는 아웃바운드 80/443은 닫음(최소권한).

> 🛡️ 6/16 추가: **Snort를 IDS→IPS(인라인)로 전환** — Block Offenders + Inline 모드(체크섬 오프로딩 OFF 선행), 켜자마자 따라오는 오탐 튜닝(Suppress=알람끄기 / Pass List=차단면제)·커스텀 drop 시그니처(hex/hash). **Kali(공격기 192.168.1.250)**를 WAN쪽에 투입해 검증 준비.

> ☘️ 6/17 추가: pfSense **OpenVPN 서버**(원격접속) 구축 — CA·서버인증서(lifetime 3650)·새 터널 /24·redirect gateway·client-export. **Security Onion(미러 NSM 센서)** 도입 시작 — SPAN 복제 트래픽을 Suricata+Zeek로 보고 SOC 콘솔로 관제(3.1). 정리: **Snort 인라인 = 막는 쪽 / Security Onion = 보는 쪽**.

> 🧅 6/18 추가: Security Onion **SOC 콘솔 Detections에 커스텀 룰 추가 → 톱니바퀴(동기화) → ~15분 뒤 반영**(분산 센서라 배포 지연 큼) → PCAP로 패킷 흐름 확인. 이론은 **NGIPS가 갖춰야 할 요건**(암호화 양방향검사·세션관리·멀티패턴·취약점스캐너/Anti-Botnet 연동·GeoIP·Inline+IDS 듀얼모드)과 **Snort/Suricata 룰 옵션 심화**(offset/depth/distance/within/nocase/http_header/fast_pattern/flow + ET 룰 2건 분석). ⚠️ **우리 랩은 VLAN(논리분리)일 뿐 망분리 아님** — 세그먼트(여러 LAN)≠VLAN≠망분리. ⚠️ **Anti-DDoS와 IPS는 한 라인에 몰면 같이 뻗음** → 역할별 분리(AD=pf / FW=OPN).

> 🗡️ 6/19 추가: **Kali 공격기 가동** — 웹서버 **nmap 포트스캔**(열린 포트=포워딩으로 연 표면만 잡힘) + **Hydra 사전공격**으로 SSH 무차별 대입(서버 auth 실패 로그 무더기 / 실패 로그 주체는 관리자 아니면 공격자 둘뿐). **IPS 인라인+전체 룰에 nmap** 쏘니 알람 폭주 → **ESM(로그 중앙 수집·상관분석) 필요성** 체감(= Security Onion의 상용 확장판). 오후 **와이어샤크로 IP 헤더** 필드 분해(이론: OSI-7Layer.md).

> 🔬 6/22 추가: 장비 증설 없이 **와이어샤크로 L4(전송계층) 직접 관찰** — `ping -l 3500`으로 단편화 강제해 MF=1/0·Fragment Offset 확인(TCP는 MSS 협상으로 안 보임), **TCP 3-way**(상태전이·시퀀스/ACK 동기화·플래그·윈도우) 캡처, **UDP**(헤더 4칸·닫힌 포트→ICMP Port Unreachable). 오후 **nmap `-sT`(Connect, 3-way 완성·로그 남음) vs `-sS`(SYN/스텔스, RST로 끊어 흔적 제거)** 를 와이어샤크로 대조(열림=SYN+ACK/닫힘=RST/필터=무응답). 이론: OSI-7Layer.md(L4)·Attack-Defense.md(포트 스캔).

### IP 배치 — 외부망(192.168.1.0/24) + 내부 관리망(192.168.2.0/24)

| 구성요소 | 주소 | 역할 |
|---|---|---|
| 공유기 | 공인 211.x.x.x / 내부 G/W 192.168.1.1 | 집 전체 인터넷 출입구 + 외부망 게이트웨이 |
| 내 본컴 | 192.168.1.83 | 메인 PC. NFS로 서브컴(LAB)에 저장공간 제공 |
| 서브컴(LAB) | 192.168.1.63 | 실습 전용 머신. 위에 ESXi 올림 |
| ESXi | 192.168.1.200 | 서브컴 위 하이퍼바이저 (관리: Management Network) |
| NFS | 192.168.1.220 | 본컴 → 서버컴(LAB) 용량 추가용 네트워크 스토리지 |
| **pfSense WAN(em0)** | **192.168.1.82/24** | 공유기 브릿지로 받은 외부 인터페이스 |
| **pfSense LAN(em1)** | **192.168.2.1/24** | 내부 관리망 게이트웨이 + DHCP 서버(2.100~200). 포트그룹 VLAN ID **4095(트렁크)** |
| **xubuntu** | **192.168.2.100** | LAN에 붙은 pfSense 관리 데스크탑(DHCP 수령) |
| **VLAN 10 G/W** | **10.10.10.1** | 운영 VLAN 게이트웨이 → Web 서버(10.10.10.10) |
| **VLAN 20 G/W** | **10.10.20.1** | 개발 VLAN 게이트웨이 → Web_Dev 서버(10.10.20.10) |
| **VLAN 30 G/W** | **10.10.30.1** | 파일공유 VLAN 게이트웨이 → Samba 서버(10.10.30.10) |
| **VLAN 40 G/W** | **10.10.40.1** | 내부 DNS VLAN 게이트웨이 → BIND9 서버(10.10.40.10) |
| **Kali (공격기)** | **192.168.1.250 static** | 본컴 VirtualBox·WAN쪽 외부 공격자 포지션 (6/16, 필요시만 가동) |
| **OpenVPN 터널** | 새 /24 가상대역 | 원격접속 클라이언트가 받는 사설 IP 대역 (6/17, redirect GW) |
| **Security Onion 센서** | 모니터링 NIC(IP없음) + 관리 NIC | SPAN 미러 수신(promiscuous) / SOC 웹 접속용 관리 NIC 분리 (6/17) |
| 다른 무선 장치 | (무선) | 공유기에 무선으로 붙는 단말들 (IDS 음영지역) |

> ⚠️ WAN(192.168.1.x)과 LAN(192.168.2.x)은 **반드시 다른 대역**. 6/10에 pfSense LAN 기본값(192.168.1.1)이 공유기 G/W와 충돌해 192.168.2.1로 바꾼 게 이 분리의 출발점.

> ⚠️ pfSense가 VLAN 10~40을 라우팅하려면 LAN 포트그룹이 **VLAN ID 4095(VGT/트렁크)** 여야 한다 — 태그를 안 지우고 게스트(pfSense)로 다 넘겨야 pfSense가 동별로 분류·라우팅. 6/11 핵심.

> 🔀 본컴(192.168.1.83)→내부 서버 직접 접근용 **NAT 포트포워딩**(6/12): WAN 80→web:80, 1010→web:1010, 2020→webdev:2020, 3030→samba:3030, 4040→dns:4040. WAN 룰의 Source는 **`ADMIN_IP` Alias**(=192.168.1.83)로 통일.

---

## 🗓️ 진행 로그

| 날짜 | 내용 | 파일 |
|---|---|---|
| 2026-06-09 | Workstation Pro·ESXi 8.0 설치 / 목표 랩 구성 / 홈 네트워크 설계 | [2026-06-09-환경구축.md](2026-06-09-환경구축.md) |
| 2026-06-10 | pfSense WAN/LAN 2장 할당 / 대역충돌 해결(LAN 2.1) / DHCP·라우팅 검증(xubuntu) | [2026-06-10-pfSense구축.md](2026-06-10-pfSense구축.md) |
| 2026-06-11 | VLAN으로 Web/Web_Dev 분리 / ESXi VGT 4095 트렁크 / pfSense VLAN 라우팅(10.10.10·20.1) / 웹서버 초기세팅 | [2026-06-11-VLAN구축.md](2026-06-11-VLAN구축.md) |
| 2026-06-12 | 삼바 파일서버(VLAN30)·내부 DNS(VLAN40) 추가 / NAT 포트포워딩(DNAT) / Alias / SSH ssh.socket 포트변경 | [2026-06-12-삼바·DNS·NAT.md](2026-06-12-삼바·DNS·NAT.md) |
| 2026-06-15 | Squid/SquidGuard 아웃바운드 URL 통제(화이트/블랙리스트) / SSL Bump 보류 / 안 쓰는 아웃바운드 닫기 / Snort IDS 4종 탐지실험 | [2026-06-15-웹프록시·Snort.md](2026-06-15-웹프록시·Snort.md) |
| 2026-06-16 | Snort IDS→IPS 인라인 전환(Block Offenders·체크섬 OFF) / 오탐 튜닝(Suppress·Pass List·SID·hex/hash) / Kali 공격기 WAN 투입 / VPN 연결 흐름 | [2026-06-16-Snort-IPS전환·Kali·VPN.md](2026-06-16-Snort-IPS전환·Kali·VPN.md) |
| 2026-06-17 | pfSense OpenVPN 구축(CA·인증서·터널 /24·redirect GW·client-export) / SSL-VPN any·VPN≠포트포워딩·닌자게이트 / Security Onion 미러 센서 도입(3.1) | [2026-06-17-OpenVPN·SecurityOnion.md](2026-06-17-OpenVPN·SecurityOnion.md) |
| 2026-06-18 | Security Onion Detections 커스텀 룰 추가·동기화(~15분)·PCAP 확인 / (이론) NGIPS 요건·Snort/Suricata 룰 옵션 분석·VLAN≠망분리·Anti-DDoS와 IPS 분리 | [2026-06-18-SecurityOnion-룰추가.md](2026-06-18-SecurityOnion-룰추가.md) |
| 2026-06-19 | Kali nmap 포트스캔·Hydra 사전공격(SSH 무차별 대입) / IPS 인라인+전체룰에 nmap→알람폭주→ESM 필요성 / 와이어샤크 IP헤더 분해 | [2026-06-19-Kali포트스캔·Hydra·패킷분석.md](2026-06-19-Kali포트스캔·Hydra·패킷분석.md) |
| 2026-06-22 | 와이어샤크 L4 관찰(ping -l 단편화·TCP 3way/헤더/윈도우·UDP+ICMP unreachable) / nmap -sT vs -sS(스텔스) 대조 | [2026-06-22-와이어샤크-전송계층·nmap스캔비교.md](2026-06-22-와이어샤크-전송계층·nmap스캔비교.md) |

---

## ➡️ 다음 할 것 (백로그)

- [x] ESXi에 pfSense 올리기 — WAN(em0) / LAN(em1) 2장 구성 ✅ 6/10
- [x] 내부망 기기(xubuntu) 올려 LAN 통신·DHCP 확인 ✅ 6/10
- [x] LAN 대역에 서버 게스트 올려 세그먼트 분리 — VLAN 10/20 Web·Web_Dev ✅ 6/11
- [x] 파일서버(삼바 VLAN30)·내부 DNS(BIND9 VLAN40) 올려 4칸 채우기 ✅ 6/12
- [x] 관리 IP를 Alias(`ADMIN_IP`)로 묶어 WAN 룰에 일괄 적용 ✅ 6/12
- [x] 아웃바운드 URL 통제를 프록시로 — Squid/SquidGuard(화이트/블랙리스트, HTTP) ✅ 6/15
- [x] IDS(Snort) 올려 탐지 실험 — C2·Nikto·디렉터리 트래버설·커스텀 SQLi ✅ 6/15
- [x] 안 쓰는 아웃바운드 닫기 — samba/dns의 80/443 제거(최소권한) ✅ 6/15
- [x] **Snort IDS→IPS(인라인) 전환** — Block Offenders + Inline(체크섬 오프로딩 OFF) ✅ 6/16
- [x] **IPS 오탐 튜닝** — Suppress(알람끄기)·Pass List(차단면제)·커스텀 drop(hex/hash·SID 3000000+) ✅ 6/16
- [x] **Kali 공격기 투입** — 192.168.1.250 static, WAN쪽 외부 공격자 포지션 ✅ 6/16
- [x] **VPN 실제 구축** — pfSense OpenVPN(CA·서버인증서·터널 /24·redirect GW·client-export) ✅ 6/17
- [x] **Security Onion 도입(미러 NSM)** — SPAN 수신·SOC 콘솔 개념까지 ✅ 6/17 (실설치는 재시도 예정)
- [x] **Security Onion Detections 콘솔 흐름** — 커스텀 룰 추가 → 톱니바퀴 동기화(~15분) → PCAP 확인 ✅ 6/18
- [ ] **SSL Bump 재시도** — 투명 intercept(ERR_DNS_FAIL) 대신 명시적 프록시(PAC/WPAD)로 HTTPS까지 필터링
- [ ] **WAF(ModSecurity + OWASP CRS)** 올려 IDS가 못 잡는 범용 SQLi/XSS 커버 (IDS≠WAF 실증)
- [ ] **Security Onion 실설치 재시도** — ISO 다시 받기 → so-status·ifconfig RX 확인 → SPAN 물려 미러 검증
- [ ] **OpenVPN 원격접속 테스트** — client-export로 실접속 + 사용자별 목적지 정책(사설 IP ≠ 프리패스) 검증
- [x] **Kali 실제 공격 발사** — nmap 포트스캔(열린 포워딩 포트만 잡힘) + Hydra 사전공격(SSH 실패 로그 무더기) ✅ 6/19
- [x] **IPS 인라인+전체룰에 nmap** → 알람 폭주로 ESM(중앙 로그수집·상관분석) 필요성 체감 ✅ 6/19
- [x] **와이어샤크로 L4 직접 관찰** — ping -l 단편화(MF)·TCP 3way/시퀀스·ACK/윈도우·UDP+ICMP unreachable ✅ 6/22
- [x] **nmap -sT vs -sS(스텔스) 대조** — 열림(SYN+ACK)/닫힘(RST)/필터(무응답), 스텔스=RST로 흔적 제거 ✅ 6/22
- [ ] **스텔스 스캔(-sS)을 IPS가 잡나** — Snort 인라인에 쏴서 `flags:S` 임계치/알람 대조 (스캔 캡처 PCAP를 Security Onion에도 올려 보는 쪽 비교)
- [ ] **Kali 공격 → 막는 쪽/보는 쪽 동시 검증** — Snort 인라인은 drop, Security Onion Alerts엔 뜨나 대조
- [ ] **Hydra 무차별 대입 차단** — SSH fail2ban / Rate Limit / IPS 임계치로 끊기 (FIMS slowapi의 인프라판)
- [ ] **Snort 룰 튜닝(잔여)** — 인터페이스별로 그 망에 맞는 룰만 남기고 오탐 대상 정리
- [ ] VLAN 10~40 사이 pfSense 방화벽 룰로 단방향 통제 (어떤 망→어떤 망 why로 설계)
- [ ] 인터넷 아웃바운드를 PMS(내부 레포) 경유로 — 서버 직접 인터넷 차단 (오늘 화이트리스트 ubuntu.com만 연 것의 다음 단계)
- [ ] pfSense 방화벽 룰로 Architecture "A/B/C/D 코스" 정책 박아보기
- [ ] WAN `any` 허용 잔재를 관리 IP(Alias)로 좁혀 공격 표면 줄이기
- [ ] NFS(192.168.1.220)로 본컴→서버컴 용량 마운트
- [ ] 유선 구간에 SPAN/TAP 걸어보고 무선이 왜 안 잡히나(음영지역) 직접 확인
