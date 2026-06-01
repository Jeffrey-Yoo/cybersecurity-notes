# 🏗️ FIMS 프로젝트 — 과정1 완전 회고

> **FIMS (Franchise Integrated Management System)**
> 소규모 프랜차이즈 사장님에게 대기업급 보안·운영 인프라를 통째로 제공하는 SaaS.
> 팀명: 9조 사장님을9해조 / URL: `https://9haejo.duckdns.org`

---

## 전체 아키텍처

```
🌍 인터넷
    ↓ HTTPS(443) / WireGuard(UDP 51820)
AWS EC2 (공인 3.35.66.99 / WG 10.8.0.1)
Apache(리버스프록시+SSL) · Wazuh Manager · ETL(:9000) · WG서버 · S3
    ↓ WireGuard 터널               ↓ WireGuard 터널
어플라이언스 (WG 10.8.0.2)        DB (WG 10.8.0.3)
Suricata · Wazuh Agent           MariaDB · socat 중계

── IDC (VirtualBox VM 6대) ── 관리망56 공통 + 서비스망 구간분리 ──
어플라이언스 ─30─ DMZ ─40─ WAS ─70─ DB ─80─ Storage
(경비실)      (로비)  (사무실) (금고앞) (금고/폐쇄망)
```

| 서버 | 관리망(56) | 서비스망 | WireGuard | 역할 |
|---|---|---|---|---|
| DNS | 56.10 | — | 없음 | bind9 내부 DNS |
| 어플라이언스 | 56.11 | 30.1 | 10.8.0.2 | Suricata · Wazuh Agent |
| DMZ | 56.12 | 40.1 | 없음 | Apache (관문) |
| WAS | 56.13 | 40.2 / 70.1 | 없음 | FastAPI :8000 |
| DB | 56.14 | 70.2 / 80.1 | 10.8.0.3 | MariaDB + socat 중계 |
| Storage | 56.15 | 80.2 | 없음 | rsync 백업 폐쇄망 |

---

## AWS 인프라

```
리전: ap-northeast-2 (서울)
VPC: fims-vpc (10.0.0.0/16)
  서브넷: fims-public (10.0.1.0/24) / fims-private (10.0.2.0/24)
IGW: fims-igw → 라우팅 테이블(0.0.0.0/0 → IGW)로 퍼블릭 서브넷만 인터넷 연결
보안그룹: fims-siem-sg (443 전체, 51820/udp 전체, 22 내 IP만)
EC2: fims-siem (Ubuntu 22.04, t3.micro, 키페어 fims-key.pem)
Elastic IP: 3.35.66.99
S3: fims-backup (퍼블릭 액세스 4개 전부 차단)
```

> 💡 서브넷이 "퍼블릭"인 건 IP가 아니라 라우팅 테이블이 IGW로 가는 길을 가졌느냐로 결정됨
> 서울 리전 프리티어는 t2 아닌 t3.micro

---

## VirtualBox VM 어댑터 구성

```
어댑터1  enp0s3  NAT                 → 인터넷 egress (apt 설치용)
어댑터2  enp0s8  내부 네트워크        → 서비스망 (30/40/70/80, VM끼리만)
어댑터3  enp0s9  호스트 전용          → 관리망 192.168.56.x (윈도우에서 SSH)
```

> VirtualBox 호스트 전용 기본 대역이 56이라 관리망으로 씀
> 내부망은 호스트·인터넷에서 안 보임 = 망분리

---

## WireGuard VPN

```ini
# /etc/wireguard/wg0.conf (AWS)
[Interface]
Address = 10.8.0.1/24
ListenPort = 51820
PrivateKey = <AWS 개인키>

[Peer]   # 어플라이언스
PublicKey = <어플라이언스 공개키>
AllowedIPs = 10.8.0.2/32

[Peer]   # DB
PublicKey = <DB 공개키>
AllowedIPs = 10.8.0.3/32
```

```bash
systemctl enable wg-quick@wg0
systemctl start  wg-quick@wg0
wg show                         # 핸드셰이크·피어 연결 확인
```

> ⚠️ 삽질: DB WireGuard 불통 = DB wg0.conf의 PrivateKey에 AWS 개인키 잘못 입력
> → DB 개인키로 교체. "개인키=신원, 짝 안 맞으면 안 붙음"

---

## socat 중계 (프로젝트 설계의 핵심)

