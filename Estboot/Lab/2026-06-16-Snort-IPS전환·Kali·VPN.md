# 2026-06-16 — Snort를 IPS로(인라인 차단) + 룰 튜닝(Suppress/Pass) + Kali 공격기 투입 + VPN 흐름

과정2 11일차. 어제(6/15)까지 Snort는 **보고만 있는 IDS**였다. 오늘은 그걸 **걸리면 막는 IPS(인라인)**로 끌어올리고, 차단을 켜면 따라오는 숙제인 **오탐 튜닝(Suppress/Pass List)**까지 만졌다. 그리고 그 IPS를 때려볼 **공격기(Kali)**를 토폴로지에 새로 박았다. 오후엔 **VPN이 실제로 어떻게 연결되나**(키·OTP·SSL VPN·입구 복호화)를 흐름으로 정리.

> 🔗 어제와의 연결: 6/15 = "탐지(alert)만 + 룰 튜닝은 다음 숙제"로 끝냈다. 오늘이 그 **"다음 숙제"의 본편** — 탐지를 차단으로 바꾸고(인라인), 차단을 켜니 곧장 필요해진 오탐 관리(Suppress/Pass)를 한 것. 어제 적어둔 "설치 직후 오탐 폭주 → 튜닝이 IDS 운영의 핵심"이 오늘 실제 작업이 됐다.

> FIMS 연결: 과정1 FIMS의 Suricata는 **탐지·로깅(IDS)**까지였다 — 라인 위에서 직접 끊는 인라인 IPS는 안 했다. 오늘 pfSense Snort를 인라인으로 돌려본 게 "FIMS에선 못 가본 차단까지" 한 걸음 더 간 셈. (그리고 인라인을 켜자마자 체크섬·오탐이라는 새 함정이 줄줄이 나온 것도 그때 Suricata fast.log 삽질의 연장)

---

## 🗺️ 오늘 바뀐 토폴로지 — Kali 공격기 추가

어제까지의 구성(WAN쪽 본컴 + pfSense + VLAN 10/20/30/40)에 **Kali Linux**를 WAN(외부) 쪽에 새로 그렸다. IPS를 만들었으면 **때려볼 공격자**가 있어야 하니까.

```
                    [ 인터넷 / 192.168.1.0/24 (WAN·관리 대역) ]
                        │              │                 │
            본컴(호스트) 192.168.1.83   Kali 192.168.1.250(static)   ← ★오늘 추가: 공격기
                        │              │   (본컴 VirtualBox에 설치 — 서브컴 자원 아끼려고)
                        └──────┬───────┘
                               ▼
                    pfSense WAN  192.168.1.82   ── Snort(WAN) : 인라인 IPS로 전환
                               │
                ┌──────────────┼───────────────┬───────────────┐
            VLAN10 web      VLAN20 webdev    VLAN30 samba     VLAN40 dns
            10.10.10.10     10.10.20.10      10.10.30.10      10.10.40.10
            Snort(VLAN10)   xubuntu+firefox  파일서버         BIND9
```

- **Kali = 192.168.1.250 static**, 본컴 VirtualBox에 올림. **서브컴 자원을 아끼려고 본컴에 설치**한 게 의사결정 포인트(공격기는 상시 켜둘 게 아니라 필요할 때만 띄우면 됨).
- 위치가 **WAN(외부) 쪽**인 게 핵심 — 외부 공격자 입장에서 pfSense WAN의 Snort 인라인 차단을 시험하기 위함. (어제 본컴/PowerShell로 흉내냈던 외부 공격을, 오늘부터는 진짜 공격용 배포판으로)

> 💡 왜 굳이 Kali냐 — 칼리는 Nikto·nmap·sqlmap·metasploit 등 **공격 도구가 기본 탑재**된 배포판. 어제 "xubuntu에 firefox 따로 깔았다"처럼 도구를 매번 설치할 필요 없이, IPS/IDS 룰을 때려보는 **표준 공격기**로 쓰기 좋다. (방어 장비를 만들었으면 같은 사거리의 공격기로 검증하는 게 정석)

> ⚠️ static IP를 250으로 박은 이유 — DHCP로 받으면 IP가 바뀌어 Suppress/Pass List·룰의 출발지 IP가 흔들린다. 공격기·관리노드는 **고정 IP**여야 룰과 로그가 일관된다. (어제 "threshold·룰은 출발지 IP 기준"의 연장)

---

## 1️⃣ Snort를 IDS → IPS로 (인라인 차단 켜기)

### 하기 전 준비 — Hardware Checksum Offloading OFF

`System → Advanced → Networking`에서 **Hardware Checksum Offloading 비활성화**(체크박스 ON = "Disable hardware checksum offload").

