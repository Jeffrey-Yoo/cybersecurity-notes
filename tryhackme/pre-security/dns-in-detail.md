# DNS in Detail — TryHackMe

**날짜:** 2026-04-06
**Room:** https://tryhackme.com/room/dnsindetail

---

## 한 줄 요약
DNS = 인터넷의 전화번호부. 도메인(google.com)을 IP(142.250.196.110)로 변환해주는 시스템.
도메인의 구조, 레코드 종류, 요청 과정, 그리고 실제 사이트에서 어떻게 쓰이는지를 배운다.

---

## DNS가 하는 일
사람: 브라우저에 "google.com" 입력
DNS: google.com → 142.250.196.110 으로 변환
컴퓨터: 142.250.196.110으로 접속
→ IP 주소를 외울 필요 없이 이름으로 접속 가능!
---

## 도메인 이름의 구조 (오른쪽부터 읽는다)
www.google.com

     .com    → TLD (Top Level Domain) 최상위
   google    → SLD (Second Level Domain) 사이트 이름
     www     → Subdomain (서브도메인) 하위 구분
아파트 비유:
.com   = 서울시 (어느 도시)
google = 강남구 (어느 구역)
www    = 101동 (구역 안에서 어디)
### TLD의 두 종류
gTLD (Generic): 용도별
.com → 상업 / .org → 기관 / .edu → 교육
ccTLD (Country Code): 국가별
.kr → 한국 / .uk → 영국 / .jp → 일본
.co.uk → 영국 상업 (ccTLD)
### 서브도메인 규칙
최대 길이: 63자
전체 도메인 최대: 253자
사용 불가: _ (언더스코어)
사용 가능: 알파벳, 숫자, - (하이픈)
같은 도메인에서 서비스 구분:
mail.google.com   → 메일
drive.google.com  → 드라이브
maps.google.com   → 지도
---

## DNS 레코드 종류

### A 레코드 — 도메인 → IPv4
가장 기본! "이 도메인의 IP가 뭐야?"
mysite.com     → A → 45.33.32.156 (메인 웹서버)
api.mysite.com → A → 45.33.32.200 (API 서버)
→ 서브도메인마다 다른 서버를 연결할 수도 있음
### AAAA 레코드 — 도메인 → IPv6
A 레코드의 IPv6 버전
mysite.com → AAAA → 2001:db8::1
왜 AAAA?
IPv4 = 32비트 = A (하나)
IPv6 = 128비트 = AAAA (네 배)
### CNAME 레코드 — 도메인 → 다른 도메인
"이건 별명이야, 진짜는 저기야"
store.mysite.com → CNAME → jeff-store.myshopify.com
고객이 store.mysite.com 접속
→ DNS: "그건 jeff-store.myshopify.com으로 가"
→ 주소창에는 내 도메인이 보이지만 실제 서버는 Shopify
### MX 레코드 — 이메일 서버 주소
"이 도메인으로 이메일 보내려면 어느 서버로?"
mysite.com → MX → aspmx.l.google.com (우선순위 10)
mysite.com → MX → alt1.aspmx.l.google.com (우선순위 20)
우선순위 숫자 낮을수록 먼저 시도
10번 서버 다운 → 자동으로 20번으로 = 백업 시스템
### TXT 레코드 — 자유 텍스트 (메모/증명)
스팸 방지 (SPF):
"v=spf1 include:_spf.google.com -all"
→ "구글 서버만 이 도메인으로 메일 보낼 수 있어"
→ 해커가 내 도메인인 척 메일 보내면 차단!
도메인 소유 증명:
"google-site-verification=abc123xyz"
→ "이 도메인 소유자가 나라는 증명코드"
### 한눈에 정리
A      → 도메인 → IPv4        "IP가 뭐야?"
AAAA   → 도메인 → IPv6        "IPv6 주소가 뭐야?"
CNAME  → 도메인 → 다른 도메인   "별명이야, 저기로 가"
MX     → 도메인 → 메일 서버    "이메일은 여기로"
TXT    → 도메인 → 텍스트       "메모/인증 정보"
---