WAS는 VPN 없음 → AWS에서 직접 못 감
DB만이 내부망(70) + WireGuard(10.8.0.3) 양쪽을 다 아는 유일한 서버
→ DB에 socat 설치해 "DB:8000 → WAS:8000" 중계

```
사용자 → AWS Apache (ProxyPass /api → 10.8.0.3:8000)
       → WireGuard 터널
       → DB(10.8.0.3:8000) → socat → WAS(192.168.70.1:8000)
                                       → MariaDB(192.168.70.2:3306)
```

```ini
# /etc/systemd/system/fims-socat.service
ExecStart=/usr/bin/socat TCP-LISTEN:8000,fork TCP:192.168.70.1:8000
# fork = 동시 연결 허용
Restart=always
```

> ⚠️ 삽질: exit-code 1 = 8000 좀비 점유 → pkill socat 후 재시작

---

## Apache 리버스 프록시 + HTTPS

```apache
<VirtualHost *:443>
    ServerName 9haejo.duckdns.org
    Alias /app /var/www/html/app
    <Directory /var/www/html/app>
        FallbackResource /app/index.html
    </Directory>
    ProxyPass        /api http://10.8.0.3:8000
    ProxyPassReverse /api http://10.8.0.3:8000
    SSLCertificateFile    /etc/letsencrypt/live/9haejo.duckdns.org/fullchain.pem
    SSLCertificateKeyFile /etc/letsencrypt/live/9haejo.duckdns.org/privkey.pem
</VirtualHost>
```

```bash
apt install certbot python3-certbot-apache -y
certbot --apache -d 9haejo.duckdns.org
certbot renew --dry-run
```

> 구글 OAuth가 HTTPS 요구 → 인증서 필요 → 도메인 필요 → DuckDNS

---

## DB 설계 (MariaDB · fims 13개 테이블)

```
companies → branches → users (멀티테넌트 FK 계층)
pos_transactions, cctv_events, reviews(sentiment)
security_alerts, blacklist_ips, backup_logs, login_logs
```

---

## FastAPI 백엔드 (WAS)

```
routers: auth, dashboard, alerts, community, ingest(수집입구), webhook(Wazuh수신)
JWT = 출입증 (SECRET_KEY 64자, 8h 세션)
uvicorn = fims-was.service로 상시 가동
```

> ⚠️ 삽질1: 라우트 순서 — /posts/detail/{id}가 /posts/{company_id} 아래라 오인 → detail 위로
> ⚠️ 삽질2: JWT sub(문자열) vs author_id(정수) → int 캐스팅

---

## React PWA 프론트엔드

- React 18 + MUI v5 + @react-oauth/google, homepage:"/app"

| 증상 | 원인 | 해결 |
|---|---|---|
| recharts 안 보임 | overflow-x → 폭 0 | 규칙 제거 + width 99% + debounce |
| 다크모드 흰배경 | 하드코딩 | rgba() 반투명 교체 |
| 탭 닫으면 로그인 풀림 | sessionStorage | localStorage + 8h 만료 |
| 업데이트 미반영 | 서비스워커 캐시 | 초기화 + Ctrl+Shift+R |

