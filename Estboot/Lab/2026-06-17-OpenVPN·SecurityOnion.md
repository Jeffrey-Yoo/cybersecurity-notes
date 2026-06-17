# 2026-06-17 — pfSense OpenVPN 구축(원격접속 터널) + Security Onion 센서 도입(미러 NSM)

과정2 12일차. 오전엔 6/16에 "개념"으로만 본 VPN을 **pfSense OpenVPN으로 실제 구축**(CA·서버 인증서·터널 대역·게이트웨이 리다이렉트·클라이언트 키 배포)했고, SSL-VPN을 왜 any로 여는지·VPN과 포트포워딩이 어떻게 다른지·재택 이중망의 "닌자게이트" 위험까지 흐름으로 정리. 오후엔 **Security Onion(미러 기반 NSM 센서)**을 토폴로지에 새로 박기 시작했다.

> 🔗 어제와의 연결: 6/16 = "VPN이 실제로 어떻게 연결되나"를 **개념**으로(키·OTP·SSL VPN·입구 SSL Inspection). 오늘이 그 **랩 구현편** — 개념의 터널을 pfSense OpenVPN으로 직접 깔고, 6/16에 만든 인라인 IPS(Snort) 옆에 **가시성 담당(Security Onion 미러 센서)**을 추가해 "막는 쪽 + 보는 쪽"을 둘 다 세우기 시작.

> FIMS 연결: 과정1 FIMS의 **WireGuard 터널**(원격접속)과 **Suricata+Wazuh(탐지+관제)**를, 오늘 각각 **OpenVPN**과 **Security Onion(Suricata+Zeek+Elastic+콘솔 올인원)**으로 다시 만난 셈. FIMS에서 손으로 엮은 걸 제품·마법사로 받는 경험.

> ⚠️ 솔직한 기록: 오후 Security Onion 구간은 **내 PC 문제로 ISO를 제때 못 받아 실제 설치를 못 따라가** 강의 집중이 깨졌다. 그래서 이 파일의 Security Onion 부분은 **노트에 남은 것(so-status·ifconfig RX 확인) + 공식 문서(3.1) 기준으로 메운 보충**이다. 실제 3.1 화면과 다르면 설치 후 본문을 갱신할 것.

---

## 🗺️ 오늘의 토폴로지 변화 (드로우 갱신)

6/16에 **Kali(공격기)**와 **Snort 인라인 IPS 전환**이 들어갔고, 오늘 **Security Onion 센서(SPAN 미러 수신)**가 추가됐다. 그림(Excalidraw)도 이 상태로 갱신.

```
        [ 인터넷 / 192.168.1.0/24 (WAN·관리 대역) ]
              │                  │
   본컴 192.168.1.83     Kali 192.168.1.250(static)  ← 공격기(6/16)
              └────────┬─────────┘
                       ▼
            pfSense WAN 192.168.1.82
            ├ Snort(WAN)  : 인라인 IPS = "막는 쪽"(6/16)
            ├ Squid/SquidGuard : 아웃바운드 URL 통제(White Pass List)
            └ OpenVPN 서버 : 원격접속 터널 + 터널 대역(새 /24)  ← ★오늘
                       │
        ┌──────────────┼───────────────┬───────────────┐
   VLAN10 web      VLAN20 webdev    VLAN30 samba     VLAN40 dns
   10.10.10.10     10.10.20.10      10.10.30.10      10.10.40.10
        │
   [스위치 SPAN/미러] ──복제──> ☘️ Security Onion 센서 = "보는 쪽"(미러 NSM)  ← ★오늘
                                 (모니터링 NIC: IP 없음·promiscuous / 관리 NIC: SOC 접속용)
```

- **막는 쪽 vs 보는 쪽**이 오늘로 둘 다 생겼다: **Snort 인라인 IPS(차단)** + **Security Onion(미러 가시성)**. 6-8 "인라인 vs 미러링"이 랩에 그대로 구현됨.
- Security Onion 센서는 **라인에 끼지 않는다**(IPS와 반대). 스위치 SPAN/TAP의 **복제본**만 옆에서 받는다 → 센서가 죽어도 통신은 안 끊김.

---

## 1️⃣ pfSense OpenVPN 구축 (원격접속 / Remote Access)

pfSense 내장 패키지 **OpenVPN**을 사용. (IPsec은 2차 프로젝트로 미룸)

### 절차 흐름

