# 2026-06-24 — SYN Flooding 실증·Snort 임계치 차단 + Zabbix(NMS) 모니터링 구축

과정2 17일차. 오전엔 DDoS 연장선으로 **칼리를 SPAN(미러)에 물려** 오가는 패킷을 와이어샤크로 보고, **SYN Flooding**을 직접 쏴서 Snort 임계치 룰로 끊었다. 오후엔 "공격이 들어오면 어디를 봐야 하나"라는 질문에서 출발해 **SNMP/NMS** 개념을 잡고, 무료 오픈소스 NMS인 **Zabbix 7.4**를 Ubuntu 24.04에 직접 올려 web_server를 모니터링했다.

> 🔗 연결: 6/22~23에 와이어샤크로 L4·UDP Flooding을 "관찰"했고, 오늘은 SYN Flood를 "쏘고 막는" 데까지 갔다. 그리고 6-23의 **골든타임**(자원이 맥스 치기 직전 끊어야 한다)을 잡으려면 평소부터 자원을 봐야 한다 → 그게 NMS(Zabbix) 도입 동기.

> FIMS 연결: 과정1에서 서비스가 살아있나 직접 확인했던 걸, 이번엔 **SNMP로 자동 수집해 한 화면(Zabbix)** 에 띄우는 인프라판으로 끌어올렸다.

---

## 1️⃣ SPAN으로 트래픽 보기 + 프로토콜을 맞춰야 도달한다

칼리를 ESXi에서 **어댑터1=WAN, 어댑터2=SPAN(미러)** 으로 줘서, SPAN으로 복제되는 패킷을 와이어샤크로 보게 세팅했다.

```
pfSense 포트포워딩 53번(DNS)으로 WAN 접근 → 내부 DNS까지 정상 도달
  → 프로토콜(UDP/53)을 정확히 맞췄으니 응용프로그램(BIND)까지 부하가 닿음
ICMP(핑)로 같은 서버를 때리면 → 서버까지 도달 못 함 (포워딩된 건 53뿐)
```

> 💡 교훈: **열린 포트/프로토콜에 정확히 실어야 부하가 응용계층까지 닿는다.** 아무 프로토콜이나 쏜다고 서버가 아프지 않다 — 안 연 프로토콜은 방화벽 앞단에서 그냥 버려진다. (Attack-Defense.md "프로토콜을 맞춰야 도달한다")

---

## 2️⃣ SYN Flooding — 왜 먹히고, 어떻게 막나

### 원리 (half-open으로 backlog 고갈)

```
3-way 중 마지막 ACK를 일부러 안 줌 → 반쪽 연결(half-open)이 backlog queue를 가득 채움
OS backlog는 보통 300~400개 → 늘려도(튜닝) 결국 한계에 봉착
→ 그래서 CPS(초당 연결 시도)가 빡세게 올라가야 큐 비울 틈을 안 줘서 터진다
```

### 위조 IP가 "실존 IP"면 오히려 방어된다 (그림 3-56)

```
SRC IP를 2.2.2.2(실존)로 위조 → 서버가 2.2.2.2로 SYN-ACK 보냄
→ 2.2.2.2는 SYN 보낸 적 없음 → RST로 끊음 → 서버 backlog에서 그 연결 삭제(회복)
```
- ⚠️ 즉 **SYN Flood 성공 조건 = SYN 이후 아무 응답도 안 와야** 한다 → 보통 라우팅 안 되는 위조 IP를 쓴다.

### Snort 임계치 룰로 차단

```
alert tcp any any -> $HOME_NET any ( \
    msg:"SYN Flood Detected"; flags:S; \
    threshold: type both, track by_src, count 20, seconds 2; \
    sid:3000010; rev:1; )
```

| 옵션 | 역할 |
|---|---|
| `flags:S` | SYN만 세팅된 패킷 매칭 (S=SYN, A=ACK, R=RST, P=PSH). 정상 SYN과 모양이 같으니 개수로 구분 |
| `threshold type both` | limit+threshold 결합 — 임계치 넘을 때 발동하되 알림 폭주 억제 |
| `track by_src` | 출발지 IP 기준 카운트 |
| `count 20 / seconds 2` | "2초에 같은 소스 SYN 20개 초과" = 공격 |