## DNS 요청 과정

브라우저에 www.google.com 입력하면:
[1단계] 내 컴퓨터 캐시
→ "최근에 찾아본 적 있나?" → 없음
[2단계] Recursive DNS 서버 (통신사 서버)
→ "우리 캐시에 있나?" → 없음
→ "내가 대신 찾아줄게"
[3단계] 찾아가는 여정:
Recursive → Root DNS 서버 (전 세계 13개)
← ".com은 이쪽 TLD 서버로 가"
Recursive → .com TLD 서버
← "google.com은 이 Authoritative 서버가 담당"
Recursive → google.com Authoritative 서버
← "142.250.196.110이야"
[4단계] 결과 전달 + 캐시 저장
Recursive → 내 컴퓨터: "142.250.196.110이야"
→ 다음에 같은 질문하면 캐시에서 바로 답!
### 각 서버 역할 비유
내 컴퓨터 캐시     = 내 머릿속 기억
Recursive 서버     = 114 안내원 ("대신 찾아드릴게요")
Root 서버          = 국가 대표 안내소 (".com은 이쪽")
TLD 서버           = 지역 안내소 ("google.com은 이쪽 담당")
Authoritative 서버 = 실제 전화번호부 ("이 IP야")
### TTL (Time To Live)
캐시 유효기간 (초 단위)
TTL = 3600 → 1시간 동안 캐시 유지
짧으면: 항상 최신, 요청 많아짐
길면:   요청 적지만, IP 바뀌었을 때 반영 느림
---

## nslookup 실습 명령어
nslookup --type=A www.website.thm       → IPv4 주소 조회
nslookup --type=AAAA www.website.thm    → IPv6 주소 조회
nslookup --type=CNAME shop.website.thm  → 별명 도메인 조회
nslookup --type=MX website.thm          → 메일 서버 조회
nslookup --type=TXT website.thm         → 텍스트 레코드 조회
---

## 심화: CNAME의 실제 사용

### Q: CNAME으로 외부 서비스를 연결한다는 건 뭘 빌려오는 거야?

CNAME 자체는 단순한 도메인 연결이지만, 그 뒤에 있는 서비스의 전체 인프라를 빌려 쓰는 것.
Shopify를 CNAME으로 연결하면 제공받는 것:
┌─────────────────────────────────┐
│ 프론트엔드: UI/디자인/장바구니     │
│ 백엔드: 결제/재고/주문/배송 추적   │
│ 보안: HTTPS/DDoS 방어/PCI 인증   │
│ 인프라: 자동 확장/CDN/백업        │
└─────────────────────────────────┘
내가 하는 것: 상품 등록, 가격 정하기, 디자인 선택
### Q: 왜 직접 안 하고 외부 서비스를 쓰는 거야?

자체 운영도 가능하다. 전부 A 레코드로 내 서버를 가리키면 됨.
하지만 현실적인 이유로 외부 서비스를 씀:
직접 운영 = 식당에서 빵도 굽고 음료도 만들고 배달도 직접
→ 다 할 수 있지만 힘들고 품질 유지 어려움
외부 서비스 = 빵은 빵집, 음료는 음료업체, 배달은 배민
→ 전문가한테 맡기니까 품질 좋고 부담 줄어듦
대기업: 자체 운영할 여력 있음
개인/스타트업: 외부 서비스가 현실적
---

## 심화: CNAME vs API — 두 가지 방법

### CNAME = 서비스 통째로 빌림 (간판만 내 것)
store.mysite.com → CNAME → myshopify.com
→ 접속하면 Shopify UI가 그대로 보임
→ 코딩 필요 없음, 커스텀 제한적
### API = 데이터/기능만 빌림 (매장은 내가 만듦)
내 서버 → Shopify API: "인기 상품 10개 줘"
Shopify → JSON 데이터로 응답
내 프론트엔드 → 받은 데이터를 내 디자인으로 표시
→ 완전한 디자인 자유
→ 프론트엔드 개발 필요
→ 실제로 많은 회사가 이 방식 사용
---

