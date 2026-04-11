# HTTP in Detail — TryHackMe

**날짜:** 2026-04-06
**Room:** https://tryhackme.com/room/httpindetail

---

## 한 줄 요약
DNS가 "주소를 찾아주는 전화번호부"라면, HTTP는 "찾은 주소에 가서 실제로 대화하는 방법".
URL 구조, 요청/응답, 메서드, 상태 코드, 헤더, 쿠키를 배운다.

---

## HTTP vs HTTPS
HTTP  = 대화 내용이 그대로 보임 (엽서)
HTTPS = 대화 내용이 암호화됨 (봉투 편지)
차이는 딱 하나: 암호화 (S = Secure)
나머지 규칙은 완전히 동일
🔒 자물쇠 있음 = HTTPS (안전)
⚠️ 자물쇠 없음 = HTTP (위험)
---

## URL 구조
http://user:password@tryhackme.com:80/view-room?id=1#task3
http://          → Scheme: 어떤 프로토콜?
user:password@   → User: 로그인 정보 (거의 안 씀)
tryhackme.com    → Host: 어느 서버?
:80              → Port: 어느 문?
/view-room       → Path: 서버 안 어느 페이지?
?id=1            → Query String: 추가 정보
#task3           → Fragment: 페이지 내 위치
건물 비유:
Scheme       = 이동 수단 (자동차/버스)
Host         = 목적지 건물
Port         = 입구 (1층 정문)
Path         = 건물 안 위치 (3층 회의실)
Query String = 세부 요청 ("1번 회의실요")
Fragment     = 회의실 안 자리 (3번 자리)
---

## HTTP 요청 (Request) — 내가 서버에 보내는 것
GET / HTTP/1.1                ← 메서드 / 경로 / 프로토콜 버전
Host: tryhackme.com           ← 어느 서버?
User-Agent: Mozilla/5.0       ← 내 브라우저 정보
Referer: https://google.com   ← 어디서 왔는지
← 빈 줄: "요청 끝!"
---