> 🔗 본격 방어는 앞단 **Anti-DDoS의 syncookies / First SYN Drop** — Anti-DDoS가 서버 대신 3-way를 먼저 맺어보고 검증된 손님만 서버로 넘긴다. (Security-Solutions.md "Anti-DDoS가 SYN Flood를 막는 실제 메커니즘")

> hping3 옵션(공격 발사): `-1` ICMP / `-2` UDP / `-8` SCAN / `-S` SYN / `-p` 목적지포트 / `-i u100` 전송주기(마이크로초) / `-c` 개수 / `--rand-source` 출발지 IP 랜덤 위조 / `--flood` 최대속도. (Attack-Defense.md "hping3")

---

## 3️⃣ 왜 NMS인가 — SNMP로 평소를 알아야 골든타임을 잡는다

공격이 들어오면 "서비스가 도는지, 어떤 어플라이언스가 죽었는지"를 봐야 한다. 그걸 한곳에 모아주는 게 **NMS**, 모으는 표준 프로토콜이 **SNMP**.

```
SNMP = 라우터·스위치·서버 상태를 원격 모니터링/설정하는 표준 프로토콜
NMS  = SNMP로 수집한 값을 한 화면에 그래프·알림으로 보여주는 시스템
```
- ⚠️ **평균값(baseline)** 이 핵심 — 평소 트래픽/파일크기엔 평균선이 있고, 침투·이상이 생기면 거기서 갑자기 벗어난다. 코어 시스템엔 필수.
- 🔗 NMS ≠ ESM: NMS=가용성/성능 모니터링, ESM=보안 로그 상관분석. (Security-Solutions.md "NMS와 SNMP")

---

## 4️⃣ Zabbix 7.4 설치 — 명령어가 "왜·어떻게 이어지는지"

> 환경: NMS 서버 10.10.60.10(MGT VLAN) / Ubuntu 24.04 / DB는 MariaDB / 접속은 pfSense 포트포워딩 WAN:1234 → 10.10.60.10:80

**큰 그림 먼저.** Zabbix는 부품 4개가 맞물린다. 설치 순서는 이 의존관계를 그대로 따라간다:
```
zabbix-server (수집 엔진)  ─ 값을 DB에 저장 ─→  MariaDB (시계열 저장소)
        │                                              ↑ 스키마(빈 테이블 틀)를 먼저 부어야 함
        └─ 웹UI를 통해 보여줌 ─→ Apache + PHP (대시보드 웹서버)
                                          ↑ 에이전트(10050)가 각 서버의 CPU/디스크/트래픽을 보고
```
→ 그래서 ① 패키지 설치 → ② DB 만들고 권한 → ③ 빈 DB에 **스키마 부음** → ④ 서버가 그 DB를 쓰도록 **비번 알려줌** → ⑤ 서비스 기동 → ⑥ 웹UI에서 DB 연결 마무리 → ⑦ 대상에 에이전트 → ⑧ 호스트 등록. 한 단계라도 빠지면 다음이 안 선다.

### 0. 사전 — root 전환
```bash
sudo -s          # 이후 명령이 전부 설치/시스템 변경이라 매번 sudo 치지 않게 root 쉘로
```

### 1. Zabbix 저장소(repo) 추가 — "어디서 받을지"부터 알려준다
```bash
wget https://repo.zabbix.com/zabbix/7.4/release/ubuntu/pool/main/z/zabbix-release/zabbix-release_latest_7.4+ubuntu24.04_all.deb
dpkg -i zabbix-release_latest_7.4+ubuntu24.04_all.deb
apt update
```
- `wget …deb` : Zabbix **저장소 등록용 deb 패키지**를 내려받는다. 이 deb는 프로그램 본체가 아니라 "apt가 zabbix를 어디서 받아올지" 알려주는 **주소록(.list + GPG 키)** 이다.
- `dpkg -i …deb` : 그 deb를 설치 → `/etc/apt/sources.list.d/`에 Zabbix 공식 저장소 주소와 서명키가 등록됨.
- `apt update` : 방금 추가된 저장소 목록을 apt가 다시 읽어들임. **이걸 안 하면** 다음 단계 `apt install zabbix-...` 가 "패키지 없음"으로 실패한다.
> 💡 즉 1단계는 다운로드가 아니라 **"공식 출처를 시스템에 등록"** 하는 것. 그래야 2단계 apt가 정품·최신 zabbix를 신뢰된 경로로 당겨온다.