## 심화: 실제 사이트 DNS 설계 실습

jeffsecurity.com 사이트를 만든다고 가정:
┌──────────────────────┬───────┬────────────────────────────────┐
│ 도메인                │ 타입  │ 값                              │
├──────────────────────┼───────┼────────────────────────────────┤
│ jeffsecurity.com      │ A     │ 45.33.32.156 (메인 서버)        │
│ www.jeffsecurity.com  │ A     │ 45.33.32.156 (메인 서버)        │
│ portfolio.jeffsecurity│ A     │ 45.33.32.200 (포트폴리오 서버)   │
│ blog.jeffsecurity.com │ CNAME │ jeffsec.wordpress.com          │
│ store.jeffsecurity.com│ CNAME │ jeff-security.myshopify.com    │
│ jeffsecurity.com      │ MX    │ aspmx.l.google.com (10)        │
│ jeffsecurity.com      │ MX    │ alt1.aspmx.l.google.com (20)   │
│ jeffsecurity.com      │ TXT   │ v=spf1 include:_spf.google -all│
│ jeffsecurity.com      │ TXT   │ google-site-verification=abc123│
└──────────────────────┴───────┴────────────────────────────────┘
사용자 입장:
jeffsecurity.com       → 메인 서버 → 메인 페이지
blog.jeffsecurity.com  → WordPress → 블로그
store.jeffsecurity.com → Shopify → 쇼핑몰
portfolio              → 별도 서버 → 포트폴리오
메일 전송               → 구글 → Gmail 수신
해커가 사칭 메일 전송    → TXT(SPF) 확인 → 차단!
하나의 브랜드(jeffsecurity.com) 아래
실제로는 5개의 다른 서버에서 서비스가 돌아가고 있음
---

## 심화: API 아키텍처와 보안

API 방식으로 외부 서비스를 연결할 때의 데이터 흐름과 보안 위협:

### WordPress API (블로그)
데이터 흐름:
내 서버 → WordPress API: "최신 글 5개 줘" (HTTPS)
WordPress → JSON 응답: {title, date, content}
내 프론트엔드 → 내 디자인으로 표시
보안 위협:
⚠️ API 키 노출 → 키를 서버에만 저장, 환경변수 사용
⚠️ 중간자 공격 → HTTPS 필수 (443 포트)
⚠️ 과다 요청   → Rate Limiting (초당 요청 수 제한)
### Shopify API (쇼핑몰)
데이터 흐름:
내 서버 → Shopify API: "상품 목록 줘" + API 키 인증
Shopify → JSON 응답: {name, price, stock}
결제 시 → Shopify 결제 페이지로 리다이렉트 (PCI 보안)
결제 완료 → Shopify가 내 서버에 "결제 됐어" 알림 (Webhook)
보안 위협:
⚠️ API 키 탈취   → 키 권한 최소화 (읽기만 허용)
⚠️ 가격 변조     → 결제는 Shopify 서버에서 직접 처리 (내 서버 안 거침)
⚠️ 가짜 결제 알림 → Webhook 서명 검증 (Shopify가 보낸 게 맞는지 확인)
### Google OAuth (로그인)
데이터 흐름:
사용자 → "구글로 로그인" 클릭 → 구글 로그인 페이지로 이동
구글에서 직접 ID/PW 입력 (내 서버를 거치지 않음!)
구글 → 내 서버에 인증 코드 전달
내 서버 → 구글에 코드+API키로 토큰 요청
구글 → 토큰 발급 → 사용자 이름/이메일 조회 가능
보안 핵심:
🔐 사용자 비밀번호를 내 서버에 저장하지 않음
→ 내 사이트가 해킹당해도 구글 비밀번호는 안전!
⚠️ 토큰 탈취 → 유효기간 짧게 + HTTPS 필수
⚠️ 가짜 로그인 페이지 → 리다이렉트 URL 사전 등록
### 모든 API 통신의 OSI 7계층 경로
7계층 Application:  HTTPS로 API 요청
6계층 Presentation: TLS 암호화
5계층 Session:      API 서버와 세션 유지
4계층 Transport:    TCP 포트 443으로 전송
3계층 Network:      API 서버 IP로 라우팅
2계층 Data Link:    게이트웨이 MAC으로 전달
1계층 Physical:     전기 신호로 전송
---