```
① 인증서 만들기
   - CA(인증기관) 생성
   - 서버 Certificate 생성 → 서버측 lifetime = 3650 (≈10년)
② OpenVPN 서버 마법사 설정
   - 터널 네트워크(Tunnel Network) = 기존과 안 겹치는 "새 VPN 대역"을 /24(24비트)로
     → 접속 클라이언트는 이 가상 대역에서 사설 IP를 발급받음
   - Redirect Gateway 체크 = "WAN을 게이트웨이로 쓰겠다" 선언
       · 게이트웨이는 하나다
       · 이 설정으로 다른 대역 사용자가 내 게이트웨이를 타고 트래픽을 오갈 수 있게 됨
       · ⚠️ 바로 이 지점이 "닌자게이트"(우회 경로) 위험의 출발 → 아래 4️⃣
   - 방화벽 규칙 자동 생성 체크박스 활성화 → finish
③ 클라이언트 키 배포 도구
   - 추가 패키지 openvpn-client-export 설치
     = "사용자에게 키값(설정+인증서 묶음)을 전달해주는 툴"
   - VPN → OpenVPN → Client Export → set as default →
     하단 응용프로그램(설치형 클라이언트) 다운로드 → 원하는 사용자에게 전달
```

- **lifetime 3650**: 서버 인증서 유효기간을 길게(약 10년). 자주 갱신하기 번거로운 인프라 인증서라 길게 잡되, ⚠️ 길수록 **유출 시 노출 기간도 길다**는 트레이드오프(실무는 짧게+자동갱신 쪽이 안전).
- **터널 네트워크 /24를 "새 대역"으로**: 기존 LAN/VLAN 대역과 겹치면 라우팅이 꼬인다. VPN 클라이언트는 이 가상 대역에서 IP를 받아 내부망의 일원처럼 동작.

> 🔗 개념 상세(SSL-VPN any 정책·VPN≠포트포워딩·키/OTP/SSL Inspection)는 `../IT-Infra-Secu/Security-Solutions.md` → VPN 섹션 참고. 여기선 "랩에서 실제로 뭘 눌렀나"만.

---

## 2️⃣ SSL-VPN은 왜 정책을 any로 여나

```
원래 폐쇄형 VPN → 직원이 잦게 이동(스벅·집·LTE)하니 1:1 출발지 정책은 사람이 대응 불가
 → 입구(출발지)는 any로 열 수밖에 없음 = 공개형
 → 대신 들어오는 건 SSL(인증)으로 막는다 → VPN 로그인=인증 통과해야 서버 접속
```
- ⚠️ any = 해커도 문 앞까지 옴. "문은 누구나 두드리게 열되 인증으로 거른다"가 핵심. 보안 책임이 **출발지 제한 → 인증**으로 이동.

---

## 3️⃣ VPN ≠ 포트포워딩 (꼭 구분)

```
포트포워딩(DNAT, 6/12) : 외부 포트를 내부 IP:포트로 꽂기 (서비스 하나 노출)
VPN                    : 접속자에게 사설 IP 발급 → 내부망 일원으로
```
- ⚠️ 사설 IP를 줬다고 다 들어오는 게 아니다 → **그 IP에 대한 정책이 생성돼야** 함. 특정 목적지로만 가는 정책만 부여.
- 관리자(부여자) 입장: 클라이언트에게 **권한 부여 + 허가 경로(정책)** 2단 설계.

---

## 4️⃣ 재택 이중망의 함정 — 계정 2개·닌자게이트·ISMS

```
재택에서 개발망 + 인터넷망 둘 다 쓰려면 → VPN 계정 2개 필요 (터널은 계정당 1개)
개발망 작업 중 인터넷 필요 → 두 라우팅 동시 사용 가능 → 근데 위험
```
- ⚠️ **닌자게이트**(강사님 표현) = 두 망을 몰래 잇는 우회 게이트웨이/피벗.
  - 재택 노트북이 RAT로 원격당하면 → 두 망에 동시에 붙은 노트북이 다리가 됨 → 해커가 인터넷→노트북→개발망으로 피벗 → **망분리가 노트북 한 대로 무력화**.
- **바람직**: VPN 끊고 → 인터넷 쓰고 → 다시 연결(동시 연결 금지). "번거롭다" 불평한 직원에게 강사님 *"그럼 그만둬라"* (실제 사례).
- 🔗 코로나 이후 VPN 확산 → **ISMS 심사가 이 "재택 두 망 동시연결" 취약 환경부터 확인**한다.

---

## 5️⃣ Security Onion — 미러 기반 NSM 센서 도입 (보충 포함)

> ⚠️ 위에 적은 대로 이 구간은 PC 문제로 실설치를 못 따라감 → **노트(설치 후 점검) + 공식 문서(3.1) 기준 보충**.

### 노트에 실제로 남은 것
```
- 설치 완료 후 so-status → 정상적으로 러닝 중인지 확인
- ifconfig → rx 패킷이 잘 들어오는지 확인 (미러가 실제로 들어오나)
- (강의 화면) 관리자 페이지에서 UI들이 어떻게 작동하고 로그를 어떻게 보는지 → 못 따라감
```

