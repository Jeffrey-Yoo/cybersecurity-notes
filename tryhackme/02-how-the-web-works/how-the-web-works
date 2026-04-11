# How Websites Work — TryHackMe

**날짜:** 2026-04-06
**Room:** https://tryhackme.com/room/howwebsiteswork

---

## 한 줄 요약
웹사이트는 Front End(브라우저가 보여주는 것)와 Back End(서버가 처리하는 것)로 나뉜다.
HTML/CSS/JS 기초와 두 가지 보안 취약점(민감 데이터 노출, HTML 인젝션)을 배운다.

---

## 웹사이트의 두 파트
사용자 → 브라우저에서 요청 → [인터넷] → 서버가 처리 → 응답 → 브라우저가 표시
Front End (클라이언트 사이드) = 브라우저가 보여주는 것
Back End (서버 사이드)        = 서버가 처리하는 것
### 키오스크 비유
프론트엔드 = 키오스크 (터치스크린 주문기)
→ 메뉴를 보여주고 (HTML)
→ 예쁘게 꾸미고 (CSS)
→ 버튼 누르면 반응하고 (JavaScript)
→ 주문을 주방(백엔드)에 전달
→ 주방에서 결과 오면 화면에 표시
백엔드 = 주방
→ 주문 확인 (요청 처리)
→ 재료 있는지 확인 (DB 조회)
→ 요리 시작 (데이터 처리)
→ 완성된 음식을 홀로 보냄 (HTTP 응답)
---

## HTML — 웹페이지의 뼈대