> 모바일 깨짐은 전역(html/body/#root)부터 잡고 컴포넌트로

---

## 데이터 수집 파이프라인

```
어플라이언스 collector.py (POS/CCTV 더미, 1분 cron)
    ↓ WireGuard
AWS ETL :9000 (Pandas 정제)
    ↓ DB socat
WAS :8000 /ingest → MariaDB
```

> 정적파일만 DMZ/AWS, 데이터는 JS가 WAS API 호출로 가져옴. 적재경로 ≠ 조회경로

---

## Suricata (IDS)

```bash
# local.rules
alert tcp any any -> $HOME_NET any
  (msg:"FIMS Port Scan Detected"; flags:S;
   threshold:type both,track by_src,count 5,seconds 10; sid:9000001; rev:1;)
```

> ⚠️ 삽질1: fast.log 빔 = 룰 경로 불일치 → /var/lib/suricata/rules로 복사
> ⚠️ 삽질2: 앱 알림 X = forwarder 기본 API키 vs WAS 실제 키 불일치 → 일치시킴
> 기본 룰 경로: /var/lib/suricata/rules/

---

## Wazuh (SIEM)

```
Suricata 탐지 → Agent → Manager (레벨판정 ≥12 CRITICAL / ≥7 WARN)
→ webhook → WAS → security_alerts → 앱알림
(위험 IP는 blacklist_ips → 어플라이언스 폴링 차단)
```

> ⚠️ 클라우드 캐시 사건: CVE DB 자동 다운로드로 vd 폴더 10GB+ / /var/ossec/tmp 누적
> → rm -rf /var/ossec/tmp/* + vulnerability-detection 비활성화 + Swap 2GB
> → 97% → 36%
> 디버깅: du -sh /var/ossec/* | sort -rh | head

---

## 백업 파이프라인

```
DB cron (새벽2시) → backup.sh
  ① mysqldump → ② sha256sum → ③ rsync(Storage) → ④ scp(AWS)
  ⑤ ETL /backup/receive → Groq AI 체크섬 판정
  ⑥ integrity==true 일 때만 → S3
```

저장 3중화: IDC Storage(폐쇄망) → S3 → DR

---

## 웹앱 보안 9가지

```
1. HTTPS          Let's Encrypt
2. JWT 인증        SECRET_KEY 64자, 8h
3. Rate Limit     slowapi, 로그인 5분 5회
4. 로그인 IP 기록  login_logs (감사)
5. CORS 제한       허용 도메인 한정
6. 망분리          30/40/70/80
7. WireGuard       암호화 터널
8. S3 퍼블릭 차단  백업 버킷 차단
9. PrivateRoute    인증 없으면 전체 차단
```

> 로그인 5회 차단 = slowapi (Fail2Ban 아님)

---

## 주요 삽질 모음

| 문제 | 원인 | 해결 |
|---|---|---|
| .pem 권한 거부 | 키 파일 너무 열림 | icacls로 상속제거+나만 읽기 |
| DB WireGuard 불통 | AWS 개인키 잘못 입력 | DB 개인키로 교체 |
| DB↔Storage 핑 불통 | 내부망 이름 불일치 (storage vs data) | 이름 통일 |
| socat exit-code 1 | 8000 포트 좀비 점유 | pkill socat 후 재시작 |
| Suricata fast.log 빔 | 룰 경로 불일치 | /var/lib/suricata/rules로 복사 |
| 앱 알림 안 옴 | forwarder 기본 API키 vs 실제 키 불일치 | .env 키 일치 |
| FastAPI 라우트 충돌 | detail이 {id} 아래 | detail 위로 |
| JWT sub↔author_id | 문자열 vs 정수 | int 캐스팅 |
| WAS 구글로그인 실패 | NAT 제거 후 oauth2.googleapis.com 접근불가 | NAT 어댑터 재추가 |
| Wazuh OOM | CVE DB 10GB+, tmp 누적 | tmp 삭제 + vuln비활성 + Swap 2GB |
| 전체 UTC | 기본 타임존 | Asia/Seoul + MariaDB +09:00 |
| fims-was failed | nohup+systemd 8000 충돌 | pkill uvicorn → systemd 통일 |

---

## 기술 스택 버전

```
Ubuntu 22.04 / VirtualBox 7.x (Win11 25H2) / Python 3.12
FastAPI + uvicorn / React 18 + MUI v5 / MariaDB 10.11.14
WireGuard / Apache 2.4 / Suricata 7.0.3 / Wazuh 4.14.5
socat / slowapi / Groq (llama-3.1-8b-instant) / S3 / Let's Encrypt / DuckDNS
```

---

## 실무 연결 정리

| FIMS에서 만든 것 | 실무에서의 위치 |
|---|---|
| 어플라이언스 VM (Suricata) | NGFW/UTM 소프트웨어 버전 |
| AWS 보안그룹 + UFW | 1차 방화벽 (L4) |
| VirtualBox 내부 네트워크 세그멘테이션 | 소프트웨어 망분리 (VLAN의 약한 버전) |
| WireGuard 터널 | VPN 암호화 터널 |
| socat 중계 | 망연계 게이트웨이 발상과 동일 (단, 검사/승인/로그 없음) |
| Groq AI 체크섬 검증 | 무결성 확인 자동화 |

> "오픈소스로 IDS-SIEM-차단 파이프라인을 직접 엮어봐서, 상용 NGFW/DLP 콘솔을 봐도 박스 안에서 무슨 일이 일어나는지 안다"