### 2. Zabbix 패키지 설치 — 부품 4개를 한 번에
```bash
apt install zabbix-server-mysql zabbix-frontend-php zabbix-apache-conf zabbix-sql-scripts zabbix-agent
```
| 패키지 | 역할 |
|---|---|
| `zabbix-server-mysql` | **수집 엔진 본체**(MySQL/MariaDB 연동판). 에이전트한테 값을 당겨와 임계치 판단 |
| `zabbix-frontend-php` | 웹UI(대시보드)를 그리는 **PHP 코드** |
| `zabbix-apache-conf` | 그 PHP를 **Apache 웹서버에 연결**해주는 설정 |
| `zabbix-sql-scripts` | 빈 DB에 부을 **테이블 스키마(.sql)** 모음 → 5단계에서 사용 |
| `zabbix-agent` | 이 서버 자신을 모니터링할 **에이전트**(localhost용) |

### 3. MariaDB 설치·기동·자동시작 — 데이터 그릇부터
```bash
apt install mariadb-server
systemctl start mariadb      # 지금 당장 DB 데몬 켜기
systemctl enable mariadb     # 부팅 때마다 자동 시작 등록
```
- `start` = 지금 한 번 켬 / `enable` = 재부팅해도 자동으로 켜지게 등록. **둘은 별개** — start만 하면 리부팅 후 꺼져 있다.

### 4. DB·계정 생성 — Zabbix 전용 그릇과 열쇠
```bash
mysql -uroot -p
```
```sql
create database zabbix character set utf8mb4 collate utf8mb4_bin;
create user zabbix@localhost identified by 'zabbix1234';
grant all privileges on zabbix.* to zabbix@localhost;
set global log_bin_trust_function_creators = 1;
quit;
```
- `create database zabbix … utf8mb4_bin` : zabbix 전용 빈 DB 생성. **문자셋을 utf8mb4_bin으로 못 박는 건 Zabbix 요구사항** — 안 맞추면 다음 import가 깨진다.
- `create user … identified by 'zabbix1234'` : zabbix 서버가 DB에 접속할 **전용 계정+비번** 생성. (root로 붙이지 않는 게 최소권한 원칙)
- `grant all privileges on zabbix.*` : 그 계정에게 **zabbix DB에 한해서만** 전권 부여. 다른 DB는 못 건드림.
- `set global log_bin_trust_function_creators = 1` : 다음 스키마 import엔 트리거/함수가 들어있는데, 바이너리 로그가 켜진 MariaDB는 이걸 기본 차단한다. **잠깐 1로 풀어 import를 허용** → 끝나면 6단계에서 다시 0으로 닫는다(보안 원복).

### 5. DB 스키마 import — 빈 그릇에 테이블 틀을 붓는다
```bash
zcat /usr/share/zabbix/sql-scripts/mysql/server.sql.gz | mysql --default-character-set=utf8mb4 -uzabbix -p zabbix
```
- `zcat …server.sql.gz` : 2단계 `zabbix-sql-scripts`가 깔아둔 **압축된 SQL 스키마**를 압축 푼 채로 출력.
- `| mysql -uzabbix -p zabbix` : 그 SQL을 **파이프로 mysql에 흘려넣어** zabbix DB 안에 수백 개 테이블을 한 번에 생성. (4단계에서 만든 zabbix 계정으로 접속)
> 💡 4단계가 "빈 냉장고를 들여놓은 것"이라면, 5단계는 "그 안에 선반·칸막이를 다 짜넣는 것". 이게 있어야 server가 값을 어디에 넣을지 안다.

### 6. log_bin_trust 원복 — 열어둔 문 닫기
```bash
mysql -uroot -p
set global log_bin_trust_function_creators = 0;   # import 위해 풀었던 걸 다시 잠금
quit;
```

### 7. 서버 설정파일에 DB 비번 알려주기 — 엔진↔DB 연결
```bash
nano /etc/zabbix/zabbix_server.conf
# DBPassword=zabbix1234   ← 4단계에서 만든 비번을 그대로 적는다
```
- ⚠️ **여기 비번과 4단계 DB 비번이 한 글자라도 다르면** server가 DB에 못 붙어 "Access denied"로 죽는다. (오늘 트러블슈팅 2번이 정확히 이거였음)