## HTTP 응답 (Response) — 서버가 나에게 보내는 것
HTTP/1.1 200 OK               ← 프로토콜 버전 / 상태 코드
Server: nginx/1.15.8           ← 서버 소프트웨어
Content-Type: text/html        ← 데이터 종류
Content-Length: 98             ← 데이터 크기 (빠진 데이터 없는지 확인용)
Set-Cookie: name=adam          ← 쿠키 저장해
← 빈 줄
�
← 실제 페이지 내용 Welcome ```
Q&A: HTTP/1.1에서 1.1이 뭐야?
1.1은 프로토콜이 아니라 HTTP 프로토콜의 버전 번호.
HTTP/0.9 → 맨 처음 (텍스트만 가능)
HTTP/1.0 → 헤더 추가, 이미지 전송 가능
HTTP/1.1 → 현재 가장 많이 쓰이는 버전 ← 이거!
HTTP/2   → 더 빠름 (여러 요청 동시 처리)
HTTP/3   → 최신 (UDP 기반)
HTTP/1.0 vs 1.1의 핵심 차이
HTTP/1.0:
요청 1개 → 핸드셰이크 → 받음 → 연결 끊음
요청 2개 → 핸드셰이크 → 받음 → 연결 끊음
→ 매번 SYN → SYN/ACK → ACK 반복 = 느림!

HTTP/1.1:
핸드셰이크 1번 → 요청 1,2,3 전부 처리 → 연결 끊음
→ 하나의 연결에서 여러 요청 = 빠름!
→ 세션을 유지하기 때문에 가능 (이전 강의와 연결!)
요청/응답에서 보이는 부분
GET /blog HTTP/1.1  → "나는 1.1 규칙으로 대화할게"
HTTP/1.1 200 OK     → "나도 1.1로 대화할게, 성공했어"
= 양쪽이 같은 버전으로 대화하자는 합의
HTTP 메서드 — "뭘 해달라는 건지"
GET    → "정보 줘"   예: 페이지 보기, 검색
POST   → "이거 받아" 예: 회원가입, 로그인, 글 작성
PUT    → "이거 수정해" 예: 프로필 변경, 비밀번호 변경
DELETE → "이거 삭제해" 예: 계정 삭제, 게시글 삭제
실제 요청 형태
GET (블로그 1번 글 보기):
GET /blog?id=1 HTTP/1.1
Host: tryhackme.com

POST (로그인):
POST /login HTTP/1.1
Host: tryhackme.com
Content-Length: 29

username=thm&password=letmein
→ 데이터가 빈 줄 아래에 들어감!

PUT (유저 정보 수정):
PUT /user/2 HTTP/1.1
Host: tryhackme.com
Content-Length: 14

username=admin

DELETE (유저 삭제):
DELETE /user/1 HTTP/1.1
Host: tryhackme.com
상태 코드 — 서버의 대답
1xx — 정보:         "처리 중이야"
2xx — 성공:         "잘 됐어!"
3xx — 리다이렉트:    "다른 데로 가"
4xx — 클라이언트 에러: "네가 잘못했어"
5xx — 서버 에러:      "내(서버)가 잘못했어"
반드시 외울 코드
200 OK                → 요청 성공 ✅
201 Created           → 새로 만들어짐 (회원가입 등)
301 Moved Permanently → 영구 이동 (주소 바뀜)
302 Found             → 임시 이동
401 Unauthorized      → 로그인 안 했잖아 🔒
403 Forbidden         → 권한 없어 ⛔
404 Not Found         → 그런 페이지 없어 (가장 유명!)
405 Method Not Allowed→ 그 메서드 안 돼
500 Internal Server Error → 서버가 죽었어 💀
503 Service Unavailable   → 서버 과부하/점검 중

비유:
200 = "주문하신 거예요" 😊
404 = "그런 메뉴 없는데요?" 🤷
403 = "VIP만 입장 가능" 🚫
500 = "주방에 불이 났습니다" 🔥
503 = "오늘 쉬는 날이에요" 🏖️
헤더 — 요청/응답에 붙는 추가 정보
요청 헤더 (내가 서버에 보내는 정보)
Host           → 어느 서버에 요청하는지 (필수)
User-Agent     → 내 브라우저 종류/버전
Referer        → 어디서 왔는지
Cookie         → 저장된 쿠키를 서버에 보냄
Content-Length → 보내는 데이터 크기
응답 헤더 (서버가 나에게 보내는 정보)
Set-Cookie     → "이 쿠키 저장해"
Content-Type   → 데이터 종류 (HTML? 이미지? JSON?)
Content-Length → 데이터 크기 (빠진 데이터 확인용)
Cache-Control  → 브라우저 캐시에 얼마나 저장할지
쿠키 — 서버가 내 브라우저에 남기는 메모
HTTP는 기억력이 없음(Stateless). 쿠키가 이 문제를 해결.
[1] 로그인:
나 → 서버: "ID: jeff / PW: 1234"
서버 → 나: "성공! 이 쿠키 저장해" (Set-Cookie: session=abc123)

[2] 다음 페이지:
나 → 서버: "마이페이지 줘" + Cookie: session=abc123
서버: "abc123... jeff구나!" → 마이페이지 보여줌

[3] 쿠키 없으면:
나 → 서버: "마이페이지 줘" (쿠키 없음)
서버: "누구세요? 로그인 먼저"

비유:
쿠키 = 놀이공원 손목 밴드
입구에서 표 사면(로그인) → 밴드 받음(쿠키)
놀이기구 탈 때마다 밴드 보여줌 → 입장객 확인
밴드 없으면 → "표 먼저 사세요"
노트북에서 쿠키 보는 방법 (크롬 기준)
방법 1: Application 탭
F12 → Application → Cookies 클릭

┌──────────────┬───────────────┬──────────┬──────────┐
│ Name         │ Value         │ Domain   │ Expires  │
├──────────────┼───────────────┼──────────┼──────────┤
│ session      │ abc123xyz     │ .thm.com │ 2026-04-07│
│ name         │ adam          │ .thm.com │ Session  │
└──────────────┴───────────────┴──────────┴──────────┘

Name    = 쿠키 이름
Value   = 서버에 전송되는 데이터
Domain  = 어느 사이트 쿠키인지
Expires = 유효기간 (Session = 브라우저 닫으면 삭제)

방법 2: Network 탭
F12 → Network → 아무 요청 클릭 → Headers/Cookies 탭

Request:  Cookie: session=abc123 (내가 보낸 쿠키)
Response: Set-Cookie: session=abc123 (서버가 저장하라고 보낸 쿠키)
쿠키의 보안 문제 (매우 중요!)
⚠️ 쿠키 변조:
Application 탭에서 쿠키 값을 직접 수정 가능!
name=adam → name=admin으로 변경
→ 서버가 검증 안 하면 관리자 권한 획득 😱

⚠️ 세션 하이재킹:
해커가 session=abc123 쿠키를 훔치면
→ 비밀번호 없이 그 사람인 척 로그인 가능

방어:
→ 서버에서 쿠키 값 반드시 검증
→ HTTPS로 쿠키 전송 암호화
→ 세션 유효기간 짧게 설정
→ HttpOnly 플래그 (JavaScript로 쿠키 접근 차단)
보안관제에서 HTTP가 중요한 이유
웹 방화벽(WAF) 로그에서 매일 보는 것:

"POST /login에 1분에 500번 요청"
→ 브루트포스 공격!

"GET /admin?id=1 OR 1=1"
→ SQL 인젝션 시도!

"DELETE /user/1 (권한 없는 사용자)"
→ 권한 우회 시도!

상태 코드 분석:
401 대량 발생 → 로그인 공격 중
403 반복     → 권한 우회 시도
500 갑자기   → 서버 장애 또는 공격으로 인한 크래시
전체 흐름에서 HTTP의 위치
1. 브라우저에 URL 입력
2. DNS가 도메인 → IP 변환 (DNS 강의)
3. TCP 핸드셰이크로 연결 (Packets & Frames 강의)
4. HTTP 요청 전송 ← 이 강의!
5. 서버가 HTTP 응답 반환 ← 이 강의!
6. 브라우저가 HTML을 화면에 표시
7. 쿠키 저장 → 다음 요청 때 같이 보냄

이전 강의들과 연결:
DNS (도메인→IP) → TCP (연결) → HTTP (대화) → 화면에 표시
새로 알게 된 용어
용어
뜻
HTTP
HyperText Transfer Protocol
HTTPS
HTTP + Secure (암호화)
URL
웹 리소스 주소
Scheme
URL의 프로토콜 부분 (http, https, ftp)
Host
URL의 서버 주소 부분
Path
URL의 페이지 경로 부분
Query String
URL의 추가 정보 (?id=1)
GET
정보 요청 메서드
POST
데이터 제출 메서드 (새로 만들기)
PUT
데이터 수정 메서드
DELETE
데이터 삭제 메서드
Status Code
서버 응답 결과 숫자 (200, 404 등)
Content-Length
데이터 크기를 알려주는 응답 헤더
Set-Cookie
서버가 쿠키 저장을 요청하는 헤더
Cookie
브라우저에 저장된 데이터, 요청 시 서버에 전송
Stateless
기억력 없음 — HTTP의 특성
Session Hijacking
세션 쿠키를 훔쳐서 타인인 척 로그인
HttpOnly
JavaScript로 쿠키 접근 차단하는 보안 플래그
HTTP/1.1
현재 가장 많이 쓰이는 HTTP 버전
WAF
Web Application Firewall, 웹 방화벽
궁금한 점
Burp Suite로 HTTP 요청을 어떻게 가로채고 수정하는지?
세션 하이재킹을 실무에서 어떻게 탐지하는지?
HTTP/2와 HTTP/3의 차이가 보안에 어떤 영향을 미치는지?
커밋하면 How The Web Works 모듈 두 번째 노트 완성이에요!