## 보안관제에서 DNS가 중요한 이유

### DNS로 할 수 있는 세 가지 보안 업무

**1. 의심 도메인 조사**
로그에서 내부 PC가 evil-update.com에 매 5분마다 접속 발견
$ nslookup --type=A evil-update.com → 러시아 IP
$ nslookup --type=TXT evil-update.com → (없음)
$ nslookup --type=MX evil-update.com → (없음)
정상 사이트: A + MX + TXT 전부 있음
이 사이트: A만 있음 = 악성코드 C&C 서버 가능성!
→ 도메인/IP 차단 + 감염 PC 격리
**2. 해커의 정찰 대응**
해커가 공격 전에 하는 것:
nslookup으로 모든 서브도메인 조사
→ dev 서버 외부 노출 발견 → 공격 진입점
→ SPF 설정 약함 발견 → 피싱 메일 가능
보안관제 대응:
→ 불필요한 서브도메인 외부 노출 금지
→ SPF를 ~all이 아닌 -all로 설정 (강력 차단)
**3. DNS 하이재킹 탐지**
해커가 도메인 관리 계정 탈취 후 A 레코드 변경:
mybank.com → 해커 서버 IP
고객이 접속하면 가짜 은행 페이지 → 비밀번호 탈취
탐지 방법:
매일 주요 도메인 A 레코드 자동 확인
→ IP가 갑자기 바뀌면 긴급 알림!
---

## 새로 알게 된 용어
| 용어 | 뜻 |
|------|-----|
| DNS | Domain Name System, 도메인 → IP 변환 |
| TLD | Top Level Domain (.com, .kr) |
| gTLD | Generic TLD (용도별: .com, .org) |
| ccTLD | Country Code TLD (국가별: .kr, .uk) |
| SLD | Second Level Domain (google, naver) |
| Subdomain | 서브도메인 (www, mail, blog) |
| A Record | 도메인 → IPv4 |
| AAAA Record | 도메인 → IPv6 |
| CNAME Record | 도메인 → 다른 도메인 (별명) |
| MX Record | 메일 서버 주소 + 우선순위 |
| TXT Record | 자유 텍스트 (SPF, 인증) |
| SPF | 이메일 발송 허용 서버 목록 (스팸 방지) |
| Recursive DNS | 대신 찾아주는 서버 (통신사) |
| Root DNS | 최상위 안내 서버 (전 세계 13개) |
| Authoritative DNS | 도메인의 실제 레코드 보관 서버 |
| TTL | 캐시 유효기간 (초) |
| nslookup | DNS 레코드 조회 명령어 |
| API | 외부 서비스의 데이터/기능만 가져오는 방식 |
| Webhook | 외부 서버가 내 서버에 알림을 보내는 방식 |
| OAuth | 구글 등 외부 인증으로 로그인하는 방식 |
| DNS Hijacking | DNS 레코드를 변조해서 가짜 서버로 유도하는 공격 |
| C&C Server | 악성코드에 명령을 내리는 해커의 서버 |

## 궁금한 점
- DNS over HTTPS(DoH)가 보안에 어떤 영향을 미치는지?
- 실무에서 서브도메인 열거(enumeration) 도구는 뭘 쓰는지?
- API 키 관리의 베스트 프랙티스는?