### 보충 — Security Onion 3.1 핵심 (공식 문서 기준)

```
구성 : Suricata(시그니처 NIDS + 풀 PCAP) + Zeek(프로토콜 메타로그)
       + Elastic(ES 저장/검색 · Logstash 수집 · Kibana) + SOC(웹 관제 콘솔)
버전 : 3.1.0 (Elastic 9.3.3 / Suricata 8.0.5 / Zeek 8.0.8, OS=Oracle Linux 9)
⚠️ 3.0부터 ① UI 새로 바뀜  ② Stenographer 제거 → 풀 패킷캡처를 Suricata가 담당
```

**센서 동작 (왜 ifconfig RX를 보나)**
```
스위치 SPAN(미러)/TAP ──복제 트래픽──> 센서 모니터링 NIC (promiscuous, IP 없음)
   → Suricata 탐지+PCAP / Zeek 메타로그 → Elasticsearch 적재 → SOC·Kibana 조회
* 모니터링 NIC에 IP를 안 주는 이유: 듣기만 하면 됨 + 센서 자신이 표적이 안 됨
* 관리 NIC은 따로(SOC 웹 접속용, IP 있음)
```
- ⚠️ **RX가 0이면** 룰·콘솔이 멀쩡해도 탐지 0(장님 센서) → SPAN 설정/케이블/포트그룹부터 의심. (그래서 ifconfig RX 확인이 1번 점검)
- `so-status`로 도커 서비스가 전부 running인지 = 수집/탐지 파이프라인이 살아있나 확인.

**SOC 관제 UI(좌측 메뉴)와 로그 보는 법**
```
Grid       : 노드(매니저/센서/서치) 상태판
Alerts     : 경보 모음 — 여기서 시작
Dashboards : 데이터 유형별 대시보드(NIDS 경보/Zeek 메타/Elastic Agent/방화벽 로그)
Hunt       : 위협 헌팅(쿼리로 능동 사냥)
PCAP       : 선택 스트림 전체 패킷(증거) — 3.x는 Suricata 캡처
Cases      : 의심 건을 사건화해 협업·종결
Detections : 탐지 룰(Suricata/Sigma/YARA) 관리
Kibana/CyberChef/ATT&CK Navigator : 원본 탐색 / 디코딩 / MITRE 매핑

로그 보는 흐름:  Alerts → (드릴다운) Hunt → PCAP(증거) → Cases(사건화)
```

> 🔗 이론 상세는 `../IT-Infra-Secu/Security-Solutions.md` → "Security Onion — 오픈소스 NSM 플랫폼" 섹션. SPAN/센서 위치는 6-9 "센서를 길목에" 와 동일.

---

## ⚠️ 오늘 짚어둘 것

- **OpenVPN = CA + 서버 인증서(lifetime 3650) + 새 터널 /24 + redirect gateway + client-export** 순.
- **터널 대역은 기존과 안 겹치는 새 /24** — 겹치면 라우팅 꼬임.
- **redirect gateway 체크 = WAN을 게이트웨이로** → 다른 대역이 내 G/W를 타는 우회(닌자게이트) 위험의 씨앗.
- **SSL-VPN any 정책** = 이동성 때문에 입구를 열고 인증으로 막는 것. **VPN≠포트포워딩**(사설 IP 발급 ≠ 접근 허가, 정책 필요).
- **재택 두 망 동시연결 금지** — RAT 피벗(닌자게이트), ISMS 점검 포인트.
- **Security Onion = 미러 NSM(보는 쪽), Snort 인라인 = IPS(막는 쪽)** — 역할 분담, 경쟁 아님.
- **설치 후 so-status(서비스 running) + ifconfig RX(미러 수신)** 2종 점검이 "센서가 보고 있나"의 기본.

---

## ➡️ 다음 할 것

- [ ] OpenVPN 클라이언트(client-export)로 **실제 원격 접속 테스트** — 발급 사설 IP 확인 + 정책 통과/차단 검증
- [ ] VPN 사용자별 **목적지 정책**(특정 VLAN만 허용) 박아보기 — "사설 IP ≠ 프리패스" 실증
- [ ] Security Onion **실설치 재시도**(ISO 다시 받기) → so-status·ifconfig RX 확인 → SPAN 물려 미러 들어오나 확인
- [ ] Kali로 공격 → **Security Onion Alerts에 뜨나** + Snort 인라인은 **막나** 동시 검증(보는 쪽 vs 막는 쪽 대조)
- [ ] 이월: SSL Bump 명시적 프록시, WAF(ModSecurity), VLAN 단방향 룰