```
왜? 인라인 Snort(netmap)는 패킷을 "NIC이 체크섬 채우기 직전"보다 앞에서 가로챈다
 → 오프로딩이 켜져 있으면 Snort가 보는 패킷의 체크섬이 비어있음/틀림 → 검사 꼬임·재주입 깨짐
 → 끄면 CPU가 미리 체크섬을 채워 Snort가 "완성된 패킷"을 봄 → 인라인이 정상 동작
⚠️ VirtualBox(가상 NIC)는 체크섬을 호스트가 대신 처리해 더 잘 깨짐 → VM 위 인라인 IPS엔 거의 필수
```

> 자세한 메커니즘은 `Security-Solutions.md` → "Hardware Checksum Offloading을 꺼야 인라인이 돈다" 참고.

### Block Offenders 체크 → 인라인 모드

`Services → Snort → (WAN 인터페이스 edit)`:

```
Block Offenders : ON
  설명문 그대로 "Checking this option will automatically block hosts that generate a Snort alert."
  = alert를 일으킨 호스트를 자동 차단 (기본은 Not Checked)

차단 방식(모드):
  Legacy Mode → 복제본 보고 탐지, alert IP를 pf 차단테이블(snort2c)에 등록해 "이후" 트래픽 차단(사후)
  Inline Mode → netmap으로 라인 위에서 실시간으로 그 패킷 자체를 drop  ← 오늘 켠 것
```

- 이 체크 하나로 어제까지 IDS였던 Snort가 **IPS**가 된다. (`Security-Solutions.md` "IDS=탐지만 / IPS=탐지+차단"이 체크박스로 갈리는 지점)
- drop 룰(액션이 `alert`가 아니라 `drop`인 룰)에 걸리면 실시간 폐기.

> 🔗 Legacy=미러링(사후 차단) / Inline=인라인(실시간 차단). 6/8~6/9에 정리한 "인라인 vs 미러링"이 GUI 옵션으로 그대로 나온다.

---

## 2️⃣ 차단을 켜니 따라오는 숙제 — 오탐 튜닝 3종

IPS를 켜는 순간 **오탐 = 장애**(정상 PC·서버가 끊김)다. 그래서 바로 튜닝 도구를 만졌다.

### Suppress(억제) — 알람만 끈다

`Snort → Suppress`. alert 옆 ➕ 버튼으로 자동 등록된다. 노트 한 줄 요약 그대로:

```
"ip가 계속 들어온다 → ip를 끈다 / sid로 너무 들어온다 → sid를 끈다"

① 특정 IP만:   suppress gen_id 1, sig_id 2404300, track by_dst   (그 IP/방향만 알람 X)
② SID 통째로:  suppress gen_id 1, sig_id 2404300                 (이 룰은 전부 알람 X)
```

- 실제로 등록하니 `wansuppress_xxxx` 리스트에 위 형식으로 **자동 추가**됐다(=알람 안 옴, 해당 대상에 한해).
- gen_id 1 = Snort 기본 룰엔진, sig_id = 룰 번호.

> ⚠️ Suppress는 **"차단 해제"가 아니라 "로그 음소거"**다. 노이즈를 줄여 진짜 봐야 할 alert를 보이게 하는 것.

### Pass List(통과 목록) — 차단을 면제한다

`Snort → Pass Lists`. 등재된 IP는 **alert가 떠도 절대 block 안 함**.

```
많이 쓰는 PC·서버·게이트웨이·DNS·관리자 IP → Pass List 등재 (예: passlist_945 "내 노트북 예외 처리")
 → 핵심 노드가 오탐으로 끊기는 사고 방지
```

- ⚠️ **여기서도 IP 직접 입력은 하수 — Alias로.** 대상이 바뀌면 Alias 한 곳만 수정. (6/12 "ip로 치는 건 하수"가 또 등장)

> 🔗 정리: **Suppress = 알람 끄기(로그) / Pass List = 차단 면제(가용성).** 시끄러운 룰은 Suppress, 절대 안 끊겨야 하는 호스트는 Pass List. 목적이 다르다.

### SID 네임스페이스 — 커스텀은 3000000번부터

```
SID는 겹치면 안 됨(중복 = 룰 로드 에러)
  ~999,999            Snort VRT/Talos
  2,000,000~2,999,999 ET(어제 쓴 ET Open이 여기)
  3,000,000~          우리 커스텀(local)  ← 안 겹치는 빈 대역
```

---

## 3️⃣ 커스텀 drop 시그니처 — 문자열을 hex/hash로

웹서버 admin 로그인 패턴으로 drop 룰을 짜보며 배운 핵심: **content를 평문 문자열로 넣으면 잘 안 맞는다**(인코딩·이스케이프·비출력 바이트). 그래서:

```
헥스(hex):   content:"|68 70 61 73 6D 6D 6C 64|"   ← 바이트 그대로 매칭 (68..64 = ASCII "hpasmmld")
해시(hash):  file_sha256; content:"|c7f6…b5d4|"     ← 파일 자체를 지문으로 매칭(파일 단위 탐지)
            file_md5;    content:"|a47d…4d6e|"
```