```html
<!DOCTYPE html>
<html>
  <head>
    <title>제프의 블로그</title>      ← 브라우저 탭 제목
  </head>
  <body>
    <h1>환영합니다!</h1>             ← 큰 제목
    <p>보안 공부 기록</p>            ← 문단
    <img src='img/cat-1.jpg'>       ← 이미지
    <a href="https://google.com">구글</a>  ← 링크
  </body>
</html>
주요 태그
<h1>~<h6>  → 제목 (1이 제일 큼)
<p>        → 문단
<img>      → 이미지 (닫는 태그 없음)
<a>        → 링크
<button>   → 버튼
<div>      → 구역 나누기 (보이지 않는 박스)
<input>    → 입력창
<!-- -->   → 주석 (화면에 안 보이지만 코드에는 있음)
실습: 이미지 깨진 이유
❌ <img src='img/cat-2'>       ← 확장자 없음!
✅ <img src='img/cat-2.jpg'>   ← .jpg 추가하면 보임
CSS — 웹페이지의 디자인
HTML = 뼈대 (구조)
CSS  = 옷 (색상, 크기, 배치, 폰트 등)

HTML만 있으면 → 못생긴 흰 배경에 검은 글씨
CSS 추가하면 → 예쁜 디자인의 웹사이트
JavaScript — 웹페이지의 동작
HTML = 뼈대 (정적)
CSS  = 디자인 (정적)
JS   = 움직임과 반응 (동적!)
텍스트 변경
<div id="demo">Hi there!</div>

<script>
  document.getElementById("demo").innerHTML = "Hack the Planet";
</script>

→ "demo"라는 요소를 찾아서 내용을 바꿈
→ 화면에 "Hack the Planet"이 보임
버튼 클릭 반응
<button onclick='document.getElementById("demo").innerHTML = "Button Clicked";'>
  Click Me!
</button>

버튼 클릭 전: "Hi there!"
버튼 클릭 후: "Button Clicked"
→ 새로고침 없이 내용이 바뀜!
JS가 하는 일 예시
로그인 폼에서 "비밀번호 틀렸습니다" 메시지 표시
장바구니에 상품 추가할 때 숫자 올라감
검색창에 입력하면 자동완성 뜸
→ 전부 JavaScript!
Q&A: 프론트엔드와 백엔드의 기술
Q: 백엔드는 어떤 언어로 만들어?
프론트엔드 (선택지 없음, 고정):
→ HTML + CSS + JavaScript

백엔드 (여러 언어 중 선택):
→ JavaScript (Node.js) — 프론트와 같은 언어로 가능!
→ Python (Django/Flask) — 보안 분야에서 많이 씀
→ Java (Spring) — 대기업/은행에서 많이 씀
→ PHP (Laravel) — 워드프레스가 이걸로 만들어짐
실제 로그인 과정의 전체 흐름
사용자가 로그인 버튼 클릭
        ↓
[프론트엔드] JS가 아이디/비밀번호를 담아서
        ↓  HTTP POST /api/login
[백엔드] Node.js/Python이 요청 받음
        ↓  DB 조회
[데이터베이스] "jeff라는 유저 있어? 비밀번호 맞아?"
        ↓  "맞아!"
[백엔드] 세션 쿠키 발급 + "로그인 성공" 응답
        ↓  HTTP 200 OK + Set-Cookie
[프론트엔드] "환영합니다 Jeff!" 화면 표시
보안 관점에서 핵심 차이
프론트엔드 코드:
→ 누구나 볼 수 있음 (Ctrl+U로 소스 보기)
→ 여기에 비밀 정보 넣으면 전부 노출!
→ JavaScript 수정으로 검증 우회 가능!

백엔드 코드:
→ 서버에서만 실행, 사용자가 볼 수 없음
→ DB 접속 정보, API 키, 비밀번호 검증은 여기에
→ 사용자 입력을 여기서 최종 검증해야 함

원칙: 화면 표시 = 프론트 / 진짜 처리 = 백엔드
보안 취약점 1: Sensitive Data Exposure (민감 데이터 노출)
개발자가 소스 코드에 중요한 정보를 남겨놓는 실수.
<!-- TODO: remove this before going live
     username: admin
     password: testpasswd -->
화면에서: 로그인 폼만 보임 (정상적으로 보임)
소스 코드: 주석 안에 admin / testpasswd 있음!
→ 우클릭 → "페이지 소스 보기" → 비밀번호 발견 😱

비유:
식당 메뉴판 뒷면에 금고 비밀번호를 적어놓은 것
→ 손님이 뒤집으면 바로 보임
보안관제/해커가 하는 것
1. 사이트 소스 코드 열기 (Ctrl+U)
2. 주석(<!-- -->)을 전부 검색
3. 숨겨진 비밀번호, API 키, 내부 URL 발견
4. 발견된 정보로 추가 침투
방어
→ 배포 전에 주석의 민감 정보 제거
→ 비밀번호는 절대 프론트엔드 코드에 넣지 않음
→ API 키는 백엔드(서버)에만 저장
보안 취약점 2: HTML Injection (HTML 인젝션)
사용자 입력을 검증 없이 그대로 페이지에 표시하면 생기는 취약점.
정상 사용:
입력: "Jeff"
출력: "Hi, Jeff!"

공격:
입력: <a href="http://hacker.com">여기 클릭</a>
출력: "Hi, 여기 클릭!"  ← 진짜 클릭 가능한 링크가 생김!
왜 위험한지
해커가 입력창에 넣으면:
<h1>긴급: 비밀번호를 재설정하세요!</h1>
<a href="http://hacker.com/fake-login">여기서 변경</a>

→ 페이지에 진짜 공지처럼 보이는 가짜 메시지
→ 다른 사용자가 클릭 → 해커 사이트로 이동
→ 가짜 로그인 페이지에서 비밀번호 탈취 😱

비유:
식당 게시판에 아무나 글을 쓸 수 있는데
"가게 이전했습니다, 새 주소: hacker.com"을 적어놓으면
다른 손님들이 속아서 엉뚱한 곳으로 감
방어: 입력값 검증 (Sanitization)
사용자 입력에서 HTML 태그를 무력화:
< → &lt;
> → &gt;

브라우저가 태그로 인식하지 않고 그냥 텍스트로 표시

핵심 원칙: "절대 사용자 입력을 신뢰하지 마라"
(Never trust user input)
이전 강의들과 연결
DNS:              도메인 → IP 찾기 (전화번호부)
HTTP:             서버와 대화하는 규칙 (요청/응답)
How Websites Work: 서버가 보내준 HTML/JS를 화면에 표시 ← 여기!

전체 흐름:
URL 입력 → DNS(IP 찾기) → TCP(연결) → HTTP(요청/응답)
→ 브라우저가 HTML(뼈대) + CSS(디자인) + JS(동작) = 화면 표시!
새로 알게 된 용어
용어
뜻
Front End
브라우저에서 실행되는 클라이언트 사이드
Back End
서버에서 실행되는 서버 사이드
HTML
웹페이지의 뼈대 (구조)
CSS
웹페이지의 디자인 (스타일)
JavaScript
웹페이지의 동작 (상호작용)
DOM
Document Object Model, JS가 HTML을 조작하는 방법
innerHTML
HTML 요소의 내용을 바꾸는 JS 속성
getElementById
ID로 HTML 요소를 찾는 JS 함수
Sensitive Data Exposure
소스 코드에 민감 정보가 노출되는 취약점
HTML Injection
사용자 입력이 HTML로 실행되는 취약점
Sanitization
사용자 입력에서 위험한 코드를 제거/무력화
Node.js
JavaScript로 백엔드를 만드는 기술
Django/Flask
Python으로 백엔드를 만드는 기술
궁금한 점
XSS(Cross-Site Scripting)는 HTML Injection과 어떻게 다른지?
실무에서 입력값 검증은 프론트/백 둘 다 하는지?
다음 강의 "Putting It All Together"에서 뭘 배우는지?