### 8. 서비스 기동 + 자동시작
```bash
systemctl restart zabbix-server zabbix-agent apache2
systemctl enable  zabbix-server zabbix-agent apache2
```
- 세 부품(수집엔진·로컬에이전트·웹서버)을 한 번에 재시작하고, 부팅 자동시작까지 등록. 이제 백엔드는 다 섰다.

### 9. 웹UI 초기 설정 — 마지막 연결을 GUI에서
- 브라우저: `http://<접속주소>:1234/zabbix` (pfSense 포워딩 1234 → 10.10.60.10:80)
- 초기 계정: **Admin / zabbix**
- DB 연결 입력값: type `MySQL` / host `localhost` / port `0`(기본) / name `zabbix` / user `zabbix` / pw `zabbix1234`
> 💡 7단계가 "서버 데몬용 DB 비번", 9단계는 "웹UI용 DB 연결" — **둘 다 같은 zabbix DB를 가리키지만 설정 위치가 달라** 둘 다 맞춰줘야 한다.

### 10. ⚠️ 트러블슈팅
- **locale Fail** → `apt install locales -y` 후 `locale-gen en_US.UTF-8` (시스템 로케일이 없어 웹 사전점검 실패)
- **DB 접속 실패(Access denied)** → 원인은 7단계 `DBPassword` ↔ 실제 DB 비번 불일치. `ALTER USER 'zabbix'@'localhost' IDENTIFIED BY '…';` 로 통일 후 conf 수정 → `systemctl restart zabbix-server`.

---

## 5️⃣ 모니터링 대상에 에이전트 심기 + 호스트 등록

### 대상 서버에서 (에이전트 = 첩보원)
```bash
# (NMS와 동일하게 zabbix repo 추가 후)
apt install zabbix-agent -y
nano /etc/zabbix/zabbix_agentd.conf
#   Server=10.10.60.10        ← 수동 체크(passive): 이 NMS만 나에게 값 요청 허용
#   ServerActive=10.10.60.10  ← 능동 보고(active): 내가 NMS로 먼저 값 push
#   Hostname=서버이름          ← 웹UI 호스트명과 일치해야 매칭됨
systemctl restart zabbix-agent
systemctl enable  zabbix-agent
```
- `Server` vs `ServerActive` 차이: 전자는 **NMS가 당겨가는**(passive) 통로 허용 목록, 후자는 **에이전트가 밀어 보내는**(active) 대상. 둘 다 적어두면 양쪽 다 동작.

### 웹UI에서 호스트 추가
`Data collection → Hosts → Create host`
- Host name: 서버이름(에이전트 Hostname과 동일) / Templates: **Linux by Zabbix agent** / Host groups: Linux servers
- Interfaces: Add → Agent → IP: 대상 내부IP, **Port: 10050**(Zabbix 에이전트 표준 포트)

### 현재 모니터링 중
```
Zabbix server (127.0.0.1:10050)  → ZBX 연결 완료(초록)
web_server    (10.10.10.10:10050)→ ZBX 연결 완료(초록)
```
- Filesystems 대시보드에서 `/`·`/boot` 사용률 파이차트, 시간대별 사용량 그래프 정상 수집 확인. → **이제 평소 baseline이 그래프로 쌓인다 = 골든타임 감지의 토대.**

---

## 오늘의 삽질 / 핵심 정리

- ⚠️ **repo 추가(1단계)를 "그냥 다운로드"로 오해하기 쉽다.** 실제론 apt에 정품 출처를 등록하는 단계 → `apt update`까지 해야 install이 먹는다.
- ⚠️ **DB 비번 3곳(생성·server.conf·웹UI)을 일치**시켜야 한다. 하나만 어긋나도 Access denied로 막힌다.
- ⚠️ `log_bin_trust_function_creators`는 **import 동안만 1, 끝나면 0** — 열고 닫는 한 쌍으로 기억.
- ⚠️ `systemctl start` ≠ `enable` — start는 지금, enable은 부팅 자동. 둘 다 해야 리부팅 후에도 산다.
- SYN Flood는 **응답 없는 위조 IP**로 쏴야 효과 — 실존 IP 위조하면 그쪽 RST로 오히려 회복됨.

> 🔗 이론 정리본: SYN Flood/Snort 룰·프로토콜 도달 → **Attack-Defense.md**, Anti-DDoS syncookies·NMS/SNMP·Zabbix 구조 → **Security-Solutions.md**, MTU 확인(`netsh`) → **OSI-7Layer.md**.