- 이게 **SKT USIM 침해사고 후 공유된 공식 탐지룰(공문)** 형태 — `drop TCP $HOME_NET any -> $EXTERNAL_NET $HTTP_PORTS` 로 **안→밖(outbound)** 방향. 감염 단말이 밖으로 빠져나가는/ C2로 나가는 걸 잡는 구조.
- hex = 인코딩 무관 정확 매칭(단 짧으면 오탐), hash = 파일 통째 매칭(단 1바이트만 바뀌어도 못 잡음=변종 회피).

> 🔗 `Security-Solutions.md` "content는 바이너리(헥스)도 본다" + "역방향(트로이목마) outbound 룰"의 실제 사례. 해시의 한계(변종)는 결국 행위기반(EDR·애노멀리)이 메운다.

---

## 4️⃣ Kali Linux 설치

```
- 본컴 VirtualBox에 설치 (서브컴 자원 절약 — 공격기는 필요할 때만 띄움)
- IP: 192.168.1.250 static (WAN 대역, 외부 공격자 포지션)
- 용도: pfSense WAN Snort 인라인 IPS를 때려보는 표준 공격기 (Nikto·nmap·sqlmap 등 기본 탑재)
```

> ⚠️ 공격기는 고정 IP로 — Suppress/Pass List·룰의 출발지 IP 기준이 흔들리지 않게.

---

## 5️⃣ VPN — 실제로 어떻게 연결되나 (개념 정리)

오후엔 VPN이 "스벅의 내 노트북 → 회사 서버"로 어떻게 이어지는지를 흐름으로 봤다.

```
[스벅의 나] ─ 인터넷 ─> [회사 VPN 게이트웨이] ─ 정책 검사 ─> [서버망 PC/우분투/NAS]
  IPsec key(서로 보유)            서버쪽 VPN 정책 통과해야만 서버 도달
```

- **터널 + 정책 2단** — VPN으로 묶였다고 끝이 아니라, 서버쪽 VPN에 걸린 정책을 통과해야 서버에 닿는다.
- **Site-to-Site** = 지역↔지역(망↔망). 서버가 pfSense IPsec이면 반대편도 IPsec 장비 필요(표준이라 이론상 호환되나 실무는 짝 맞추는 게 편함).
- **Client-to-Site** = 클라이언트가 아무 망에서 와서 로그인·인증 후 접속. 보통 **휴대폰 OTP(MFA)**, 그 OTP는 유일한 한 개라 복제 불가 → 잃어버리면 막힘.
- **SSL-VPN** = 인증서(TLS) 통신. 크롬으로 포털 접속→로그인→OTP. 인증서/OTP 분실 시 타격 큼.
- **입구에서 SSL Inspection** — SSL-VPN으로 들어온 직원 트래픽은 들어오자마자 복호화(SSL Inspection)해서 관제 상황판에서 다 본다. 터널만 믿지 않고 안에서 또 본다(=ZTNA 발상).

> 🔗 상세는 `Security-Solutions.md` → VPN "실제로 어떻게 연결되나 (6-16 심화)" 참고. 비유: VPN 터널=정문까지 오는 통근버스, 내린 뒤 사옥 안 행동은 로비 검색(SSL Inspection)+CCTV 상황판(관제)이 따로 본다.

---

## ⚠️ 오늘 짚어둘 것

- **인라인 IPS 전엔 Hardware Checksum Offloading OFF** — 특히 VirtualBox는 거의 필수. 안 끄면 인라인 검사가 꼬인다.
- **Block Offenders 체크 = IDS→IPS** — Legacy(사후 차단) vs Inline(실시간 drop) 모드 구분.
- **IPS 켜면 오탐=장애** — Pass List(차단 면제)·Suppress(알람 끄기)가 곧장 필요. 둘은 목적이 다르다.
- **Pass List/룰도 IP 직접입력은 하수 → Alias.**
- **커스텀 SID는 3000000+** (ET=2xxxxxx와 안 겹치게).
- **content는 hex/hash로** — 평문 문자열은 인코딩·이스케이프로 빗나간다. 해시는 파일단위지만 변종엔 무력.
- **Kali·공격기는 고정 IP** — 룰/로그의 출발지 기준이 흔들리지 않게.

---

## ➡️ 다음 할 것

- [ ] Kali로 **실제 공격 → 인라인 drop 검증** (Nikto/nmap/sqlmap을 WAN에서 쏴서 Block 동작·Blocked 탭 확인)
- [ ] Block Offenders **오탐 한 번 유도해보고** Pass List로 복구 (가용성 사고 시나리오 체험)
- [ ] 어제 이월: **SSL Bump 명시적 프록시**, **WAF(ModSecurity)**, VLAN 단방향 룰 마저
- [ ] VPN 실제 구축(pfSense IPsec / OpenVPN) — 오늘 개념을 랩으로 (2차 프로젝트)
